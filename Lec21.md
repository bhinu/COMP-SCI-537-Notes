# CS 537 Lecture 21: FFS Policy + LFS

Notes mapped to your Midterm 3 pattern summary. Cross-references to specific exam questions in **bold**.

---

## Part 1: FFS Placement Policy

### The complete policy (slide 10)

| Object | Placement rule |
|---|---|
| File inode | Same group as parent directory |
| Directory inode | New group with fewer-than-average used inodes |
| First data block | Same group as inode |
| Subsequent data blocks | Near previous block |
| Large file: data after 48KB | Spill to new group |
| Large file: every 1MB after spill | Move to another group with fewer-than-avg blocks |

### Why these rules

The FS is one big tree: every file is "related" to every other file through the root. Placing every related thing together means making no choice. FFS picks a few strong locality rules and breaks elsewhere:

- **File inodes go WITH parent dir.** Reason: `ls -l` reads each file's inode after listing dir entries. Same group = sequential reads, fast `ls -l`.
- **Dir inodes BREAK from parent.** Reason: a single big tree filling one group is bad for everyone. Spread directories so the tree fans out.
- **Data blocks WITH inode.** Reason: open then read is the dominant access pattern.
- **Large files SPILL.** Reason: one large file fills a group and starves all small files that share the dir.

### Maps to exam questions (Tier 1 in your summary)

- **"Inode points to data block in another cylinder group"** (SP22 Q15): TRUE for large files. The 48KB spill rule is the mechanism. Slide 9 to 10.
- **"Directory inode + dir data in same group"** (SP22 Q16): TRUE. Rule from slide 5: "put data near inode."
- **Modified FFS, file inodes in different group from dir, what slows down?** (S23 Q36, also slide 48 quiz):
  - `ls foo`: UNAFFECTED. Only reads dir data. Dir inode and dir data are still co-located.
  - `ls -l foo`: SLOWER. Stats each file = reads each file inode = cross-group seek.
  - `open("/foo/bar")`: SLOWER. Path lookup of bar reads bar's inode in the displaced group.
  - **Answer: only plain `ls` is unchanged.**

### Other FFS features (slide 11)

For MC recall:
- Large blocks (with libc buffering and fragments for small file efficiency)
- Long file names
- Atomic rename
- Symbolic links

---

## Part 2: LFS Motivation and Strategy

### Why LFS exists (slides 14 to 15)

The performance problem: gap between sequential and random I/O grows over time. RAID-5 in particular kills small random writes because of read-modify-write parity overhead.

LFS bet: **design for writes to use the disk purely sequentially**.

### Strategy

1. Buffer writes in memory until you have a full segment (MB scale).
2. Write the segment sequentially to a free location on disk.
3. Never overwrite. Old copies remain on disk. This is **Copy-on-Write (COW)**.

Bonus property: if you crash mid-write, the old version is still intact.

### What LFS removes from FFS (slide 21, 26)

Both bitmaps go away:
- **Inode bitmap**: not needed (you write inodes wherever the log head is)
- **Data bitmap**: not needed (same reason)

This is the slide 30 "Why no bitmap?" gotcha. Allocation is always at the log head; free space is tracked at segment granularity, not block granularity.

---

## Part 3: The Inode Number Problem and IMAP

### Attempt 1: name inodes by disk offset (slides 21 to 25)

If inode "name" = disk offset, then every inode rewrite changes its name. The directory entry pointing to it must update. The parent directory's inode then changes location, so its parent must update. Cascade goes all the way to root. Worse, those cascade writes themselves change inode numbers, so it never terminates cleanly.

### Attempt 2: IMAP (slides 26 to 28)

**IMAP** = mapping from inode number to current on-disk inode location.

Properties:
- Inode numbers are stable
- Only the IMAP entry changes when an inode is rewritten
- IMAP is too big to keep entirely in memory
- IMAP is written in pieces inside segments, alongside data and inodes
- Pointers to those IMAP pieces are kept in memory

### Reading a file in LFS

Three steps once IMAP pointers are cached:
1. Look up inum in IMAP (memory hit) -> get inode disk location
2. Read inode -> get data block pointers
3. Read data blocks

**One disk read per inode lookup once IMAP is in memory.** This is the SP22 Q14 gotcha (TRUE).

### Maps to exam questions

- **SP22 Q60: IMAP updated on every file write AND on cleaning.** TRUE. Every inode rewrite produces a new disk location, so IMAP must update. Cleaning rewrites inodes too.

---

## Part 4: What Goes Together in a Segment

The key gotcha pair from your pattern summary, confirmed by slide 48:

| Question | Answer | Why |
|---|---|---|
| When data block written, is its inode in same segment? | **TRUE** | Inode points to the new block, gets rewritten, bundled in same segment write |
| When inode written, are ALL its data blocks in same segment? | **FALSE** | If only some blocks changed, only those are rewritten. Old blocks stay put. |

**SP22 Q17:** Directory entries pointing to a new inode are NOT necessarily in the same segment. The directory only needs updating on create/unlink/rename, not on every file write.

---

## Part 5: Garbage Collection

### Mechanism: segment summary (slides 39 to 44)

Two cases for liveness:

**Inode liveness: easy.** Look up inum in IMAP. If IMAP doesn't point to this inode location, this is an old version. Dead.

**Data block liveness: hard.** Naive approach scans every inode. Fix: each segment carries a **Segment Summary (SS)** with (inode_number, offset_in_file) for each block.

Liveness check for a data block:
1. Read SS, get (inum, offset)
2. Look up inum in IMAP, get current inode location
3. Read the current inode
4. Check what it has at that offset. If it points to THIS block, alive. Otherwise dead.

### Policy: which segments to clean (slide 45)

Heuristics:
- Clean most-empty first (cheapest reclaim)
- Clean coldest segments (least likely to be rewritten soon)
- Hybrid policies exist

### Cleaning cost math

Segment with fraction $u$ live:
- Read 1 segment (cost 1)
- Write $u$ to a fresh location (cost $u$)
- Original segment now fully free

To reclaim space then write 1 segment of new user data:
$$\text{total cost} = 1 + u + 1 = 2 + u$$

vs writing to an already-empty segment: cost 1.

**SP22 Q46 example:** $u = 0.8$ -> total $= 2.8$ -> ~3x. Closest MC answer is 3X.

**SP22 Q47:** Cleaning continuously vs cleaning when out of space -> continuously does MORE total work because you may clean segments that would later be invalidated naturally.

### Write amplification (slide 46)

$$\text{WA} = \frac{\text{total bytes read+written by storage}}{\text{bytes written by app}}$$

Same data written 10 times in LFS:
- App: 10 * 100KB = 1000KB
- Each rewrite eventually triggers cleaning of the previous version: + 100KB write per cleaning, + 100KB read per cleaning
- WA grows with rewrite frequency

Critical for SSDs: flash cells wear with each erase cycle, so amplification multiplies wear.

---

## Part 6: Crash Recovery

### What's lost in a crash

In-memory IMAP pointers. They must be rebuilt.

### Naive approach
Scan entire log, rebuild IMAP from scratch. Slow.

### Checkpoint approach (slides 50 to 54)

Periodically (e.g., every 30s) write IMAP pointers + log tail position to a fixed checkpoint region.

On reboot:
1. Read checkpoint -> IMAP pointers as of last checkpoint
2. Read log tail position from checkpoint
3. **Roll forward**: scan past the tail to recover writes that happened after the checkpoint

### Two-region checkpoint (slides 56 to 61)

Risk: crash WHILE writing checkpoint -> corrupted checkpoint, and you have no IMAP pointers at all.

Solution: TWO checkpoint regions, alternating writes. Each carries timestamp/checksum. On boot, pick the newest valid one. Old one is your safety net.

This is the pattern summary's "checkpoint alternates between two locations" gotcha.

---

## Part 7: LFS vs FFS Workload Performance

| Workload | LFS | Why |
|---|---|---|
| Random writes | EXCELLENT | Buffered, written sequentially as a segment |
| Sequential writes | Excellent | Same mechanism |
| Sequential reads after sequential writes | Good | Data still in order on disk |
| Sequential reads after MANY random writes | **BAD** | Reads hop between segments where each rewrite landed |
| Random reads | OK | One IMAP lookup, one inode read, one data read |

**S23 Q40:** Random read after random write is the LFS killer. Slide 62: LFS bets that "future reads cached in memory" hides read fragmentation.

### Slide 63: what actually happened to LFS

GC unpredictability hurt adoption. Journaling (next lecture) captured most of the write benefit without GC headaches. Then SSDs flipped the script: every modern SSD's FTL is essentially LFS. The design lives on inside hardware.

---

## Quick-Recall Flashcards

1. **Q:** FFS placement rule for file inodes?
   **A:** Same group as parent directory.

2. **Q:** FFS placement rule for directory inodes?
   **A:** New group with fewer-than-average used inodes.

3. **Q:** At what file size does FFS spill data to a new group?
   **A:** 48KB (when an indirect block is needed). Then every 1MB after that.

4. **Q:** Modified FFS where file inodes are in a different group from parent dir. Which operations slow down?
   **A:** `ls -l` (stats every file) and `open(path)` (loads file inode). Plain `ls` unchanged.

5. **Q:** Two motivations for LFS?
   **A:** (1) Growing seq vs random I/O gap, (2) RAID-5 small-write penalty.

6. **Q:** Why does LFS not need inode/data bitmaps?
   **A:** Allocation always happens at the log head; free space is tracked at segment level.

7. **Q:** What is IMAP?
   **A:** Map from inode number to current on-disk inode location.

8. **Q:** Where is IMAP stored?
   **A:** Pieces are written into segments alongside data; pointers to those pieces are kept in memory and persisted at checkpoint time.

9. **Q:** When LFS writes a data block, is the file's inode rewritten too?
   **A:** Yes, in the same segment.

10. **Q:** When LFS rewrites an inode, are ALL its data blocks rewritten too?
    **A:** No, only changed blocks.

11. **Q:** How does LFS check if a data block is live?
    **A:** Read segment summary -> get (inum, offset) -> look up inum in IMAP -> read current inode -> check if it points to this block.

12. **Q:** How does LFS check if an inode is live?
    **A:** Look up inum in IMAP. If it doesn't point to this location, dead.

13. **Q:** Two GC policies LFS considers?
    **A:** Clean most-empty first; clean coldest first.

14. **Q:** Cleaning cost: u live, write 1 segment of new data. Total disk work?
    **A:** read 1 + write u + write 1 = 2 + u.

15. **Q:** How does LFS recover from a crash?
    **A:** Read latest checkpoint -> get IMAP pointers + log tail -> roll forward by scanning past the tail.

16. **Q:** Why two checkpoint regions?
    **A:** To survive a crash during a checkpoint write. Use timestamp/checksum to pick newest valid one.

17. **Q:** Worst LFS workload?
    **A:** Random reads after many random writes (data scattered across segments, no spatial locality).

18. **Q:** Define write amplification.
    **A:** Total bytes written/read by storage / bytes written by app.

19. **Q:** Why is write amplification bad for SSDs?
    **A:** Flash cells wear out per erase cycle; amplification multiplies wear.

20. **Q:** For 100 MB/s disk with 10ms seek, what segment size gives 50% of peak transfer?
    **A:** 1MB. At 50% transfer, half the time on seeks: 50 seeks/sec, 50 MB transferred/sec, so 50 MB / 50 seeks = 1MB per segment.

---

## Predicted Questions for This Lecture

### True/False (8 to 10 likely on exam)

1. In LFS, when a data block is written, the inode that points to it is also written to the same segment.
   **TRUE**

2. In LFS, when an inode is written, all data blocks it points to are written to the same segment.
   **FALSE** (only changed blocks)

3. LFS removes the inode bitmap from FFS but keeps the data bitmap.
   **FALSE** (both removed)

4. The FFS large-file policy spills data to a new group at 1MB.
   **FALSE** (spills at 48KB, then every 1MB after)

5. Reading an inode in LFS requires reading the IMAP from disk on every access.
   **FALSE** (IMAP pointers cached in memory)

6. LFS keeps two checkpoint regions to handle crashes during a checkpoint write.
   **TRUE**

7. In LFS, an inode's number changes whenever the inode is rewritten.
   **FALSE** (inum is stable; IMAP location changes)

8. To check if a data block is live, LFS scans every inode in the file system.
   **FALSE** (uses segment summary then IMAP lookup)

9. FFS places directory inodes in the SAME group as the parent directory.
   **FALSE** (places in NEW group with fewer-than-avg used inodes)

10. FFS places file inodes in the same group as their parent directory.
    **TRUE**

11. LFS write amplification is always 1 because writes are sequential.
    **FALSE** (cleaning adds extra reads and writes)

12. LFS performs well on random reads following many random writes.
    **FALSE** (data ends up scattered)

### Multiple Choice (3 to 5 likely)

1. **Which is NOT in an inode?**
   a. Owner
   b. Permissions
   c. Filename extension
   d. Size
   e. Reference count
   **Answer: c** (filename and extension live in directory entries)

2. **Modified FFS places file inodes in a different group from parent directory. Which operation is unaffected?**
   a. ls foo
   b. ls -l foo
   c. open("/foo/bar")
   d. cat /foo/bar
   **Answer: a** (only reads dir data, no file inode access)

3. **In LFS, why is no inode bitmap needed?**
   a. The OS always knows which inodes are used
   b. Inodes always written sequentially at the log head; allocation is segment-level
   c. The IMAP stores allocation status
   d. Inodes are pre-allocated at fs creation
   **Answer: b**

4. **LFS segment with 80% live data; reclaim it and write a new segment of data. Total disk work relative to writing to an empty segment?**
   a. 1x
   b. 1.8x
   c. ~3x
   d. ~5x
   **Answer: c** (read 1 + write 0.8 + write 1 = 2.8, ~3x)

5. **Disk: 200 MB/sec, 5ms seek. Segment size for 80% peak transfer?**
   a. 250 KB
   b. 800 KB
   c. 4 MB
   d. 8 MB
   **Answer: c.** At 80% transfer, seek time fraction is 0.2. In 1 sec: 0.2s on seeks at 5ms each = 40 seeks; 0.8s transferring at 200 MB/s = 160 MB. 160 / 40 = 4 MB per segment.

6. **Which is FALSE about LFS garbage collection?**
   a. Cleans entire segments
   b. Uses segment summary to find data block liveness
   c. Always cleans the segment with the most live data first
   d. Increases write amplification
   **Answer: c** (cleans MOST EMPTY first)

7. **After a crash, LFS recovers by:**
   a. Scanning the entire disk
   b. Replaying the journal
   c. Reading checkpoint then rolling forward from log tail
   d. Restoring IMAP from a redundant in-place copy
   **Answer: c**

8. **Two-region checkpoint exists because:**
   a. Performance: writes can interleave
   b. Crash safety: protects against crash during checkpoint write
   c. Capacity: one region too small
   d. Compatibility: alternates with FFS
   **Answer: b**

9. **LFS bets on which assumption to justify ignoring read locality?**
   a. Reads are rare
   b. Reads will hit the cache
   c. Reads can be reordered by the disk scheduler
   d. SSDs are fast enough that locality doesn't matter
   **Answer: b** (slide 62: "assume future reads cached in memory")

10. **After many random writes to an LFS file, sequential read performance is:**
    a. Same as on FFS
    b. Faster than FFS because of segment buffering
    c. Slow because data is scattered across segments
    d. Triggers automatic defragmentation
    **Answer: c**

---

## What's NOT in this lecture (cross-reference)

These pattern-summary topics show up in adjacent lectures:
- **Journaling** (data/ordered/writeback, recovery walkthrough): next lecture
- **SSDs** (direct map vs log): later lecture, but slide 63 hints at the connection
- **FSCK** (consistency vs recoverability): FFS crash consistency lecture
- **NFS, RPC, TCP/UDP**: distributed systems block
- **Disk scheduling, RAID**: earlier lectures
