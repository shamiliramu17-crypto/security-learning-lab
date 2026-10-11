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


---

# Day 2: Docker hardening

Goal: see what a container can do by default, then remove privileges one at a time and watch what breaks. All tests ran in the Kali VM (Docker 28.5.2).

## 1. Capabilities (what root can do)
Linux splits root's power into about 40 pieces called capabilities. Docker gives containers a reduced default set of 14 (`capsh --decode` showed `cap_chown`, `cap_dac_override`, `cap_fowner`, `cap_fsetid`, `cap_kill`, `cap_setgid`, `cap_setuid`, `cap_setpcap`, `cap_net_bind_service`, `cap_net_raw`, `cap_sys_chroot`, `cap_mknod`, `cap_audit_write`, `cap_setfcap`).

| Container | CapEff | Meaning |
|---|---|---|
| Default | `00000000a80425fb` | 14 capabilities |
| `--cap-drop ALL` | `0000000000000000` | none |
| `--privileged` | `000001ffffffffff` | every capability |

| Test | Default | `--cap-drop ALL` | `--privileged` |
|---|---|---|---|
| `chown` a file | works | Operation not permitted | n/a |
| `mount -t tmpfs` | permission denied | n/a | works |

- The user was still root (uid 0) with `--cap-drop ALL`, but root no longer had the power. The kernel checks capabilities, not the name "root".
- `--privileged` gives almost all capabilities and host access. A compromise can become a host compromise. It is the power used in container escapes, so avoid it.
- Best practice: drop all capabilities, then add back only what the app needs (for example `--cap-add NET_BIND_SERVICE`).

## 2. Read-only filesystem
- `--read-only`: writing to `/etc` failed (Read-only file system). It stops an attacker from changing files or leaving tools behind.
- `--read-only --tmpfs /tmp`: writing to `/tmp` worked. The app gets one small scratch space in RAM that disappears when the container stops.
- `tmpfs` with `noexec,nosuid`: a program copied into `/tmp` failed to run (Permission denied). An attacker can drop a tool but cannot run it.

## 3. Running as a non-root user
- Default: `uid=0(root)`. With `--user 1000:1000`: `uid=1000`, and writing to `/etc` gave Permission denied.
- On the host, `ps` showed one `sleep` owned by root and one owned by kali. The host sees the container's real UIDs because Docker does not remap user IDs (shared user namespace, seen on Day 1).
- So container root is real root on the host. If someone escapes a root container, they arrive as root. Running as non-root limits the damage.

## 4. no-new-privileges
- `NoNewPrivs` was 0 by default and 1 with `--security-opt no-new-privileges`.
- It stops a process from gaining more privileges later, for example through setuid programs.

## 5. Seccomp and layered defense
- Seccomp filters which system calls a process may make. Default container: `Seccomp: 2`, `Seccomp_filters: 1`. With `seccomp=unconfined`: `Seccomp: 0`, `Seccomp_filters: 0`.

| Test: `unshare -U echo hi` | Result | Blocked by |
|---|---|---|
| Default | Operation not permitted | Seccomp |
| Seccomp unconfined | Permission denied | AppArmor |
| Seccomp and AppArmor unconfined | `hi` | nothing |

- `unshare -U` tries to create a new user namespace. Two independent layers blocked the same action, and only removing both allowed it. This is defense in depth.
- `unconfined` is only for comparison tests, never for real containers.

## ## 6. Everything combined

```bash
docker run --rm -it \
  --read-only --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --user 1000:1000 \
  --pids-limit 100 -m 128m --cpus 0.5 \
  alpine sh
```

Results inside: `uid=1000`, `/etc` not writable, `/tmp` writable but not executable, `CapBnd` all zeros, `NoNewPrivs: 1`, `Seccomp: 2`.


## Summary of layers
| Control | Limits |
|---|---|
| `--cap-drop ALL` | what root can do |
| `--read-only` and `--tmpfs noexec` | changing files and running dropped tools |
| `--user` | running as root |
| `no-new-privileges` | gaining privileges later |
| seccomp (default) | which system calls are available |
| AppArmor (default) | which files and actions a program can use |
| `-m`, `--cpus`, `--pids-limit` | resource abuse |

## Interview answer
Containers share the host kernel, so I use several layers. I drop all capabilities, use a read-only filesystem, run as a non-root user, and set no-new-privileges. Seccomp and AppArmor add two more layers. In my lab, disabling seccomp alone did not allow a blocked action because AppArmor still blocked it. I avoid `--privileged` because it gives the container almost all capabilities and host access, so a compromise can become a host compromise.


How would I write a custom seccomp profile for one app?