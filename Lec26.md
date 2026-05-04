# CS 537 Midterm 3: Lecture 26 (Virtualization) Notes

Exam: **Mon May 4, 12:25-2:25 PM**. Virtualization is the last topic on the exam (slide 3 confirms scope: I/O through "today" plus P5/P6/P7).

## Why this lecture matters

Across the three sample midterms, only **S23 Q58-Q61** tested virtualization (Swift section). The Lecture 26 in-class quiz on slides 41-43 is essentially the same four questions. **Treat slide 43 as direct exam predictions.**

| S23 Q | Topic | Answer |
|---|---|---|
| 58 | Workload with smallest slowdown in a VM? | Sequential array scan |
| 59 | Guest user setting a timer = how many VMM executions? | 2 typical |
| 60 | TLB caches what mapping inside a VM? | gVA to hPA |
| 61 | If not all privileged instructions trap in user mode, best approach? | Direct-execute user code, emulate in privileged mode |

---

## Tier 1 facts (memorize cold)

### Two approaches to virtualization (slides 8-9)

**Emulation** (slide 8): software decodes every instruction. Cross-architecture (ARM on x86 via QEMU). About **10x slowdown**.

**Limited Direct Execution** (slide 9): run guest instructions on the real CPU. Only intercept privileged operations. This is what real VMMs do.

### What needs special handling (slide 10)

Privileged code only. User-mode code already runs in a virtual address space and can't touch devices, so it runs at native speed.

Privileged things: page tables, interrupt vectors, device drivers, timers.

### Trap-and-emulate (slides 15-18)

- Guest OS runs in **user mode**, not kernel mode.
- Guest OS *thinks* it is privileged.
- Privileged instruction in guest → traps to VMM → VMM emulates → returns to guest.
- VMM is the only thing in real kernel mode.

### Syscall timeline in a VM (slide 29)

Critical sequence for any guest syscall:

1. Process executes a syscall trap.
2. Trap goes to the **VMM**, not the guest kernel (because guest kernel is in user mode).
3. VMM invokes guest OS trap handler at user-mode privilege.
4. Guest kernel runs the syscall body.
5. Guest kernel issues `return-from-trap` (a privileged instruction).
6. That return-from-trap **also traps to VMM**.
7. VMM does the real return-from-trap to the user process.

So a single guest syscall causes **2 VMM executions minimum** (entry trap + the return-from-trap). If the guest kernel does extra privileged ops in between (e.g., `lidt` to set a timer), each adds another VMM execution.

This is why a webserver (lots of syscalls) suffers way more than matrix multiply (mostly user-mode arithmetic).

### lidt example (slides 19-27)

- Real OS uses `lidt` to set the interrupt descriptor table base.
- Guest in user mode: `lidt` traps to VMM.
- VMM stores the guest's intended IDT address (this is shadow state) but installs its own IDT at the hardware level.
- When a hardware timer fires, control goes to VMM's IDT entry. VMM decides whether to switch VMs (slide 28: VMM tick handler) or invoke the guest's tick handler.

Two timer handlers exist: VMM's (decides which OS to run) and the guest OS's (decides which process to run).

### Memory virtualization: two translations (slides 32-33)

- gVA → gPA: the guest page table, owned by the guest OS.
- gPA → hPA: the host (or nested) page table, owned by the VMM.

Native x86-64: 4 memory accesses for translation (slide 34).

### Shadow paging (Approach 1, slides 35-36)

- VMM combines both translations into a single shadow page table mapping gVA → hPA.
- Hardware uses the shadow table directly: **at most 4 memory accesses** per translation, same as native.
- Guest page table is marked **read-only**. Any guest write traps to VMM, which then updates the shadow.
- Cost: page table updates are slow. Translations are fast.

### Nested paging (Approach 2, slide 37)

- Hardware walks both tables. New registers: gCR3 (guest), ncr3 (nested/host).
- Each gPA encountered during the guest walk has to itself be translated to hPA. This produces a 2D walk.
- **At most 24 memory accesses** per translation: 5 + 5 + 5 + 5 + 4.
- Cost: slow translations on TLB miss. Page table updates are **in-place and fast** (no traps).

### Shadow vs nested tradeoff (slide 38)

| | Shadow paging | Nested paging |
|---|---|---|
| Translation on TLB miss | 4 accesses (fast) | 24 accesses (slow) |
| Page table update | Trap to VMM (slow) | In-place (fast) |
| Best for | Stable address spaces, lots of translations | Frequent page table churn (fork/exec, mmap) |

### TLB contents in a VM (slide 43, S23 Q60)

The TLB caches the **final** mapping that hardware uses: **gVA → hPA**. It does not cache gVA → gPA, regardless of whether you use shadow or nested paging. Under nested paging, the TLB still memoizes the full gVA → hPA result so a TLB hit avoids the 24-access walk.

### When trap-and-emulate breaks (slide 43, S23 Q61)

Original x86 had ~17 privileged instructions that, executed in user mode, **silently did the wrong thing** instead of trapping (e.g., `popf` would mask the interrupt-flag write rather than trap). With no trap, there's nothing for the VMM to intercept.

Best response: **direct execution of guest user-mode code, but emulate (or binary-translate) all instructions when running in privileged mode**. This was VMware's pre-VT-x approach.

### Workload performance (slides 31, 41-43)

- **Sequential array scan**: about the same as native. Cache-friendly, predictable, hits TLB, stays in user mode.
- **Random hash table or tree**: a little slower. TLB misses, and each miss is now a 24-access walk under nested paging.
- **Webserver / syscall-heavy**: much slower. Every syscall = 2+ VMM entries.

### Device virtualization (slides 39-40)

- **Emulation**: VMM intercepts device-register reads/writes and emulates the device. Slow, transparent.
- **Replacement / paravirtualization**: write a guest driver that calls into VMM directly (e.g., virtio, VMware tools). Much faster, requires guest cooperation.

### VMM architectures (slides 45-47)

- **Hypervisor (Type 1)**: VMM on bare metal. Implements its own scheduler, memory mgmt, drivers. Examples: VMware ESX, Xen.
- **Hosted (Type 2)**: VMM runs as a process inside a host OS, uses host services. Examples: VMware Workstation, KVM, Hyper-V, VirtualBox, QEMU.

### Uses (slides 48-53)

- **Suspend / resume**: save all VM state to disk. Like hibernate but for arbitrary guests.
- **Migration**: iteratively copy memory while VM runs. Mark RO, copy modified pages, repeat, brief pause at end. Disk on a network FS so it doesn't have to be copied.
- **Clone**: multiple copies (honeypots, parallel test).
- **Time travel**: revert to an earlier saved state.
- **Introspection**: VMM looks at guest memory directly to detect rootkits that hide from `ps` or `/proc`.
- **Subversion**: rootkit installed in the hypervisor can hide from the guest entirely (e.g., boot-sector hypervisor that loads the OS).

### Containers vs VMs (slide 54)

Containers virtualize **user mode only** (namespaces + cgroups). One shared kernel. VMs virtualize the whole machine including kernel mode.

---

## Predicted T/F (high confidence)

1. Under nested paging, the TLB stores gVA-to-hPA translations, not gVA-to-gPA. **True.**
2. Trap-and-emulate requires every privileged instruction to trap when executed in user mode. **True.**
3. The guest OS in a trap-and-emulate VMM runs in kernel mode. **False** (user mode).
4. A pure software emulator like QEMU runs about 10x slower than native. **True.**
5. With shadow paging, a guest write to its page table updates that table in place without involving the VMM. **False** (it traps; guest PT is read-only).
6. Under nested paging, a single TLB miss on a 4-level page table can require up to 24 memory accesses. **True.**
7. A guest user-mode system call causes exactly one entry into the VMM. **False** (typically 2: entry trap and the return-from-trap).
8. Sequential array scanning suffers more slowdown in a VM than random pointer chasing in a tree. **False** (sequential suffers least).
9. Containers virtualize the kernel separately for each container. **False** (user mode only; one shared kernel).
10. The VMM and the guest OS each have their own timer interrupt handler. **True.**
11. A hosted VMM runs inside a host OS and uses the host's drivers. **True.**
12. Live migration requires stopping the VM for the entire duration of the memory copy. **False** (iterative copy keeps VM running, brief pause at the end).
13. Page table updates are faster under nested paging than under shadow paging. **True.**
14. Address translation on a TLB miss is faster under shadow paging than under nested paging. **True.**
15. A webserver in a VM slows down more than a CPU-bound numerical workload. **True.**
16. Under nested paging, the VMM still has to trap on every guest page-table update. **False** (that's the shadow-paging cost; nested updates are in-place).
17. The TLB is disabled inside a virtual machine. **False.**

## Predicted MC

**Q1.** Which workload sees the smallest slowdown when run in a VM?
- A. Webserver handling many short HTTP requests
- B. Database doing random pointer chasing through a B-tree
- C. Sequential scan over a large array doing arithmetic
- D. Compiler doing many fork/exec calls

**Answer: C.** User-mode, cache-friendly, predictable TLB.

**Q2.** Under nested paging, the worst-case number of memory accesses for one address translation is approximately:
- A. 4   B. 8   C. 16   D. 24

**Answer: D.** 5 + 5 + 5 + 5 + 4 = 24.

**Q3.** When a guest user process makes a syscall under simple trap-and-emulate, control transfers to the VMM how many times?
- A. 0   B. 1   C. 2   D. 4

**Answer: C.** Once on the syscall trap, once on the guest's return-from-trap.

**Q4.** Some old privileged instructions silently fail in user mode instead of trapping. Which approach handles this most efficiently?
- A. Pure trap-and-emulate
- B. Pure software emulation of all guest code
- C. Direct execution of user-mode code; emulate (or binary-translate) all instructions when in privileged mode
- D. Run the guest in real kernel mode

**Answer: C.** Slide 43 verbatim.

**Q5.** A VMM that runs as a regular process inside Linux and uses Linux drivers is best described as:
- A. A hypervisor (Type 1)
- B. A hosted VMM (Type 2)
- C. A container runtime
- D. A binary translator

**Answer: B.**

**Q6.** Inside a virtualized x86 system, what does the TLB cache?
- A. gVA to gPA mappings only
- B. gPA to hPA mappings only
- C. gVA to hPA mappings (the final translation)
- D. Both halves separately in two TLBs

**Answer: C.**

**Q7.** With shadow paging, the guest page table is:
- A. Marked read-only; writes trap to the VMM
- B. Read-write, used directly by hardware
- C. Disabled; the host page table is used instead
- D. Cached only in the TLB

**Answer: A.**

**Q8.** Why is live VM migration possible without significant downtime?
- A. Memory is copied iteratively while the VM runs; a short pause copies remaining dirty pages
- B. Memory does not need to be copied at all
- C. The VM is fully stopped for the entire transfer
- D. Only the registers are migrated, memory stays on the source

**Answer: A.**

## Flashcards

| Front | Back |
|---|---|
| Trap-and-emulate: guest kernel runs in what mode? | User mode |
| Why does a guest syscall produce 2 VMM entries? | Trap on syscall + trap on the privileged return-from-trap |
| Shadow paging max accesses per translation | 4 |
| Nested paging max accesses per translation | 24 (5+5+5+5+4) |
| Shadow paging weakness | PT updates trap to VMM |
| Nested paging weakness | Slow translations on TLB miss |
| TLB caches what mapping in a VM? | gVA to hPA |
| Pure emulator slowdown vs native | About 10x |
| Best workload for VM | Sequential, user-mode, CPU-bound |
| Worst workload for VM | Syscall-heavy or random pointer chasing |
| Hypervisor vs hosted | Bare metal (ESX, Xen) vs in-OS (KVM, VMware Workstation, VirtualBox) |
| Containers virtualize what? | User mode only (namespaces + cgroups) |
| When does trap-and-emulate fail? | When privileged instructions silently no-op in user mode |
| Fix when trap-and-emulate fails | Direct-execute user code, binary-translate / emulate in privileged mode |
| How does live migration avoid downtime? | Iteratively copy memory while VM runs, short pause at end |
| Guest page table under shadow paging | Read-only, writes trap |
| Two timer handlers in a VM | VMM's (chooses VM) and guest OS's (chooses process) |

## If you only have 5 minutes before the exam

1. **Trap-and-emulate**: guest OS in user mode, privileged ops trap to VMM.
2. **Syscall = 2 VMM entries** (entry + return-from-trap), more if guest does extra privileged ops.
3. **TLB caches gVA to hPA**, not the intermediate.
4. **Shadow paging**: 4 accesses on translation, slow PT updates. **Nested paging**: 24 accesses, fast PT updates.
5. **Best workload**: sequential array scan (same as native). **Worst**: syscall-heavy.
6. **If privileged instructions silently no-op in user mode**: direct-execute user code, emulate in privileged mode (binary translation).
7. **Hypervisor (Type 1)** on bare metal vs **Hosted (Type 2)** inside an OS.
8. **Containers** virtualize user mode only (namespaces, cgroups), one shared kernel.

Good luck.
