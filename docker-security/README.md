# Docker security: Day 1 notes

Lab environment: Kali Linux VM (VMware, NAT network), Docker 28.5.2, kernel 6.19.14.

## Main idea
A container is not a small virtual machine. It is a normal Linux process that shares the host's kernel. Namespaces make it feel like it has its own environment, and cgroups limit how many resources it can use.

## Namespaces (what a container can see)
Namespace types: pid, net, uts, time, user, mnt, ipc, cgroup.

| Namespace | What it isolates | What I saw |
|---|---|---|
| uts | Hostname | The container's hostname was a random ID (26a17ee88138), not `kali` |
| pid | Process list | The container showed only 2 processes starting at PID 1. Kali had about 230 |
| mnt | Filesystem view | `ls /` inside the container showed Alpine's own filesystem |
| net, ipc, cgroup | Network, shared memory, cgroup view | Different from Kali |
| user, time | User IDs, clocks | **Same as Kali** |

I compared `/proc/<PID>/ns` for the nginx container (PID 20760 on Kali) with `/proc/self/ns` on Kali. Different numbers in brackets mean different namespaces.

## Security point: shared user namespace
The user namespace was the same for the container and Kali. By default, root inside a container is also root on the host. If an attacker escapes the container, they arrive as root. That is why extra protections are needed: AppArmor, seccomp, dropped capabilities, and eBPF-based policies.

## Cgroups (what a container can use)
Cgroups limit and manage resources such as memory and CPU.
- Without `-m`, the container's memory limit was the whole VM's RAM (3.817 GiB).
- With `-m 100m`, `docker stats` showed a 100 MiB limit (104857600 bytes).
- With `-m 20m` and `tail /dev/zero`, the process was killed and the exit code was **137**. The 137 code means the process was killed by the out-of-memory killer. Kali was not affected.
- `docker stats` shows only running containers.

## AppArmor
`docker inspect -f '{{.AppArmorProfile}}' web` printed `docker-default`. Docker applies this AppArmor profile to containers by default. `aa-status` showed it loaded in enforce mode.

## Kernel check
Security modules on my kernel: lockdown, capability, landlock, yama, apparmor, tomoyo, bpf, ipe, ima, evm. AppArmor, Landlock and BPF are all available for the next labs.

## Commands I used
| Command | Purpose |
|---|---|
| `docker run -it --rm alpine sh` | Start a disposable container |
| `docker inspect -f '{{.State.Pid}}' web` | Get the container's PID on the host |
| `sudo ls -l /proc/<PID>/ns` | List a process's namespaces |
| `docker run -m 100m ...` | Set a memory limit (cgroup) |
| `docker stats --no-stream` | Show resource use of running containers |

