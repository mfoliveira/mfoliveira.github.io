---
title: "page_owner Part 2: optimizing output"
date: 2026-02-23
tags:
- igalia
- page_owner
---

This blog post is [part of a series]({{< ref "tags/page_owner" >}}) about the `page_owner` debug feature in the Linux memory management subsystem, related to the talk _[Improving `page_owner` for profiling and monitoring memory usage per allocation stack trace](http://www.youtube.com/watch?v=qFdjO3t5F9I)_ presented at [Linux Plumbers Conference 2025](https://lpc.events/event/19/contributions/2202/).

- [Part 1]({{< ref "posts/2026-02-23-page_owner-part-1-quick-introduction" >}}) is a quick introduction to `page_owner` and its debugfs files.
- Part 2 describes challenges with processing `page_owner` files over time and a solution with new debugfs files in Linux v6.19.

# Problem: stack traces over time

As described in Part 1, `page_owner`'s debugfs files contain stack traces for the most part:
- `/sys/kernel/debug/page_owner` has one stack trace per allocated page, and
- `/sys/kernel/debug/page_owner_stacks/show_stacks` lists the stack traces that allocated pages.

Reading and processing a significant amount of stack traces incurs a non-trivial computational cost in CPU and memory (copying to, and processing in, userspace) and storage usage, as the total size for such long strings might become large. This shouldn't be an issue if done only once, but it does pose a concern if done repeatedly.

Take the processing of stack traces one step further and that concern materializes into a technical problem:
> How to store information (say, number of pages) _per-stack trace_ and _over time_?

For that, the stack trace must become a _key_ to be assigned _values_ from multiple reads over time. However, keys are usually numbers or somewhat short identifiers, not such long strings as stack traces (although doable, that is computationally more expensive in CPU and memory usage).

# Workaround: stack trace hashing

One possible solution to this problem is hashing the stack traces and using the resulting hash values as keys.

However, this is inefficient with `page_owner` since there is significant duplication of stack traces on both debugfs files:
- In the `page_owner` file, even on a single read, some stack traces may have tens/hundreds/thousands of duplicates; and they compound on multiple reads over time.
- In the `show_stacks` file, there are no duplicates on a single read, but duplicates frequently happen on multiple reads over time.

With a high ratio of duplication, the dominant component in computational cost is the hashing step, which is significantly more expensive than the remaining step that simply use the resulting keys for storing values.

Additionally, the hashing step is usually repeated with the same data set (stack traces present in previous reads), which means that most of the calculations are discarded and done again on every read -- wasting time and computational resources.

For illustration purposes, compare the execution time of [script `page_owner-to-show_stacks.py`]({{< ref "posts/2026-02-23-page_owner-part-2-optimizing-output/#script-1-page_owner-to-show_stackspy" >}}), which parses the `page_owner` file hashing the stack traces (with the [extremely fast](https://github.com/Cyan4973/xxHash?tab=readme-ov-file#benchmarks) `XXH3_64`) and accumulating the number of pages per stack trace, reporting it at the end -- basically mimicking `show_stacks` -- with just reading the equivalent file.

The single read with hashing is 38.55 times slower:

```shell
# time ./page_owner-to-show_stacks.py </sys/kernel/debug/page_owner >/dev/null

real    0m1.542s
user    0m1.486s
sys     0m0.057s

# time cat /sys/kernel/debug/page_owner_stacks/show_stacks >/dev/null

real    0m0.040s
user    0m0.000s
sys     0m0.040s
```

So, considering the single-read results with the `page_owner` file, it's not compelling to use it for multiple reads. However, multiple reads of the `show_stacks` file instead should perform better, though, as it contains unique stack traces and likely a lower ratio of duplication on multiple reads than in a single read of the former file.

Check the execution time of [script `show_stacks-over-time.py`]({{< ref "posts/2026-02-23-page_owner-part-2-optimizing-output/#script-2-show_stacks-over-timepy" >}}), which parses copies of `show_stacks` (collected over time), similarly hashing the stack traces and storing the number of pages per stack trace over time (that is, per copy).

For 100 copies, the execution time is almost 1 second:

```shell
# time ./show_stacks-over-time.py show_stacks.{1..100} >/dev/null

real    0m0.944s
user    0m0.900s
sys     0m0.044s
```

That is a great improvement (comparing to processing a single read of the `page_owner` file), but this is just a particular case on a lightly stressed, small VM with 1 GiB RAM. There is still the computational cost of hashing, which might increase processing time in cases with more stack traces (that is, a greater number of different code paths for memory allocation were exercised in the kernel).

# Solution: stack trace handle numbers

The hashing of stack traces is only required in order to obtain a _unique identifier_ for each stack trace, so that it can be used as a _key_. However, if such an identifier were already available, the hashing step (and associated computational cost) could be avoided altogether.

Fortunately, that is now the case with Linux 6.19! The stack trace storage used by `page_owner` ( [`stackdepot`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/stackdepot.h?h=v6.19)) provides a _handle number_ to uniquely refer to stack traces -- which meets the requirement. 

Linux 6.19 [contains two new debugfs files](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/Documentation/mm/page_owner.rst?h=v6.19&id=0de9a442eeba4a6435af74120822b10b12ab8449) with _handle numbers_ for optimized output:
- `/sys/kernel/debug/page_owner_stacks/show_handles`: this lists `nr_base_pages:` per `handle:` (instead of per stack trace as in `show_stacks`)
- `/sys/kernel/debug/page_owner_stacks/show_stacks_handles`: this lists `handle:` per stack trace (for resolving handle numbers to stack traces)

For the example in the previous post, `show_stacks` contains:

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

While, for the same snippet, `show_handles` contains:

```
...
handle: 27000838
nr_base_pages: 9643

...
```

And the handle number can be resolved to a stack trace with `show_stacks_handles`:

```
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
handle: 27000838

...
```

## Comparison: `show_stacks` vs. `show_handles`

From the previous post, for `show_stacks`:
```
# time cat /sys/kernel/debug/page_owner_stacks/show_stacks \
  | wc --bytes | numfmt --to=iec
402K

real    0m0.042s
user    0m0.004s
sys     0m0.046s
```

Now, for `show_handles`:

```shell
# time cat /sys/kernel/debug/page_owner_stacks/show_handles \
  | wc --bytes | numfmt --to=iec
31K

real    0m0.015s
user    0m0.004s
sys     0m0.019s
```  

That is only 7.7% of the size and 35.7% of the time! Nice improvements.

Finally, compare the execution time of [script `show_handles-over-time.py`]({{< ref "posts/2026-02-23-page_owner-part-2-optimizing-output/#script-3-show_handles-over-timepy" >}}) with the previous one; it uses handle numbers as keys for stack traces instead of hashing them.

For 100 copies, the execution time is approximately 1/3 of a second, roughly 3 times faster.

```shell
# time ./show_handles-over-time.py show_stacks_handles show_handles.ln.{1..100} >/dev/nul

real    0m0.348s
user    0m0.319s
sys     0m0.030s
```

# Conclusion

The original debugfs files provided by `page_owner` consist mainly of stack traces, which isn't an efficient format for reading and processing repeatedly.

In order to store the number of pages used per stack trace over time, the stack traces must be converted to keys for storing values over time, for which hashing can be used. However, even efficient hashing algorithms incur a significant overhead.

In order to address this issue, Linux 6.19 provides new debugfs files for `page_owner` with _handle numbers_, which are unique identifiers for stack traces and can be used as keys, instead of hashing. 

This optimizes the reading and processing of `page_owner` information, as it reduces the amount of data copied from kernel to userspace and allows storing the number of pages per stack trace over time without the overhead of hashing.

# Scripts

## Script 1: `page_owner-to-show_stacks.py`
```python
#!/usr/bin/env python3
# SPDX-License-Identifier: GPL-2.0
#
# Script to parse /sys/kernel/debug/page_owner, hashing the stack trace
# of each page and accumulating the number of pages per stack trace.
# At the end, print all stack traces and their number of pages in a format
# like /sys/kernel/debug/page_owner_stacks/show_stacks.
#
# Usage: page_owner-to-show_stacks.py </sys/kernel/debug/page_owner
#
# Author: Mauricio Faria de Oliveira <mfo@igalia.com>

import re
import sys
import xxhash

re_page = re.compile('^Page allocated via order ([0-9]+)')
re_stack = re.compile('^ ')
re_empty = re.compile('^$')

pages = {}  # key -> number of pages
stacks = {} # key -> stack trace

for line in sys.stdin:

    # middle lines: try stack trace first as it occurs more often
    if re_stack.match(line):
        stack = stack + line
        continue
        
    # first line
    match = re_page.match(line)
    if match:
        order = int(match.group(1));
        stack = ''
        continue

    # last line
    if re_empty.match(line):
        key = xxhash.xxh3_64_hexdigest(stack)
        nr_pages = 2 ** order

        if key in pages:
            pages[key] += nr_pages
        else:
            pages[key] = nr_pages
            stacks[key] = stack

        continue

for key in stacks.keys():
    print(" " + stacks[key].strip())
    print("nr_base_pages: " + str(pages[key]))
    print()
```

## Script #2: `show_stacks-over-time.py`

```python
#!/usr/bin/env python3
# SPDX-License-Identifier: GPL-2.0
#
# Script to parse /sys/kernel/debug/page_owner_stacks/show_stacks in multiple
# reads, hashing each stack trace and recording the number of base pages per
# stack trace in each read.
# At the end, print all stack traces and their number of pages in each read.
#
# Usage: show_stacks-over-time.py <read1> <read2> <read3> ... <read N>
#
# Author: Mauricio Faria de Oliveira <mfo@igalia.com>

import re
import sys
import xxhash

re_pages = re.compile('^nr_base_pages: ([0-9]+)')
re_stack = re.compile('^ ')
re_empty = re.compile('^$')

stacks = {}	# key -> stack trace (all reads)
pages = {}	# key -> array of number of pages (per read)
read = 0	# number of the current read

if len(sys.argv) < 2:
	exit(1)

files = sys.argv[1:]
nr_files = len(files)

for file in files:
	with open(file, 'r') as fd:
		stack = ''
		for line in fd:
	
			# first lines
			if re_stack.match(line):
				stack = stack + line
				continue
				
			# next to last line
			match = re_pages.match(line)
			if match:
				nr_pages = int(match.group(1));
				continue
		
			# last line
			if re_empty.match(line):
				key = xxhash.xxh3_64_hexdigest(stack)
		
				if key not in stacks:
					stacks[key] = stack;

				if key not in pages:
					pages[key] = {}

				pages[key][read] = nr_pages
		
				stack = ''
				continue

		read += 1

for key in stacks.keys():
	print(" " + stacks[key].strip())

	pages_per_read = []
	for read in range(nr_files):
		nr_pages = 0
		if read in pages[key]:
			nr_pages = pages[key][read]
		pages_per_read.append(str(nr_pages))
	
	print(' '.join(pages_per_read))
	print()
```

## Script #3: `show_handles-over-time.py`

```python
#!/usr/bin/env python3
# SPDX-License-Identifier: GPL-2.0
#
# Script to parse /sys/kernel/debug/page_owner_stacks/show_handles in multiple
# reads, collecting handle numbers and recording the number of base pages per
# handle number in each read.
# At the end, print all stack traces and their number of pages in each read,
# resolving handle numbers with /sys/kernel/debug/page_owner_stacks/show_stacks_handles.
#
# Usage: show_handles-over-time.py <show_stacks_handles> <read1> <read2> <read3> ... <read N>
#
# Author: Mauricio Faria de Oliveira <mfo@igalia.com>

import re
import sys
import xxhash

re_pages = re.compile('^nr_base_pages: ([0-9]+)')
re_stack = re.compile('^ ')
re_empty = re.compile('^$')
re_handle = re.compile('^handle: ([0-9]+)')

stacks = {}	# handle number -> stack trace (all reads)
pages = {}	# handle number -> array of number of pages (per read)
read = 0	# number of the current read

if len(sys.argv) < 3:
	exit(1)

resolver = sys.argv[1]
files = sys.argv[2:]
nr_files = len(files)

for file in files:
	with open(file, 'r') as fd:
		for line in fd:
	
			# first line
			match = re_handle.match(line)
			if match:
				handle = int(match.group(1))
				continue
				
			# next to last line
			match = re_pages.match(line)
			if match:
				nr_pages = int(match.group(1));
				continue
		
			# last line
			if re_empty.match(line):
				key = handle
		
				if key not in pages:
					pages[key] = {}

				pages[key][read] = nr_pages
		
				continue

		read += 1

with open(resolver, 'r') as fd:
    stack = ''

    for line in fd:

        # first line
        if re_stack.match(line):
            stack = stack + line
            continue

        # next to last line
        match = re_handle.match(line)
        if match:
            handle = int(match.group(1))
            continue
        
        # last line
        if re_empty.match(line):
            stacks[handle] = stack
            stack = ''
            continue

for key in pages.keys():
	print(" " + stacks[key].strip())

	pages_per_read = []
	for read in range(nr_files):
		nr_pages = 0
		if read in pages[key]:
			nr_pages = pages[key][read]
		pages_per_read.append(str(nr_pages))
	
	print(' '.join(pages_per_read))
	print()
```
