# Lecture 17: SSDs and Files

Post-midterm content. Two halves: SSDs (slides 1-29) and Files API (slides 30-69).

---

## EXAM PRIORITY MAP (based on SP22, F15, S23 sample midterms)

> **Tier 1: drill heavily**
> - Part A Section 3 (Three NAND ops + costs): conceptual base for every SSD Q
> - Part A Section 6-7 (FTL direct vs log + GC): SP22 Q49, Q50; S23 Q44
> - Part B Section 4 (Path translation cost): related to inode/path traversal Tier 1 pattern; S23 Q24 directly
>
> **Tier 2: know cold, lighter weight**
> - Part B Section 7-8 (FD model + offset semantics): S23 Q28, Q29, Q30, Q31 (a four-question block)
> - Part B Section 11 (Deletion: unlink + close + refcount): S23 Q42 (open + fork + unlink scenario)
> - Part B Section 13 (Atomic file update idiom): conceptual foundation for crash consistency
>
> **Tier 3: skim, low yield**
> - Part A Section 1 (Workloads framing)
> - Part A Section 2 (Cell types - just know "more bits = denser, slower, less reliable")
> - Part A Section 5 (Striping)
> - Part A Section 10 (Throughput numbers)
> - Part B Section 1-3 (file/inode/path intro - background only)
> - Part B Section 5 (Special directory entries)
> - Part B Section 9 (lseek mechanics)
> - Part B Section 12 (rename internals)

---

# Part A: SSDs and Flash Storage

> **Note**: Part A repeats most of L16's SSD material. If you've already drilled L16, you can use this as review and focus your time on Part B (the FD/offset semantics).

## 1. Workloads (the framing for everything that follows)  `[TIER 3]`

| Type | What it looks like | Metric | Bottleneck |
|---|---|---|---|
| Sequential | Read/write multiple MBs in a row (video stream, dataset load) | MB/sec | Transfer time |
| Random | Read/write small chunks (<64 KB) scattered (databases, email) | IOPS (ops/sec) | Seek + rotate time |

**HDD math to remember:**
- Sequential threshold: ~20 MB makes transfer time dominate seek/rotate (5 ms/MB transfer vs 10 ms seek/rotate, so 20 MB transfer = 100 ms transfer >> 10 ms seek).
- Random IOPS for HDD: ~100 (10 ms per seek/rotate).

## 2. NAND Flash Cell Types  `[TIER 3]`

A flash cell stores bits using distinct voltage levels.

| Type | Bits/cell | Reliability (erases before failure) | Speed |
|---|---|---|---|
| SLC (Single Level) | 1 | ~100,000 | Fastest |
| MLC | 2 | Lower | Slower |
| TLC | 3 | Even lower | Even slower |
| QLC (Quad Level) | 4 | ~1,000-1,600 | Slowest |

**More bits per cell = denser = cheaper, but slower and less reliable.** That sentence is all you need.

## 3. The Three NAND Operations and Costs  `[TIER 1]`

> **Sample exam hits**: This is the foundation for SP22 Q49, Q50 and S23 Q44 (FTL throughput math).

| Op | What it does | Granularity | Latency |
|---|---|---|---|
| Read | Returns contents of a page | Page (~4 KB) | 25-75 us |
| Program (write) | Flips selected 1s to 0s | Page | 200-1400 us |
| Erase | Resets all bits in a block to 1 | **Block** (128-256 KB) | 1.5-4.5 ms |

**Critical asymmetry:** read is fast, program is slower, erase is roughly 1000x slower than read.

**The killer constraint:** to write a page, you must first erase the entire block it lives in. You cannot just "rewrite" a page in place.

Compare to HDD: 4-9 ms seek + 4-7 ms rotational latency. SSD random reads crush HDDs (75 us vs 8+ ms).

## 4. Block vs Page  `[TIER 1]`

- **Page:** the unit of read and program (~4 KB).
- **Block:** the unit of erase (128-256 KB, holds many pages).

This mismatch is the entire reason FTL exists.

> **Direct trap question** (likely T/F): "Read, Erase, and Program all operate on ~4 KB segments." FALSE. Only read and program. Erase is block-level.

## 5. Striping  `[TIER 3]`

Page addresses are striped across multiple flash chips like array indices. A single request can span chips in parallel. This gives natural load balancing without any explicit RAID-style logic.

## 6. Flash Translation Layer (FTL)  `[TIER 1 - HIGHEST YIELD]`

> **Sample exam hits**: SP22 Q49 (direct map throughput = erases/sec ~500/sec), SP22 Q50 (log throughput = page writes/sec ~10000/sec), S23 Q44 (same comparison)

FTL is firmware (or sometimes software) that sits between the OS's logical block view and the physical NAND. It has three jobs:

1. **Translate** logical block reads/writes into physical read/erase/program ops.
2. **Reduce write amplification** (extra writes caused by the erase-block constraint).
3. **Wear leveling:** distribute writes evenly across blocks so no single hot block burns out first.

### FTL Approach 1: Direct Mapping  `[TIER 1]`

Logical page N maps 1-to-1 to physical page N.

- **Read:** straightforward, just fetch the physical page.
- **Write a single page:** read the entire block into RAM, modify the one page, erase the entire block, then re-program the whole block.

**Two big problems:**
1. **Write amplification.** A 4 KB write triggers a full block read + erase + program. That's bad latency and bad wear.
2. **Poor reliability.** Repeated writes to the same logical block hammer the same physical block. It dies fast. Worse, if power fails between the erase and the rewrite, you lose data.

> **Exam math**: random write throughput = erases per second. SSD spec sheet says 500 erases/sec? Then ~500 random 4KB writes/sec.

### FTL Approach 2: Log-Based Mapping  `[TIER 1]`

Treat the SSD like an append-only log.

- Every write goes to the next free physical page (the log head).
- Maintain an in-memory **logical-to-physical map**: logical page X currently lives at physical page Y.
- When you overwrite logical page 92, you don't modify the original physical page. You write the new version to a fresh physical page and update the map. The old physical page becomes **garbage**.

**Wins:**
- Avoids read-modify-write of whole blocks.
- Naturally spreads writes (good wear leveling) even if logical access is hot.

**Trade-off:** garbage accumulates, so you eventually need garbage collection.

> **Exam math**: random write throughput = page writes per second. ~100 us per program -> ~10000 random writes/sec. Roughly **20x better than direct map**.

### Walkthrough (memorize this pattern)

```
Initial: all blocks erased. logHead = page 0.

write(logical=92, data=w0):
  program(physical_page 0, w0)
  map: 92 -> 0
  logHead = 1

write(logical=17, data=w1):
  program(physical_page 1, w1)
  map: 92 -> 0, 17 -> 1
  logHead = 2

... (more writes fill block 0)

write(logical=92, data=w4):  // overwrite!
  erase(block 1) if needed   // block 1 is fresh, just program
  program(physical_page 4, w4)
  map: 92 -> 4 (was 0; physical page 0 is now garbage)
  17 -> 1, 33 -> 2, 68 -> 3
```

## 7. Garbage Collection  `[TIER 1]`

**Problem:** overwritten pages are garbage but still occupy space, and they share blocks with valid data.

**Procedure:**
1. Pick a block that has lots of garbage and some valid pages.
2. Read all valid pages from that block.
3. Write valid pages to the end of the log.
4. Update the logical-to-physical map for those pages.
5. Erase the original block (now fully reclaimed and ready to reuse).

**The cost:** GC adds extra read + write traffic the user didn't ask for. This is a major source of write amplification in real SSDs.

## 8. Overprovisioning  `[TIER 2]`

The SSD reports a smaller logical capacity than its actual physical capacity. The hidden pages are reserve space.

**Why:** keeps free pages available so the FTL can defer GC to a background task instead of running it on the critical write path.

> **Trap T/F**: "Overprovisioning hurts performance." FALSE. It helps by deferring GC.

## 9. Wear Leveling (two flavors)  `[TIER 2]`

- **Dynamic wear leveling:** spreading active writes across blocks (the log-based approach already does this for hot data).
- **Static wear leveling:** periodically shuffling cold blocks (data that never gets overwritten) to other locations so those underused blocks also get exercised. Without this, cold blocks would just sit there while hot blocks get reused over and over.

## 10. SSD vs HDD (Throughput Reference)  `[TIER 3 - reference]`

| Device | Random Read | Random Write | Seq Read | Seq Write |
|---|---|---|---|---|
| Samsung 960 Evo SSD | 85.5 MB/s | 244 MB/s | 2664 | 2508 |
| Crucial BX100 SSD | 24.5 | 73.5 | 466 | 392 |
| Seagate 15K HDD | 2 | 2 | 223 | 223 |

**Takeaways:**
- Random workloads: SSDs crush HDDs (10x to 50x).
- Sequential workloads: SSDs win but margin is smaller for cheaper SSDs.
- HDDs are still ~10x cheaper per bit.

## 11. Likely Exam Traps for SSDs  `[TIER 1 reference]`

| Claim | Truth |
|---|---|
| "Random workloads perform similarly on SSDs and HDDs" | **False.** SSDs are dramatically faster for random. |
| "Sequential workloads perform similarly on SSDs and HDDs" | **False** for high-end SSDs (way faster), but **closer** for cheap SSDs. |
| "Erase is comparatively expensive on SSD and should be minimized" | **True.** ~1000x slower than read. |
| "Read, Erase, and Program all operate on ~4 KB segments" | **False.** Read and Program are page-level (~4 KB). Erase is block-level (128-256 KB). |
| "Direct mapping is simple and fast" | **False.** Direct mapping is simple but slow because every page write triggers a full block read-modify-write. |
| "Log-based mapping needs garbage collection" | **True.** Overwritten pages accumulate as garbage and must be reclaimed. |
| "Overprovisioning hurts performance" | **False.** It helps by deferring GC off the critical path. |

## 12. One-Sentence Summary

SSDs are way faster than HDDs for random IO and competitive on sequential, but the asymmetric read/program/erase costs and the erase-block-before-write rule force the FTL to use log-style writes plus garbage collection plus wear leveling to hide the weirdness from the OS.

---

# Part B: Files and the FD API

> The FD/open-file-table model is a guaranteed exam topic per S23 (Q28-Q31, a four-question block). Path traversal cost is also Tier 1.

## 1. What is a File?  `[TIER 3]`

An **array of persistent bytes** that can be read/written. The file system is the collection of all files plus the OS subsystem managing them.

Files have **three kinds of names**:
- **inode number:** unique numeric ID (internal use)
- **path:** human-friendly string (`/usr/lib/file.so`)
- **file descriptor (fd):** integer index into a per-process table (used during reads/writes)

## 2. inode Basics  `[TIER 1 conceptual]`

> **Sample exam hits**: SP22 Q34, F15 Q56, S23 Q33 all test "filename is NOT in the inode" as T/F traps. Pattern summary high-frequency gotcha #1.

An inode stores **metadata** about a file:
- location (where data blocks live on disk)
- size
- permissions
- timestamps
- link count
- (and importantly: NOT the file name)

The inode is the canonical identity of a file. The name is just a label sitting in some directory.

## 3. Why Not Just Use inode Numbers in System Calls?  `[TIER 3]`

The naive API:
```c
read(int inode, void *buf, size_t nbyte)
write(int inode, void *buf, size_t nbyte)
```

Problems:
- inode numbers are unfriendly to humans.
- No organization or hierarchy.
- Offset semantics across processes are undefined (where does the next read pick up from?).

So we evolve the API.

## 4. Paths and Directories  `[TIER 1]`

> **Sample exam hits**: S23 Q24 (path `/this/exam/is/hard` requires 5 inode reads). Pattern summary calls path traversal Tier 1.

A **directory** is a special file whose contents are a mapping from string names to inode numbers:
```
"readme.txt" -> 3
"hello"      -> 0
```

Directories form a **tree** rooted at `/`. File names only need to be unique **within their directory**, so `/usr/lib/file.so` and `/tmp/file.so` can coexist.

### Path Translation Cost (LIKELY EXAM QUESTION)

To open `/etc/bashrc` from scratch:

1. Read root inode (well-known location).
2. Read root data block to find `etc -> 0`.
3. Read inode 0 (etc directory).
4. Read etc's data block to find `bashrc -> 3`.
5. Read inode 3 (bashrc file).
6. Read bashrc's data block.

**Total: 6 reads** for a path two levels deep. Generalize:
```
Inode reads      = depth + 1     (root + each component)
Directory reads  = depth         (one for each level of traversal)
Total cold reads = 2*depth + 1  (excluding final file's data block)
```

Drill table for path lookup inode counts:
| Path | Inode reads |
|---|---|
| `/foo` | 2 (root, foo) |
| `/foo/bar` | 3 |
| `/a/b/c/d` | 5 |
| `/this/exam/is/hard` | 5 (S23 Q24 answer) |

The OS caches prefix lookups (`/a/b`, `/a/bb`, `/a/bbb` all share `/a`).

This is **the** reason `open` is separate from `read`/`write`: traversal is expensive, do it once.

### Special Directory Entries  `[TIER 3]`

Every directory contains:
- `.` (self) inode = this directory's inode
- `..` (parent) inode = parent directory's inode

Visible in `ls -la` output.

## 5. Why No `writedir`?  `[TIER 3]`

Directories are managed indirectly through `mkdir`, `rmdir`, file creation, and `unlink`. You don't write raw bytes to a directory because the file system needs to maintain its internal structure (inode pointers, etc.) carefully. Random user writes would corrupt the FS.

`readdir` is fine because reading is safe.

## 6. The FD-Based File API  `[TIER 2]`

```c
int fd = open(char *path, int flag, mode_t mode);  // do the traversal once
read(int fd, void *buf, size_t nbyte);              // uses cached state
write(int fd, void *buf, size_t nbyte);
close(int fd);
```

**Wins over the path-based API:**
- Traversal happens once at `open`.
- Per-fd offset is well-defined.
- String paths (human-friendly), hierarchical, and no repeated lookup cost.

## 7. The Three-Layer FD Model  `[TIER 2 - drill]`

> **Sample exam hits**: S23 Q28-Q31 (four questions on this exact mechanism).

```
Per-process FD Table       System-wide Open File Table       inode Table
+----------+               +-------------------+              +---------------+
| 0 stdin  |               | refcnt: 2         |              | inode #1000   |
| 1 stdout |               | offset: 10        | -----------> | location, size|
| 2 stderr |               | inode ptr -------+|              +---------------+
| 3 -------+-------------> +-------------------+
| 4 -------+-------> +-------------------+
| 5        |        | refcnt: 1         |
+----------+        | offset: 0         |
                    | inode ptr -------+|
                    +-------------------+
```

Three levels:
1. **FD table (per-process):** integer index -> pointer to an open-file-table entry.
2. **Open file table (system-wide):** stores the offset, inode pointer, ref count, mode flags.
3. **inode table:** the actual file metadata.

### xv6 Source Reference

```c
struct file {
    struct inode *ip;
    uint off;
};

struct proc {
    struct file *ofile[NOFILE];  // per-process fd table
};

struct {
    struct spinlock lock;
    struct file file[NFILE];     // global open file table
} ftable;
```

## 8. Critical: Offset Semantics  `[TIER 2 - HIGH YIELD]`

> **Sample exam hits**: S23 Q28 (`dup` shares offset), Q29 (`fork` shares offset), Q30 (separate `open` does NOT share), Q31 (parent read advances child's view of offset).
>
> Per pattern summary, this whole topic is a four-question block in S23. Drill these three rules until reflex.

**Three rules** that get tested as T/F traps:

| Operation | New open file entry? | Shared offset with original? |
|---|---|---|
| `open()` (second call on same path) | **YES, new entry** | **NO, independent** |
| `dup(fd)` | **NO, same entry** | **YES, shared** |
| `fork()` | **NO, child shares parent's entries** | **YES, shared** |

### Worked Practice Problem (from slide 60)

```c
int fd1 = open("file.txt");   // returns 12
int fd2 = open("file.txt");   // returns 13
read(fd1, buf, 16);
int fd3 = dup(fd2);           // returns 14
read(fd2, buf, 16);
lseek(fd1, 100, SEEK_SET);
```

Tracing:
- After `int fd1 = open(...)`: fd1 has its own open-file entry. offset_fd1 = 0.
- After `int fd2 = open(...)`: fd2 has its own (separate) open-file entry. offset_fd2 = 0.
- After `read(fd1, buf, 16)`: offset_fd1 = 16. offset_fd2 still 0.
- After `int fd3 = dup(fd2)`: fd3 points to **fd2's** open-file entry. They share offset.
- After `read(fd2, buf, 16)`: shared offset becomes 16. So offset_fd2 = offset_fd3 = 16.
- After `lseek(fd1, 100, SEEK_SET)`: offset_fd1 = 100.

**Final answers:**
- **offset_fd1 = 100**
- **offset_fd2 = 16**
- **offset_fd3 = 16** (shared with fd2)

> **Drill mantra**: open = NEW entry (independent). dup = SAME entry (shared). fork = SAME entry (shared across processes).

## 9. lseek  `[TIER 3]`

```c
off_t lseek(int fd, off_t offset, int whence);
```

| `whence` | New offset |
|---|---|
| `SEEK_SET` | `offset` (absolute position) |
| `SEEK_CUR` | `current + offset` |
| `SEEK_END` | `file_size + offset` |

Just modifies the offset field in the open file table entry.

## 10. fsync (durability)  `[TIER 2]`

> **Sample exam hits**: SP22 Q22, S23 Q26 ("write returns before data hits disk - only fsync guarantees durability"). High-frequency gotcha #3 in pattern summary.

The FS buffers writes in memory for performance (batching, coalescing, deferred allocation). On crash, buffered data is lost.

```c
fsync(int fd);
```

Forces buffers to flush to disk **and** tells the disk to flush its internal write cache. Only after fsync returns is the data durable.

> **Single-line gotcha**: `write()` returning is NOT a durability guarantee. Only `fsync()` is.

## 11. Deleting Files  `[TIER 2]`

> **Sample exam hits**: S23 Q42 (open + fork + parent unlinks + parent exits + child exits scenario - inode freed when child exits). Lecture 19 quiz on slide 51 also reproduces this.

There is **no `delete` system call**. Files vanish through reference counting:

- `unlink(path)` removes a directory entry (a name -> inode link). Decrements the inode's link count.
- `close(fd)` or process exit removes the fd reference to the open file table entry. Decrements its refcnt.

**The inode (and its data blocks) is freed only when:**
- link count = 0 (no directory entry points here), AND
- open fd count = 0 (no process has it open).

### Classic exam scenario: open + fork + unlink

```c
int fd = open("file.txt");  // parent
fork();                      // child inherits fd
// parent
unlink("file.txt");          // link count -> 0, but file still has 2 fds
exit();                      // parent's fd closes, refcnt -> 1
// child
exit();                      // child's fd closes, refcnt -> 0, NOW the inode is freed
```

**Answer:** the inode is marked free **when the child exits**. Not at unlink, not at parent exit.

## 12. rename  `[TIER 3]`

```c
rename(char *old, char *new);
```

- Deletes the old link.
- Creates a new link to the **same inode**.
- Does **not move data**, even when "moving" between directories. Just changes the name in directory entries.

Atomic on most file systems (the rename either fully takes effect or doesn't).

### Crash-Safety Issue

If the system crashes between the "delete old link" and "create new link" steps, you can end up with no link (file orphaned) or both links (depending on order). This is why journaling exists.

## 13. The Atomic File Update Pattern  `[TIER 2 - foundational]`

> Not directly tested in samples, but it's the canonical "communicating requirements" example and underlies crash consistency lecture.

To safely update `file.txt` so that on crash you see either fully-old or fully-new contents:

```
1. write new data to file.txt.tmp
2. fsync(file.txt.tmp)            // make new data durable
3. rename(file.txt.tmp, file.txt) // atomic swap
```

Why this works:
- Until the rename, only the temp file has new data. file.txt is untouched.
- rename is atomic, so observers see one or the other, never partial.
- fsync before rename ensures the new data is actually on disk before the rename takes effect.

**Forgetting fsync** is a real-world bug. The rename could complete but the new data could still be sitting in a write buffer that hasn't reached disk.

## 14. Likely Exam Trap Patterns  `[TIER 2 reference]`

| Question pattern | Right answer |
|---|---|
| Two separate `open`s on the same file share an offset | **False.** Separate open file table entries. |
| `dup` creates an independent fd with its own offset | **False.** dup shares the open file table entry. |
| `fork` causes the child to have its own offset | **False.** Parent and child share the open file table entry. |
| Reading sequentially through a path costs ~1 disk read | **False.** ~2 reads per directory level + 2 for the file. |
| `unlink` immediately frees the inode | **False.** Only if no fds are open. |
| `rename` moves data when crossing directories | **False.** Just rewires the directory entry. |
| `fsync` is necessary for durability | **True.** Buffered writes are not durable. |
| Directories can be written via the `write` syscall | **False.** Use mkdir/rmdir/creat/unlink. |

---

## 15. What to Drill (ranked by sample exam frequency)

1. **FTL direct map vs log throughput math** (Tier 1, Part A). SP22 Q49-Q50 and S23 Q44 directly test this.
2. **Path traversal cost** (Tier 1, Part B Section 4). Inode reads = depth + 1. S23 Q24 tested this.
3. **FD offset semantics: open vs dup vs fork** (Tier 2, Part B Section 8). S23 Q28-Q31 was a four-question block.
4. **Inode does not contain filename** (Tier 1 gotcha). SP22 Q34, F15 Q56, S23 Q33.
5. **fsync vs write durability** (Tier 2). SP22 Q22, S23 Q26.
6. **Open + fork + unlink scenario** (Tier 2). S23 Q42 tested exactly this.
7. **Page vs block granularity** (Tier 1 SSD trap). One-line T/F.

## 16. What to Skim (low yield)

- Cell type table (SLC/MLC/TLC/QLC details).
- HDD vs SSD throughput numbers.
- lseek `whence` flag table.
- rename internals beyond "doesn't move data".
- Special directory entries (`.` and `..`).
