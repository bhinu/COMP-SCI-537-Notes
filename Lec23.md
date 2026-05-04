# CS 537 Midterm 3: Lecture 23 — RAID

Companion to your existing midterm pattern summary. RAID is Tier 1 material; this lecture maps tightly to the patterns you already documented across SP22, F15, and S23 sample exams.

---

## Frame and notation (slides 2 to 10)

Five primitives drive every RAID question:

| Symbol | Meaning |
|---|---|
| N | number of disks |
| C | capacity of one disk |
| S | sequential throughput of one disk |
| R | random throughput of one disk |
| D | latency of one small I/O |

Three goals trade off: **capacity, reliability, performance**. Different RAID levels make different trade-offs. Mapping is either dynamic (table/tree) or static (math). Static wins for RAID 0/1/4/5 because it is O(1) lookup with no metadata.

---

## RAID-0 striping (slides 11 to 15)

**Mapping (chunk size 1, N disks):**
```
disk   = A % N
offset = A / N
```

For chunk size > 1, the mapping uses `(A / chunk) % N` for disk and `(A / chunk) / N * chunk + (A % chunk)` for offset. Worth practicing on a 4-disk, chunk-2 layout.

**Analysis:**
- Capacity: `N * C`
- Failures tolerated: **0**
- Latency (random small I/O): `D`
- Throughput: `N * S` sequential, `N * R` random

**Reliability gotcha:** stripes amplify failure rate. With per-disk failure rate `f`, the array's failure rate is `N * f` (any disk down kills the array). MTTF goes from 3 years (1 disk) to roughly 0.3 years on 10 disks. This is the opposite of what you naively want from "more disks."

---

## RAID-1 mirroring (slides 16 to 30)

**Layout:** 2 disks hold one copy each. With 4 disks you stripe across 2 mirrored pairs (RAID-1+0 in your summary).

**Analysis:**
- Capacity: `N/2 * C`
- Failures tolerated: **at least 1, at most N/2** (depending on which disks fail; if both halves of a mirror fail you lose data even with 2 of N down)
- Latency: `D`

**Throughput (slide 19):**
- Random read: `N * R` (any mirror serves)
- Random write: `(N/2) * R` (must hit both copies)
- Sequential read: `(N/2) * S` (in practice, since you must read both halves of each stripe; this is the lecture's pessimistic assumption)
- Sequential write: `(N/2) * S`

**Consistent-update problem (slides 20 to 29):** crash mid-write leaves the two mirrors disagreeing. After reboot you cannot tell which copy is correct. Hardware fix: NVRAM in the controller logs the write. Software RAID like Linux md does not have this and must accept the risk.

**Reliability:** mirrors only fail if the second disk fails during the repair window. With 1-day repair and 3-year MTTF per disk, MTTF of the pair is around 3000 years. This is why mirroring exists.

---

## RAID-4 parity disk (slides 31 to 45)

**Parity rule (slide 41):** parity is XOR across the stripe. `P = D0 ⊗ D1 ⊗ D2 ⊗ D3`. Recovery: missing block = XOR of all surviving blocks plus parity.

The handwritten annotation on slide 41 shows: if D1 changes from B to B', new parity = `A ⊗ B' ⊗ C ⊗ D`. This is **additive parity** (recompute from scratch).

**Additive vs subtractive parity (slide 45):** the optimization that matters for random writes.
- Additive: read all N-1 data blocks, XOR them with new value, write data + parity. Cost: N-1 reads + 2 writes.
- Subtractive: `P_new = D_old ⊗ D_new ⊗ P_old`. Cost: 2 reads (old data, old parity) + 2 writes (new data, new parity). Same cost on a 5-disk array, but subtractive cost stays at 4 ops regardless of N.

**Random write throughput on RAID-4:** `R / 2`. Two disks (the data disk and the parity disk) each serve one I/O per write request, so each random write costs 2 disk ops on the parity disk side. The parity disk is the bottleneck; even though there are N-1 data disks, the parity disk caps your write rate.

**Analysis:**
- Capacity: `(N-1) * C`
- Failures tolerated: **1**
- Latency: `D` for read, `2D` for random write (read + write parity)

**Throughput (slide 44):**
- Sequential read: `(N-1) * S`
- Sequential write: `(N-1) * S` (full-stripe writes avoid the read-modify-write penalty)
- Random read: `(N-1) * R`
- Random write: `R / 2`

---

## RAID-5 rotated parity (slides 46 to 50)

Same data layout as RAID-4, but parity rotates across disks per stripe. Eliminates the parity-disk hot spot for random writes.

**Random write throughput (slide 50):** `N * R / 4`. Each random write hits 2 disks (data + its parity, on different disks per stripe). With N disks, you get N total ops per second, divided by 4 ops per write (read data, read parity, write data, write parity) = `N * R / 4`.

**Analysis:**
- Capacity: `(N-1) * C`
- Failures tolerated: **1**
- Latency: `D` read, `2D` random write

**Throughput (slide 49):**
- Sequential read: `(N-1) * S`
- Sequential write: `(N-1) * S`
- Random read: `N * R` (parity blocks also serve reads, unlike RAID-4)
- Random write: `N * R / 4`

This is why your pattern summary notes "RAID-5 random read better than RAID-1+0 with same number of disks." Confirmed: RAID-5 gets `N*R` random reads, RAID-1+0 gets `N*R` too actually, but RAID-5 has more usable capacity. The summary's claim holds in the typical comparison where you fix data capacity and RAID-5 needs fewer disks.

---

## Master comparison table (slide 51)

| | Reliability | Capacity | Read lat | Write lat | Seq Read | Seq Write | Rand Read | Rand Write |
|---|---|---|---|---|---|---|---|---|
| RAID-0 | 0 | `N*C` | `D` | `D` | `N*S` | `N*S` | `N*R` | `N*R` |
| RAID-1 | 1 | `N/2*C` | `D` | `D` | `N/2*S` | `N/2*S` | `N*R` | `N/2*R` |
| RAID-4 | 1 | `(N-1)*C` | `D` | `2D` | `(N-1)*S` | `(N-1)*S` | `(N-1)*R` | `R/2` |
| RAID-5 | 1 | `(N-1)*C` | `D` | `2D` | `(N-1)*S` | `(N-1)*S` | `N*R` | `N*R/4` |

**Memorize the asymmetries:**
- RAID-1 random read is `N*R`, write is `N/2 * R` (2x gap)
- RAID-4 and RAID-5 random write differ massively: `R/2` vs `N*R/4`. That is the entire point of RAID-5.
- RAID-5 is the only level where random read uses all N disks (parity blocks serve reads too).

---

## Quick-recall flashcards

| Front | Back |
|---|---|
| RAID-0 random write throughput | `N * R` |
| RAID-1 random read vs random write | `N*R` read, `N/2 * R` write |
| RAID-4 random write bottleneck | parity disk, throughput `R/2` |
| RAID-5 random write throughput | `N * R / 4` |
| RAID-5 random read throughput | `N * R` (parity blocks also serve reads) |
| Subtractive parity formula | `P_new = D_old ⊗ D_new ⊗ P_old` |
| Why RAID-1 has consistent-update problem | crash mid-write leaves mirrors disagreeing; cannot tell which is correct |
| HW solution to consistent-update | NVRAM in RAID controller |
| RAID-0 capacity | `N * C` |
| RAID-1 capacity | `N/2 * C` |
| RAID-4/5 capacity | `(N-1) * C` |
| RAID-1 max failures tolerated | `N/2` (one per mirror pair); minimum 1 (if both halves of one pair fail) |
| Sequential write latency penalty in RAID-4? | None at full-stripe write, `2D` for partial |
| RAID-0 reliability with 10 disks at 3-yr MTTF | ~0.3 yr (3.5 months) MTTF |

---

## Predicted exam questions

### True/False candidates (high confidence based on past exam patterns)

1. **T/F:** RAID-5 random reads use all N disks, while RAID-4 random reads use only N-1. → **TRUE.** RAID-5 rotates parity, so every disk holds data blocks for some stripes.
2. **T/F:** RAID-1 sequential read throughput equals random read throughput. → **FALSE typically.** Sequential reads are pessimistically `N/2 * S` (must read both halves), random reads are `N * R` (any mirror serves).
3. **T/F:** Subtractive parity in RAID-5 requires reading all data blocks in a stripe before writing. → **FALSE.** That's additive parity. Subtractive only reads old data + old parity.
4. **T/F:** A RAID-5 array with one failed disk continues to serve reads with no performance penalty. → **FALSE.** Reads of blocks on the failed disk require reading all surviving blocks + parity to reconstruct.
5. **T/F:** A 5-disk RAID-5 array can tolerate simultaneous failure of any 2 disks. → **FALSE.** RAID-5 tolerates exactly 1 disk failure. RAID-6 tolerates 2.
6. **T/F:** RAID-0 with 10 disks has a higher MTTF than a single disk. → **FALSE.** It is 10x worse (any disk failing kills the array).

### Multiple choice candidates

**Q (RAID throughput):** A 6-disk RAID-5 array with per-disk random throughput R. What is the random write throughput?
- (a) `R/2`
- (b) `R`
- (c) `6R/4 = 1.5R` ✓
- (d) `6R`

**Q (RAID capacity):** A RAID-1+0 (mirrored stripes) array with 8 disks of 2TB each. What is usable capacity?
- (a) 16 TB
- (b) 14 TB
- (c) 8 TB ✓
- (d) 4 TB

### Multi-part question candidate (RAID layout)

Given a 5-disk RAID-5 array with chunk size 4 KB, left-symmetric layout, and parity rotating right-to-left:

```
Stripe 0: D0  D1  D2  D3  P
Stripe 1: D4  D5  D6  P   D7
Stripe 2: D8  D9  P   D10 D11
Stripe 3: D12 P   D13 D14 D15
Stripe 4: P   D16 D17 D18 D19
```

If you read logical block 11, which physical (disk, stripe) pair holds it?
- Block 11 is in stripe 2 (blocks 8 to 11), position 3 within the data blocks of that stripe.
- In stripe 2, parity is on disk 2; data blocks are on disks 0, 1, 3, 4 holding D8, D9, D10, D11 respectively.
- **Answer: disk 4, stripe 2.**

This is the kind of mapping question that appears on F15. Practice with several RAID levels.

---

## Formula sheet additions

```
Slide 14, RAID-0:    capacity = N*C, throughput = N*S sequential, N*R random
Slide 18-19, RAID-1: capacity = N/2*C
                     read = N*R random, write = N/2*R random
                     seq read = seq write = N/2*S
Slide 41, RAID-4 parity:    P = D0 ⊗ D1 ⊗ ... ⊗ D(N-2)
Slide 45, subtractive:      P_new = D_old ⊗ D_new ⊗ P_old
Slide 43-44, RAID-4:        capacity = (N-1)*C
                             rand write = R/2 (parity bottleneck)
                             rand read = (N-1)*R
                             seq read = seq write = (N-1)*S
Slide 50, RAID-5:           rand write = N*R/4 (subtractive parity, rotated)
                             rand read = N*R (parity blocks serve reads too)
Slide 51 master table:      memorize columns
```

---

## Where this fits in your study sequence

Slot into **Day 2 (RAID throughput)** of your existing 9-day plan: add the slide-anchored derivations for `R/2` and `N*R/4`. Practice the additive vs subtractive parity distinction.

The single biggest gap in your existing summary is **RAID-5 random write derivation as `N*R/4`**, which you noted but is worth practicing to derive on the fly. RAID will be 8 to 15 questions on the exam including throughput math, mapping, and reliability.
