![[Container_Namespaces.png]]

![[linux-namespaces-container.png]]

How are containers isolated?
Linux Namespaces + cgroups + Layering mechanism.
- cgroups : manages cpu, mem, block I/O for containers.
- Layering mechanism : Each containers get its own writeable layer even though it is created from same image, 
If we create 2 nginx containers from same nginx image, and change a html in single container it gets only modified in that container, so even though they are created from single image they get writable layer when run in container.

To view namespace taken by container:

```bash
lsns -p <container_pid>
```
# Linux Namespaces

## Definition
Linux namespaces are a kernel feature that provide **isolation of system resources** for a group of processes.

They make a process think it has its **own system view** while still sharing the same Linux kernel.

**Used by:** Docker, Kubernetes, Containers.

---

## Problem Solved

Without namespaces:
- All processes share the same view of the system.
- Can see the same processes, network interfaces, hostname, and mounts.

With namespaces:
- Each container gets its own isolated view of resources.
- Containers can run independently on the same host.

---

## Types of Namespaces

| Namespace | Isolates |
|------------|----------|
| PID | Process IDs and process tree |
| NET | Network interfaces, IPs, routing tables |
| MNT | Filesystem mount points |
| UTS | Hostname and domain name |
| IPC | Shared memory, message queues, semaphores |
| USER | User and Group IDs |
| CGROUP | Visibility of cgroup resource limits |

---

## Docker & Namespaces

When a container starts, Docker creates multiple namespaces:

```text
PID + NET + MNT + UTS + IPC + USER + CGROUP


![[Docker-breakdown.png]]