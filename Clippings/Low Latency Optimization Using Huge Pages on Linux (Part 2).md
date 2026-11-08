---
title: "Low Latency Optimization: Using Huge Pages on Linux (Part 2)"
source: "https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-2/"
author:
  - "[[ndepalo]]"
published: 2022-12-02
created: 2026-11-07
description: "Welcome to the second part of our series on improving latency through the use of huge pages! While the first installment was focused on potential hardware benefits, this post will hone in on the practicality of using huge pages to run software – and the most important caveats to be aware of."
tags:
  - "clippings"
---
##### Written

##### Topic

##### Published

Welcome to the second part of our series on improving latency through the use of huge pages! While the [first installment](https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-1/) was focused on potential hardware benefits, this post will hone in on the practicality of using huge pages to run software – and the most important caveats to be aware of.

### Why is huge page allocation tricky?

One might wonder why this post is even necessary — why can’t one just edit some config file and have the operating system allocate huge pages? It is important to understand that even after configuring Linux to support allocating some 2MiB pages, the OS still uses 4KiB pages for most of its internal workings. Therefore, allocating a huge page is not just about finding one free page; the OS must identify 512 contiguous 4KiB pages, starting at an address that’s a multiple of 2MiB, to create one huge page. This can be quite involved.

### Memory fragmentation

If there is a lot of free memory on the machine, creating a new huge page is usually not a problem. But when free memory becomes constrained, memory often gets fragmented and there might not be any suitable set of 512 contiguous physical free 4KiB pages. The OS must then defragment some of the memory. Typically, the memory used by programs will be moved from one physical page to another. The programs’ data will, of course, not change — it will simply be stored in a different part of the RAM. However, moving a program’s memory involves suspending it for some amount of time if it’s currently running. Therefore, defragmenting can be quite disruptive <sup>1</sup> and make some huge page allocations expensive.

<sup>1</sup> Recent Linux versions [proactively](https://lwn.net/Articles/817905/) defragment to “smooth out” the latency spikes.

### Transparent huge pages

One very easy way to get the memory of long running processes to be backed by huge pages is to use a facility called [*Transparent Huge Pages*](https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html) (THP). It was introduced in Linux 2.6.38 over 10 years ago, so it is commonly available.

When THP is in-use, the operating system will replace the physical backing of processes with huge pages on its own when it deems it possible/necessary <sup>2</sup>. Linux supports two THP modes: the *always* mode, and the *madvise* mode. In the always mode, most memory allocated by programs is eligible to be backed by huge pages. In the madvise mode, programs must explicitly tell the kernel to use huge pages for parts of their memory using the [*madvise*](https://man7.org/linux/man-pages/man2/madvise.2.html) system call.

The always mode is by far the simplest way to access the benefits of huge pages. However, programs have no control over which memory might end up being backed by huge pages and when. In the madvise mode, they can decide which memory could be promoted, but this requires modifying the source code or inserting madvise calls at runtime (using a [LD\_PRELOAD](https://man7.org/linux/man-pages/man8/ld.so.8.html) library for example).

THP is a good choice for a variety of workloads, especially those with long-running memory-hungry programs that can tolerate some occasional latency spikes in exchange for lower latency during most memory accesses. One could also imagine turning on THP during a program’s initialization hoping to maximize the huge page usage, then turning it off to avoid latency spikes. Note that Linux will break down THP transparently if necessary (under memory pressure for example), and that can also be a source of additional latency.

The “magic” of THP comes at the expense of complexity in the operating system; while this feature has existed for a long time, it took a few years to become stable enough for production use. Therefore, a fairly recent kernel is recommended when trying it out. It should also be noted that ‘always mode’ of THP is not suitable for all programs and [some](https://www.couchbase.com/blog/often-overlooked-linux-os-tweaks/) [prominent](https://www.mongodb.com/docs/manual/tutorial/transparent-huge-pages/) [open](https://www.percona.com/blog/2019/03/06/settling-the-myth-of-transparent-hugepages-for-databases/) [source](https://github.com/redis/redis/blob/unstable/src/latency.c#L368) software recommends disabling THP altogether.

<sup>2</sup> Some [tricks](https://mazzo.li/posts/check-huge-page.html) can be used to encourage the kernel to allocate huge pages

### Hugetlbfs

The other way to allocate huge pages is through a pseudo filesystem called [*hugetlbfs*](https://www.kernel.org/doc/html/v4.18/admin-guide/mm/hugetlbpage.html). Some documents also refer to this facility as *explicit huge pages*.

Unlike THP, hugetlbfs uses a specific pool of huge pages. The minimum size of the pool is pre-allocated, while the maximum is a hard upper limit. If pages are available in the pool, allocating a huge page is as fast as allocating a 4KiB page. When no free page is available in the pool but the maximum has not been reached, Linux will attempt to create a fresh new huge page.

Memory in the hugetlbfs pool must be explicitly requested by programs and is unavailable to the operating system for its own use. Therefore, sizing the pool appropriately is of the utmost importance in order not to starve the operating system and programs that will not be using huge pages. On the flip side, hugetlbfs pages cannot be swapped out to the disk (but can be [moved](https://lwn.net/Articles/849860/)).

Be aware that hugetlbfs has an initially surprising and important caveat: when a program requests a new huge page but the pool is already maxed out, the OS will send [SIGBUS](https://man7.org/linux/man-pages/man7/signal.7.html) to the process which, if unhandled, will kill it. To avoid this problem, hugetlbfs will by default reserve pages in its pool during allocation (i.e during [*mmap()*](https://man7.org/linux/man-pages/man2/mmap.2.html)). But it’s still possible to unexpectedly run out of pages during [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) (after a [fork()](https://man7.org/linux/man-pages/man2/fork.2.html), for example <sup>3</sup>). Therefore, as a rule of a thumb, it’s safer not to use hugetlbfs in a process that needs to spawn child processes.

<sup>3</sup> Only the child process can receive SIGBUS after a copy-on-write fault. There are other situations where huge pages allocation can fail and trigger a SIGBUS but they are more rare such as specific [NUMA policy](https://man7.org/linux/man-pages/man2/set_mempolicy.2.html) or [MAP\_NORESERVE](https://man7.org/linux/man-pages/man2/mmap.2.html) mappings.

### Support for huge page allocation in software

Generally, most of the memory usage of programs come from what’s called anonymous memory. It’s obtained by calling a memory allocator. Most of the high performance memory allocators ([jemalloc](http://jemalloc.net/), [tcmalloc](https://google.github.io/tcmalloc/temeraire.html), [mimalloc](https://microsoft.github.io/mimalloc/environment.html), etc.) support allocating 2MiB pages backed by either THP, hugetlbfs, or both. Some programs also natively support allocating huge pages using hugetlbfs for their memory, examples include the [Java VM](https://www.oracle.com/java/technologies/javase/largememory-pages.html) and [postgresql](https://www.postgresql.org/docs/current/kernel-resources.html#LINUX-HUGE-PAGES).

One glaring exception used to be the [glibc](https://www.gnu.org/software/libc/) allocator, which is, for most, the default allocator on Linux. Fortunately, glibc 2.35 introduced native support for allocating hugetlbfs pages controlled by the [glibc.malloc.hugetlb](https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation-Tunables.html) tunable**.**

When using earlier versions of glibc (<2.34 <sup>4</sup>), a [library](https://github.com/libhugetlbfs/libhugetlbfs) can be plugged into the main heap segment allocator <sup>5</sup> to use hugetlbfs pages or THP in madvise mode.

<sup>4</sup> Glibc 2.34 removed support for external libraries to provide huge pages but does not have native support for huge pages.  
<sup>5</sup> Additional segments allocated by the glibc allocator do not use morecore() and therefore will use 4K mappings.

### Running programs from huge pages

A [few](https://github.com/libhugetlbfs/libhugetlbfs) [projects](https://github.com/intel/iodlr) even support running programs out of hugetlbfs pages. These work by copying the program code onto huge pages during initialization. Note that some tools might not understand what happened and struggle to realize that this memory is coming from a program. Therefore, it is recommended to carefully measure your workload ([this paper](https://web.archive.org/web/20230921193747/https://www.intel.com/content/dam/develop/external/us/en/documents/run-perf-opt-bp-large-code-pages-q1update.pdf) has good examples) before looking into running software out of huge pages in order to verify that it does not break your debugging pipeline.

Another drawback to consider is that running programs out of anonymous pages prevents the OS from sharing the pages containing the code between multiple instances. This can significantly increase the memory footprint of running programs and induce L3 thrashing. [Libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs) provides a way to share the remapped segments but it comes with its own set of [caveats](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO).

Another option using THP is the [tmpfs](https://man7.org/linux/man-pages/man5/tmpfs.5.html) filesystem mounted with the ‘huge’ option. Files can be copied to this filesystem before running and the program’s instructions will reside on a THP-enabled region while running <sup>6</sup>. Shared libraries can also be copied and loaded from this filesystem.

<sup>6</sup> Note that before Linux 5.7 a specific configuration option needs to be enabled for this option to be available (CONFIG\_TRANSPARENT\_HUGE\_PAGECACHE)

### Fine tuning the TLB usage

As [part 1](https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-1/) explained, making better use of the TLB is the main benefit of huge pages. Until now, we have described the TLB as an indiscriminate array of entries that store memory translations. However, like the [CPU caches](https://en.wikipedia.org/wiki/CPU_cache), the TLB is split into instructions and code entries, and uses multiple levels. This could affect software performance and should be taken into account.

On x86 CPUs, the TLB has 2 levels. The first level — which has the lowest latency — is split into an instruction specific TLB (*iTLB*) and a data TLB (*dTLB*). Depending on the specific CPU, each level will have specific 4KiB, 2MiB or 1GiB slots in the TLB ([cpuid](http://www.etallen.com/cpuid.html) can be used to retrieve this information).

Here is the output for a [Cascade Lake](https://en.wikichip.org/wiki/intel/microarchitectures/cascade_lake) CPU:

```js
cache and TLB information (2):
   0x63: data TLB: 2M/4M pages, 4-way, 32 entries
         data TLB: 1G pages, 4-way, 4 entries
   0x03: data TLB: 4K pages, 4-way, 64 entries
   0x76: instruction TLB: 2M/4M pages, fully, 8 entries
   0xb6: instruction TLB: 4K, 8-way, 128 entries
   0xc3: L2 TLB: 4K/2M pages, 6-way, 1536 entries
```

On this CPU, each core has a L1 dTLB with 32 entries for 2MiB pages, 64 entries for 4KiB pages and 4 entries for 1GiB pages. The L2 TLB has 1536 entries shared by instruction and data 4KiB and 2MiB pages. Therefore to get the lowest latency, keeping some hot pages as 4KiB might be required.

### Conclusion

In this second post concerning huge pages, we covered the practical use of huge pages and some of the caveats to keep in mind. Various kernel facilities and libraries allow backing both anonymous and file-backed code and data mappings with huge pages, but each have their idiosyncrasies.

We recommend carefully reviewing the benefits and drawbacks of both hugetlbfs and THP. As both come with their own set of caveats, running benchmarks is paramount in order to decide which solution will best fit a given workload. Ultimately, using huge pages optimally can be complicated but worthwhile — we hope this post is a useful starting point.

If you find this content interesting and would like to know more about HRT, check out our [website](https://www.hudsonrivertrading.com/) or consider applying to [join our team](https://www.hudsonrivertrading.com/careers/)!