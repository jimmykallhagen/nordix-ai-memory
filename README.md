# ZFS as a Memory Extension Layer for Local AI Inference

 - A framework for ZFS-backed memory extension in local AI inference.


---

**Author:** Jimmy Källhagen  
**Project:** [Nordix](https://github.com/jimmykallhagen/Nordix)  
**License:** SPDX-License-Identifier: PolyForm-Noncommercial-1.0.0  
**Copyright (c) 2025 Jimmy Källhagen**

---

## The Problem

Large language models are growing faster than hardware can keep up. A 70 billion parameter model quantized to 4-bit precision requires approximately 40 GB of memory. A 405 billion parameter model needs over 200 GB. The standard answer is to buy more GPUs or rent cloud compute. But there is another path — one rooted in how operating systems already manage memory.

This document explores a systems-level approach: using ZFS, compressed memory, and Linux kernel memory tiering to extend available memory for AI inference on commodity desktop hardware. The goal is not to replace datacenter infrastructure, but to make large models accessible on hardware that a small business or a motivated individual can afford.

---

## Why This Matters for Businesses

Every company has internal knowledge - documentation, contracts, procedures, customer data, compliance records. Running a local language model against this data means:

- **No data leaves the building.** Sensitive documents stay on your hardware, not on someone else's server.
- **No per-token cost.** Cloud AI billing scales with usage. Local inference has a fixed hardware cost.
- **No dependency on external services.** Your AI capability does not disappear when a provider changes pricing, terms, or availability.

The barrier has been hardware cost. Running a capable model locally has traditionally required enterprise-grade GPUs with large VRAM pools. The approach described here aims to lower that barrier significantly by making smarter use of the memory and storage already present in a modern workstation.

---

## The Core Insight: Compression as Memory Multiplication

ZFS users are already familiar with this principle. When compressed ARC is enabled, ZFS stores cached blocks in their compressed form and only decompresses on read. The result is that physical RAM holds significantly more useful data than its raw capacity would suggest.

| Compression | Typical Ratio | 64 GB RAM → Effective ARC |
|-------------|---------------|---------------------------|
| lz4         | ~2.0x         | ~80–100 GB                |
| zstd-3      | ~2.5x         | ~100–120 GB               |
| zstd-7      | ~3.0x         | ~120–150 GB               |

This is not theoretical. These are production numbers from real ZFS systems. The same principle can be applied to AI model storage.

AI model weights, particularly quantized weights, exhibit patterns that compress well. A 40 GB model file stored on a ZFS dataset with lz4 compression may occupy only 25–30 GB of effective space in cache. This means a system with 64 GB of RAM can potentially hold model data that would otherwise require 80–100 GB.

---

## The Architecture: Three Memory Tiers

Modern AI inference engines like llama.cpp process transformer models layer by layer. At any given moment, only one or two layers need to be in active memory. This sequential access pattern is the key - it means we can tier memory without catastrophic performance loss.

The proposed architecture has three tiers:

```
Tier 0: GPU VRAM (24–32 GB)
  Fastest. Active computation happens here.
  Holds the layers currently being processed.
        ↕ PCIe bus
Tier 1: System RAM / DDR5 (64–256 GB)
  Fast. Holds layers queued for GPU processing.
  With compressed ARC, effective capacity is 1.5–3x physical.
        ↕ Kernel memory tiering
Tier 2: ZFS zvol on NVMe (1–8 TB effective)
  Large. Holds the full model and overflow data.
  Compressed with lz4 for near-free decompression.
  NVMe Gen5 striping provides 50+ GB/s sequential read.
```

The critical point: **the application does not need to know about these tiers.** If Tier 2 is exposed as swap or as a memory-tiered NUMA node, the AI inference engine simply sees "more RAM." The Linux kernel handles page placement and promotion transparently.

---

## Why ZFS Is Uniquely Suited

Other storage systems can provide fast NVMe access. But ZFS offers specific advantages for this use case:

**Compressed ARC serves as Tier 1.5.** When a model is stored on a ZFS dataset, frequently accessed blocks are cached in ARC in their compressed form. This is not swap - it is genuine caching with LRU eviction, hit rate tracking, and adaptive sizing. Hot model layers stay in compressed ARC automatically.

**zvol provides a clean block device interface.** A ZFS zvol can be used directly as a swap device or as a DAX-capable block device for memory tiering. The guest (the AI engine) sees a block device. ZFS handles compression, checksumming, and caching underneath.

**Snapshots provide rollback for model experiments.** Testing different quantization levels, fine-tuned models, or configuration changes becomes trivial when you can snapshot before and rollback after.

**Tunable compression algorithms** allow balancing CPU cost against memory gain. lz4 decompresses at 5–6 GB/s per core with minimal CPU overhead, making it effectively free on modern multi-core processors. zstd provides better ratios when CPU headroom exists.

---

## Linux Kernel Memory Tiering

Since kernel 5.15, Linux has native support for memory tiering through the NUMA balancing subsystem. The kernel can:

- Recognize that different NUMA nodes have different performance characteristics
- Automatically place frequently accessed ("hot") pages in faster memory
- Demote infrequently accessed ("cold") pages to slower memory tiers
- Promote pages back to fast memory when they become hot again

This was originally built for Intel Optane Persistent Memory but the mechanism is generic. It works with any memory device that can be exposed as a NUMA node, including NVMe devices configured through the DAX (Direct Access) subsystem.

The configuration is straightforward:

```bash
# Enable memory tiering mode
echo 2 > /proc/sys/kernel/numa_balancing

# Possible values:
# 0 = disabled
# 1 = normal NUMA balancing
# 2 = memory tiering (promotes hot pages, demotes cold pages)
```

When combined with a ZFS zvol exposed as swap on fast NVMe storage, the kernel manages page placement automatically. The AI inference engine allocates memory normally, and the kernel ensures that the most frequently accessed model weights stay in DRAM while less active data resides on compressed NVMe storage.

---

## CXL: The Future of This Approach

Compute Express Link (CXL) is an emerging standard that formalizes exactly this concept. CXL memory expanders attach additional memory pools via the PCIe bus, and the CPU can access them coherently - slower than local DRAM, but faster than storage.

Samsung, Lenovo, and others are already shipping CXL memory expanders for server platforms. The Linux kernel (6.3+) has native CXL support, and tools like Samsung's open-source SMDK (Scalable Memory Development Kit) provide a full software stack for managing tiered CXL memory.

The ZFS zvol approach described here is conceptually identical to CXL memory tiering, implemented with hardware that is available today on consumer platforms. As CXL hardware becomes available in workstation-class systems, the same principles apply — the only change is that the slow tier gets faster.

---

## Practical Test: What You Can Do Today

This approach can be tested on any Linux system with ZFS and at least one NVMe drive. No custom kernel, no patched software, no specialized hardware.

### Step 1: Create a compressed zvol

```bash
# Create a zvol with lz4 compression on your NVMe pool
zfs create -V 100G \
  -o volblocksize=16K \
  -o compression=lz4 \
  -o primarycache=all \
  -o sync=disabled \
  your-pool/ai-swap
```

### Step 2: Configure as swap

```bash
mkswap /dev/zvol/your-pool/ai-swap
swapon -p 10 /dev/zvol/your-pool/ai-swap
```

### Step 3: Tune swap behavior

```bash
# Increase swappiness to allow kernel to use swap proactively
sysctl vm.swappiness=80

# This tells the kernel it is acceptable to move inactive pages
# to swap before RAM is fully exhausted, enabling smoother tiering
```

### Step 4: Run inference

```bash
# Run a model larger than your physical RAM
# llama.cpp will allocate memory normally
# The kernel handles the rest via swap on compressed zvol

./llama-server -m /path/to/large-model.gguf \
  --ctx-size 4096 \
  --threads 16
```

### Step 5: Observe

```bash
# Monitor ARC hit rate (target: >85%)
awk '/^hits/{h=$3} /^misses/{m=$3} END{printf "%.1f%%\n",h/(h+m)*100}' \
  /proc/spl/kstat/zfs/arcstats

# Monitor swap usage and I/O
zpool iostat -v your-pool 2

# Monitor tokens per second in llama.cpp output
```

---

## Benchmarking Considerations

ZFS benchmarking requires careful methodology. Synthetic benchmarks like CrystalDiskMark or fio produce misleading numbers because they do not account for ARC caching behavior, TXG write batching, or compression ratios.

For AI inference testing specifically:

**Write benchmarks are not meaningful.** ZFS write performance depends on when transaction groups flush, how much dirty data is buffered, and whether sync is enabled. A 1 GB write test may complete entirely in RAM and report absurd speeds. This is not representative of real workload performance.

**Read benchmarks are useful as reference points** when ARC behavior is understood. A cold-start read (after clearing ARC) shows NVMe throughput. A warm read shows ARC-cached performance. Both are relevant but must be reported separately.

**The only meaningful benchmark is the actual workload.** For AI inference, this means measuring tokens per second across multiple runs, with and without the zvol swap tier, on the same model and prompt. Report cold-start (first run) and warm (subsequent runs) separately.

---

## Hardware Scaling

The approach scales across different hardware budgets:

| System | Physical RAM | Effective with compression | GPU VRAM | Practical model size |
|--------|-------------|---------------------------|----------|---------------------|
| Budget desktop | 32 GB | ~50–70 GB | 8–12 GB | 13B–33B quantized |
| Mid-range workstation | 64 GB | ~100–150 GB | 24 GB | 33B–70B quantized |
| High-end workstation | 128 GB | ~200–300 GB | 24–48 GB | 70B–120B quantized |
| Dual-GPU workstation | 256 GB | ~400–600 GB | 48–64 GB | 200B–405B quantized |

Adding NVMe zvol swap extends each tier further. A 4x NVMe Gen5 stripe with lz4 compression provides an additional 4–12 TB of effective memory capacity at approximately 25–30% of DRAM bandwidth.

---

## Limitations and Honest Expectations

This approach does not make a desktop perform like a datacenter. Specific limitations include:

**Latency increases with tier depth.** Every page fault from DRAM to NVMe adds microseconds. For interactive chat, this manifests as variable token generation speed — some tokens arrive quickly (data in DRAM/ARC), others take longer (data paged in from NVMe).

**Throughput is bounded by PCIe bandwidth.** Even the fastest NVMe stripe cannot match DDR5 memory bandwidth. Models that fit entirely in RAM will always be faster than models that spill to storage.

**Compression ratios vary.** Quantized model weights compress reasonably well, but the ratio depends on the specific quantization method and model architecture. Always test with your actual workload before planning capacity.

**NVMe endurance matters.** Using NVMe as swap means write cycles. Modern enterprise NVMe drives handle this well, but consumer drives with lower endurance ratings may wear faster under sustained swap workloads.

**This is systems engineering, not magic.** The goal is to run models that would otherwise be completely inaccessible on a given hardware budget. The tradeoff is speed — you are trading tokens per second for the ability to run the model at all.

---

## Conclusion

The building blocks for memory-extended local AI inference already exist in the Linux ecosystem. ZFS provides compressed caching and block-level storage management. The Linux kernel provides transparent memory tiering. Modern NVMe drives provide the bandwidth to make storage-backed memory practical.

What has been missing is the recognition that these systems-level tools - originally built for databases, virtual machines, and file servers — apply directly to the problem of running large AI models on commodity hardware.

For businesses considering local AI deployment, this approach offers a path that does not require specialized hardware, cloud subscriptions, or sending sensitive data to external providers. It requires a Linux system, competent configuration, and an understanding of how memory hierarchies work.

The models are open. The hardware is affordable. The software is free. The missing piece is the systems knowledge to connect them.

---

## Related Work

- [Nordix VM — ZFS, ARC, zvol and Virtual Machines](https://github.com/jimmykallhagen/nordix-vm) - Detailed documentation on ZFS zvol configuration for virtual machines, including benchmarks and compressed ARC analysis.
- [Nordix](https://github.com/jimmykallhagen/Nordix) - A Linux distribution built around the philosophy of RAM as workspace, NVMe as storage, and ZFS as the foundation.
- [GreenBoost](https://www.phoronix.com/news/Open-Source-GreenBoost-NVIDIA) - An open-source Linux kernel module for transparently extending NVIDIA GPU VRAM with system RAM and NVMe storage.
- [Linux Kernel Memory Tiering](https://stevescargall.com/blog/2022/06/using-linux-kernel-memory-tiering/) - Guide to configuring memory tiering in Linux kernel 5.15+.
- [Samsung SMDK](https://github.com/OpenMPDK/SMDK) — Scalable Memory Development Kit for CXL memory management.
- [VMware ESXi 9.0 Memory Tiering over NVMe](https://lenovopress.lenovo.com/lp2288-implementing-memory-tiering-over-nvme-using-vmware-esxi-90) - Enterprise implementation of NVMe as tiered memory.

---

*This document presents a conceptual framework and practical starting point, not a finished product. The approach described here has not been formally benchmarked at scale. Contributions, testing results, and corrections are welcome.*
