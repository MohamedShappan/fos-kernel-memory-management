# FOS Kernel — Memory Management & Scheduling

Kernel-level memory management for FOS, the x86 educational operating system used in the Operating
Systems course at Ain Shams University: paging, frame allocation, page-fault handling, working-set
tracking, and the scheduler that runs on top of it.

> **Scope of this repository.** FOS ships as a skeleton kernel where the course provides the boot
> path, drivers, and test harness, and students implement the memory-management and scheduling
> internals. This repository is a **source snapshot of the `inc/` and `kern/` trees** — it is a
> record of the implementation work, **not a buildable project**: the top-level makefile, `boot/`,
> `lib/`, and `user/` directories are not included, and neither are the course-provided sources
> that some headers here declare. See [Building](#building) below.

## Overview

An operating system has to hand each process a private, contiguous-looking address space that is
actually made of scattered physical frames, and it has to keep working when physical memory runs
out. This project implements that machinery: the page tables that create the illusion, the frame
allocator that backs it, the fault handler that fills pages in on demand, and the replacement policy
that decides what to evict when frames are exhausted.

## Architecture

```
                 user program touches an unmapped address
                                  │
                            CPU raises #PF
                                  │
   kern/trap/trap.c ──────────────┴─→ page_fault_handler()
                                            │
   kern/mem/memory_manager.c   page tables, frame alloc/free, map/unmap
   kern/mem/working_set_manager.c   per-environment working set, LRU-approx replacement
   kern/disk/pagefile_manager.c     page file backing store
                                            │
   kern/cpu/sched.c            Round Robin / MLFQ scheduling over ready queues
   kern/proc/user_environment.c     environment (process) lifecycle
```

## What is implemented here

**Paging and frame management** (`kern/mem/memory_manager.c`, 554 lines)

- `initialize_paging()` — page directory setup and the kernel's initial mappings
- `allocate_frame()` / `free_frame()` / `decrement_references()` — physical frame allocator with
  reference counting, so shared frames are only reclaimed when the last mapping goes away
- `get_page_table()` / `create_page_table()` — page-table walk with allocate-on-demand
- `map_frame()` / `unmap_frame()` / `loadtime_map_frame()` — virtual→physical mapping, including a
  load-time variant that maps without triggering a fault
- `tlb_invalidate()` — TLB shootdown on mapping changes
- `calculate_available_frames()` — free/modified/buffered frame accounting

**Boot-time allocator** (`kern/mem/boot_memory_manager.c`, 437 lines) — the bump allocator used
before the real frame allocator exists, plus the frame-info array it carves out.

**Working set & page replacement** (`kern/mem/working_set_manager.c`, 244 lines) — per-environment
working-set tracking with an approximated-LRU list policy (`PG_REP_LRU_LISTS_APPROX`), and dynamic
working-set resizing (`double_WS_Size`, `half_WS_Size`).

**Scheduling** (`kern/cpu/sched.c`, `kern/proc/priority_manager.c`) — Round Robin and
Multi-Level Feedback Queue schedulers over new/ready/exit queues, with program priorities.

## Tech Stack

C · x86 assembly · QEMU · GCC/GNU Make (in the full course environment)

## Building

This snapshot does not build on its own. To run the code you need the complete FOS project
skeleton from the Ain Shams OS course, into which `inc/` and `kern/` from this repository can be
dropped, then:

```bash
make        # build the kernel
make qemu   # boot it under QEMU
```

The course kernel exposes a command prompt; the test suites in `kern/tests/`
(`test_dynamic_allocator.c`, `test_kheap.c`, `test_priority.c`) are invoked from there.

## Engineering Decisions

**Reference-counted frames.** `FrameInfo.references` is what makes shared memory and copy-on-write
safe: `unmap_frame()` decrements rather than frees, and a frame returns to the free list only when
nothing maps it. Getting this wrong produces use-after-free bugs that surface as random corruption
much later.

**Allocate page tables lazily.** `get_page_table()` walks the directory and only calls
`create_page_table()` when a mapping actually needs one, so a sparse address space costs sparse
memory.

**Approximated LRU over exact LRU.** True LRU needs a timestamp update on every memory access,
which is impossible without hardware support. The list-based approximation uses the accessed bit the
MMU already maintains, trading precision for a policy that is actually affordable.

## Limitations

- Not buildable standalone (see above)
- Some headers here (`kheap.h`, `chunk_operations.h`, `shared_memory_manager.h`,
  `paging_helpers.h`) declare functions whose sources live in parts of the course tree that are
  not part of this snapshot
- No automated CI — the course tests run interactively from the kernel command prompt
