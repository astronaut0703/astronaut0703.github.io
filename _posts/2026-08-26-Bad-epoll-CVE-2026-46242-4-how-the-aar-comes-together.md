---
layout: post
title: "[Bad Epoll CVE-2026-46242] Addendum: Completing the AAR"
date: 2026-08-26 21:38:00 +0900
categories: [1-day analysis]
tags: [linux-kernel, bad-epoll, cve-2026-46242, uaf, aar, kaslr]
---

## Introduction

While writing the three Bad Epoll blog posts, I thought I had understood the root process of Bad Epoll. But I hadn't.

While talking about Bad Epoll's AAR with the people I study with, I realized that I did not fully understand **how the AAR is actually done**. So I'm writing this additional post.

### What I learned from this process

**1. Complete understanding means reaching a state where I can explain it back to myself, exactly as it is.**

**2. When you read a PoC top to bottom, what stays in your hands is a story: "race → UAF → leak → root."** But knowing the story and knowing *why each step had to take that shape* are different things. I mistook having memorized the story for understanding.

---

## 1. The starting point — a dangling `struct file`

Before getting into the AAR, let's be precise about what we're actually holding.

It is a dangling `struct file`: **a live `epitem` still pointing at a `file` that has already been freed.** Here is the path that gets us there.

### 1-1. Setting the stage

```c
epoll_ctl(ep_waiter, ADD, ep_target, &ev);
```

We make an epoll watch another epoll. At this point `file_target->f_ep` points to `&ep_target->refs`. This `f_ep` is the **only marker** that says "is there an epoll watching this file?" Later, this single field is what brings everything down.

### 1-2. Thread A — the marker is cleared first

```c
close(ep_waiter);
  → __ep_remove()
      WRITE_ONCE(file->f_ep, NULL);   // clear the marker FIRST
      ...                             // ← Race Window
      hlist_del_rcu(&epi->fllink);    // actual list cleanup comes LATER
```

The problem is the ordering. The marker is cleared first, and the actual list cleanup happens afterwards. The space in between is empty.

### 1-3. Race Window — only 6 instructions

When two threads run at the same time, the temporal gap in which another operation can slip in before a particular piece of code executes is called a Race Window. Here that gap is only about **6 kernel assembly instructions** wide.

That is not a width you can hit by chance, so the attacker widens it artificially. The PoC uses `timerfd_create()` for timing and repeats `close(dup(ep_target))` 250 times to stretch the window open.

Thread B slips into that widened gap.

```c
// Thread B: close(ep_target)
__fput(file)
  → eventpoll_release(file)
      if (file->f_ep != NULL) {      // ← decides based on the marker alone
          eventpoll_release_file(file);
          return;
      }
      file->f_op->release(...)
        → ep_eventpoll_release() → kfree(ep_target);
```

Because Thread A has already set `f_ep` to NULL, Thread B **concludes "nobody is watching" and skips the entire cleanup procedure.** It then calls `kfree(ep_target)`.

### 1-4. Reallocation — creating the Victim

Right after `ep_target` is freed, the attacker pushes a new object into that slot.

```c
ep_waiter2 = epoll_create1(0);
ep_target2 = epoll_create1(0);   // ← lands in the slot just freed
epoll_ctl(ep_waiter2, ADD, ep_target2, &ev);
```

This is the target structure the attacker places at the freed memory location in order to exploit the UAF — the **Victim**.

### 1-5. The 8-byte UAF write

Once the delay ends, Thread A — which had been stalled inside the window — finishes its work.

```c
hlist_del_rcu(&epi->fllink);
  → *(epi->fllink.pprev) = next;    // pprev == &ep_target2->refs.first
```

It **writes 8 bytes into a slot that already belongs to someone else.** `ep_target2->refs.first` gets pushed out.

### 1-6. What those 8 bytes cause

1. The waiter list is severed, and `ep_waiter2`'s `epitem` falls out of the list.
2. When `file_target2` is released, the waiter list is empty, so **nobody cleans up that epitem.** The epitem stays in memory as is.
3. In the end, `epitem->ffd.file` is a **dangling pointer** still pointing at the address of the now-gone `file_target2`.

> This is where part 3 ended. The problem is what comes next. I moved past it with a single sentence — "now we get an arbitrary read" — when in fact the whole rest of this post was folded inside that one sentence, and I didn't even know it was folded.

---

## 2. What is the window for reading kernel memory through that dangling file?

### 2-1. The read window — `/proc/self/fdinfo/<ep_waiter2>`

On Linux, you can read the files under `/proc/self/fdinfo/` to check the status information of the files you have opened.

When the attacker asks to read the contents of `/proc/self/fdinfo/<ep_waiter2>`, the kernel's `ep_show_fdinfo()` function is automatically invoked.

### 2-2. Dereferencing the dangling pointer

`ep_show_fdinfo()` walks the list of watch targets (`epitem`) attached to `ep_waiter2` one by one and assembles their information. At that point it runs into **the epitem whose list link was severed earlier.**

The kernel then looks at the file information that epitem points to, `epi->ffd.file`. It is a dangling pointer to already-freed memory. In the meantime, the attacker has reallocated that freed slot with their own data — a **fake `struct file`**.

### 2-3. Riding the chained addresses like dominoes

```c
// fs/eventpoll.c — ep_show_fdinfo()
struct inode *inode = file_inode(epi->ffd.file);   // = f->f_inode

seq_printf(m, "... pos:%lli ino:%lx sdev:%x\n",
    epi->ffd.file->f_pos,   // printed as-is
    inode->i_ino,           // 1 dereference
    inode->i_sb->s_dev);    // 2 dereferences
```

The kernel takes the data the attacker crafted and rides the chained addresses like dominoes, dereferencing along the way and printing the values.

| Printed field | Path | Nature |
|---|---|---|
| `pos` | `epi->ffd.file->f_pos` | printed as-is |
| `ino` | `inode->i_ino` | 1 dereference |
| `sdev` | `inode->i_sb->s_dev` | 2 dereferences |

**The kernel follows the attacker's pointers on the attacker's behalf.** That is the point where the attack becomes possible.

---

## 3. Which path do we break through for a Full AAR?

Let's focus on the `inode->i_sb->s_dev` path.

### 3-1. Why the `i_sb` path and not `i_ino`

Moving `f_inode` around and reading `i_ino` looks possible too. But **the same single `seq_printf` line also dereferences `i_sb`.** In other words, wherever you move `f_inode`, the 8 bytes sitting at the `i_sb` offset from that location must be a **valid, dereferenceable address.** Otherwise the kernel dies right there.

So the `i_ino` path is only usable when "there happens to be a valid pointer sitting next to the address you want to read." This is **Limited AAR**.

The `i_sb → s_dev` path is different. The value it produces is used directly as an address with no additional checks, and the read ends there.

### 3-2. How `inode->i_sb->s_dev` works

```c
ADDR = *(X + off(i_sb))
sdev = *(ADDR + off(s_dev))
```

- `ADDR = *(X + off(i_sb))` — read the memory located at the `i_sb` field's offset (distance) from the `inode` address (`X`) to obtain the `i_sb` address value (`ADDR`).
- `sdev = *(ADDR + off(s_dev))` — read the memory located at the `s_dev` field's offset from the `ADDR` we just obtained, to get the final `sdev` value.

### 3-3. Plugging the AAR in here

**AAR (Arbitrary Address Read)** is the technique of freely reading any address you want within kernel memory.

The attacker arranges for the address they want to read to end up in the `ADDR` slot, and then reads `/proc/self/fdinfo/`. The kernel, without suspecting anything in between, reads the memory at **(desired address) + off(s_dev)** and prints it in the `sdev` field.

---

## 4. So can we obtain the "desired address" directly?

Not directly. **If you know the address you can't change the contents; if you can fill the contents you don't know the address.** This is because of KASLR.

### 4-1. The case where you know the address but can't change the contents

- The kernel source is public, so the offsets are known. With a single successful KASLR leak, you can determine the exact memory address of a global variable.
- But that address lives in the kernel's own system region, so it cannot be overwritten with ordinary user privileges.

### 4-2. The case where you can fill the contents but don't know the address

- If the attacker allocates memory in bulk through kernel-object-creating system calls, they can fill that memory space with arbitrary data of their choosing.
- But the kernel dynamic memory (Heap/SLUB) allocator places memory at a random, variable location every time. Even after filling in the contents, you cannot know exactly where that address is.

| Candidate memory | Do you know the address? | Do you control the contents? |
|---|:---:|:---:|
| Kernel image global variable | ✓ | ✗ |
| Object sprayed onto the heap | ✗ | ✓ |
| **A user-written field in `current`** | **✓** | **✓** |

---

## 5. The solution — a user-written field in `current`

For the attack to work, we need **memory whose address we know exactly and whose contents we can also manipulate.** There is a place where both conditions hold at once.

**The `current` structure** — the kernel manages the information of the currently running process (`task_struct`) through a variable called `current`.

**User-written fields** — inside that structure there is space where values passed from user space into the kernel are stored as-is. Fields such as the name field written by `prctl(PR_SET_NAME)`, or the limit-value field written by `setrlimit()`. A single system call is enough to plant a value of my choosing inside kernel memory.

And the address of `current` itself — we obtain it with the **Limited AAR** described earlier. By picking only safe locations that happen to have a valid pointer next door and reading a few times, we can reach the address of `task_struct`.

> This is the part I had been missing. The limited read **bootstraps itself into a full read.** They are not two separate primitives; one is the raw material for the other.

To summarize:

```
A = current + offsetof(task_struct, field)   ← we know the address (leaked via Limited AAR)
*A = "desired address"                       ← we control the contents (planted via a system call)
```

We use this `A` as the springboard for the AAR.

---

## 6. How do we line up the offset?

We set the fake `struct file`'s `f_inode` **pulled back by the offset**, so that when the kernel reads `i_sb` it lands exactly on `A`.

```c
f_inode = A - offsetof(struct inode, i_sb);
```

Then the kernel's execution flows like this:

```
inode->i_sb              →  reads A  →  "desired address"
"desired address"->s_dev →  prints the 4 bytes at ("desired address" + off(s_dev)) as sdev
```

### A 4-byte AAR obtained

**Why 4 bytes?** Because the data type of the `s_dev` field (`dev_t`) is 4 bytes. If you need 8 bytes, split the target address into `T` and `T+4` and read twice.

---

## Closing

Compressed into a single line, the whole flow is this:

> An 8-byte UAF write obtained from a 6-instruction race severs the waiter list and turns a `struct file` into a dangling pointer; the fake file placed on top of it is then read out for us by `/proc/self/fdinfo`, and with `current` as a springboard, the limited read is promoted into a full arbitrary read.

When I was writing part 3, I only knew the first half of that sentence. The second half was lumped together as "and then we get an arbitrary read," and I didn't even know it had been lumped together.

Most write-ups compress the hardest stretch into a single sentence. And when reading, that is the sentence we pass over the fastest. From now on, when I meet a sentence like that, I intend to stop and unfold whatever is folded up inside it.

---

### References

- [oss-sec: CVE-2026-46242 ("Bad Epoll") disclosure](https://seclists.org/oss-sec/2026/q3/105)
- [Zero Hunt — Bad Epoll technical analysis](https://zerohunt.ai/blog/bad-epoll-cve-2026-46242-linux-kernel-root/)
- [J-jaeyoung/bad-epoll (PoC)](https://github.com/J-jaeyoung/bad-epoll)