Oct 5, 2026 · @Michael

20 questions to check whether the Stage 1 fundamentals are solid. Answer each one aloud or in writing before reading the reference answers at the end.

## Questions

**A. Driver and interrupts**

1. Between the NIC receiving a frame and the driver getting it, how do DMA and the RX ring descriptors work together? What happens when the ring is full, and where do you see the count?
2. What problem does NAPI solve? Walk through the full sequence from the hard interrupt firing, to poll completing, to the interrupt being re-enabled.
3. How much work can net\_rx\_action do in one softirq run? Which parameters control it? What happens if packets remain when the budget runs out?
4. When does ksoftirqd take over softirq processing? How does it affect latency?
5. What do the first three columns of /proc/net/softnet\_stat mean?

**B. sk\_buff and offloads**

6. What do the head, data, tail and end pointers of an sk\_buff point to? Which pointer does each of skb\_reserve, skb\_put, skb\_push and skb\_pull move?
7. What is the difference between the linear area and frags (skb\_shared\_info)? Why are non-linear skbs needed?
8. What is the difference between skb\_clone and skb\_copy? When is each used?
9. What is the difference between GRO and LRO? Why must a forwarding box disable LRO but can keep GRO?
10. What is the difference between TSO and GSO? Where on the transmit path does GSO segment?
11. What do CHECKSUM\_UNNECESSARY and CHECKSUM\_COMPLETE on receive, and CHECKSUM\_PARTIAL on transmit, each mean?

**C. Multi-queue**

12. What problem does each of RSS, RPS, RFS and XPS solve? Which are hardware and which are software?
13. How do the RSS hash and the indirection table work together? Which commands view and change them?
14. With RSS already configured as one hardware queue per CPU, is RPS still needed?
15. How does RFS choose the target CPU? What does aRFS add on top of RFS?

**D. Transmit and queuing**

16. From the application calling send() to the NIC transmitting, which layers does the packet pass through? Where is the qdisc?
17. How do the mq root qdisc and fq\_codel relate? What is the kernel's default qdisc versus the effective default on mainstream distributions, and which sysctl controls it?
18. What problem does BQL (Byte Queue Limits) solve?

**E. Observability**

19. What does each of these show: `ethtool -S`, `ethtool -g`, `ethtool -c`, `ethtool -l`, `nstat`, `ss -tmi`?
20. Where on the path does tcpdump capture on receive and on transmit? How does that affect deciding which layer dropped a packet?