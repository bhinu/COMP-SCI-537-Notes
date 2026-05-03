# Lecture 22: Journaling, Exam-mapped Notes

## Where this lecture lands in the midterm

Per your pattern summary, journaling is **Tier 1** (Topic #8) and shows up on all three sample midterms. This lecture also reinforces Topic #7 (FFS crash consistency) and Topic #11 (FSCK).

Lecture's three exam-relevant beats:

1. The **crash consistency matrix** (slide 11) ↔ the summary's crash matrix in Topic #7.
2. **Three journaling modes** + atomicity guarantees ↔ S23 Q42, Q55, Q56, Q57; SP22 Q18.
3. **Ordering and barriers** ↔ recovery timelines like F15's 11-write sequence.

---

## 1. Consistency vs Atomicity (the central distinction)

The lecture is built on this distinction. Be able to state both crisply.

- **Consistency**: the FS's data structures don't contradict each other (no inode pointing to a block the bitmap says is free, etc.). FSCK guarantees this.
- **Atomicity (for persistence)**: a collection of writes is all-or-nothing across crashes. Either all of state B is visible, or all of state A.  Journaling guarantees this.

Slide 15's diagram is the key visual. Consistent states form a subset of all states. FSCK lands you in *some* consistent state (could be A, could be B, could be neither correctly). Journaling lands you in *exactly A or exactly B*.

**Exam trap (S23 Q41):** "FSCK guarantees consistency, NOT recoverability of the in-progress operation." Old vs new state is unpredictable after FSCK.

---

## 2. Why inconsistency happens

Three causes layered together:

1. **Redundant data**: the FS has multiple structures encoding overlapping facts (bitmap says block N is free, inode says block N belongs to file). Definition (slide 5): if knowing A constrains values of B, A and B are redundant.
2. **Multiple writes per logical op**: appending requires 3+ writes (data block, data bitmap, inode).
3. **Crashes interrupt mid-sequence**: power loss, kernel panic, reboot. I/O scheduling can also reorder writes, making it worse.

---

## 3. The crash matrix (slide 11)

Append to file requires updating: **inode**, **data bitmap**, **data block**.

| Subset written | Outcome | Why |
|---|---|---|
| bitmap only | **block leaked** | bitmap claims used, no inode references it |
| data only | **nothing bad** | just stale bytes in a still-free block |
| inode only | **points to garbage** | inode references a block the bitmap thinks is free; another file may overwrite |
| bitmap + data | **block leaked** | data is correct but no inode points to it |
| bitmap + inode | **points to garbage** | inode references uninitialized block |
| data + inode | **reuse risk** | another file can reallocate, since bitmap still says free |

This *is* the crash consistency matrix in your pattern summary. The vocabulary "leak" vs "points to garbage" is what FFS uses to label the failure mode.

---

## 4. Atomicity intuition (slides 16 to 27)

Replacing X with Y where f(X) is some redundant data (e.g., bitmap entry, parent inode pointer).

**Without journaling:**
- Write Y first: f(X) is now wrong, **bad crash window**.
- Write f(Y) first: X still on disk, **bad crash window**.
- No safe order exists.

**With journaling:**

1. Write Y to journal.
2. Write f(Y) to journal.
3. Write **commit** block (the magic single-write atomic point).
4. Checkpoint: write Y in place.
5. Checkpoint: write f(Y) in place.
6. Free journal entry.

After a crash, recovery reads the journal:

- Commit present → replay (idempotent rewrite of Y and f(Y) in place).
- Commit absent → discard journal, FS is in old state A.

Punchline: **the commit block is the atomic switch.** One sector write flips you from A to B atomically.

---

## 5. Journal layout (slide 29)

```
[Super | Journal | Group 0 | Group 1 | ... | Group N]
```

A transaction in the journal:

```
[ TxB | metadata blocks | (data blocks if full journaling) | TxE/commit ]
   ^                                                              ^
   begin                                                          end
```

**Terms to memorize:**

- *Journal*: reserved disk region.
- *Transaction*: writes for one logical operation.
- *Commit block*: final block whose presence signals "transaction is durable".
- *Checkpoint*: writing journaled data to its final in-place location.
- *Free*: marking the journal entry reusable after checkpoint completes.

---

## 6. The three journaling modes (highest-yield section)

Direct hits in your summary: S23 Q42, Q55, Q56, Q57; SP22 Q18.

### 6a. Data journaling (full)

Journal **everything**: metadata + data.

- Cost: every block is written twice (once to journal, once in place).
- Benefit: simplest recovery, strongest guarantee.

Sequence for "write A to block 5, B to block 2":

1. Write `[5,2 | A | B | TxE=0]` to journal (commit not yet set).
2. Set commit bit, journal becomes `[5,2 | A | B | TxE=1]`.
3. Checkpoint: write A to block 5, B to block 2.
4. Clear journal entry (commit=0 again).

### 6b. Writeback journaling

Journal **metadata only**. Data goes to its in-place location whenever convenient (before, during, or after the transaction).

**Problem (slide 50):** if metadata is journaled and replayed but data was never written, the inode now points to whatever stale bytes were on disk before. Could be **someone else's old data**, including sensitive content. This is the "leak of sensitive data" risk.

### 6c. Ordered journaling

Journal **metadata only**, but with a strict rule: **write the data to its in-place location BEFORE writing the journal commit.**

Sequence:

1. Write D (data) to its final location.
2. Write `[TxB | I' | B' | TxE]` to journal.
3. Checkpoint metadata in place.

Why it works: if you crash after step 1 but before step 2, data is on disk but bitmap says the block is free and the inode doesn't reference it. You **lose D**, but D was new data, so losing it is acceptable. No security leak, no inconsistency.

**Exam trap (S23 Q42, SP22 Q18):** ordered = data first, then commit metadata. Not the reverse.

### Comparison table

| Mode | Journaled | Data write timing | Cost | Risk on crash |
|---|---|---|---|---|
| Data (full) | metadata + data | written twice | 2x I/O on data | none (just slow) |
| Writeback | metadata only | anytime | 1x data I/O | leak of stale/sensitive data |
| Ordered | metadata only | before commit | 1x data I/O | new data lost, no corruption |

---

## 7. Ordering and barriers (slide 41)

Total ordering of every write is too slow (defeats disk write cache parallelism). Instead, use **barriers** at three specific points:

1. **Before journal commit**: all journal transaction entries (TxB, metadata, data if full) must be on disk first. Otherwise commit could be persisted while the journaled blocks are still in cache, and replay would read garbage.
2. **Before checkpoint**: journal commit must be on disk. Otherwise a crash mid-checkpoint could leave the in-place blocks half-updated and no journal to replay from.
3. **Before freeing the journal entry**: all checkpoint writes must complete. Otherwise you could free a journal entry whose checkpoint never finished.

Memorize as:

```
journal entries  ▸ BARRIER ▸ commit  ▸ BARRIER ▸ checkpoint  ▸ BARRIER ▸ free
```

---

## 8. Recovery rules

After a crash, scan the journal:

| Journal state | Action | FS state |
|---|---|---|
| TxB present, no commit | Discard | Old (A) |
| TxB present, commit present | Replay all journaled blocks to in-place locations | New (B) |
| Replay partially done before second crash | Replay again (idempotent) | New (B) |
| No journal entry | Nothing to do | Whatever's on disk |

**Your F15 11-write timeline drops out of this rule:**

- Writes 1 to 4: header + 3 journaled blocks. Commit not yet on disk, no replay, old state.
- Write 5: commit lands, replay kicks in, new state.
- Writes 6 to 10: in-place writes happen; doesn't matter for recovery, replay handles missing ones.
- Write 11: free journal. State is new throughout 5 to 11.

**Your S23 Q55/Q56 pattern also drops out:**

- Journal `{5,2; A; B; commit=0}` → commit absent → block 5 keeps old contents.
- Journal `{4,6; C; F; commit=1}` → commit present → block 4 = C after replay.

---

## 9. Optimizations

### Write buffering (delayed checkpoint)
After commit, the journal already has a durable copy. No urgency to checkpoint. Delay it, batch many transactions, checkpoint together. Reduces random I/O.

Difficulty: journal space is finite. Solution: keep multiple uncheckpointed transactions live in the journal.

### Batched updates
If two ops both touch the same inode bitmap block, the in-memory bitmap is dirtied once and journaled once for both ops. Reduces redundant journaling.

### Circular log
Journal is a fixed-size circular buffer. Old transactions are overwritten once their checkpoints complete.

```
T1 T2 T3 T4 ...
0                       128 MB
```

---

## 10. Predicted exam questions

### T/F (high confidence)

1. Ordered journaling writes data to its final location before writing the commit block. → **T**
2. Data journaling writes data twice (once to journal, once in place). → **T**
3. Writeback journaling can leak stale data into a file after a crash. → **T**
4. FSCK can recover the in-progress operation if the FS is consistent. → **F** (consistency only)
5. Journaling guarantees atomicity; FSCK guarantees consistency. → **T**
6. After a crash with no commit block, the FS is in the new state. → **F** (old state)
7. Replay of a journaled transaction must be idempotent. → **T**
8. The commit block is the single atomic point that distinguishes "old" from "new" state. → **T**
9. Writeback journaling is always slower than data journaling. → **F** (writeback is faster)
10. Barriers are needed before journal commit, before checkpoint, and before freeing the journal entry. → **T**
11. Ordered journaling can leak sensitive data on crash. → **F** (only writeback can)
12. After commit lands, FS performance temporarily improves because checkpointing can be deferred. → **T**

### MC: which mode is this?

"Writes data first, then writes metadata plus commit to journal."

- a) Data journaling
- b) Writeback journaling
- c) **Ordered journaling** ← answer
- d) FSCK

### Multi-step recovery problem template

Given a journal entry `[TxB | I'=X | B'=Y | TxE=1]` and the in-place blocks are still old `I=W, B=Z`.

After recovery:

- in-place `I` = **X** (replay)
- in-place `B` = **Y** (replay)

Same problem with `TxE=0`:

- in-place `I` = **W** (no replay)
- in-place `B` = **Z** (no replay)

### F15-style timeline question

Data-journaled transaction with 3 metadata + 0 data + commit + 3 in-place writes, plus barriers.

| Writes completed | State after recovery |
|---|---|
| 1 (TxB) | old |
| 2 to 4 (metadata journaled) | old |
| 5 (TxE/commit) | **new** (replay kicks in) |
| 6 to 8 (in-place writes) | new |
| 9 (free journal entry) | new |

The flip happens exactly at the commit write.

---

## 11. Flashcards

| Front | Back |
|---|---|
| Define: consistency | FS data structures don't contradict each other. |
| Define: atomicity (persistence) | All-or-nothing across crashes; either old or new state, never a mix. |
| Why is FSCK insufficient? | Lands in *some* consistent state, not necessarily the correct old or new one. |
| Role of the commit block? | Single-sector write that atomically marks a transaction as durable. |
| Three journaling modes? | Data (full), ordered, writeback. |
| Data journaling cost? | 2x writes for everything. |
| Writeback journaling risk? | Leaked sensitive data: metadata committed but data never written. |
| Ordered journaling rule? | Data to in-place location before journal commit. |
| Three barrier points? | Before commit, before checkpoint, before free. |
| Replay rule? | Commit present: replay all journaled blocks idempotently. Commit absent: discard. |
| Crash, commit absent? | Old state. |
| Crash, commit present? | New state (after replay). |
| Why must replay be idempotent? | A second crash could interrupt replay itself; redo must be safe. |
| Define: checkpoint | Writing journaled data to in-place locations. |
| Why circular log? | Bounded journal space; old transactions overwritten after checkpoint. |
| Batched updates benefit? | Multiple ops touching same metadata block journal it once. |
| Delayed checkpoint benefit? | Batches random in-place writes; safe because journal is durable. |

---

## 12. Cheatsheet (rules and decisions)

```
Atomicity test          = is there a single atomic write that flips A to B?
                          For journaling, that's the commit block.

Recovery decision       = commit on disk?
                            Yes -> replay all journaled blocks
                            No  -> discard journal

Crash-state outcome     = state A until commit lands; state B after commit.

Mode -> journaled bytes = data:      metadata + data
                          ordered:   metadata only (data written first)
                          writeback: metadata only (data anytime)

Barrier rule            = entries -> commit -> checkpoint -> free
                          (barrier between every arrow)

Crash matrix (append)   = bitmap only      -> leak
                          data only        -> harmless
                          inode only       -> garbage
                          bitmap + data    -> leak
                          bitmap + inode   -> garbage
                          data + inode     -> reuse risk

FSCK vs Journal         = FSCK:    any consistent state, slow disk scan
                          Journal: exactly A or B, fast replay
```

---

## What this lecture does NOT cover (for completeness)

The slides at the end mention "Advanced journaling topics" and FSCK is referenced but not deeply expanded. If your exam includes:

- FSCK pass-by-pass details (superblock, free blocks, inode state, link counts, directory check)
- ext3/ext4 specific tunables
- NVRAM-based shortcuts

...those will need a different lecture's slides or the textbook chapter.
