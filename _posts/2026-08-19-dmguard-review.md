---
title: "[DMGuard Review] The CPU Freed the Page, but the GPU Is Still Looking at It"
date: 2026-08-19 21:00:00 +0900
categories: [Paper Review]
tags: [kernel, use-after-free, gpu, memory-safety, page-table, usenix, android]
description: A review of DMGUARD (USENIX Security 2026), the first runtime defense against physical-page use-after-free across CPU, GPU, and IOMMU translation domains.
toc: true
math: false
mermaid: false
image:
  path: /assets/img/dmguard-1.png
  alt: CPU and GPU sharing physical pages through separate page tables
---

> **DMGUARD: Safeguarding Kernels from Physical-Page Use-After-Free Vulnerabilities**
> Juhee Kim, Jaeyoung Chung, Dae R. Jeong, Byoungyoung Lee (Seoul National University)
> USENIX Security 2026 · [Paper](https://www.usenix.org/conference/usenixsecurity26/presentation/kim-juhee-dmguard)
{: .prompt-info }

## Introduction

Kernel memory safety research has long been built around a single question: *is this pointer valid?* ASLR, CFI, DFI, KASAN, MTE, PAC. Every one of these defenses rests on the same quiet assumption — **that the physical page a virtual address points to is the correct one**.

DMGUARD says that assumption is already broken. And not in some exotic corner of the stack, but in a very ordinary place: inside your Android phone.

---

# Part 1. The Paper

## 1. Background — Where Do These Bugs Come From?

### The Shape of the Problem

Integrated GPUs in mobile SoCs (ARM Mali, Qualcomm Adreno) have no dedicated VRAM, so they share system DRAM with the CPU. The trouble is in *how* they share it.

- The CPU has its own page tables and its own page pool (the buddy allocator).
- The GPU driver has **its own page tables and its own page pool**, kept separately.
- To avoid copying data, the same physical page is mapped into **both** page tables at once (zero-copy).

In other words, the lifecycle of a single physical page is **split across different subsystems**. The GPU driver allocates it, the kernel maps it, and something else frees it. The moment the ordering slips, you have a bug.

![CPU and GPU each maintain their own page tables and allocation pools while sharing the same physical pages](/assets/img/dmguard-1.png)
_Source: DMGUARD paper — CPU and GPU each hold their own page tables and page pools while sharing the same physical pages (0x5000, 0x6000)._

### A Second Cause: PFN-Based Mapping

Low-level interfaces like `VM_PFNMAP` and `vmf_insert_pfn_prot()` use a **raw PFN — just a number** — instead of a `struct page`. As a result, no reference count is taken. Once a driver builds a mapping this way, the kernel can free the page without ever knowing that mapping exists.

---

## 2. Two Dangerous Scenarios

The normal order is `Alloc → Map → Unmap → Free`. There are two ways to break it.

- **Free-Before-Unmap** — the page is freed while a mapping still exists. The page is already gone, but the old mapping remains, leaving it in a *dangling* state where it no longer points at a valid physical page.
- **Map-After-Free** — a new mapping is created for a page that has already been freed. Invalid from the very first moment.

In both cases, once the physical page is reallocated for some other purpose, an attacker can read and write someone else's data. Combine this with page spraying to steer a **kernel page table** or a **credential structure** into that slot, and you get **privilege escalation**.

![Timeline comparison of valid mapping, free-before-unmap, and map-after-free](/assets/img/dmguard-2.png)
_Source: DMGUARD paper — timeline comparison of (a) valid mapping, (b) free-before-unmap, and (c) map-after-free._

### Why Existing Defenses Don't Help

- **CFI, Intel MPK, ARM PAC, MTE** — all of them stand on the premise that *address translation is correct*. Once the translation layer itself is compromised, they are powerless.
- **CONFIG_PAGE_TABLE_CHECK (ptcheck)** — counts mappings only in the CPU context. It never looks at GPU or IOMMU page tables, cannot detect map-after-free even in principle, and is vulnerable to race conditions.

---

## 3. A Real Case — CVE-2025-0072 (Mali GPU)

While managing GPU memory, the Mali driver allocates a page twice. It is supposed to keep track of where each page was allocated, but allocating the second page **overwrites the record of the first**.

```c
T1: reserve VA1        →  allocate P1, store in queue->phys
T3: reserve VA2        →  allocate P2, overwrite queue->phys (P1 record lost)
T5: access VA2         →  page fault → create VA2 → PA2 mapping
T6: munmap(VA1)        →  VA1 was never mapped, so nothing happens
T7: free queue->phys(=P2)  →  freed while VA2 → PA2 mapping is still live 💥
```

The correct sequence would be `VA2 → P2 mapping → unmap → free P2`. In this bug, P2 is already freed while VA2 still points at it — a stale mapping.

This is **not** a classic memory corruption bug. Each individual operation — allocating, mapping, freeing — is a perfectly legitimate API call. What went wrong is the *state management and ordering*: **when** should what be freed. In short, bugs that mismanage a page's lifecycle and mapping state can be just as severe as bugs that misuse memory directly.

---

## 4. The Core Idea of DMGUARD

### 4-1. Page Lifecycle as a State Machine

Every physical page is viewed along two axes — allocation status × mapping status — producing four states.

| State | Meaning |
| --- | --- |
| **Freed–Unmapped** | A free page sitting in the allocator's pool (initial state) |
| **Allocated–Unmapped** | Allocated, but not yet present in any page table |
| **Allocated–Mapped** | Allocated and mapped (normal use) |
| **Freed–Mapped** | **A state that must never exist = dangling mapping** |

The invariant — *a page may be mapped only while it is allocated* — can be violated by exactly two transitions: **Free-Before-Unmap** and **Map-After-Free**.

![State machine of a page lifecycle, with red arrows marking vulnerable transitions](/assets/img/dmguard-3.png)
_Source: DMGUARD paper — the page lifecycle state machine. Red arrows are the vulnerable transitions._

### 4-2. Two Watchdogs

|  | MapCount | Page Tag |
| --- | --- | --- |
| Prevents | Free-Before-Unmap | Map-After-Free |
| Tracks | Mapped ↔ Unmapped | Freed ↔ Allocated |
| Mechanism | Per-page mapping counter | Random tag per allocation cycle |
| Check point | Block free if `MapCount != 0` | Block map if tags mismatch |

![Overview of DMGUARD's page state transition tracking](/assets/img/dmguard-4.png)
_Source: DMGUARD paper — where each of the two mechanisms sits on the state machine._

#### MapCount — No Freeing While Mappings Remain

For each physical page, count the number of user-space mappings. Increment on map, decrement on unmap. At free time, if `MapCount != 0`, flag it as free-before-unmap and block it.

`ptcheck` does something similar, but counts **only CPU mappings**. DMGUARD counts mappings in **GPU and IOMMU page tables** as well. As accelerators with their own page tables — GPUs, NPUs, SmartNICs — keep multiplying, this extension is the whole point.

#### Page Tag — No Mapping if the Tags Disagree

Two tags are compared:

- **PageTag** — the current allocation-cycle ID, stored in the physical page's metadata.
- **RefTag** — the same ID, embedded in the top byte of any pointer referencing that page (`struct page*` or a PFN).

```c
on alloc  →  generate a new random PageTag; the returned pointer's RefTag gets the same value
on map    →  PageTag == RefTag ?
             match    → valid reference, proceed
             mismatch → stale reference to a freed/reallocated page → block
on free   →  invalidate PageTag
```

The implementation uses ARM64's **TBI (Top Byte Ignore)** to store the tag in the pointer's top byte (x86's LAM works the same way). With an 8-bit tag, the detection rate is **254/255 = 99.6%**. For the "map immediately after free" case, DMGUARD guarantees the new tag always differs from the previous one, giving **100% detection**.

#### Lockless Design

Page management sits on an extremely performance-sensitive path, so locks are off the table. DMGUARD uses atomic operations combined with memory barriers instead.

This matters in practice on weak memory models like ARM and RISC-V, where instruction reordering can skip a check entirely. The paper shows concretely that without the barrier in `dmcMap`, execution can reorder into `L20 (tag check) → L41 (tag invalidation) → L44 (MapCount check) → L17 (MapCount increment)` and miss a map-after-free. They then validated the concurrent `Map-Free`, `Alloc-Map`, and `Free-Unmap` scenarios exhaustively using **LKMM litmus tests**.

As a bonus, comparing PageTag against RefTag also **detects double-free**.

---

## 5. Evaluation

### 5-1. Security

**24 vulnerabilities total** = 19 real CVEs + 5 synthetic.

|  | Empirically tested | Theoretically analyzed | Total |
| --- | --- | --- | --- |
| Count | 15 (10 real + 5 synthetic) | 9 | **24** |
| DMGUARD | ✅ 15/15 | ✅ 9/9 | **24/24** |
| ptcheck | ❌ 5/15 | ❌ 0/9 | **5/24** |

- Subsystems covered: io_uring, usb, Linux mm, **Mali, Adreno, PowerVR**
- Nearly everything `ptcheck` missed involved mappings in a **GPU or IOMMU context**
- **Zero** false positives and false negatives
- **A new vulnerability found via Syzkaller**: in the upstream Mali driver, GPU regions freed on process termination without `munmap()` leave entries behind in the GPU page table. The authors **responsibly disclosed this to ARM**. ARM acknowledged the issue but judged exploitability low, since the dangling mappings are created only after GPU tasks have already terminated.

### 5-2. Performance (Pixel 8)

| Metric | Overhead |
| --- | --- |
| Geekbench 6 CPU single | **−0.44%** |
| Geekbench 6 CPU multi | **−1.29%** |
| Geekbench 6 GPU (OpenCL) | **−0.22%** |
| LMBench syscall (geomean) | +2.96% |
| Phoronix real workloads (geomean) | +1.26% |
| Kernel boot time | +3.53% (0.510s → 0.528s) |
| App cold start (10-app average) | +3.10% (max +13ms) |
| Memory | +1.05% (~50MB / 7.7GB) |

**Reading the numbers**: comprehensive protection for GPU workloads at a **0.22% GPU slowdown** is the most striking result in the paper. The one real cost is `fork` (+10–12%), because creating a new process sets up a large number of PTEs at once.

### 5-3. Robustness

The Android **VTS** `vts-kernel` suite (Kselftest + Linux Test Project) was run **10 times consecutively** with no crashes or hangs — all tests passed. MapCount and tags stayed accurate even through complex page operations like huge page splitting and page migration.

---

# Part 2. Review

## Q1. What Security Threat Is This Paper Responding To?

It responds not to *buggy code*, but to **buggy collaboration**.

On the surface the target is physical-page use-after-free. The real problem underneath is **resource lifecycle management with diffused responsibility**.

A traditional UAF happens inside one component. Code that calls malloc/free reuses a pointer after freeing it. One cause, one place to fix. The UAF in this paper is different:

- **Who allocated the page**: the GPU driver's own page pool
- **Who mapped the page**: the CPU kernel's page fault handler
- **Who freed the page**: the GPU driver's cleanup routine

**Every one of these actors behaved correctly from its own point of view.** Nobody broke a rule, and yet the system as a whole ended up in a broken state. In CVE-2025-0072, `munmap(VA1)` doing nothing was exactly right, and `kbase_mem_pool_free_pages(queue->phys)` was also exactly right by its own bookkeeping.

Three structural pressures converge here:

- **Heterogeneous computing is now the norm** — GPUs, NPUs, DPUs, and SmartNICs all carry their own MMUs and page tables and reach directly into host memory. The number of translation domains keeps growing.
- **Zero-copy performance pressure** — avoiding copies means exposing the same physical page to multiple domains. That is precisely the opposite of safe isolation.
- **A bias in the defense stack** — memory safety research has been conducted almost entirely above the virtual address space. The translation layer beneath it was simply assumed correct.

So this threat is not a one-off bug but a **structural consequence**. The paper's 24 vulnerabilities are spread evenly across io_uring, usb, mm, Mali, Adreno, and PowerVR — that spread is the evidence.

---

## Q2. Questions I Would Ask

- **What mitigation policy was considered instead of `BUG()`?** Refusing the free in a free-before-unmap case turns the bug into a memory leak. Is there any path to reclaim that leaked page? And is a DoS — an attacker triggering this path repeatedly — inside the threat model at all?
- When a huge page is split, the paper says MapCount is "correctly distributed" across the 512 underlying 4KB pages. **Is that distribution itself atomic?** What happens if a concurrent free lands in the middle of it? Were huge page split/merge scenarios included in the LKMM litmus tests?
- **Isn't the real fix architectural rather than a state machine bolted on top** — namely, GPU drivers not keeping separate page pools in the first place? DMGUARD monitors the symptom, not the root cause. Compared to architectural consolidation (a single dmabuf-based allocator, for example), was runtime monitoring chosen purely for deployability?

---

## Q3. How Does It Differ from Prior Work?

|  | What it protects | What it assumes | Relation to DMGUARD |
| --- | --- | --- | --- |
| **KASAN, MTE, BUDAlloc, SeaK** | Heap object UAF | VA→PA translation is correct | Different layer. **Orthogonal, complementary** |
| **CFI, DFI, PAC** | Control/data flow integrity | Page tables are correct | Different layer. **Orthogonal** |
| **Intel HLAT/VT-rp, Apple SPTM, ptcheck** | Page table integrity | Page management **logic is correct**; only corruption needs blocking | **This is the real point of contrast** |

The third row is the crux. Hardware-based page table protection defends the page tables **against malicious corruption**. Its worldview is that the kernel's page management code is correct, and the job is to stop attacks that bypass it to overwrite PTEs.

DMGUARD claims the opposite: **the kernel's page management code is itself wrong**. In CVE-2025-0072 the attacker corrupted nothing. They called `mmap` twice and then `munmap`. Every PTE update was performed by legitimate kernel code through legitimate APIs. Neither HLAT nor SPTM can stop that, and there is no reason they should — it looks exactly like normal page management.

Shifting the problem from **"protection against corruption"** to **"protection against incorrect logic"** is where this paper stands.

---

## Closing — Where Does This Paper Belong?

*"When a page's lifecycle is managed across multiple translation domains, no single actor enforces its consistency."* Simply stating that clearly is what makes this a good paper. The 24 CVEs are evidence that it is not a theoretical worry but something already happening, and the state machine abstraction is the tool that makes the problem tractable.

At the same time, I am reserved about how long this defense will hold. DMGUARD's coverage hinges on a **social and organizational premise — that drivers use the designated interfaces**. Nothing technically enforces that premise. Every time a new accelerator appears, a human has to read the code and find the instrumentation points. In the end, the problem hasn't been eliminated so much as **converted into a form that can be watched**.

So I read this paper as a **starting point, not a destination**. How should physical memory lifecycles be managed in heterogeneous systems? DMGUARD is the first serious answer to that question, and almost certainly not the last. The fundamental solution probably lies in architecture — a unified allocator, lifecycle enforcement at the type level, hardware support — rather than in monitoring. But hundreds of millions of devices need protecting until that architecture arrives, and as the practical defense that fills the gap, DMGUARD is well built.