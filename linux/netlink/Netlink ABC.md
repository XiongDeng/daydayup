
# Question: can we say netlink is a special socket that kernel and application use it to exchange fundamental network information. kernel and application both play as a client and also as a server.

Mostly yes, but two parts of that sentence need adjusting.

**"Fundamental network information" is too narrow.** Netlink (the `AF_NETLINK` socket family) started as the way to configure networking. `NETLINK_ROUTE` is what `ip` uses for routes, addresses, links, and neighbors. Today it's a general kernel↔user-space communication channel used for much more:

- device hotplug events (`NETLINK_KOBJECT_UEVENT`, which udev listens on)
- the audit subsystem (`NETLINK_AUDIT`)
- SELinux notifications
- Generic Netlink, which many subsystems build on, such as nl80211 for Wi-Fi, taskstats, and thermal events

User-space processes can also talk to each other over netlink, though that's rare.

**"Both client and server" is better described as "bidirectional peers."** Netlink isn't really a client/server model. It's a message-based, full-duplex channel where either side can start a conversation:

- **Request/response:** user space sends a request, such as "dump all routes" or "add this address," and the kernel replies. This is the most common pattern, and it looks client/server-like, with the kernel answering.
- **Asynchronous notifications:** the kernel sends messages on its own, usually to multicast groups that applications subscribe to, such as "a link went down" or "a USB device was plugged in." The application sent no request.

Each socket is identified by a port ID, and the kernel's is 0. Messages are datagrams and can be dropped if the receiver's buffer fills, in which case you get `ENOBUFS`.

A more accurate one-liner:

> Netlink is a socket-based IPC mechanism (`AF_NETLINK`) that lets the kernel and user-space processes exchange messages in both directions, for requests, replies, and asynchronous event notifications. It's best known for network configuration, but it's also used by many other kernel subsystems.

# Question: what the unix domain socket is?
A Unix domain socket (`AF_UNIX`, also called `AF_LOCAL`) is an IPC mechanism for two processes **on the same machine** to talk to each other. It uses the normal socket API (`socket`, `bind`, `listen`, `connect`, `send`, `recv`), but data never goes through the network stack. The kernel copies it directly from one process to the other, so it's faster than TCP over `localhost`.

**Addressing:** instead of an IP and port, a Unix socket is identified by:

- a **filesystem path**, such as `/var/run/docker.sock` or `/tmp/.X11-unix/X0`, which shows up as a special file (type `s` in `ls -l`), or
- on Linux, an **abstract name** that lives in a kernel namespace with no file on disk (written with a leading null byte, shown as `@name`), or
- no name at all, as with `socketpair()`, which creates two connected sockets, typically shared between a parent and child process.

**Socket types:**

- `SOCK_STREAM`: connection-oriented byte stream, like TCP
- `SOCK_DGRAM`: message-based, like UDP but **reliable and ordered**, since nothing is lost in the local kernel
- `SOCK_SEQPACKET`: connection-oriented, but message boundaries are preserved

**Things TCP can't do:**

- **Pass file descriptors** between processes (`SCM_RIGHTS`), so one process can hand an open file or socket to another.
- **Check the peer's identity**: the receiver can get the sender's PID, UID, and GID (`SO_PEERCRED` / `SCM_CREDENTIALS`), which is useful for access control.
- **Use file permissions** on the socket path to control who can connect.

**Common uses:** the Docker daemon, systemd and D-Bus, X11 and Wayland display servers, MySQL/PostgreSQL local connections, `ssh-agent`, and `syslog` (`/dev/log`).

**Compared with netlink:** a Unix socket is for **process ↔ process** communication on the same host, and it follows a real client/server model: one side binds and listens, the other connects. Netlink is mainly for **kernel ↔ user-space** communication.