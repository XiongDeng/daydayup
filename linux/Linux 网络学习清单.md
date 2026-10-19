 
Oct 5, 2026 · @Michael

## 学习路线

按五个阶段推进，前一阶段是后一阶段的基础；每阶段的具体材料见下面各节。

1. **收发包路径**：网卡中断、NAPI、sk\_buff、GRO/GSO、RSS/RPS/XPS、qdisc。目标是能画出一个包从网卡到 socket 再到网卡的完整路径，并知道每一环的可调参数。
2. **netlink 与路由子系统**：rtnetlink 消息、FIB 结构（IPv4 LC-trie、IPv6 fib6）、nexthop 对象、策略路由、VRF。这是 FRR、fpmsyncd、SONiC 与内核交互的接口层。
3. **eBPF 与 XDP**：程序类型、verifier、map、XDP 与 TC 挂载点、AF\_XDP。
4. **Linux 上的 bridge、VXLAN、EVPN**：bridge 的 FDB 与 VLAN 过滤、VXLAN 设备与 FDB、FRR 的 EVPN 实现、与内核的对接。
5. **交换芯片抽象**：switchdev（内核原生模型）与 SAI（SONiC 路线）的对比。

## 书籍

内核网络栈的书都停在 3.x/4.x 内核，用来建立结构，细节以当前源码和 LWN 为准。

| 书 | 作者 / 年份 | 对应阶段 | 读法 |
| --- | --- | --- | --- |
| [Linux Kernel Networking: Implementation and Theory](https://www.apress.com/9781430261964) | Rami Rosen，Apress，2014 | 1、2 | 主读：sk\_buff、路由子系统、邻居子系统各章；无线、蓝牙章节跳过 |
| Understanding Linux Network Internals | Christian Benvenuti，O'Reilly，2005 | 1、2 | 2.6 内核，过时但对 bridge、邻居子系统、路由缓存的设计动机讲得最透；只查阅 |
| [Cloud Native Data Center Networking](https://www.oreilly.com/library/view/-/9781492045595/) | Dinesh Dutt，O'Reilly，2019 | 2、4 | 主读：Clos、BGP、EVPN 均以 FRR + Linux 举例，作者曾任 Cumulus 首席科学家 |
| BGP in the Data Center / EVPN in the Data Center | Dinesh Dutt，O'Reilly 报告，2017 / 2018 | 4 | 各约 100 页，前者讲 unnumbered 与 RFC 7938，后者讲 Linux 上的 EVPN；可作为上一本的速读版 |
| [Learning eBPF](https://cilium.isovalent.com/hubfs/Learning-eBPF%20-%20Full%20book.pdf) | Liz Rice，O'Reilly，2023 | 3 | 约 230 页，Isovalent 官网提供免费 PDF；重点读 verifier、CO-RE、网络章节 |
| BPF Performance Tools | Brendan Gregg，2019 | 3 | 只读网络章节（socket、TCP、网卡层的追踪工具） |

## 必读长文与官方文档

这一类比书更新、更贴近当前内核，是学习的主体。

| 材料 | 阶段 | 说明 |
| --- | --- | --- |
| [Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/) | 1 | Packagecloud 长文，从网卡 DMA 到 socket 逐环节讲代码路径和对应的监控、调优点；以 igb 驱动为例 |
| [同上：Sending Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data) | 1 | 发包方向的配对篇，覆盖 qdisc、XPS、驱动发送队列 |
| [Scaling in the Linux Networking Stack](https://docs.kernel.org/networking/scaling.html) | 1 | 内核官方文档，RSS、RPS、RFS、aRFS、XPS 的原理与配置 |
| How SKBs work（David Miller，vger.kernel.org/\~davem/skb.html） | 1 | 前网络子系统维护者写的 sk\_buff 指针布局说明，短 |
| [Netlink Handbook](https://docs.kernel.org/userspace-api/netlink/index.html) | 2 | 内核官方：Classic 与 Generic Netlink、消息格式、YAML 协议规范；配合 man 7 rtnetlink |
| [The eXpress Data Path（CoNEXT 2018 论文）](https://research.redhat.com/?p=22875) | 3 | XDP 的设计论文，讲清它与 DPDK 内核旁路的取舍 |
| Cilium 的 BPF and XDP Reference Guide（docs.cilium.io） | 3 | eBPF 程序类型、map、verifier、XDP/TC 挂载的工程参考 |
| [VXLAN: BGP EVPN with FRR](https://vincent.bernat.ch/en/blog/2017-vxlan-bgp-evpn) | 4 | Vincent Bernat 的实操文：Linux bridge + VXLAN 设备 + FRR EVPN 的分工；同作者的 VXLAN & Linux 一文讲内核 VXLAN 设备本身 |
| FRR 官方文档的 EVPN 章节（docs.frrouting.org） | 4 | 当前 FRR 的 EVPN 配置与多归属实现，以它为准 |
| [Ethernet switch device driver model (switchdev)](https://docs.kernel.org/networking/switchdev.html) | 5 | 内核原生交换芯片模型：每个前面板口一个 netdev，bridge FDB 下发到芯片；读完与 SAI 对比 |

## 博客

持续跟踪用 LWN 和 Cloudflare 两个就够，其余按主题查。

| 博客 | 主题 | 推荐入口 |
| --- | --- | --- |
| LWN.net | 内核网络子系统的变更与设计讨论 | 按关键词搜：XDP、page\_pool、nexthop objects、BIG TCP、devmem TCP |
| [Cloudflare 博客](https://blog.cloudflare.com/how-to-receive-a-million-packets) | 生产环境中的收包性能、内核网络调优与故障分析 | How to receive a million packets per second；How to drop 10 million packets per second（对比 iptables、tc、XDP 各层丢包的代价） |
| [Vincent Bernat](https://vincent.bernat.ch/en/blog/2017-vxlan-bgp-evpn) | Linux 上的 VXLAN、EVPN、路由实现 | 上节已列的 EVPN 文及 VXLAN & Linux |
| Brendan Gregg 的博客 | 网络相关的追踪与性能工具 | 网络章节的 bpftrace/BCC 工具说明 |
| Arthur Chiao 的博客（arthurchiao.art） | 中文；Packagecloud 收发包长文的译注版，及 Cilium、eBPF 系列 | 适合与英文原文对照，速度更快 |
| netdev 邮件列表（lore.kernel.org/netdev） | 网络子系统补丁与讨论的一手来源 | 不需要通读，查某个特性的来龙去脉时用 |

## 视频

Netdev 会议是 Linux 网络子系统开发者自己的会，质量最高；先看下表中的单个演讲，再按主题在频道里挑。

| 视频 | 阶段 | 说明 |
| --- | --- | --- |
| [Netdev Conference 频道](https://www.youtube.com/c/netdevconf) | 全部 | 每届按天分播放列表；各届 [会议网站](https://netdevconf.info/0x16/news/videos-are-released.html) 的 session 页同时提供幻灯片和论文，比直接看视频更好检索 |
| [Improving Route Scalability with Nexthop Objects](https://lpc.events/event/4/contributions/434/)（David Ahern，LPC 2019） | 2 | 内核 nexthop 对象的设计动机：路由与下一跳解耦，ECMP 路由插入开销与路径数无关；直接对应 SONiC 的 NHG 和 BGP PIC |
| [XDP - challenges and future work](https://people.netfilter.org/hawk/presentations/LinuxPlumbers2018/presentation-lpc2018-xdp-future.pdf)（Jesper Brouer、Toke Høiland-Jørgensen，LPC 2018） | 3 | XDP 论文作者的演讲，含 AF\_XDP 与 DPDK 的性能对比；链接为幻灯片，视频在 LPC 2018 网络分会场 |
| Netdev 每届的 XDP Workshop | 3 | 年度 XDP 进展汇总，看最近一两届即可 |
| Linux Plumbers Conference 频道的 Networking Track | 2、3、5 | 内核网络与 switchdev、硬件卸载相关的设计讨论 |
| Brendan Gregg：Linux Performance Tools 系列演讲 | 1 | 网络部分对应收发包路径上每一层的观测工具 |

## 来源

本清单中的链接均为检索时实际打开或确认存在的页面；未附链接的条目（How SKBs work、Cilium 参考指南、FRR EVPN 文档、Arthur Chiao 博客）请按括号内的站点名查找。