**必须掌握的知识（按层次）**

1. **进程与线程**：fork/exec、信号、调度策略（CFS、实时优先级）、cgroups 和 namespaces（容器的底层）。
2. **内存**：虚拟内存、page cache、mmap、huge pages、NUMA、OOM 机制。
3. **I/O 模型**：文件描述符、阻塞/非阻塞、epoll、io_uring。
4. **网络栈**：socket API、TCP 状态机与拥塞控制、sk_buff 收发路径、NAPI、netlink/rtnetlink（路由和接口事件都靠它）、XDP 与 eBPF。
5. **并发**：futex、内存序（memory ordering）、原子操作、无锁结构、RCU。
6. **调试与性能**：gdb + core dump、strace、perf、ftrace、bpftrace、ASan/Valgrind。
7. **链接与 ABI**：ELF、动态链接、符号解析、glibc 与系统调用的关系。

**书籍**

- **The Linux Programming Interface**（Michael Kerrisk）：系统编程的首选参考，比 APUE 更贴近 Linux。
- **Advanced Programming in the UNIX Environment**（Stevens）：经典，和上一本二选一即可。
- **UNIX Network Programming, Vol. 1**（Stevens）：socket 编程的根基。
- **Systems Performance, 2nd ed.** 和 **BPF Performance Tools**（Brendan Gregg）：性能分析方法论和工具的标准读物。
- **Computer Systems: A Programmer's Perspective**（CS:APP）：链接、内存、缓存的底层视角。
- **Is Parallel Programming Hard, And, If So, What Can You Do About It?**（Paul McKenney，免费）：RCU 和内存序的权威。
- **Linux Kernel Development**（Robert Love）：内核概念入门，版本较老，读思路不读细节。

**博客与网站**

- **LWN.net**：内核动态的第一手来源，内核相关问题几乎都能在这里找到深度解析。
- **brendangregg.com**：性能工具和火焰图的原始出处。
- **Packagecloud 博客的 "Monitoring and Tuning the Linux Networking Stack"**（收包、发包两篇）：网络栈收发路径讲得最完整的长文。
- **Cloudflare 博客**：大量内核网络、eBPF、生产事故分析。
- **jvns.ca**（Julia Evans）：把 strace、网络等主题讲得直观易懂。
- **preshing.com**：内存序和无锁编程。

**视频（YouTube）**

- **Netdev Conference 频道**：Linux 网络子系统开发者的演讲，网络方向最有价值。
- **Linux Plumbers Conference** 和 **Kernel Recipes**：内核各子系统的现状与设计讨论。
- **Brendan Gregg 的演讲**，如 "Linux Performance Tools"、"Broken Linux Performance Tools"。
- **Herb Sutter 的 "atomic<> Weapons"**（C++ and Beyond 2012）：内存模型讲得最清楚的视频之一。
- **MIT 6.S081 课程**（基于 xv6）：想补操作系统原理时用。

时间有限的话，优先级可以是：**TLPI 作参考书随查 + Systems Performance + LWN + Netdev 演讲**，其余按需。


那这几本可以跳过了。你读那批书之后，Linux 有几块是实质性的新东西，经典书里都没有，值得补：

**近十年的主要变化**

- **eBPF**：从包过滤器变成了内核里的通用可编程层——观测（bpftrace）、网络（XDP、TC）、安全、甚至调度。BTF 和 CO-RE 解决了跨内核版本兼容的问题。
- **io_uring**（5.1，2019）：基于共享环形队列的异步 I/O，取代 aio，减少系统调用开销。
- **调度器**：6.6 用 EEVDF 取代了 CFS；6.12 合入 sched_ext，允许用 BPF 写调度策略；同在 6.12，PREEMPT_RT 正式进入主线。
- **cgroup v2 与 PSI**：统一层级，PSI（Pressure Stall Information）直接给出 CPU、内存、I/O 的资源压力指标。
- **网络数据面**：XDP 和 AF_XDP（内核内的高速收包路径，部分替代 DPDK 的场景）、page_pool、busy polling、BBR、devlink 和 switchdev（内核原生的交换芯片抽象，和 SONiC 走的 SAI 路线是两条思路，面试时能对比说出来有用）。
- **内存管理**：multi-gen LRU、folio。
- **Rust 进入内核**（6.1 起）。

**对应的读物**

- **Learning eBPF**（Liz Rice，2023）：薄，讲清 eBPF 的程序模型、verifier、CO-RE，补这一块效率最高。
- **BPF Performance Tools**（Brendan Gregg，2019）：和你读过的 Systems Performance 是配套的，偏工具。
- **Efficient IO with io_uring**（Jens Axboe 写的 PDF）：io_uring 作者自己的设计说明。
- **Understanding Software Dynamics**（Richard Sites，2021）：Google 资深工程师讲如何用低开销追踪定位尾延迟。和你做过的尾延迟工作直接相关，如果没读过可以看。
- **LWN 上的专题**：搜 EEVDF、sched_ext、folio、io_uring，每个主题都有系列文章讲来龙去脉。
- **内核文档 Documentation/networking/switchdev.rst**：短，十几分钟读完，用来理解内核原生交换模型和 SAI 的区别。

如果只选两个：**Learning eBPF** 和 **switchdev 文档**，和你当前准备的方向最贴近。