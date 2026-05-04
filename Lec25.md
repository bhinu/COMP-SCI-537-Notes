# Lecture 25: Distributed Systems + NFS (Notes mapped to exam patterns)

## Pattern map

This lecture feeds directly into:
- **Tier 1: NFS (stateless, idempotent, file handles)** — appears on SP22 and S23, very likely on yours
- **Tier 2: TCP/UDP guarantees, RPC mechanics** — high T/F probability
- **Tier 1: Crash recovery semantics** — write buffer scenario is a classic FRQ candidate

Specific recurring exam questions this lecture answers:
- "Does a file handle contain a server file descriptor?" (slide 69 quiz, S23 Q52)
- "Does TCP guarantee in-order, exactly-once?" (S23 Q45, Q46, Q53, Q54)
- "Is `mkdir` / `append` / `pwrite` idempotent?" (S23 Q42 family, slide 65)
- "Server crashes and restarts: does the client need to re-open?" (S23 Q51)
- "What gets lost when an NFS server with non-persistent write buffer crashes?" (slide 80, FRQ-shaped)

---

## 1. What is a distributed system (slides 3 to 5)

**Definition:** more than one machine cooperating to solve a problem.

Lamport's joke captures the central pain: failure modes you cannot directly observe. A machine you have never heard of can take down your program.

**Why distribute:**
1. More compute
2. More storage
3. Fault tolerance (replication)
4. Data sharing

**Why this matters for the exam:** every NFS design choice traces back to this list. The reason the API is stateless, the reason operations must be idempotent, the reason file handles exist instead of fds: all of it is because some other machine you cannot see may have crashed or dropped your message.

---

## 2. Pipes vs sockets: why the network is harder (slides 6 to 20)

| Property | Pipe (same machine) | Network socket (across machines) |
|---|---|---|
| Backpressure | Kernel blocks writer when buffer full | Router or remote kernel may silently drop |
| Loss | Impossible (in-kernel memory) | Possible (router buffer full, link error) |
| Reordering | Impossible | Possible at IP layer |
| Failure visibility | Process state visible to OS | Remote machine is a black box |

**Key exam-relevant claim (slide 20):** "From A's view, network and B are largely a black box." This is the justification for ACK + timeout + retry. You cannot inspect the receiver, so you must build reliability through protocol.

---

## 3. Reliable messaging primitives (slides 21 to 32)

Three layered techniques that turn an unreliable channel into a reliable logical connection. **Memorize the order; this is a likely T/F or MC.**

1. **ACK:** receiver confirms receipt. If sender never sees ACK, sender does not know if message arrived (slide 28 case 1 vs case 2 ambiguity).
2. **Timeout:** if no ACK within T, retry. T must be adaptive: too short causes redundant traffic and worsens overload, too long feels unresponsive.
3. **Sequence numbers + receiver memory:** receiver tracks "all messages before N seen" plus a buffer of out-of-order messages above N. Suppresses duplicates.

**TCP (slide 32) = adaptive timeout + sequence numbers + in-order buffering.** This is the mental model.

### Exam-critical ambiguity (slide 28)

Sender times out. Two indistinguishable cases:
- **Case 1:** message never reached receiver
- **Case 2:** message reached receiver, ACK was lost

Sender cannot tell. If the operation is "increment counter," retrying double-counts. **This is the entire motivation for idempotence.**

### TCP/UDP guarantees (predicted T/F)

| Claim | Truth |
|---|---|
| UDP messages arrive in order | False |
| UDP messages arrive at most once | True (no built-in retry, but still no guarantee of arrival) |
| TCP delivers in order | True |
| TCP delivers exactly once at the message layer | False at the application layer; TCP segments are deduped via seq num, but if app retries on its own (e.g., RPC retry on connection break) you can get at-least-once semantics |
| TCP is reliable because of ACKs and timeouts | True |
| Sender of TCP message always knows whether receiver got it | False (slide 28 logic still applies if connection breaks before final ACK) |

S23 Q45/46/53/54 map directly here.

---

## 4. NFS architecture (slides 33 to 46)

### Goals (slide 34)

1. **Transparent access:** POSIX semantics, client cannot tell it is over the network
2. **Fast and simple crash recovery** for both client and server
3. Reasonable performance

The first two goals constrain everything that follows. Goal 2 in particular is why the protocol is stateless.

### Mount model (slides 38 to 45)

NFS exports a remote local FS at a mount point. Client requests get translated to NFS protocol messages, then back into local FS calls on the server.

Looks like: `/dev/sda1 on /`, `/dev/sdb1 on /backups`, `NFS on /home`. The client's VFS routes calls under `/home` to the NFS layer.

---

## 5. API evolution: Strategy 1 to Strategy 4 (slides 47 to 57)

This is the most testable conceptual progression in the lecture. Each strategy gets killed by a specific failure mode. **Memorize the failure that kills each one.**

| Strategy | API | What kills it |
|---|---|---|
| **1. Wrap UNIX calls** | `open` returns server's fd; `read(fd)` sent to server | Server crash invalidates fd; on reboot, fd table is empty or reused. Client holds a stale handle. |
| **2. Stateless with paths** | `pread(path, buf, size, off)` on every call | Path lookup on every operation is too slow (full traversal each time) |
| **3. Inode-based** | `open(path)` returns inode number; subsequent calls use inode | Inode numbers are reused after deletion, so a stale inode reference can hit a freshly-created different file |
| **4. File handles** | `<volume ID, inode #, generation #>` | Generation number bumps on each inode reuse, so stale handles are detectable. **Opaque to client.** This is what NFS uses. |

### Why generation numbers matter (slide 57, hand-annotation "check generation")

When the server receives a READ request, it uses the FH to get volume + inode, reads the inode from disk or cache, **and verifies the generation number matches**. If the inode has been freed and reallocated since the FH was issued, generation mismatches and the server returns a stale handle error.

**Exam phrasing to recognize:** "client opens a file, another client unlinks it, original client tries to read." Behavior depends on whether inode has been reallocated. If not reallocated, read may succeed against the stale-but-still-valid inode. If reallocated, generation mismatch fires.

### Predicted MC: which of the following can NFS server return on a stale FH?
- Stale file handle error (correct)
- Silent wrong data (no, generation check prevents this)
- Different file's contents (no, same reason)

---

## 6. The LOOKUP / READ flow (slide 58)

This is a likely FRQ-shaped diagram question. Memorize the sequence:

**Client `open("/foo")`:**
1. Client sends `LOOKUP(rootdir_FH, "foo")`
2. Server looks up "foo" in root dir, returns foo's FH plus attributes
3. Client receives reply, allocates fd in **client-side** open file table, stores FH and current offset (= 0), returns fd to app

**Client `read(fd, buf, MAX)`:**
1. Client indexes its open file table with fd, gets FH and offset
2. Client sends `READ(FH, offset=0, count=MAX)`
3. Server uses FH to get volume + inode, **checks generation**, reads inode from disk or cache, computes block location from offset, reads data, returns it
4. Client receives reply, advances offset by bytes read, returns data to app

**Two key observations:**
- **The server stores no per-client state.** No open file table on the server. The FH plus offset in every request is sufficient.
- **The client's open file table is local-only.** The fd you get back from `open()` is a client-side abstraction. Server never sees it.

This answers slide 69's quiz: "Does a file handle contain the server file descriptor?" → **No.** And: "In NFS read() system calls are performed completely locally on the client?" → **No, they round-trip to the server**, but the *fd-to-FH translation* is local.

---

## 7. Idempotence (slides 59 to 65)

Definition (slide 62): `f()` has the same effect as `f(); f(); ...; f()`.

**Why this matters:** because of slide 28's case 1 vs case 2 ambiguity, the client may retry. If the operation is non-idempotent, retry corrupts state.

### The cheat sheet

| Operation | Idempotent? | Why |
|---|---|---|
| `pread(fh, off, len)` | Yes | Pure read, no side effects |
| `pwrite(fh, off, len, buf)` | Yes | Absolute offset, overwrites same bytes |
| `append(fh, buf, len)` | **No** | Each call shifts offset, content grows |
| `mkdir(path)` | **No** | Second call returns EEXIST |
| `creat(path)` | **No** (in general) | Truncates if O_TRUNC, fails if O_EXCL |
| `unlink(path)` | **No** | Second call returns ENOENT |
| `lookup` / `getattr` | Yes | Pure reads |

### The pwrite vs append diagrams (slides 63, 64)

**pwrite ABBA at offset 0** to file `AAAA` three times: result is `ABBA` after each call. Stable.

**append B** to file `A` three times: `AB`, `ABB`, `ABBB`. State drifts with retries.

This is exactly why NFSv2 has `pwrite` and not `append`. The client computes the offset locally and sends it, so a retried `pwrite` overwrites the same bytes.

### Predicted T/F set

| Claim | Truth |
|---|---|
| In NFS, the server stores no per-client offset | True |
| In NFS, the client specifies the offset on every read | True |
| `mkdir` is idempotent so it is safe in NFS | False |
| If a `pwrite` retry executes twice, the file ends up corrupted | False (idempotent) |
| The reason NFS does not support `append` is performance | False (it is correctness under retry) |

---

## 8. Stateful suppression vs stateless idempotence (slides 60 to 62)

TCP suppresses duplicates **statefully** via sequence numbers. If the server crashes and reboots, the seq num state is gone, so a retried client message would be re-executed.

NFS does not rely on TCP-level dedup for correctness. It relies on idempotent operations so that re-execution is harmless. **This is the core design choice.**

**Exam-shaped claim:** "NFS uses TCP and therefore avoids duplicate execution after server crash." → **False.** TCP dedup is per-connection and lost on crash.

---

## 9. Client-side fd object (slides 66, 67)

The client builds a normal POSIX `fd` interface on top of stateless server APIs.

**Client open file table entry contains:**
- File handle (FH) for the server
- Current offset (managed locally)

When app calls `read(5, buf, 1024)`:
1. Client looks up fd 5 in its table → FH and offset
2. Client sends `pread(FH, offset, 1024)` to server
3. Server returns data
4. Client updates offset locally, returns data to app

**Why this is testable:** it lets you correctly answer "is the server stateful per-client?" (no) while also answering "does the client present a stateful POSIX interface?" (yes). Both can be true.

---

## 10. Write buffering problem (slides 70 to 82)

This is the most FRQ-shaped scenario in the lecture. **Walk through it once and remember the failure mode.**

### Setup

Server acknowledges write **before** the data is on disk (write buffer in memory). Faster, but vulnerable to crash.

### The damaging trace (slides 72 to 80)

```
client:                server mem:        server disk:
write A to 0           A                  
write B to 1           A B                
write C to 2           A B C              
                                          [flush]    A B C
write X to 0           X B C              
write Y to 1           X Y C              
                                          [partial flush] X B C
                       (crash, mem lost)  X B C
write Z to 2           Z (mem)            X B C
                                          [flush]    X B Z
```

**Final disk state: X B Z.**

This is **not a state the client ever observed**. Client thought every write succeeded (got an ACK for each) but disk shows a state that does not correspond to any prefix of the client's writes. Y was acknowledged and lost.

### The two solutions (slides 81, 82)

1. **Don't use a server-side write buffer.** Persist to disk before ACKing. Correct but slow (disk write on every call).
2. **Persistent write buffer.** Battery-backed RAM or NVRAM. Survives crash. Cost: more expensive hardware.

### Predicted FRQ wording

> "An NFS server uses a non-persistent in-memory write buffer and ACKs writes before they reach disk. Describe a sequence of client writes after which the disk state cannot be explained by any prefix of client operations. Explain what guarantee is violated."

You answer with the slide 80 trace. Guarantee violated: **durability of acknowledged writes.**

---

## 11. NFS summary (slide 84) and the full mental model

Two design pillars, both directly testable as MC:

| Property | Why |
|---|---|
| **Stateless server** | Server crash is transparent: client retries, server has nothing to recover. Goal 2 from slide 34. |
| **Idempotent operations** | Retries due to lost ACKs are harmless. Resolves slide 28 ambiguity without per-client state. |

Add to this the supporting machinery:
- **File handles** for stable, generation-checked references
- **Client-side fd table** for POSIX semantics on top of a stateless protocol
- **Adaptive timeouts** (inherited from TCP)

---

## Predicted exam questions

### True / False (15)

1. NFS servers maintain an open file table per client. **F**
2. A file handle in NFS contains the server's file descriptor. **F**
3. The generation number in a file handle helps detect inode reuse. **T**
4. `pwrite` is idempotent. **T**
5. `append` is idempotent. **F**
6. `mkdir` is idempotent. **F**
7. After an NFS server crash and reboot, the client must re-open all files. **F**
8. NFS clients specify the offset on every read request. **T**
9. NFS read system calls execute entirely on the client. **F** (round-trips to server)
10. TCP guarantees in-order delivery. **T**
11. UDP guarantees in-order delivery. **F**
12. The sender of a message in a distributed system can always determine whether the receiver got it. **F**
13. Adaptive timeouts wait longer between successive failed retries. **T**
14. NFS relies on TCP to deduplicate retried operations across server crashes. **F**
15. If an NFS server uses a non-persistent write buffer, an acknowledged write may be lost on crash. **T**

### Multiple choice (5)

**MC1.** A client sends `pwrite(FH, 0, 4, "ABBA")`, the server executes it and crashes before the ACK arrives. The client times out and retries. Final state of the file?
- a) Corrupted (ABBA written twice causes overflow)
- b) `ABBA` (idempotent) ✅
- c) Server returns error on retry
- d) Depends on whether server persisted the first call

**MC2.** Why does NFSv2 not support `append`?
- a) Performance reasons
- b) `append` is not idempotent under retry, so the API would be incorrect ✅
- c) The local FS does not support it
- d) Generation numbers cannot encode append offsets

**MC3.** A client opens a file, getting FH X. Another client unlinks it, the inode is freed and reallocated to a new file. The first client tries to read using FH X. What happens?
- a) Reads the new file's contents
- b) Reads the original file's stale contents
- c) Server returns stale file handle error due to generation mismatch ✅
- d) Server hangs

**MC4.** A server uses an in-memory write buffer and acks writes before flushing. After several writes and a crash, the disk shows state that does not match any prefix of the client's history. The violated guarantee is:
- a) Atomicity of individual writes
- b) Ordering of writes on disk
- c) Durability of acknowledged writes ✅
- d) Idempotence

**MC5.** After an NFS server reboots, the client's existing open file descriptors:
- a) Become invalid and must be re-opened
- b) Continue to work because the FH is still valid and the server is stateless ✅
- c) Continue to work only if the server enables crash recovery
- d) Continue to work only if TCP connection survived

---

## Flashcards (front / back)

| Front | Back |
|---|---|
| Lamport's definition of a distributed system | One where a machine you have never heard of can crash your program |
| Three techniques for reliable messaging | ACK, timeout, sequence numbers (with receiver memory) |
| Why slide 28 case 1 vs case 2 matters | Sender cannot distinguish lost message from lost ACK; this motivates idempotence |
| Definition of idempotent | f() has same effect as f() called any number of times |
| Three idempotent NFS operations | pread, pwrite, lookup/getattr |
| Three non-idempotent operations | append, mkdir, unlink |
| What kills Strategy 1 (server fds) | Server crash, fd table empty or reused on reboot |
| What kills Strategy 3 (raw inodes) | Inode reuse after delete |
| Components of an NFS file handle | volume ID, inode number, generation number |
| What generation number prevents | Stale handle silently reading reallocated inode |
| Where the open file table lives in NFS | On the client only (server is stateless) |
| Server's role on every READ | Use FH → check generation → read inode → compute block → read data |
| Why NFS does not support `append` | Not idempotent, retries would corrupt state |
| Two fixes for unsafe write buffer | No buffer (slow), or persistent battery-backed buffer |
| Guarantee violated by lost write buffer | Durability of acknowledged writes |
| Does a server crash require re-open in NFS | No, stateless protocol means client retries transparently |
