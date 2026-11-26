# nftables Design Summary

*2026-11-25*

A summary of how Linux nftables is designed, written from the upstream kernel (net/netfilter/nf_tables_*) and userspace (nft, libnftables, libnftnl) implementation. No specific source document was supplied, so this describes the upstream design in general rather than one kernel release.

## 1. Purpose and goals

nftables replaces iptables, ip6tables, arptables and ebtables with one packet classification framework built on the existing Netfilter hooks. Its main goals are:

- **One framework for all families:** a single kernel subsystem and a single tool (nft) instead of four separate codebases.
- **Generic kernel, smart userspace:** the kernel offers a small set of generic building blocks; protocol knowledge and syntax live in userspace, so new matches rarely need kernel changes.
- **Atomic, incremental updates:** rulesets change through transactions, without dumping and reloading the whole table as iptables-restore does.
- **Scalable matching:** native sets and maps let one rule replace hundreds of linear rules.

## 2. Layered architecture

| Layer | Component | Role |
|---|---|---|
| CLI | nft | Parses the nft language, prints rulesets, runs scripts |
| Library | libnftables | Parser, evaluator, linearizer that turns rules into expressions; JSON API |
| Netlink helpers | libnftnl, libmnl | Build and parse nf_tables netlink messages |
| Transport | nfnetlink (NFNL_SUBSYS_NFTABLES) | Carries batched requests, dumps and event notifications |
| Kernel core | nf_tables_api.c | Object lifecycle, transactions, notifications |
| Kernel datapath | nf_tables_core.c + expression modules | Evaluates rules per packet at Netfilter hooks |

The userspace compiler does the heavy lifting: it resolves names, checks types, and "linearizes" a high level statement such as `tcp dport { 22, 80 } accept` into a sequence of primitive kernel expressions (payload load, set lookup, verdict).

## 3. Object model

- **Tables:** top level namespaces, each bound to an address family: ip, ip6, inet (dual stack), arp, bridge or netdev. Tables hold everything else and can be owned by a process or marked dormant.
- **Chains:** a base chain attaches to a Netfilter hook (prerouting, input, forward, output, postrouting, ingress, egress) with a type (filter, nat, route), a numeric priority and a default policy. Regular chains have no hook and are reached by jump or goto.
- **Rules:** an ordered list of expressions inside a chain, each with a kernel assigned handle used for positional inserts and deletes.
- **Sets and maps:** typed collections, either anonymous (owned by one rule) or named (shared). Maps bind keys to data; verdict maps bind keys to verdicts such as `jump web_chain`.
- **Stateful objects:** named counters, quotas, limits, ct helpers, ct timeouts, ct expectations, secmarks and synproxy settings that several rules can reference.
- **Flowtables:** software or hardware fast path entries for established flows on a set of devices.

## 4. The rule virtual machine

Each rule is a small program for a register based virtual machine.

- **Registers:** a verdict register plus general data registers, addressable as 16 byte or 4 byte slots, so expressions can pass values to each other.
- **Expressions:** small kernel modules with eval, init, destroy and dump callbacks. Core ones include payload (load packet bytes), meta (interface, mark, length), ct (conntrack state), cmp, bitwise, byteorder, range, lookup (set membership), immediate (load a constant or verdict), counter, limit, nat, masq, redir, log, reject, fwd, dup and queue.
- **Evaluation:** expressions run in order; a failing match breaks out of the rule; a verdict expression sets the outcome. Hot expressions (cmp, payload, meta, bitwise, lookup) are called directly instead of through indirect calls to reduce retpoline cost.
- **Verdicts:** accept, drop, queue, continue, return, jump (push a return point) and goto (no return). Jump depth is bounded to stop loops, and the ruleset is checked for loops when it is committed.
- **Blobs:** at commit time a chain's rules are packed into a compact, read only blob that the datapath walks under RCU.

## 5. Sets

Sets are central to nftables performance. The kernel picks a backend from the set's flags and the size and policy hints userspace sends:

- **hash / rhash:** exact match on fixed keys, with growable resizing for dynamic sets.
- **rbtree:** interval matching for ranges and prefixes.
- **bitmap:** small keys up to 16 bits, such as ports.
- **pipapo:** concatenated keys with ranges, for example address plus port range, with AVX2 accelerated lookups where available.

Sets support element timeouts, per element counters and comments, and dynamic updates from the datapath (`add @set` and `update @set`) for features such as rate limiting by source address.

## 6. Transactions and atomic commit

All changes arrive as one netlink batch bounded by NFNL_MSG_BATCH_BEGIN and NFNL_MSG_BATCH_END.

1. **Prepare:** each message is validated and queued as a transaction entry, and new objects are created but marked inactive in the next generation.
2. **Generation mask:** every object carries a genmask with two bits, current and next. Readers see only objects active in the current generation, so the old ruleset keeps working while the new one is staged.
3. **Commit:** if the whole batch validated, the generation counter flips and the new ruleset becomes visible at once. Deleted objects are released after an RCU grace period.
4. **Abort:** if any message fails, every queued change is undone and the running ruleset is untouched.

A per network namespace commit mutex serializes writers. Each commit bumps a generation id that userspace uses to detect concurrent changes, and the kernel sends notifications that `nft monitor` can follow.

## 7. Packet path

1. A packet reaches a Netfilter hook where one or more base chains are registered, ordered by priority.
2. nft_do_chain walks the chain's rule blob, evaluating expressions with the packet and per packet registers.
3. Jumps push onto a small call stack; return or the end of a chain pops back. Reaching the end of a base chain applies its policy.
4. The final verdict goes back to Netfilter, which accepts, drops or queues the packet. An accept in one base chain does not stop later base chains at the same hook, but a drop is final.

NAT chains work with conntrack so that only the first packet of a flow runs through NAT rules. Flowtables let established flows skip the classic forwarding path, and with hardware offload skip the CPU entirely.

## 8. Compatibility and tooling

- **iptables-nft:** the iptables command line backed by nf_tables. It uses native expressions where it can and the nft_compat module to run legacy xtables matches and targets where it can't.
- **Translation:** iptables-translate and ip6tables-translate convert legacy rules into nft syntax.
- **Debugging:** `meta nftrace set 1` with `nft monitor trace` shows the rule by rule path of a packet. `nft -a list ruleset` prints rule handles.
- **Automation:** libnftables offers a JSON interface for programs such as firewalld, which uses nftables as its default backend.

## 9. Key design decisions and trade-offs

| Decision | Benefit | Cost |
|---|---|---|
| Generic expression VM instead of per match modules | New matches mostly need only userspace changes | Harder to read raw kernel dumps; the compiler is complex |
| No built in tables or chains | Nothing is registered at hooks unless used, so empty hooks cost nothing | Users must create their own skeleton |
| Native sets and maps | Lookups stay fast as rules scale | Several set backends to maintain |
| Generation mask transactions | Atomic swaps with no packet loss window | Extra memory while two generations coexist |
| Single inet family | One ruleset for IPv4 and IPv6 | NAT and some matches still need care across families |

## 10. Where to read the code

- **Kernel:** net/netfilter/nf_tables_api.c (control plane and transactions), nf_tables_core.c (datapath), nft_*.c (expressions), nft_set_*.c (set backends), nf_flow_table_*.c (flowtables), include/net/netfilter/nf_tables.h (core structures).
- **Userspace:** the nftables repository (src/parser_bison.y, evaluate.c, netlink_linearize.c, netlink_delinearize.c) and libnftnl.
