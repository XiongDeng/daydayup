dockerd is a higher-level daemon built on top of containerd. dockerd handles the Docker-specific developer experience, and delegates the actual work of running containers to containerd.

**The stack**

```
docker CLI
   │  REST API over /var/run/docker.sock
dockerd            ← Docker Engine: API, builds, networking, volumes
   │  gRPC
containerd         ← container lifecycle, image pull/storage, snapshots
   │  ttrpc
containerd-shim-runc-v2   (one per container)
   │  fork/exec
runc               ← OCI runtime: creates namespaces/cgroups, execs your process, then exits
```

**Who does what**

- **dockerd** owns everything that makes Docker feel like Docker: the Docker API the CLI and Compose talk to, image builds (via BuildKit), networking (bridge networks, port publishing, iptables/nftables rules, embedded DNS), named volumes, logging drivers, Swarm, and authentication to registries.
- **containerd** is a general-purpose container runtime daemon: pulling and unpacking images, managing content and snapshots (overlayfs layers), creating and supervising containers and tasks, and exposing events. It knows nothing about Docker networks, Compose, or Dockerfiles.
- **containerd-shim** sits between containerd and each container. It holds the container's stdio and reports its exit status, so containerd (and dockerd) can restart or upgrade without killing running containers. That's what makes dockerd's `live-restore` possible.
- **runc** is the low-level OCI runtime. It reads an OCI bundle (rootfs + `config.json`), makes the clone/unshare/pivot_root/cgroup calls, execs the entrypoint, and exits. The shim becomes the container's parent.

**How it got this way**

Early Docker was one monolithic daemon. In 2016 (Docker 1.11) it was split: the low-level runtime became runc (donated to the OCI), and lifecycle management became containerd, which was later donated to the CNCF. dockerd kept the user-facing features and became a client of containerd.

**Why it matters in practice**

- **Kubernetes doesn't need dockerd.** The kubelet talks to containerd directly through containerd's CRI plugin. Kubernetes removed dockershim in 1.24, so most clusters run containerd (or CRI-O) with no Docker daemon at all. Images built with Docker still run fine, since they're standard OCI images.
- **Separate namespaces inside containerd.** containerd has its own namespacing (unrelated to Linux namespaces). Docker's containers live in the `moby` namespace, and Kubernetes' in `k8s.io`. So on a node running both, `docker ps` won't show Kubernetes pods, and vice versa.
- **Other clients.** `ctr` is containerd's bare debugging CLI, `nerdctl` is a Docker-compatible CLI that talks to containerd directly, and `crictl` talks to the CRI interface Kubernetes uses.
- **Image storage is converging.** Historically dockerd kept its own image store (graph drivers like overlay2) separate from containerd's. Newer Docker releases use containerd's image store (snapshotters) instead, which is what enables things like multi-platform images in the local store. Check your Docker version's docs for whether it's the default on your install.

In short, containerd is the runtime engine, dockerd is a product built on top of it, and runc does the kernel-level work for both.