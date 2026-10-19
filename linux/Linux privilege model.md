1. Root vs. capabilities

Historically Unix had two states: uid 0 could do anything, everyone else was restricted. Since Linux 2.2, root's powers are split into about 40 capabilities (man 7 capabilities). Examples:

┌──────────────────────┬───────────────────────────────────────────────────────────┐
│      Capability      │                          Allows                           │
├──────────────────────┼───────────────────────────────────────────────────────────┤
│ CAP_NET_BIND_SERVICE │ Bind to ports below 1024 (FRR's ZCAP_BIND)                │
├──────────────────────┼───────────────────────────────────────────────────────────┤
│ CAP_NET_RAW          │ Raw/packet sockets                                        │
├──────────────────────┼───────────────────────────────────────────────────────────┤
│ CAP_NET_ADMIN        │ Interfaces, routes, SO_BINDTODEVICE, MD5 keys and similar │
├──────────────────────┼───────────────────────────────────────────────────────────┤
│ CAP_SYS_ADMIN        │ A very broad set, including setns()                       │
└──────────────────────┴───────────────────────────────────────────────────────────┘

CAP_SYS_ADMIN is nearly as strong as root, so granting it weakens the isolation considerably.

2. Capability sets (per thread)

- Permitted: the upper bound of what the thread may have.
- Effective: the capabilities the kernel actually checks on each syscall.
- Inheritable / Ambient: what is passed across execve.
- Bounding: a hard limit for the process tree.

A process can move a capability between permitted and effective freely. That is exactly what frr_with_privs does: the capability stays in permitted all the time, and the block adds it to effective and then removes it. Dropping from permitted is permanent.

3. Basic security requirements

4. Least privilege: hold only the capabilities you need.
5. Minimize the window: hold them only while doing the privileged operation.
6. Don't run as root: run as a dedicated unprivileged user, with no shell and no home directory.
7. Fail safe: privileges must drop on every code path, including errors.
8. Don't let untrusted input reach privileged code: parsing a BGP UPDATE should not happen with capabilities in the effective set.

9. Best practices

Process level
- Start with the needed privileges, then drop to an unprivileged user and keep only the required capabilities (FRR's zprivs_init).
- Use setresuid/setresgid and check the return values. Drop supplementary groups with setgroups.
- Set PR_SET_NO_NEW_PRIVS so execve can't regain privileges through setuid binaries.
- Lock down the bounding set so children can't acquire more.

Code level
- Use RAII/scope-based patterns (FRR's cleanup-attribute macro), not manual raise/lower.
- Raise privileges around the single syscall, not around large functions.
- Don't run complex code (parsing, allocation-heavy work) inside a privileged block.
- Prefer a file descriptor created once with privileges over re-raising repeatedly. For example, bind the listener once at startup and handle connections without privileges.

System level (defence in depth)
- systemd hardening: User=, AmbientCapabilities=, CapabilityBoundingSet=, NoNewPrivileges=, ProtectSystem=, PrivateTmp=, RestrictAddressFamilies=, SystemCallFilter=.
- seccomp: filter syscalls so a compromised daemon can't call mount, ptrace and so on.
- MAC: SELinux or AppArmor profiles.
- Namespaces and containers: drop capabilities in the container runtime.
- File permissions: sockets, config and VTY sockets owned by the frr user and the frrvty group, mode 0660 or tighter.

5. Common pitfalls

- Forgetting to drop on an error path. This is what the cleanup macro prevents.
- Using CAP_SYS_ADMIN as a catch-all.
- Dropping the effective set but leaving the capability in the permitted or ambient set. A later bug or exploit can re-raise it.
- Treating "I'm not root" as the end of the story. A process with CAP_NET_ADMIN can reconfigure the network.
- Time-of-check/time-of-use races on files opened with elevated rights.

6. How this maps to FRR

- At startup, frr_preinit/zprivs_init sets up the frr user and the _caps_p list.
- The daemon runs unprivileged, with those capabilities in permitted only.
- Each socket, bind or setns operation uses frr_with_privs(&bgpd_privs).
- If the capability library isn't available (some non-Linux platforms), FRR falls back to uid-based raising and lowering. Non-Linux code paths work differently, but the API is the same.

If you want to see this in practice, check lib/privs.c (zprivs_init, _zprivs_raise, _zprivs_lower) and run grep Cap /proc/$(pidof bgpd)/status on a live system to see the permitted vs. effective bitmasks. capsh --decode=<hex> turns those into names.