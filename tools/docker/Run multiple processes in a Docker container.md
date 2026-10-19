Understood. I'll keep this to Docker.

**What a container is at the process level**

A container isn't a VM. It's an ordinary process tree on the host kernel, placed in its own namespaces (PID, mount, network, UTS, IPC, user) and constrained by cgroups. When you `docker run`, the runtime (containerd → containerd-shim → runc) clones one process into those namespaces and execs your `ENTRYPOINT`/`CMD`. That process becomes **PID 1** in the container's PID namespace.

Anything PID 1 forks inherits the same namespaces and cgroup, so it's automatically "inside the container." Running multiple processes is mechanically trivial: PID 1 just has to start them. The hard parts are the responsibilities PID 1 inherits.

**Why PID 1 is special**

1. **Signals.** The kernel doesn't apply default actions to PID 1 for signals it hasn't installed a handler for. If your app ignores SIGTERM, `docker stop` waits out its timeout (10s by default) and then SIGKILLs everything. PID 1 also has to forward signals to its children, or they never get a clean shutdown.
2. **Zombie reaping.** Orphaned processes are reparented to the namespace's PID 1. If PID 1 never calls `wait()`, they accumulate as zombies and eventually exhaust the PID limit.
3. **Lifetime.** The container lives exactly as long as PID 1. When PID 1 exits, the kernel SIGKILLs every other process in the namespace, whatever state they're in.

**Ways to run multiple processes**

- **Shell wrapper script.** This is the simplest approach, but you have to handle signals and exits yourself:
    
    ```bash
    #!/usr/bin/env bash
    trap 'kill -TERM $p1 $p2 2>/dev/null' TERM INT
    nginx -g 'daemon off;' & p1=$!
    my-app & p2=$!
    wait -n                      # returns when either child exits (or a signal arrives)
    kill -TERM $p1 $p2 2>/dev/null
    wait
    ```
    
    Keep each service in the foreground. Daemonizing double-forks the process out from under you.
    
- **A tiny init (tini, dumb-init, or `docker run --init`).** It runs as PID 1, forwards signals, and reaps zombies, but supervises only one child. A common pattern is `ENTRYPOINT ["tini", "--", "/start.sh"]`, which puts tini in front of the script above.
    
- **A process supervisor (supervisord, s6-overlay, runit).** It starts several services, restarts them on crash, handles ordering and dependencies, and does PID 1 duties. s6-overlay is designed specifically for containers. supervisord is Python and heavier.
    
- **systemd as PID 1.** This is possible: Podman supports it natively, and Docker needs cgroup mounts and privileges. It's heavyweight and usually only worth it when porting something that assumes systemd.
    
- **`docker exec`.** This doesn't start processes from PID 1. The runtime enters the container's namespaces (`setns`) and spawns a new process there. That process is in the container's PID namespace, but its parent is the shim on the host side. Useful for debugging; not a way to run services.
    

**The usual alternative: multiple containers sharing namespaces**

The general guidance is one concern per container. That way the orchestrator owns restarts, health checks, logs (each container's stdout/stderr), and resource limits per process. When processes need to be tightly coupled, you share namespaces instead of packing them into one container:

- `--network=container:<id>` gives a shared network stack, so processes talk over localhost.
- `--pid=container:<id>` lets them see and signal each other's processes.
- A Kubernetes pod is exactly this: a pause container holds the namespaces, and app and sidecar containers join them.

Multi-process single containers make sense when the processes are inseparable: a legacy app that expects cron plus a daemon, an nginx + PHP-FPM bundle, or a dev/test image. Otherwise, separate containers in a shared pod or network namespace give you the coupling without making PID 1 into a homemade init system.