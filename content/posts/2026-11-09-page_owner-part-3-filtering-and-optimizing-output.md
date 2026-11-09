---
title: "page_owner Part 3: filtering (and optimizing!) output"
date: 2026-11-09
tags:
- igalia
- page_owner
---

This blog post is [part of a series]({{< ref "tags/page_owner" >}}) about the `page_owner` debug feature in the Linux memory management subsystem, related to the talk [_Linux page_owner: memory profiling at page granularity with stack traces_](https://www.youtube.com/live/Uq6HAXrwA_Q?t=28077s) presented at [FOSForums 2026](https://mfo.dev.br/posts/2026-09-02-fosforums-2026-talking-about-page_owner/).

- [Part 1]({{< ref "posts/2026-02-23-page_owner-part-1-quick-introduction" >}}) is a quick introduction to `page_owner` and its debugfs files.
- [Part 2]({{< ref "posts/2026-02-23-page_owner-part-2-optimizing-output.md" >}}) describes challenges with processing `page_owner` files over time and a solution with new debugfs files in Linux v6.19.
- Part 3 describes the filtering features for `page_owner` in Linux 7.3.

# Filtering features

Linux 7.3 added _filtering features_ to `page_owner`: the [`print_mode` filter](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8570b65220e80f0dafa517edd07fb5e63eff5344) and the [NUMA node filter](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bed80036125a18bb3784c59e644a12027a17f4a7), contributed by Zhen Ni, for more efficient debugging — specially in large-memory systems, and also helpful for high-frequency profiling.

The _print mode_ filter enables an output optimization for `/sys/kernel/debug/page_owner` (detailed-output file) similar to `/sys/kernel/debug/page_owner_stacks/show_handles` (optimized-output file [introduced](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3b52b9e31a860df97bb46e9f3bfdab9f36d5d893) in Linux 6.19). It replaces stack traces with their handle numbers, which reduces duplicate information — saving processing time and storage usage.

The handle numbers can be resolved to stack traces with `/sys/kernel/debug/page_owner_stacks/show_stacks_handles` (also [introduced](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5513cfbcf4da97ba0476d63e7670fdfcde59580f) in Linux 6.19), which lists stack traces along with their handle numbers.

The _NUMA node_ filter, intuitively, filters the pages listed in `/sys/kernel/debug/page_owner` by NUMA node(s). 

Both filters are described in their commit messages, and the files `show_handles` and `show_stacks_handles` are covered in [Part 2]({{< ref "posts/2026-02-23-page_owner-part-2-optimizing-output.md" >}}).

# Configuration interface

The configuration interface has an elegant design, [suggested](https://lore.kernel.org/all/agHRhlfIYbaqHHMK@tiehlicka/) by Michal Hocko. The user first writes the filter(s) — that is, `mode=...` and/or `nid=...` line(s) — to the `page_owner` file, and then reads it as usual. The resulting contents have such filter(s) applied. 

This not only avoids adding more files in `debugfs` for configuring the filters, but it also keeps the filter configuration _per file descriptor_ when accessing `/sys/kernel/debug/page_owner`, which allows concurrent users to have independent configuration, unaffected by each other.

The [`page_owner_filter`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/mm/page_owner_filter.c?h=v7.3) tool has been added to help with that, with the options `--mode <stack|handle|stack_handle>` for printing only stack traces (default), only handle numbers, or both, and `--node <nodes>` for selecting NUMA nodes, which can take a list, range, or both mixed.

The shell also works just fine. The trick is, the same file descriptor must be used for both writing the filter configuration and reading the contents. So, it is required to manually open/close and write/read with redirection, otherwise the shell uses different file descriptors.

For example, using file descriptor 3:

1) Open the file in read-write mode as file descriptor 3: \
`# exec 3<>/sys/kernel/debug/page_owner`
2) Write the filter line(s) into it: \
`# echo 'mode=handle' >&3` \
and/or \
`# echo 'nid=1' >&3`
3) And read it, optionally into a file: \
`# cat <&3 >page_owner.filtered`
4) Finally, close it: \
`# exec 3>&-`

## Examples

Example with `mode=handle` (note there are only handle numbers, no stack traces):

```
# exec 3<>/sys/kernel/debug/page_owner
# echo 'mode=handle' >&3

# head <&3
Page allocated via order 0, mask 0x0(), pid 0, tgid 0 (swapper), ts 125421401 ns
PFN 0x100000 type Unmovable Block 2048 type Unmovable Flags 0x200000000000004(referenced|node=0|zone=2)
handle: 1048577

Page allocated via order 0, mask 0x0(), pid 0, tgid 0 (swapper), ts 125422032 ns
PFN 0x100001 type Unmovable Block 2048 type Unmovable Flags 0x200000000000004(referenced|node=0|zone=2)
handle: 1048577

Page allocated via order 0, mask 0x0(), pid 0, tgid 0 (swapper), ts 125422143 ns
PFN 0x100002 type Unmovable Block 2048 type Unmovable Flags 0x200000000000004(referenced|node=0|zone=2)

# exec 3>&-
```

Example with `mode=handle` and `nid=1` (note there are only `node=1` pages):
```
# exec 3<>/sys/kernel/debug/page_owner
# echo 'mode=handle' >&3
# echo 'nid=1' >&3

# head <&3
Page allocated via order 0, mask 0x0(), pid 0, tgid 0 (swapper), ts 215517152 ns
PFN 0x480000 type Unmovable Block 9216 type Unmovable Flags 0x600000000000000(node=1|zone=2)
handle: 1048577

Page allocated via order 0, mask 0x0(), pid 0, tgid 0 (swapper), ts 215517302 ns
PFN 0x480001 type Unmovable Block 9216 type Unmovable Flags 0x600000000000000(node=1|zone=2)
handle: 1048577

Page allocated via order 0, mask 0x252000(__GFP_NOWARN|__GFP_NORETRY|__GFP_COMP|__GFP_THISNODE), pid 0, tgid 0 (swapper/0), ts 224577206 ns
PFN 0x480002 type Unmovable Block 9216 type Unmovable Flags 0x600000000000200(workingset|node=1|zone=2)

# exec 3>&-
```

Example with default configuration (without `mode=handle` or `nid=1`):
```
# head /sys/kernel/debug/page_owner
Page allocated via order 0, mask 0x0(), pid 0, tgid 0 (swapper), ts 125421401 ns
PFN 0x100000 type Unmovable Block 2048 type Unmovable Flags 0x200000000000004(referenced|node=0|zone=2)
 register_early_stack+0x32/0x65
 init_page_owner+0x33/0x330
 page_ext_init+0x2d2/0x320
 mm_core_init+0x10c/0x130

Page allocated via order 0, mask 0x0(), pid 0, tgid 0 (swapper), ts 125422032 ns
PFN 0x100001 type Unmovable Block 2048 type Unmovable Flags 0x200000000000004(referenced|node=0|zone=2)
 register_early_stack+0x32/0x65
```

# Measuring improvements

The improvements in processing time and storage usage obtained with the `print_mode` filter depend on how many characters are reduced by replacing stack traces with handle numbers. It is proportional to the number of allocated pages / stack traces printed, and the length of each stack trace (i.e., number of stack frames/lines, and length of function names and instruction offsets). Thus, of course, it varies with the workload.

For demonstration purposes, a high duplicate stack trace rate is fit by `stress-ng`'s `mmap` stressor, which can allocate an arbitrary amount of memory with a single `mmap()` call (99% of total memory, in this example).

That is paired with `strace`'s delay injection starting in the third `munmap()` (found empirically) in order to keep  memory allocated for 5 minutes (300 seconds), providing enough time for collecting data and maintaining page allocations as similar as possible.

```
# strace --inject=munmap:delay_enter=300s:when=3 --follow-forks \
  -- stress-ng --mmap 1 --mmap-bytes '99%' --timeout 300s \
  >/dev/null 2>&1 &

# free -g
               total        used        free      shared  buff/cache   available
Mem:              31          31           0           0           0           0
Swap:              0           0           0
```

The baseline is `page_owner` with the default configuration (no filters). It is 4.2 GiB in size, taking approximately 18 seconds to be read (measured three times).

```
# time wc --bytes </sys/kernel/debug/page_owner | numfmt --to=iec-i
4.2Gi

real	0m18.407s
user	0m0.093s
sys	0m18.314s
```

The test is `page_owner` with the `mode=handle` filter. It is 2.1 GiB in size, taking approximately 3 seconds to be read (measured 3 times).

```
# exec 3<>/sys/kernel/debug/page_owner \
  && echo 'mode=handle' >&3 \
  && time wc --bytes <&3 | numfmt --to=iec-i \
  && exec 3>&-
2.1Gi

real	0m2.794s
user	0m0.112s
sys	0m2.682s

# killall -9 strace stress-ng
```
The measured improvement is _50% of the size in 1/6 of the time!_

Remember that, for `mode=handle`, the `show_stacks_handles` file is required and must also be considered. However, its size is negligible compared to the `page_owner` file (~0.01%):

```
# wc --bytes </sys/kernel/debug/page_owner_stacks/show_stacks_handles | numfmt --to=iec-i
298Ki
```

And any additional processing time required for resolving handle numbers to stack traces with `show_stacks_handles` is usually limited to just the handle numbers/stack traces of interest, only in post-processing/presentation stages, which should also be negligible compared to processing all stack traces in every read of `page_owner`.

# Conclusion

The new filtering features for `page_owner` in Linux 7.3 provide optimized output for `/sys/debug/page_owner`, which is significantly faster to read and smaller to store, that is specially relevant on large-memory systems and helpful for high-frequency profiling.
