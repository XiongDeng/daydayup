The image isn't really running natively on all three. It's a Linux image, and on Mac and Windows it runs inside a hidden Linux VM.

**Why it needs Linux**

A container shares the host's kernel; there's no kernel in the image. An image is just a stack of filesystem layers (tarballs) plus a config JSON (entrypoint, env, user, and so on). The binaries in those layers are Linux ELF executables that make Linux syscalls and rely on namespaces, cgroups, and overlayfs. So a Linux image needs a Linux kernel underneath it, full stop.

**What happens on each host**

- **Linux:** Native. dockerd/containerd/runc talk to the host kernel directly. The container is just a process tree with namespaces and cgroups, no virtualization involved.
- **macOS:** Docker Desktop (or Colima, OrbStack, Podman machine) boots a small Linux VM using Apple's Virtualization.framework (older versions used HyperKit). The Docker daemon runs inside that VM, and the `docker` CLI on your Mac talks to it over a socket. Bind mounts are shared into the VM through a file-sharing layer (virtiofs), which is why heavy file I/O on mounted volumes is slower on Mac. Published ports are forwarded from the VM back to localhost.
- **Windows:** Docker Desktop runs the Linux daemon inside WSL2, which is a real Linux kernel in a lightweight Hyper-V utility VM (older versions used a dedicated Hyper-V VM). Same architecture as Mac: CLI on the host, daemon and containers in the VM.

The VM is so well hidden that it feels native, but `docker run` on a Mac is really "run this in a Linux VM."

**CPU architecture is a separate question**

OS compatibility is handled by the VM. CPU compatibility is handled differently:

- **Multi-arch images.** A tag like `nginx:latest` usually points to a manifest list (an OCI image index) with one image per platform: `linux/amd64`, `linux/arm64`, and so on. When you pull, the client picks the variant matching your machine. That's how the same tag works on an Intel Linux server and an Apple Silicon Mac, which get different binaries. You build these with `docker buildx build --platform linux/amd64,linux/arm64`.
- **Emulation.** If the image only exists for amd64 and you're on an arm64 Mac, Docker Desktop runs it under emulation (Rosetta for x86-64 Linux binaries in the VM, or QEMU via binfmt_misc). It works, but it's slower, and some things (certain JITs, specific CPU instructions) can break.

**Windows containers are a different thing**

There are also native Windows container images, built on Windows Server Core or Nano Server. They need a Windows kernel and run only on Windows hosts, using process isolation or Hyper-V isolation. Docker Desktop on Windows switches between "Linux containers" and "Windows containers" modes. A Windows image won't run on Linux or Mac at all, since there's no lightweight Windows VM path for that.

**In short:** the image format is portable, but the binaries inside it aren't. "Runs everywhere" works because Docker supplies a Linux kernel via a VM where the host doesn't have one, and supplies the right CPU architecture via multi-arch manifests or emulation.