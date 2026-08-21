---
layout: post
title: "[Pwn2Own 2025 Canon Review] The Allocator Handed Out One Pointer, but the Code Gave Back Four"
date: 2026-08-21 10:00:00 +0900
categories: [Paper Review]
tags: [pwn2own, canon, dryos, rtos, heap-exploitation, cve-2025-14233, firmware]
description: "How a single misplaced free() in Canon's proprietary CPCA protocol turned into a $10,000 remote code execution on a networked office printer — plus my own review of the research."
toc: true
---

> This post is based on the original write-up by **Team PetoWorks**, *"Pwn2Own 2025 Ireland"* —
> <https://peto.works/techblog/pwn2own-2025-ireland-kor>
> All technical credit belongs to the original authors. Part 2 is my own commentary.

---

## TL;DR

Inside Canon's **proprietary CPCA protocol** there was a simple mistake: code that should have freed **one contiguous buffer** instead freed **each element of it, one by one**.

That single mistake became a chain:

**Arbitrary address free → heap grooming → function pointer table hijack → remote code execution.**

It was demonstrated live on the Pwn2Own stage and earned **$10,000**.

---
---

# Part 1. About the Research

## 0. What security threat is this responding to?

> **"An embedded device sitting quietly on the internal network, allowing unauthenticated remote code execution."**

That is the threat this work is aimed at. Five things make it real:

### Printers are a blind spot

Printers and MFPs are permanently attached to the corporate network, but nobody watches them the way they watch servers and workstations. No EDR agent, no log review, often not even an entry in the asset inventory.

### The pre-auth attack surface is wide

*Pre-auth* means an attacker can reach the code **without logging in first**.

Because printing and network discovery have to "just work," a printer keeps a large number of ports open with no authentication at all: HTTP/HTTPS, LPD, IPP, Jetdirect, NetBIOS, SNMP, SLP, WSD, mDNS — **plus the vendor's own proprietary protocols**.

In other words: for the sake of user convenience, a printer permanently exposes many communication paths that require no login. That is a large attack surface by design.

### Proprietary protocols are never audited

There is no public RFC for Canon's CPCA. It has essentially never been exposed to outside researchers.

> *RFC (Request for Comments): the international standards documents that define internet and networking technologies.*

### RTOS mitigations are weak

An embedded RTOS like DryOS has far weaker exploit mitigations (ASLR, CFI, and friends) than a desktop OS. Once you get a memory corruption primitive, you go straight to code execution.

> *RTOS (Real-Time Operating System): an ultra-lightweight OS optimized for handling commands within a guaranteed time bound. Instead of offering the broad feature set of Windows or Linux, it focuses on being small and fast — smart appliances, medical devices, embedded systems.*
>
> *DryOS: Canon's in-house RTOS, developed for its digital cameras and printers.*

### The "standard protocol = safe" illusion

Even when an RFC is well defined, **one small implementation mistake becomes an exploitable vulnerability**. The specification being safe says nothing about the implementation being safe.

---

## 1. Background — Pwn2Own and the target

Pwn2Own is the "hacking olympics," hosted by Trend Micro's Zero Day Initiative. It runs three times a year, and the host city and target categories change every time.

- Every device is running the **latest firmware, fully patched**
- You must demonstrate the exploit **on stage, within a time limit**
- Success pays a bounty; the vulnerability then goes to the vendor under **responsible disclosure**

### The target: Canon imageCLASS MF654Cdw

- The OS is **DryOS**, Canon's own RTOS
- An RTOS is built so that *"it always finishes within a fixed time"* comes first — which is a fundamentally different design posture from Linux, where security layers are stacked one on top of another

---

## 2. Recon — opening up the device

### Where does the firmware live?

- Bootloader → flash memory
- Firmware image → **eMMC**
- At boot, the bootloader copies the firmware from eMMC into RAM and executes it

### The decisive stroke of luck: a live UART debug interface

The board still had UART pins exposed. Connecting to them dropped the researchers straight into a DryOS shell.

```
DryOS version 2.3, release #0059
Dry-MK 2.66, Dry-DM 1.21, Dry-FSM 0.10 ...
```

### Hidden debug commands

At first glance, memory dump (`xd`) and memory write (`xm`) commands appeared to have been removed from the shell. But once they looked at the firmware in a decompiler:

> **They had not been removed at all — they were only hidden from the help output.**

Searching for the string `usage:` in IDA surfaced *another* memory dump command, and that command became the debugging tool for the rest of the exploit development.

**The lesson here is one line:** *hiding* a debug feature in production firmware is not the same as *removing* it.

---

## 3. Mapping the attack surface

What the printer exposes to the network:

| Category | Protocols |
| --- | --- |
| Web management | HTTP / HTTPS |
| Printing | LPD, IPP, Jetdirect |
| Discovery | NetBIOS, SNMP, SLP, WSD, mDNS |
| **Canon proprietary** | **CPCA and others (no public RFC)** |

Three attack paths Team PetoWorks considered:

1. **Sloppy re-implementations of RFC protocols** — the place where vendors rewrite a standard their own way and make mistakes
2. **Parser bugs** — complex inputs: fonts, PDF, PostScript, PCL, images
3. **Proprietary protocols** ← this is where it actually broke

They combined static analysis, code auditing, and fuzzing. The real vulnerability was **triggered by fuzzing**, and the crash printed directly to the UART console — meaning **the UART shell doubled as the crash monitor**, which is normally the hardest part of embedded fuzzing.

---

## 4. The vulnerability — the CPCA `deleteFiles` handler

**CPCA (Common Peripheral Controlling Architecture)** is Canon's proprietary protocol that controls nearly every function of the MFP: printing, copying, scanning, mailbox management.

### Wire format

```
[ 20-byte header ] + [ variable-length payload ]

Header: magic (0xCDCA) | version | response code | opcode
        | payload length | reserved fields | padding
```

Example opcodes:

| Opcode | Function |
| --- | --- |
| `0x01` | Echo |
| `0x5f` | **Delete Files** ← the vulnerability |

### The bug

```c
ptr = alloc(sizeof(DeleteFiles));        // allocate the struct
ptr->file_ids = alloc(4 * file_count);   // ★ ONE contiguous buffer

if (error)
    goto free_logic;

free_logic:
for (i = 0; i < file_count; i++) {
    free(&ptr->file_ids[i]);   // ★★ BUG: frees each element individually
}
free(ptr);
```

### What exactly is wrong?

`file_ids` is **a single buffer allocated in one shot**, of size `4 * file_count`. Releasing it should therefore be a single `free(ptr->file_ids)` and nothing more.

Instead, the code treats `file_ids` as if it were *"an array of individually malloc'd pointers"* and calls `free()` on the address of element 0, then element 1, then element 2, and so on.

### Why is this fatal?

The key sentence from the original write-up:

> *"If we can control the contents of `file_ids`, and the free logic treats each element as a pointer, then we end up controlling the address passed to free. In other words, we can perform an **Arbitrary Address Free**."*

To put it plainly:

1. The contents of `file_ids` are **values the attacker sent in the CPCA payload**
2. Those values are passed to `free()` as addresses
3. → The attacker can tell the heap manager to **"release this address"** — any address

This is not a crash. It is a **primitive**.

---

## 5. Exploitation — from arbitrary free to RCE

### The DryOS heap

```
chunk = 16-byte aligned
[ tag | size | next pointer | Reserved ]
```

The important detail: **the `Reserved` field is not used by the heap manager at all — it exists purely to maintain alignment.** Which means an attacker can put values there and the allocator will never care. Room to manoeuvre.

### Step 1 — Heap grooming

An arbitrary free aimed at the wrong place just kills the device. To make it reliable, you have to shape the heap layout in advance.

1. Coax the allocator into carving a **large chunk** out of the top chunk
2. Free it
3. Re-allocate **smaller chunks** into the same region → the desired layout
4. Once the layout is ready, trigger the bug

The tool used for this was the **Echo handler (`0x01`)**:

- Allocation size = a 2-byte value the user sends
- Whatever data you send is copied verbatim into the allocated region

→ **A universal instrument for creating a chunk of any size, filled with any content.** The unsung protagonist of this exploit.

### Step 2 — Hijacking the PJCC handler table

DryOS contains a **PJCC handler table** — a table of function pointers.

1. Use the arbitrary free to release the **start address of the handler table**
2. The heap manager now believes that region is free space
3. Induce a re-allocation of the same size so **our data occupies that slot**
4. Use the Echo handler to fill it with a **fake handler table**
5. Send one more CPCA request

→ The firmware consults our fake table and calls an address of our choosing. **Program Counter control acquired.**

### Step 3 — The ROP chain

Controlling PC alone is not enough — the stack has to be ours too.

The gadget they found:

```asm
adds r0, r4, #4      ; [1] R4 + 4 → R0
mov  sp, r0          ; [2] R0 becomes the stack pointer
ldr  r0, [sp, #0x34] ; [3] load from the stack
...
pop  {r0, pc}        ; [4] return
```

From the original:

> *"This appears to be code used during task switching. Thanks to it, we can set up most registers and the stack as we want, based on the address stored in R4."*

→ Effectively a **stack pivot** gadget. Control R4 and you inherit the entire execution context.

> *R4 is a general-purpose register in the ARM architecture. Here it acts as the anchor: the attacker plants the address of a fake stack in R4 ahead of time, and the gadget overwrites SP with `R4 + 4`. The result is that a memory region under the attacker's control replaces the registers and the stack wholesale — a stack pivot — handing over full control of execution.*

### Step 4 — Bypassing W^X via DACR

**The problem.** When they tried to run shellcode, DryOS was enforcing per-region memory permissions.

- Every writable region was non-executable
- Attempting to jump there → **prefetch abort**

> *Prefetch abort: the error raised when the CPU tries to execute code in a memory region it has no execute permission for. "Prefetch" is the CPU reading instructions from memory ahead of execution; "abort" is it detecting the permission violation and halting.*

In short, DryOS's security policy left writable regions non-executable, so the CPU stopped the moment it tried to run the shellcode.

**The solution.** Use ARM's **Domain** mechanism.

- ARM MMU access control is managed through **16 domains**
- Each domain has a **2-bit permission field**
- Those fields live together in one register: the **DACR (Domain Access Control Register)**
- Setting a domain to **manager mode bypasses the memory access check entirely**

→ Use ROP to overwrite DACR, neutralize W^X, and run the shellcode.

> *W^X ("Write XOR Execute"): the principle that a memory region may be either writable or executable, but never both.*

So: when the W^X defense stopped their injected code from running, they used ROP to force the CPU's own permission-check switch — DACR — into manager mode ("don't check"), disabling the protection entirely and executing their code.

---

## 6. Results

| Item | Detail |
| --- | --- |
| CVE | **CVE-2025-14233** — *"Invalid free in CPCA file deletion processing"* |
| CWE | **CWE-763** (Release of Invalid Pointer or Reference) |
| CVSS | **9.8 (v3.1) / 9.3 (v4.0)** — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| Bounty | **$10,000** |
| Reliability | **~90%** (on failure the device simply reboots, so you can retry) |
| Affected | Many Canon Satera / imageCLASS / i-SENSYS models, **firmware v06.02 and earlier** |
| Disclosure | 2026-01-15, Canon PSIRT **CP2026-001** |

---

## 7. What the original write-up leaves us with

- A standardized protocol does **not** make its implementation safe.
- Printers have many pre-auth attack paths — by function, a lot of ports must stay open without authentication.
- DryOS has limited mitigations, so **classic heap techniques still work**.
- Printers sit inside the internal network and **nobody is watching them**.

---
---

# Part 2. Review

## Q1. What is the core of this work?

**The root of the vulnerability is not protocol design — it is a misunderstanding of memory ownership.**

How many times you call `free()` is a statement about *"who owns this memory, and in how many pieces."* This bug mistook **one ownership for N ownerships**, and C has no type system to enforce the answer.

---

## Q2. The questions I would ask

### ① How did they know the addresses? — the conditions for a leak-free exploit

> If `file_ids` is 32-bit (4 bytes), the entire address space is effectively exposed. Was the reason they could locate the handler table **without an information leak** simply the absence of ASLR?

Modern exploits are usually two-stage: **① leak an address → ② take control.** This chain has no stage ①. It **starts** already knowing where the handler table is.

**My guess:** DryOS has no ASLR, so as long as the firmware version matches, addresses are always identical — and the hidden memory-dump command in the UART shell let them harvest those addresses during development. Put differently, **"being able to hold the hardware in your hands" substituted for a leak primitive.**

Which raises the question back: **would this chain have survived if ASLR alone had been present?** Probably yes — an arbitrary free can operate on relative positions inside the heap. But that is my opinion, not a demonstrated fact.

### ② Why did the heap manager stop nothing?

> Does the DryOS heap have **any integrity checking at all** on its free list? Anything like tcache? Any double-free detection?

glibc has spent twenty years piling checks into `free()` — chunk alignment validation, size-field sanity checks, tcache double-free detection, cross-verification of `fd`/`bk` on unlink. **Had DryOS carried even one of them**, the arbitrary free would have aborted on the spot.

**My guess:** there were none. The fact that the original write-up explicitly notes *"the Reserved field is not used by the heap manager and exists only for alignment"* is itself evidence that **the allocator trusts its metadata and never validates it.**

What I would want to ask: was this **deliberately omitted for performance** (the RTOS demand for deterministic execution time), or is it **simply an old design**? The answer changes the defensive recommendation completely — the first makes "RTOS heap hardening is structurally hard" a general claim; the second means it is just a thing to go fix.

### ③ Why was the parsing task running privileged?

> Was flipping DACR to manager mode possible from user privilege? Which is to say — was the CPCA handler **already running in privileged mode**? And if so, isn't that a separate design flaw in its own right?

On ARM, `mcr p15, 0, r0, c3, c0, 0` (writing DACR) **executes only in privileged mode**. The fact that the trick worked implies exactly one thing: **the code parsing unauthenticated network packets was already running privileged.**

That is not a vulnerability so much as an **architectural decision**. And I consider it more fundamental than CVE-2025-14233 itself:

- The heap bug is **fixed by one line** — and indeed Canon patched it that way
- But the structure *"the network parser runs privileged"* means **the next heap bug leads to RCE all over again**

The original write-up frames the DACR bypass as *"a technical obstacle we overcame."* I read it differently: **the fact that it could be overcome is itself the finding.** I would want to know whether the authors reported that point to the vendor as a separate issue.

---

## Q3. How does this differ from existing methodology?

| | Typical embedded vulnerability research | This research |
| --- | --- | --- |
| Entry point | Web interface (CGI), standard protocol parsers | **Undocumented vendor-proprietary protocol** |
| Target OS | Embedded Linux (usually BusyBox) | **RTOS (DryOS)** — the Linux toolchain does not apply |
| Bug class | Command injection, stack overflow | **Arbitrary free (heap metadata attack)** |
| Observation | Custom hardware mods, JTAG | **A UART shell the manufacturer left behind + hidden debug commands** |
| NX bypass | Finish in pure ROP, or call mprotect | **DACR manipulation — ARM's domain feature disables the MMU check itself** |

**The difference in one line:**

> *Rather than carrying the grammar of Linux embedded research over to an RTOS unchanged, they used the debug channel the device already had — and a legacy feature of the ARM architecture — as their levers.*

---

## Q4. Would this approach transfer to other RTOS devices?

The candidates were cameras, automotive ECUs, and medical devices. I picked **the two with the most analytical value**.

- **Pick ①: Canon EOS mirrorless / DSLR** — *same OS, same vendor.* Shows the upper bound of portability.
- **Pick ②: VxWorks-based smart infusion pump** — *different OS, same structural weakness.* Tests whether the methodology generalizes.
- **Excluded: automotive ECU** — the reason is in ③ below. The reason for exclusion is itself what draws the boundary of this methodology.

### ① Canon EOS mirrorless / DSLR

**Why this device**

DryOS was originally Canon's RTOS **for cameras**. The printer is closer to a derivative deployment. Which means everything this research produced — knowledge of heap chunk layout, the stack pivot gadget pattern, the DACR bypass — **transfers almost verbatim.**

| Condition | Canon printer (this research) | Canon EOS camera | Portability |
| --- | --- | --- | --- |
| OS | DryOS 2.3 | **DryOS (same family)** | As-is |
| CPU | ARM32 | **ARM32 (DIGIC SoC)** | As-is |
| Heap allocator | 16B aligned, unvalidated | **Same-family implementation** | Nearly as-is |
| ASLR | None | **None** | Addresses can be hardcoded |
| DACR bypass | Works | **Works (same ARM MMU)** | As-is |
| Network surface | CPCA/BJNP over IP | **PTP over USB/Wi-Fi** | Protocol differs |
| Debug channel | UART shell | **UART + accumulated Magic Lantern knowledge** | Actually richer |

**There is already a decisive precedent**

In 2019, Check Point found six vulnerabilities in the PTP (Picture Transfer Protocol) implementation of the **Canon EOS 80D** and successfully **planted ransomware over Wi-Fi**. More than 30 Canon camera models were affected.

What deserves attention is **the matching pattern**:

- **2019, camera:** PTP (unauthenticated vendor implementation) → memory corruption → DryOS RCE
- **2025, printer:** CPCA/BJNP (unauthenticated proprietary protocol) → arbitrary free → DryOS RCE

Six years apart: **same OS, same structural weakness (unauthenticated vendor protocol + absent mitigations), same ending.**

**My judgment**

> **Yes, it transfers. And it has already transferred once.**
>
> That said, cameras **are not permanently connected to the internet**, which lowers real-world risk. A printer holds an IP on the corporate network 24 hours a day; a camera is exposed only while its Wi-Fi is on. **The vulnerability ports easily, but the threat scenario is far more serious on the printer.**
>
> The most realistic follow-up would be to check **whether CPCA-family code is linked into camera firmware at all.** If Canon's scan/transfer stack is shared, the file-deletion handler itself may exist there unchanged.

### ② VxWorks-based smart infusion pump

**Why this device**

If the camera demonstrates *"same OS, so of course it works,"* the medical device tests the real question: **does the methodology hold when the OS is different?**

| Condition | Canon printer | Infusion pump (VxWorks) | Portability |
| --- | --- | --- | --- |
| OS | DryOS | **VxWorks** | Completely different |
| CPU | ARM32 | ARM or PowerPC | Mixed |
| Address space isolation | None (single space) | **Effectively none on VxWorks ≤ 6.x** | Similar |
| ASLR | None | **None on older versions** | Similar |
| Heap validation | None | **Limited** | Similar |
| Unauthenticated network surface | CPCA/BJNP | **IPnet TCP/IP stack, wireless management interface** | Present |
| Patch cadence | Months | **Years (regulatory recertification)** | Far worse |
| Physical acquisition | Cheap on the used market | **Used market exists, but expensive** | — |

**The decisive precedent**

**URGENT/11**, published by Armis in 2019 — eleven vulnerabilities in the VxWorks IPnet TCP/IP stack, six of them RCE. The FDA issued a safety communication directly, and a range of medical devices including **BD Alaris infusion pumps** were affected. Armis later showed that the same IPnet stack had been **ported into RTOSes other than VxWorks** as well.

In other words, the precondition of this research — **"RTOS + unauthenticated network stack + absent mitigations"** — has already been confirmed at industry scale in medical devices.

**But there are differences**

1. **The DACR trick may not work.** On an MPU-based configuration without an MMU, or on PowerPC, that stage has to be redesigned from scratch.

**My judgment**

> **The technique does not port. The methodology ports completely.**
>
> You cannot bring the DACR bypass or the specific gadgets. But the actual skeleton of this research — *"reverse the unauthenticated vendor protocol → target the unvalidated heap allocator → ride the absence of mitigations straight to RCE"* — is OS-agnostic. URGENT/11 already proved that.

### ③ Why I excluded automotive ECUs — the boundary of the methodology

This was the only one of the three I judged **low-applicability**, so I left it out of the main analysis. The reason is exactly what shows this methodology's limits.

| Item | Automotive ECU | Why the methodology does not fit |
| --- | --- | --- |
| CPU | Cortex-R / Cortex-M / TriCore | **No MMU; they use an MPU → the DACR trick does not exist** |
| Memory allocation | **Dynamic allocation is frequently forbidden** (MISRA-C, AUTOSAR) | **No heap means no heap exploitation** |
| Attack surface | Mainly CAN/LIN bus, not IP | A remote pre-auth scenario barely holds |
| Execution model | Static schedule, fixed at compile time | Little room to swap a function pointer table at runtime |

**The second row is the crux.** Safety-critical software standards **forbid runtime dynamic allocation outright, or restrict it to initialization**. This vulnerability requires `malloc`/`free` to exist — and that premise is simply absent.

> The automotive industry banned dynamic allocation **for safety, not security** — deterministic execution time and protection against memory exhaustion. Yet that decision **incidentally eliminated this entire vulnerability class.**

---

## References

- [Team PetoWorks — *Pwn2Own 2025 Ireland*](https://peto.works/techblog/pwn2own-2025-ireland-kor) (original write-up)
- [CVE-2025-14233 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-14233)
- [Canon PSIRT — CP2026-001](https://psirt.canon/advisory-information/cp2026-001/)
- [Pwn2Own Ireland 2025: Day One Results — Zero Day Initiative](https://www.zerodayinitiative.com/blog/2025/10/21/pwn2own-ireland-2025-day-one-results)
- [DEF CON 2019: Picture Perfect Hack of a Canon EOS 80D DSLR — Threatpost](https://threatpost.com/hack-of-a-canon-eos-80d-dslr/147214/)
- [URGENT/11 — Armis](https://www.armis.com/research/urgent-11/)
- [Interpeak IPnet TCP/IP stack vulnerability — BD](https://www.bd.com/en-us/about-bd/cybersecurity/bulletin/interpeak-ipnet-tcp-ip-stack-vulnerability)