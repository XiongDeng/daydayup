This plan covers everything besides SONiC and C++: coding practice, networking fundamentals, Linux networking, troubleshooting drills, project stories, behavioral questions, system design, and logistics. I checked your earlier project chats ("Narrowing down next career opportunity" and "Running SONiC on a laptop") so this builds on the three interview stories and the 40-problem LeetCode list you already have instead of repeating them.

**1. Coding (C first, Python second)**

Your LeetCode list was chosen with NVIDIA in mind. For this role, add practical problems that look like networking code:

- Longest-prefix match (a trie keyed on IP prefixes) and an LRU cache
- A token-bucket rate limiter and a ring buffer
- Parsing packet headers from raw bytes (bit fields, byte order)
- Producer/consumer with a mutex and condition variable
- Dijkstra's shortest path, which is the SPF calculation you already know from IS-IS
- Python: parse a log or `show` output, and diff two route tables (for example, the FRR routes against the ASIC_DB entries)

Practice without autocomplete, explaining out loud as you go. Apple phone screens usually use a plain shared editor.

**2. Networking fundamentals, updated for today's data centers**

- **ASIC forwarding pipeline:** parser, L2 lookup, L3 longest-prefix match, ACL/TCAM, ECMP, header rewrite, queuing. For each stage, know which table can run out of space and what happens when it does.
- **ECMP:** hashing, polarization (when every switch hashes the same way and traffic piles onto some paths), and resilient hashing.
- **BGP in the data center:** eBGP as the only routing protocol (RFC 7938), how AS numbers are assigned, BGP unnumbered, multipath, and add-path.
- **VXLAN/EVPN:** route types 2, 3, and 5, and symmetric IRB. Keep it at a level you can explain honestly; the L3VPN comparison is your way in.
- **MLAG:** very common in SONiC deployments. It's close to your stacking work, but different enough that you should read up on it.
- **QoS basics:** PFC, ECN, and buffer profiles. You already know these from the RoCE side.

**3. Linux networking** (a minimum qualification, and SONiC relies on the kernel for everything the ASIC doesn't handle)

- **Tools:** `ip route/rule/neigh`, VRF devices, bridges with VLAN filtering, bonding, network namespaces, `tc`, `ethtool -S`, and tcpdump filters.
- **How SONiC uses the kernel:** front-panel ports appear as kernel interfaces (Ethernet0 and so on), ARP and neighbor resolution happen in the kernel, and fpmsyncd reads routes from FRR through the FPM interface.
- **Hands-on:** build small topologies with namespaces and veth pairs, then break them on purpose.

**4. Troubleshooting drills** (likely the most heavily weighted part of the interview)

Write out 8–10 scenarios and answer each in the same order: symptom → scope → which layer to split at → the command that gives evidence → root cause → fix → prevention. Scenarios to use:

- A route is in FRR but missing from ASIC_DB
- ECMP traffic is uneven across links
- ARP resolves but traffic isn't forwarded
- LACP keeps flapping
- The BGP session is up but no routes arrive
- An MTU black hole over VXLAN
- CRC errors on a port
- Drops at an egress queue
- High CPU from traffic sent to the CPU, which you can tie to your IS-IS flooding-storm story

**5. Project deep dives**

Your three existing stories (bonding driver and NIC defect, container controller redesign, P99 latency) are ready. Add four from Huawei:

- The IS-IS flooding storm in Madrid
- The LAN switch stacking design: how split-brain was handled and how forwarding tables stayed in sync across units. This maps directly to SONiC's multi-ASIC and chassis work.
- Moving IGMP processing to the line cards
- Porting the router control plane to low-end switch hardware

These are over 15 years old, so write down the details now: numbers, design choices, what went wrong. For each story, prepare a 2-minute version and a 15-minute version. Interviewers usually ask why you made a choice and what you'd do differently.

**6. Behavioral and career story**

- **A 2-minute "walk me through your career":** Huawei → startups → Alibaba → energy storage → back to networking. Make the return to networking sound deliberate, not accidental.
- **Why Apple, why this team, and why leave Wärtsilä.**
- **Stories about:** working with vendors (your third-party control-plane vendor experience is close to working with ASIC vendors), influencing people without authority over them, a disagreement, and a failure (your remaining-jitter story works well for this).
- **Level:** with 20 years of experience, expect to be judged for a senior level, around ICT4/ICT5 on Apple's scale. Ask your classmate which level this role is meant for.

**7. Design questions** (network-software flavored)

Prepare for questions like these:

- "Detect mismatches between what the control plane intended and what the ASIC actually has programmed, across 10,000 switches"
- "Roll out configuration to a switch fleet safely, with rollback"
- "Build a telemetry pipeline for switch counters"

Your Wärtsilä experience managing large device fleets carries over well.

**8. Logistics**

- Ask your classmate about the team's focus, the manager's style, and what the interview loop looks like.
- Prepare 4–5 questions to ask the interviewers, such as SONiC version, ASIC vendors, and fabric scale.
- Do 2–3 mock interviews, one each for coding, troubleshooting, and project deep dives. I can run these with you.

**Pacing** (4–6 weeks, alongside your SONiC and C++ plans)

- **Weeks 1–2:** sections 2, 3, and 5, since the old stories take the longest to reconstruct.
- **Weeks 3–4:** sections 4 and 7, plus coding every day.
- **Final 1–2 weeks:** behavioral preparation and mock interviews.

If your referral moves quickly, do sections 4 and 5 first, because they carry the most weight in the loop.

I can also turn this into a Claude Doc so you can check items off as you go.