---
title: "page_owner Part 1: a quick introduction"
date: 2026-02-20
tags:
- igalia
- page_owner
---

This blog post is [part of a series]({{< ref "tags/page_owner" >}}) about the `page_owner` debug feature in the Linux memory management subsystem, related to the talk _[Improving `page_owner` for profiling and monitoring memory usage per allocation stack trace](http://www.youtube.com/watch?v=qFdjO3t5F9I)_ presented at [Linux Plumbers Conference 2025](https://lpc.events/event/19/contributions/2202/).

# What is `page_owner`?

In the Linux kernel, `page_owner` is a debug feature that tracks the memory allocation (and release) of pages in the system -- so as to tell the '_owner of a page_' ;-).

For each memory allocation, `page_owner` stores its order, [GFP flags](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/gfp_types.h?h=v6.19), stack trace, timestamp, command, process ID (PID) and thread-group ID (TGID), and more. It also stores some information when pages are freed (stack trace, timestamp, PID and TGID).

With `page_owner`, one can find out "_What allocated this page?_" and "_How many pages are allocated by this particular stack trace, PID, or comm_", for example.

This is [struct page_owner](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_owner.c?h=v6.19#n24) in Linux `v6.19`. It stores additional information per-page, as an extension of `struct page` with `CONFIG_PAGE_EXTENSION`.

```c
struct page_owner {
        unsigned short order;
        short last_migrate_reason;
        gfp_t gfp_mask;
        depot_stack_handle_t handle;
        depot_stack_handle_t free_handle;
        u64 ts_nsec;
        u64 free_ts_nsec;
        char comm[TASK_COMM_LEN];
        pid_t pid;
        pid_t tgid;
        pid_t free_pid;
        pid_t free_tgid;
};
```

# Usage

In order to use `page_owner`, build the kernel with `CONFIG_PAGE_OWNER=y` (see `mm/Kconfig.debug`) and boot the kernel with `page_owner=on`. 

The debugfs file `/sys/kernel/debug/page_owner` provides the information in `struct page_owner` for every page, listed per `PFN` (page frame number). 

This example shows the entry for a page (line continuation added for clarity) -- it tells "_What allocated this page?_":

```
# cat /sys/kernel/debug/page_owner
...
Page allocated via order 0, \
  mask 0xd2cc0(GFP_KERNEL|__GFP_NOWARN|__GFP_NORETRY|__GFP_COMP|__GFP_NOMEMALLOC), \
  pid 5640, tgid 5640 (stress-ng-brk), ts 414987114269 ns
PFN 0x114 type Unmovable Block 0 type Unmovable Flags 0x200(workingset|node=0|zone=0)
 get_page_from_freelist+0x1416/0x1600
 __alloc_frozen_pages_noprof+0x18c/0x1000
 alloc_pages_mpol+0x43/0x100
 new_slab+0x349/0x460
 ___slab_alloc+0x811/0xd90
 __kmem_cache_alloc_bulk+0xb8/0x1f0
 __prefill_sheaf_pfmemalloc+0x42/0x90
 kmem_cache_prefill_sheaf+0xa9/0x240
 mas_preallocate+0x32f/0x420
 __split_vma+0xdc/0x300
 vms_gather_munmap_vmas+0xa4/0x240
 do_vmi_align_munmap+0xe9/0x180
 do_vmi_munmap+0xcb/0x160
 __vm_munmap+0xa7/0x150
 __x64_sys_munmap+0x16/0x20
 do_syscall_64+0xa4/0x310

...
```

One can use [tools/mm/page_owner_sort](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/mm/page_owner_sort.c?h=v6.19) to process the information in the file, or come up with custom commands, scripts, or programs.

For example: calculate the total size of pages allocated by `stress-ng-brk` with any order, in MiB:

```shell
# COMM=stress-ng-brk
# cat /sys/kernel/debug/page_owner \
  | awk -F '[ ,]' \
    '/^Page allocated via order .* \('${COMM}'\)/ { PAGES+=2^$5 } 
     END { print PAGES*4096/2**20 " MiB" }'
0.0429688 MiB
```

More information about `page_owner` is available in [Documentation/mm/page_owner.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/Documentation/mm/page_owner.rst?h=v6.19).

# Problem: output size

In the `page_owner` file, note the significant amount of text that is produced _per-page_: 745 bytes, in the example above.

Considering a system with 1 GiB of RAM and 4 kB pages, fully allocated, with similarly sized entries per page, the output size might reach approximately 186 MiB! (`745 [bytes/page] * (2**30 [bytes of RAM] / 4096 [bytes/page]) / 2**20 [bytes/MiB]`)

For validation, a test VM with 1 GiB of RAM after just a warm-up level of stress (`stress-ng --sequential --timeout 1`) produced 125 MiB, which was not quick to read even in idle state:

```
# time cat /sys/kernel/debug/page_owner \
  | wc --bytes | numfmt --to=iec
125M

real    0m3.009s
user    0m0.512s
sys     0m3.542s
```

While this might not be a serious issue for reading and processing the file only once, it can likely impact a sequence of operations.

# Alternative: optimized output

Fortunately, another debugfs file, `/sys/kernel/debug/page_owner_stacks/show_stacks`,  provides an optimized output for obtaining the memory usage per stack trace. Even though it doesn't address all needs as the generic output, it resembles the default operation of `page_owner_sort` (without `PFN` lines) and provides an often interesting information for kernel development or analysis.

This example shows the entry for a stack trace -- it tells "_How many pages are allocated by this particular stack trace?_"

```shell
# cat /sys/kernel/debug/page_owner_stacks/show_stacks
...
 get_page_from_freelist+0x1416/0x1600
 __alloc_frozen_pages_noprof+0x18c/0x1000
 alloc_pages_mpol+0x43/0x100
 folio_alloc_noprof+0x56/0xa0
 page_cache_ra_unbounded+0xd9/0x230
 filemap_fault+0x305/0x1000
 __do_fault+0x2c/0xb0
 __handle_mm_fault+0x6f4/0xeb0
 handle_mm_fault+0xd9/0x210
 do_user_addr_fault+0x205/0x600
 exc_page_fault+0x61/0x130
 asm_exc_page_fault+0x26/0x30
nr_base_pages: 9643

...
```

The `nr_base_pages` field tells the number of base pages (i.e., not huge pages) allocated by a stack trace. So, this particular stack trace for _readahead_ (`page_cache_ra_unbounded()`) has allocated approximately 37 MiB (`9643 [pages] * 4096 [bytes/page] / 2**20 [ bytes/MiB]`).

Note this file is more efficient for this particular purpose: just 402 KiB in less than 0.05 seconds. (That is 0.3% of the size and 1.7% of the time):

```
# time cat /sys/kernel/debug/page_owner_stacks/show_stacks \
  | wc --bytes | numfmt --to=iec
402K

real    0m0.042s
user    0m0.004s
sys     0m0.046s
```

# Conclusion

The `page_owner` debug feature (enabled with `CONFIG_PAGE_OWNER=y` and `page_owner=on`) provides information about the memory allocation of pages in the system in debugfs files `/sys/kernel/debug/page_owner` with  a generic format (dense description per-page) and `/sys/kernel/debug/page_owner_stacks/show_stacks` with an optimized format (number of base pages per stack trace).

