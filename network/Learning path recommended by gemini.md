第1阶段：宏观路径
《深入理解Linux网络》 + PackageCloud 调优博客
  -> 建立数据包在 Ring Buffer、NAPI、sk_buff、Socket Buffer 间流动的完整物理概念。

第2阶段：骨干架构与协议栈
《深入理解LINUX网络内幕》 + 结合 Linux 源码 (net/core/, net/ipv4/)
  -> 攻克 dev.c、sock.c、route.c 及 TCP 状态机迁移。

第3阶段：现代高性能网络与数据面
Netdev Conf 论文 + XDP/eBPF 实践 + SmartNIC/Offload
  -> 掌握 Bypass/In-kernel 数据路径优化与云原生高性能网络方案。


### 二、 核心在线资源与技术网站

- **Linux Kernel 官方网络文档 (`Documentation/networking/`)**
    
    - 源码树中最权威的最新参考，重点必读：
        
        - `scaling.rst`：RSS、RPS、RFS、XPS 等多队列/多核扩展机制。
            
        - `napi.rst`：轮询中断混合处理规范。
            
        - `netdevices.rst`：网络设备驱动生命周期与接口规范。
            
- **Netdev Conf (netdevconf.info)**
    
    - Linux 官方网络子系统的年度核心开发者大会。
        
    - 包含所有议题的技术论文、Slide 和录播，涵盖 TC（Traffic Control）、XDP、RDMA、RoCE、Netfilter 以及最新的网络硬件卸载（SmartNIC / Offloading）。
        
- **LWN.net (Linux Weekly News) - Networking 专栏**
    
    - 追踪内核网络子系统重大 Patchset、重构讨论及设计决策演进的首选站点。
        
- **PackageCloud Blog: Monitoring and Tuning the Linux Networking Stack**
    
    - 业界公认神作博客（分 Receiving Data 和 Sending Data 两个长篇），按步骤图解数据包从物理介质到用户态内存的每一个内核环节及关键监控指标。