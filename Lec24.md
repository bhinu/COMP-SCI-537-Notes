# CS 537 Midterm 3: Lecture 24 — Containers

Companion to your existing midterm pattern summary. **This is new material.** None of your three sample midterms (SP22, F15, S23) covers containers, so this is the highest-uncertainty section on the exam.

---

## Why this is a wildcard

SP26 added this lecture, so it is genuinely new exam material. Tej Chajed's framing is "callbacks to L4 (CFS), L11 (process isolation), L22 (filesystem layering)" so expect questions that link container mechanisms back to earlier scheduler, isolation, and FS material.

Treat this as Tier 1 by default, but expect the question depth to be **conceptual rather than calculation-heavy** (no analog of RAID throughput math here).

---

## What is a container? (slides 5 to 7)

A group of processes with three properties:
1. **Private view** of global kernel resources (PIDs, mounts, network, etc.)
2. **Bounded resource usage** (CPU, memory, I/O, process count)
3. **Packaged filesystem image**

All enforced by the host kernel. **No guest OS.** This is the punchline.

**Process vs Container vs VM (slide 6):**

| Layer | Process | Container | VM |
|---|---|---|---|
| App binary | shared | isolated | isolated |
| Libraries | shared | isolated | isolated |
| OS kernel | shared | **SHARED** | isolated |
| Hardware | shared | shared | emulated |

The container vs VM distinction is the most likely conceptual question on the exam. Containers share the host kernel, which makes them fast and lightweight but means a kernel bug affects every container.

---

## Three building mechanisms (slide 9)

| Mechanism | What it gives | OSTEP parallel |
|---|---|---|
| namespaces | private view of kernel names | virtualization |
| cgroups | bounded CPU/memory/I/O/processes | isolation |
| overlayfs | filesystem from stacked image layers | persistence |

Plus chroot (slide 21) for filesystem subdirectory isolation, though chroot alone is insufficient.

---

## Namespaces (slides 10 to 15)

**Eight namespace kinds (slide 11), with what each prevents:**

| Namespace | Virtualizes | Without it, container could... |
|---|---|---|
| mount | filesystem tree | read host files |
| PID | process IDs | `kill -9` any host process |
| net | interfaces, ports | bind host's port 80 |
| UTS | hostname | rename the host |
| IPC | SysV IPC, shared memory | read neighbor's shared memory |
| user | UID/GID mappings | run as real root on host |
| cgroup | cgroup hierarchy view | edit another container's limits |
| time | clock offsets | skew neighbor's clock |

**PID namespace example (slide 12):**
- Host sees: systemd (1), sshd (812), dockerd (2104), and inside the container nginx (8237), worker (8240), worker (8241).
- Container sees: nginx (1, init), worker (2), worker (3).

The container cannot see PIDs outside its namespace. If you `kill -9` PID-1 inside, **the entire namespace dies** (because PID 1 is init for that namespace, and when init dies all children die).

**User namespace rootless trick (slide 14):** UID 0 inside maps to UID 1000 outside. The process can `apt install` inside its namespace, but cannot touch host `/etc`. This is how grading servers can give students "root" safely.

**Three syscalls for namespaces (slide 15):**
- `clone(flags | CLONE_NEWPID | ...)`: new process + new namespace(s)
- `unshare(CLONE_NEWNET)`: leave current namespace, create your own
- `setns(fd, CLONE_NEWUSER)`: join an existing namespace

**Honest caveat (slide 15):** namespaces are **not a security boundary on their own**. A kernel bug touches every container. Production systems also drop capabilities and install seccomp filters.

---

## cgroups (slides 16 to 20)

Namespaces isolate **what you see**. cgroups isolate **what you can use**.

**Controllers (slide 17):**

| Controller | Limits | Docker flag |
|---|---|---|
| cpu | CPU time (shares + quota) | `--cpus`, `--cpu-shares` |
| memory | RAM + swap | `--memory` |
| io | disk bandwidth + IOPS | `--device-read-bps`, etc. |
| pids | number of processes | `--pids-limit` |
| devices | which `/dev` nodes are usable | `--device` |

Configured via virtual filesystem: `/sys/fs/cgroup/mygroup/memory.max`, `/sys/fs/cgroup/mygroup/cgroup.procs`.

**Callback to L4 / CFS (slide 18):**
- `cpu.weight` is the per-process CFS weight from L4, applied to a whole group
- `cpu.max` is hard CFS quota. `--cpus=2` programs CFS quotas through cgroups
- Two groups with equal weight split CPU 50/50 when both busy

**OOM containment (slide 19):** when a container exceeds `memory.max`, the kernel OOM killer fires **inside the cgroup**. Host processes and other containers are untouched. Without cgroups, the OOM killer picks globally, and one bad tenant could take down anything.

**Discussion question on slide 20 (answer: c, pids):** a fork bomb spawns billions of processes. Only the **pids controller** prevents it. CPU and memory limits won't help (forking itself uses tiny resources per process). The pids controller caps the total process count.

---

## chroot and overlayfs (slides 21 to 28)

**chroot (slide 21):** confines a process to a subdirectory by changing what `/` points to. Alone it is **insufficient** because:
- Easy to escape (the original 1979 critique)
- Mount and umount syscalls leak out to the host mount table
- Need a **mount namespace** for real isolation

**Container image (slide 23):**
- A stack of layers, each is a tarball of files + metadata
- Layers are content-addressed (SHA-256). Identical bytes = same hash = shared on disk
- Every image based on `ubuntu:22.04` reuses the same base layer
- Images live in registries (Docker Hub, ghcr.io). Pulling = downloading layers you do not have

**Dockerfile to layers (slide 24):**
```
FROM ubuntu:22.04        → base layer (shared)
RUN apt-get install ...  → deps layer
COPY ./app /app          → your code layer
```
Each `RUN`/`COPY`/`ADD` becomes a new layer. Reorder them and you change which layers rebuild on edit.

**overlayfs (slide 25):**
- Multiple read-only `lowerdir` layers stacked
- One read-write `upperdir` per container
- Reads hit the first layer that has the file
- Writes use **copy-on-write**: file is copied up to upperdir, then modified there. Lower layers are never touched.

**Why this is cheap (slide 26):**
- Starting a container does not copy the image, just creates an empty upperdir
- Startup measured in milliseconds, not seconds
- 100 containers from same image ≈ 1 image on disk + 100 tiny upperdirs
- Layers shared across images too (your Python image and mine share one Python layer)

**Discussion question on slide 27 (answer: c, near zero):** second container from same `myapp:v1` uses near-zero extra disk because the lowerdirs are shared via content addressing. Only writes consume extra space.

**Callback to journaling and RAID (slide 28):** "filesystem" abstraction is composable. Journaling layers transactions on a FS. overlayfs stacks tarballs. RAID transforms disk API into a more reliable disk API. All under the same `open`/`read`/`write` API.

---

## docker run lifecycle (slide 32)

Memorize this sequence; it ties all three mechanisms together:

1. **Pull** image layers from a registry → overlayfs
2. **Stack** layers with overlayfs to form root FS → overlayfs
3. **clone(CLONE_NEW...)** for new namespaces → namespaces
4. **Create and join cgroup**, set limits → cgroups
5. **pivot_root** into the overlay mount → namespaces
6. Drop capabilities, install seccomp filter → extras
7. **execve()** the container entrypoint

---

## Beyond kernel sharing (slide 33)

When sharing the host kernel is too risky, two alternatives:
- **gVisor:** intercept and emulate syscalls in userspace
- **Firecracker:** micro-VMs with per-tenant kernel, seconds to boot. Used by AWS Lambda and Fly.io.

These exist because "containers share the kernel" is a real security limit.

---

## Quick-recall flashcards

| Front | Back |
|---|---|
| Three mechanisms behind containers | namespaces, cgroups, overlayfs (+ chroot) |
| Container vs VM in one word | containers share the kernel |
| Eight namespace kinds | mount, PID, net, UTS, IPC, user, cgroup, time |
| Three namespace syscalls | clone, unshare, setns |
| What controller stops a fork bomb | pids |
| What `cpu.weight` corresponds to | per-process CFS weight, applied to a group |
| What `cpu.max` corresponds to | hard CFS quota |
| Where do writes go in overlayfs? | upperdir (copy-on-write) |
| Why is starting a container cheap? | only creates empty upperdir; lowerdirs shared |
| Layer sharing key | content-addressed by SHA-256 |
| Are namespaces a security boundary? | No on their own; need capabilities + seccomp |
| What does pivot_root do? | switches container's root to overlay mount |
| User namespace trick | UID 0 inside maps to UID 1000 outside |
| chroot alone is insufficient because... | mount/umount leak; easy to escape |
| When kernel sharing is too risky? | gVisor (syscall intercept) or Firecracker (micro-VM) |

---

## Predicted exam questions

### True/False candidates (medium confidence, new material)

1. **T/F:** A process inside a PID namespace can see processes outside it via `/proc`. → **FALSE.** The namespace gives a private view; outside processes are invisible.
2. **T/F:** Containers running on the same host can share parts of their disk image. → **TRUE.** Content-addressed layers are deduplicated.
3. **T/F:** A process running as UID 0 inside a user namespace can modify the host's `/etc/passwd`. → **FALSE.** UID 0 inside maps to a non-privileged UID outside.
4. **T/F:** chroot alone provides sufficient filesystem isolation for multi-tenant hosts. → **FALSE.** chroot can be escaped and mount syscalls leak out.
5. **T/F:** A kernel vulnerability affects every container on the host. → **TRUE.** This is the fundamental container security limit.
6. **T/F:** Killing PID 1 inside a container's PID namespace terminates only that one process. → **FALSE.** Killing PID 1 of a namespace terminates the entire namespace.
7. **T/F:** When a container exceeds its memory cgroup limit, the host's OOM killer may pick a process outside the container. → **FALSE.** The OOM killer fires inside the offending cgroup; other processes are untouched.
8. **T/F:** overlayfs writes propagate to the lower layers. → **FALSE.** Writes go to upperdir via copy-on-write; lower layers are read-only.

### Multiple choice candidates

**Q (container mechanism mapping):** Which mechanism prevents one container from seeing another container's processes?
- (a) cgroups
- (b) PID namespace ✓
- (c) chroot
- (d) overlayfs

**Q (container resource limits):** A grading server runs untrusted student submissions. A submission runs `:(){ :|:& };:`. Which cgroup controller protects the host?
- (a) cpu
- (b) memory
- (c) pids ✓
- (d) io

**Q (overlayfs):** A container's image is 500 MB. You start 10 containers from this image. Each container writes 5 MB of data. Approximately how much disk does this consume?
- (a) 5000 MB
- (b) 550 MB ✓ (one image + 10 × 5 MB upperdirs)
- (c) 50 MB
- (d) 500 MB

**Q (container vs VM):** Which statement distinguishes containers from VMs?
- (a) Containers cannot run any program a VM can run
- (b) Containers share the host kernel; VMs do not ✓
- (c) Containers are slower to start than VMs
- (d) Containers cannot be limited in CPU usage

---

## Formula sheet additions

```
Container "math" (mostly conceptual, not numeric):
- Disk usage with K identical-image containers = image_size + K * upperdir_size
- Container startup ≈ ms (no image copy)
- VM startup ≈ s (kernel boot)
- Container shares kernel; vulnerability blast radius = host
```

---

## Where this fits in your study sequence

Slot into **Day 8 (VMs/extras)** of your existing 9-day plan: swap in containers as a new block. Practice the docker run sequence (slide 32) and the namespace-to-symptom table.

Containers will likely be 4 to 8 questions on the exam (T/F + 1 to 2 MC + maybe a discussion-style). Since you have not seen these on prior exams, **over-prepare on namespaces vs cgroups vs overlayfs distinctions**.

The single biggest "gotcha" worth flagging: **namespaces are not a security boundary** (slide 15). That's the kind of T/F where the obvious answer is wrong.
