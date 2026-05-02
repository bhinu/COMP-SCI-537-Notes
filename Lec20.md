# Lecture 20: Fast File System (FFS) — Study Notes

Mapped to Midterm 3 question patterns. Slide references in (slide N).

---

## Part 1: FS structure recap (slides 4 to 14)

### What's on disk

A file system carves the disk into typed regions:

| Region | What it stores |
|---|---|
| Superblock (S) | FS configuration: block size, # inodes, group layout |
| Inode bitmap (IB) | 1 bit per inode: free or allocated |
| Data bitmap (DB) | 1 bit per data block: free or allocated |
| Inode table (I) | The actual inode structs |
| Data blocks (D) | File contents, directory entries, indirect blocks |

(slide 14 layout: `S IB DB I I I I I` then 7 rows of 8 D blocks each)

### What's in an inode

- File type (regular, directory, symlink) and mode bits
- Size, owner, permissions
- Timestamps (atime, mtime, ctime)
- **Link count** (how many directory entries point at this inode)
- Block pointers: direct + indirect + double indirect + triple indirect

**Not in the inode (high-frequency T/F trap):**
- ❌ Filename. Name lives in the directory entry, not the inode.
- ❌ Pointer to the directory that contains this file.
- ❌ File extension as a separate field.

### Multi-level indexing (slide 5)

Like multi-level page tables. As the file grows, you need more indirection:

```
Direct       → block of data
Indirect     → block of direct pointers      → data
Double ind.  → block of indirect pointers    → block of direct pointers → data
Triple ind.  → block of double ind. pointers → ...
```

**Pattern:** more metadata reads needed for bytes deeper in the file. First few KB are 1 read, middle range needs 2 reads (inode + indirect), tail needs 3 or 4.

### Directory entries (slide 7)

A directory is just a file whose data blocks contain a list of entries:

```
valid | name             | inode #
  1   | .                | 134
  1   | ..               | 35
  1   | foo              | 80
  1   | bar              | 23
```

`unlink("foo")` flips the valid bit to 0. The slot can be reused for the next entry. Inode is freed only when **link count hits 0 AND no process has it open** (this matters for the fork+unlink question).

---

## Part 2: I/O cost of FS operations (slides 16 to 20)

This is the highest-yield part of the lecture for FRQ questions. Memorize the pattern, then count for any path depth.

### `open(/foo/bar)` — 5 reads, 0 writes

1. Read root inode
2. Read root data (find `foo` entry → inode #)
3. Read foo inode
4. Read foo data (find `bar` entry → inode #)
5. Read bar inode

**Pattern:** for path of depth N, open does (N + 1) inode reads + N directory data reads = roughly **2N + 1 reads**. For `/foo/bar` (depth 2): 3 inode reads + 2 dir data reads = 5.

### `read(/foo/bar)` (already opened) — 2 reads, 1 write

1. Read bar inode (for block pointers, may be cached from open)
2. Read bar data block
3. **Write** bar inode (update atime)

The atime write is why even reads cause writes. (Most modern systems mount with `noatime` or `relatime` to skip this.)

### `close()` — 0 disk ops

Nothing to do on disk. Just kernel bookkeeping.

### `create(/foo/bar)` — 5 reads, 6 writes (slide 19, quiz Q2)

Reads (path traversal + finding free slots):
1. Root inode
2. Root data
3. foo inode
4. foo data
5. inode bitmap (find free inode for bar)

Writes:
1. **inode bitmap** (mark bar's inode allocated)
2. **bar inode** (initialize: type, size 0, link count 1)
3. **foo data** (add `bar` entry pointing to bar's inode #)
4. **foo inode** (update mtime, size if entry list grew)
5. **data bitmap** (only if create includes a data block)
6. **bar data block** (write the 1 block of content)

**Quiz Q2 (slide 30) phrasing:** "create `/537/midterm-answers` and write 1 data block. How many blocks updated?" Answer: **6**. They count: midterm-answers inode, /537 dir data (add entry), data block, inode bitmap, data bitmap, /537 inode.

### `write(/foo/bar)` (already opened) — 2 reads, 3 writes

1. Read inode (already cached probably)
2. Read data bitmap (find free block)
3. Write data bitmap
4. Write data block
5. Write inode (update size, block pointer, mtime)

---

## Part 3: Old UNIX FS problems (slides 33 to 36)

### Why it sucked: 2% of disk bandwidth

Original layout: superblock at front, then all inodes, then all data blocks. Free list = linked list embedded in inodes/data blocks.

Three specific failures:

1. **Free list scrambling.** New FS allocates contiguous, but as files are deleted and created, the free list gets out of order. Allocation becomes random across the disk.
2. **Tiny block size (512 bytes).** Even sequential reads need many small I/Os.
3. **Disk-unaware layout.** All inodes at the front, all data far away. Reading a file means seeking to inode region, then seeking to data region. Every file access is a long seek.

### Aging effect (slide 34)

- New FS: 17.5% of disk bandwidth
- After a few weeks: 3% of disk bandwidth

Hacky pre-FFS solutions that didn't really work: occasional defrag, keep free list sorted.

**The deeper problem:** old FS treats disk like RAM. Random access is fine in RAM, terrible on disk.

---

## Part 4: FFS innovations (slides 39 to 53)

### Innovation 1: Cylinder groups (slides 39 to 41)

Instead of one global layout, divide disk into groups. **Each group is a self-contained mini-FS** with its own superblock copy, inode bitmap, data bitmap, inode table, data blocks.

```
| S B I D ... | S B I D ... | S B I D ... | ...
   group 0       group 1       group 2
```

In FFS specifically, groups are ranges of cylinders → "cylinder groups." In ext2/3/4, groups are ranges of blocks → "block groups."

**Goal:** keep an inode and its data in the same group, so accessing the file is one short seek instead of two long ones.

### Innovation 2: Bitmaps replace free lists (slide 9)

Bitmap = 1 bit per block, 0 = free, 1 = allocated. Stored in fixed location at start of each group.

**Why bitmaps win:**
- Easy to find contiguous runs of free blocks (scan for zeros)
- No fragmentation of the free list itself
- Constant cost to check if a specific block is free

### Innovation 3: Replicated superblocks at varied offsets (slides 42 to 43)

Old FS kept superblock copies all on the top platter. If the top platter dies, all copies die. **Correlated failure.**

FFS stores each group's superblock at a **different offset** within the group, so any one platter or track failure can't take out all copies.

### Innovation 4: Smart placement policy (slides 45 to 51)

**Naive rule:** put related data near each other.

**Problem:** everything in the FS is "related" through the root. If you put everything near everything, you've made no decision.

**Real rule:** put **more related** stuff close, put **less related** stuff far apart.

#### The four placement rules (slide 48 + 51)

1. **File inodes:** put in the same group as the parent directory's inode. (Files in the same dir tend to be accessed together. `ls -l` reads every inode in a dir.)
2. **Directory inodes:** put in a **new group** with **fewer than average used inodes**. (Spreads dirs out so child files have room nearby.)
3. **First data block of a file:** put near the file's inode.
4. **Subsequent data blocks:** put near the previous data block (sequential layout).

#### Large file exception (slides 49 to 51)

Single large file can fill its whole group, displacing everything else. Most files are small, so prioritize many small files getting their groups.

**Threshold rule:**
- After **48 KB** (when you start using the indirect block), spill to a new group.
- Every subsequent **1 MB**, move to another group with fewer-than-average used data blocks.

The cost is one extra seek per 1 MB chunk, which is negligible compared to the 1 MB of sequential transfer.

### Other FFS features (slide 52)

- Larger blocks (4 KB instead of 512 B), with sub-block "fragments" to avoid wasting space on small files (libc handles buffering)
- Long filenames (no longer capped at 14 chars)
- Atomic rename
- Symbolic links

---

## Part 5: Modified FFS / placement policy gotchas

These are the FRQ patterns from your summary doc (S23 Q36).

### "What if file inodes lived in a different group from the directory?"

What slows down:
- **`ls -l`** — needs to stat every file → reads every file's inode → cross-group seeks for each
- **`open(/foo/bar)`** — path lookup needs to read foo dir's inode, then foo's data, then bar's inode (now in a different group)

What stays fast:
- **plain `ls`** — only reads dir data block, never touches file inodes
- **read/write of an already-open file** — fd already has the inode in memory

### "What if subdirectories shared a group with parent dir?"

What slows down:
- Anything that recursively walks the tree (find, du)
- Creating many subdirectories in one parent (group fills up)

What stays fast:
- Operations on files within a single dir

---

## Part 6: Multi-level index math (slide 5 + quiz Q3, Q4)

**Pattern:** plug numbers into the formula. This is a guaranteed FRQ.

```
P = pointers per indirect block = block_size / pointer_size
direct_capacity = num_direct_pointers × block_size
indirect_capacity = P × block_size
double_indirect_capacity = P² × block_size
triple_indirect_capacity = P³ × block_size

max_file_size = sum of all the above
```

### Worked examples from the lecture

**Slide 30 Q3: 4 KB blocks, 4 direct + 1 indirect, 4 B pointers**
- P = 4096 / 4 = 1024
- direct = 4 × 4 KB = 16 KB
- indirect = 1024 × 4 KB = 4096 KB
- max = **4112 KB** ✓ (slide answer correct)

**Slide 30 Q4: 1 KB blocks, 4 direct + 1 indirect, 4 B pointers**
- P = 1024 / 4 = 256
- direct = 4 × 1 KB = 4 KB
- indirect = 256 × 1 KB = 256 KB
- max = **260 KB**

⚠️ **Slide says 258 KB. That's wrong. Use 260.** Your summary doc has 260, which agrees with the formula. The slide arithmetic dropped 2 KB somewhere.

### Likely exam variants

| Block | Direct | Indirect levels | Ptr size | Max size |
|---|---|---|---|---|
| 4 KB | 12 | 1 + 1 dbl + 1 tri | 4 B | ~4 GB (bounded by size field, more in raw blocks) |
| 4 KB | 8 | 1 indirect only | 4 B | 32 KB + 4096 KB = 4128 KB |
| 1 KB | 4 | 1 indirect only | 4 B | 260 KB |
| 4 KB | 4 | 1 indirect only | 4 B | 4112 KB |
| 4 KB | 0 | 1 indirect, 1 dbl | 4 B | 4 MB + 4 GB |

Just memorize: P = block / ptr_size. Then add up direct + P + P² + P³.

---

## Part 7: When is an inode actually freed? (slide 30 Q1)

**Setup:** open file in parent, fork, parent unlinks the file, parent exits, child exits. When is the inode freed?

**Answer: when the child exits.**

Why: an inode has two reference counts.
- **Link count** (on-disk): number of directory entries pointing at this inode. `unlink` decrements this.
- **Open count** (in-memory): number of file descriptors referencing the inode across all processes.

Inode is reclaimed only when **both hit zero**.

Trace:
1. Parent opens file → open count = 1, link count = 1
2. Fork → child inherits the fd → open count = 2, link count = 1
3. Parent unlinks → link count = 0. Inode NOT freed. (open count still 2)
4. Parent exits → kernel closes parent's fds → open count = 1. Still not freed.
5. Child exits → open count = 0. **Now the inode is freed.**

Key fact: this is why you can `unlink` a file you have open and keep using it. Unix temp file trick.

---

## Flashcards

| Front | Back |
|---|---|
| What's NOT in an inode? | Filename, file extension, pointer to containing dir |
| What's IN a directory entry? | valid bit, name, inode number |
| How does FFS find a free block? | Scan the data bitmap for a 0 bit |
| Why are bitmaps better than free lists? | Constant-time check, easy to find contiguous runs, doesn't get scrambled |
| How many disk reads to open `/a/b/c/d`? | 5 inode reads + 4 dir data reads = ~9 reads (or 2N+1 for depth N) |
| How many writes to create a file with 1 data block? | 6: inode bitmap, data bitmap, file inode, file data, dir data, dir inode |
| What did the old UNIX FS achieve as % of disk bandwidth? | 2% steady-state, 17.5% new, 3% aged |
| Three reasons old FS was slow? | Free list scrambling, 512 B blocks, no locality between inode and data |
| What is a cylinder group? | A range of cylinders containing its own superblock copy, bitmaps, inode table, data blocks |
| Where do file inodes go in FFS? | Same cylinder group as the parent directory's inode |
| Where do directory inodes go in FFS? | A new group with fewer-than-average used inodes |
| Where does the first data block of a file go? | Same group as the file's inode |
| What's the FFS large-file threshold? | After 48 KB (when indirect block kicks in), spill to a new group |
| How often does a large file move groups? | Every 1 MB after the first spill |
| Why replicate superblocks at varied offsets? | Avoid correlated failure if one platter or track dies |
| When is an inode freed after unlink? | When link count = 0 AND open count = 0 |
| What's the formula for max file size? | direct + P + P² + P³ blocks, where P = block_size / ptr_size |
| What does `ls` need to read? Just `ls` not `ls -l` | Only directory data block |
| What does `ls -l` need to read? | Directory data + every file's inode (so cross-group hurts a lot) |

---

## Predicted T/F (15 likely)

These map to the question patterns in your summary doc.

1. **The filename of a file is stored in its inode.** → False. Lives in directory entry.
2. **A directory inode and its directory data block live in the same cylinder group.** → True.
3. **A file's inode and its first data block live in the same cylinder group.** → True.
4. **A directory inode lives in the same cylinder group as its parent directory's inode.** → False. Dir inodes go to a new group with below-average used inodes.
5. **For a 5 MB file in FFS, all data blocks live in the same cylinder group as the inode.** → False. After 48 KB, blocks spill to other groups; every 1 MB they move again.
6. **Bitmaps make it harder to find contiguous free blocks than a free list does.** → False. Easier (scan for zeros).
7. **The old UNIX FS achieved about 2% of disk bandwidth.** → True.
8. **FFS replicated superblocks all on the top platter for fast access.** → False. That was the OLD FS problem. FFS varies the offset.
9. **In FFS, `ls -l` is fast because file inodes live near the directory inode.** → True.
10. **`unlink` immediately frees the on-disk inode regardless of open file descriptors.** → False. Inode is freed only when both link count and open count hit zero.
11. **Reading a file in FFS never causes any disk writes.** → False. Atime updates write the inode (unless mounted noatime).
12. **In a directory entry list, deleting an entry shifts all subsequent entries to fill the gap.** → False. The valid bit is flipped to 0; the slot can be reused.
13. **FFS uses 512 byte blocks like the original UNIX FS.** → False. FFS introduced larger blocks (typically 4 KB) with fragments for small files.
14. **Path lookup for `/a/b/c` requires reading 4 inodes.** → True (root, a, b, c).
15. **The superblock contains the inode bitmap.** → False. Superblock has FS config (block size, # inodes, layout). Bitmap is a separate region.

## Predicted MC (8 likely)

**Q1.** A file system has 4 KB blocks, 4 byte pointers, and inodes with 12 direct pointers, 1 indirect, 1 double indirect. Max file size?
- (a) 4 MB
- (b) 4 GB ✓
- (c) 16 MB
- (d) 4 TB

P = 1024. Direct = 48 KB, indirect = 4 MB, double = 4 GB. Total dominated by double indirect.

**Q2.** You open `/a/b/c/d` for the first time (no caching). How many disk reads?
- (a) 4
- (b) 7
- (c) 9 ✓ (5 inodes + 4 dir data blocks)
- (d) 10

**Q3.** You create `/foo/bar` and write 1 data block. Number of disk writes (no journaling)?
- (a) 4
- (b) 5
- (c) 6 ✓ (inode bitmap, data bitmap, bar inode, bar data, foo data, foo inode)
- (d) 8

**Q4.** Which FFS placement rule applies to a directory inode?
- (a) Same group as its parent directory's inode
- (b) New group with fewer-than-average used inodes ✓
- (c) Same group as the root inode
- (d) Group with the most free data blocks

**Q5.** What's the FFS threshold for spilling a large file's data to a new group?
- (a) When the file exceeds the direct pointers (48 KB) ✓
- (b) When the file exceeds 1 MB
- (c) When the file exceeds the cylinder group size
- (d) Never; FFS keeps all blocks in one group

**Q6.** Why did the old UNIX FS achieve only 2% of disk bandwidth?
- (a) CPU was too slow
- (b) Free list scrambling, 512 B blocks, and inode/data separation ✓
- (c) Disk seek times were measured incorrectly
- (d) The cache was too small

**Q7.** A modified FFS puts file inodes in a different group from their parent directory. Which operation is hurt the most?
- (a) `cd` into the directory
- (b) `ls` (no flags)
- (c) `ls -l` ✓
- (d) `read` from an already-open file

**Q8.** When is an inode freed after `unlink`?
- (a) Immediately
- (b) When the next `sync` runs
- (c) When link count = 0 and no process has it open ✓
- (d) When `fsck` runs next

---

## Formula sheet (anchored to slides)

```
P = pointers_per_indirect = block_size / pointer_size       (slide 5)

direct_cap     = num_direct × block_size
indirect_cap   = P × block_size
double_cap     = P² × block_size
triple_cap     = P³ × block_size
max_file_size  = direct + indirect + double + triple        (slide 5)

open(path with N components) reads
   = (N+1) inodes + N directory data blocks                 (slide 16)
   = 2N + 1 disk reads

create(file, K data blocks) writes
   = 4 metadata writes (inode bitmap, file inode, dir data, dir inode)
   + 2K writes per data block (data bitmap once, data blocks K times)
   = 4 + 1 + K     for K=1, total 6                         (slides 19, 30)

write(K new blocks to existing file)
   = 2 reads (inode, data bitmap)
   + (1 + 1 + K) writes (data bitmap, inode, K data blocks)  (slide 20)

FFS large-file threshold
   = 48 KB before first spill, then every 1 MB              (slide 51)

Old FS bandwidth: 2% steady, 17.5% new, 3% aged              (slides 33, 34)
```

---

## Study sequence for this lecture

1. Drill the I/O cost of each operation (open, read, create, write, close). Be able to count for any path depth.
2. Memorize the four placement rules and the 48 KB / 1 MB large-file thresholds.
3. Run the multi-level index formula on 4 different (block size, ptr size, num direct) combinations until it's automatic.
4. Trace the unlink + fork + open count question on paper.
5. Be ready to answer "what gets slower / faster" if FFS placement is modified.

When you paste the next lecture (LFS, journaling, etc.), I'll do the same treatment.
