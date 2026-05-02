# Lecture 16: I/O Devices, Hard Disks, and SSDs

**CS 537 - Spring 2026 (Persistence Module Start)**

---

## EXAM PRIORITY MAP (based on SP22, F15, S23 sample midterms)

> **Tier 1: drill heavily, multiple Qs across multiple exams**
> - Section 6 (HDD mechanics + RPM math): SP22 Q36, Q37; F15 Q1-Q4; S23 Q4-Q11
> - Section 7 (SSD ops, page vs block, cell types): S23 Q44; conceptual base for SSD Qs
> - Section 8 (FTL: direct map vs log): SP22 Q49, Q50; S23 Q44
>
> **Tier 2: know cold, lighter weight (1-2 Qs typical)**
> - Section 2 (Polling vs Interrupts): SP22 Q1; S23 Q3 (interrupt latency T/F gotcha)
> - Section 3 (PIO vs DMA): SP22 Q48 (DMA = device transfers, NOT CPU)
> - Section 5 (Drivers): SP22 Q2-Q3 (every device has SOME driver, but network driver doesn't talk to all NICs)
>
> **Tier 3: skim, conceptual familiarity only (rarely tested)**
> - Section 1 (Device model basics)
> - Section 4 (NVMe queue mechanism)
> - Section 9 (Throughput reference table)

---

## TL;DR (read first if cramming)

1. **OS talks to devices** via memory-mapped registers (Status, Command, Data). Loop: wait for ready -> write data -> write command -> wait for done.
2. **Polling vs Interrupts**: polling wastes CPU but is fast for quick devices; interrupts free CPU but have overhead. Hybrid is common.
3. **PIO vs DMA**: PIO has CPU shuffle data byte-by-byte; DMA lets device read RAM directly while CPU does other work.
4. **Hard disk time** = `seek + rotation + transfer`. Sequential is fast (no seek/rotate), random is brutal (3400x slower).
5. **SSD ops**: Read (fast, ~50us) << Program/write (~200-1400us) << Erase (~1.5-4.5ms). You **can't overwrite a page**, must erase the whole block first.
6. **FTL (Flash Translation Layer)**: maps logical pages to physical pages, uses log-based writes + garbage collection + wear leveling.

---

## 1. How OS Talks to I/O Devices  `[TIER 3]`

### Canonical Device Model
Every device exposes 3 registers to the OS:
- **Status**: is the device busy/ready/done?
- **Command**: what should it do?
- **Data**: what to read/write

Hidden internals (OS doesn't see these): microcontroller (CPU+RAM), extra RAM, special-purpose chips.

### Standard Write Protocol (memorize this)
```
while (STATUS == BUSY) ;       // 1. spin until ready
Write data to DATA register     // 2. transfer data
Write command to COMMAND reg    // 3. tell device to go
while (STATUS == BUSY) ;        // 4. spin until done
```

### Bus Hierarchy
```
CPU <-> RAM <-> Memory Bus
              |
              v
         General I/O Bus (PCI) -> Graphics
              |
              v
         Peripheral I/O Bus (SATA, USB, SCSI) -> Disks, etc.
```
Faster devices sit closer to CPU on faster buses.

---

## 2. Polling vs Interrupts  `[TIER 2]`

> **Sample exam hits:** SP22 Q1 (false: "interrupts have lower latency than polling" - it's the opposite), S23 Q3 (true: interrupts have higher latency for fast devices)

| | Polling | Interrupts |
|---|---|---|
| CPU usage | Wastes cycles spinning | Free to do other work |
| Latency | Low (sees done immediately) | Higher (interrupt overhead) |
| Best for | Fast devices | Slow devices |

### Trade-offs to know
- **Fast device** -> polling wins (interrupt overhead > device time)
- **Slow/unknown device** -> interrupts win OR hybrid (spin briefly, then sleep on interrupt)
- **Flood of interrupts** -> can cause **livelock** (CPU does nothing but handle interrupts). Fix: ignore interrupts while making progress, or **interrupt coalescing** (batch several into one).

### Sync vs Async events
- **Synchronous**: OS launches, device completes (read disk, send packet). Polling can work.
- **Asynchronous**: triggered externally (keystroke, packet arrives). Interrupts are needed since the OS doesn't know when to poll.

> **Gotcha to memorize**: "Interrupts always have lower latency than polling" is FALSE. Interrupts have lower CPU cost; polling has lower latency for fast devices.

---

## 3. PIO vs DMA  `[TIER 2]`

> **Sample exam hits:** SP22 Q48 (false: "with DMA the CPU transfers data" - DMA means the device does it)

### PIO (Programmed I/O)
CPU directly copies data byte-by-byte between RAM and device register. Wastes CPU on data movement.

### DMA (Direct Memory Access)
- CPU just gives device the **memory address** of the data.
- Device reads/writes RAM **on its own** while CPU does other work.
- Device interrupts when done.

> **Single-sentence answer for exam:** With DMA, the **device** transfers data, NOT the CPU. This is the entire point.

---

## 4. Batching: I/O Request Queues (NVMe-style)  `[TIER 3]`

For multiple outstanding requests:
- **Submission Queue (SQ)** in host memory: driver puts commands here.
- **Completion Queue (CQ)** in host memory: device puts results here.
- **Doorbells** (device registers): SQ doorbell tells device "new work added"; CQ doorbell tells driver "new completion".
- Both queues are **circular**.

This enables many in-flight requests with minimal per-request overhead.

> Not seen in any sample exam. Skim only.

---

## 5. Device Drivers  `[TIER 3]`

> **Sample exam hits:** SP22 Q2 (true: every device has a driver), Q3 (false: a network driver does NOT talk to all NICs - each NIC needs its own driver).

### Why drivers exist
Tons of devices, each with its own protocol. Solution:
- OS defines a **class interface** (network, block, sound).
- Each **driver** implements that class interface for its specific device.
- Driver talks to device using device-specific protocol.

### Some devices share interfaces
- Storage: USB storage, SATA, NVMe (block class)
- Keyboard, mouse

**Fun fact**: drivers are ~70% of Linux source code. Drivers must be written separately for Linux/Windows/macOS.

---

## 6. Hard Disks (HDDs)  `[TIER 1 - DRILL HEAVILY]`

> **Sample exam hits**:
> - SP22 Q36 (6000 RPM -> avg rotation 5 ms), Q37 (8 MB track @ 6000 RPM -> 800 MB/s)
> - F15 Q1-Q4 (geometry: heads, capacity, cylinder size, sector size)
> - S23 Q4 (RPM vs diameter effect on sequential), Q5 (smaller diameter = shorter seek), Q6 (3000 RPM -> 10 ms avg), Q7 (random vs sequential), Q11 (disk scheduling fairness)
> - Implicit: this section underlies every disk question

### Anatomy
- **Platter**: round disk coated with magnetic film. Stores data.
- **Track**: one concentric ring on a platter.
- **Sector**: a slice of a track. Typically **4096 bytes**. This is the minimum read/write unit.
- **Head**: reads/writes data, mounted on a moving **arm**.
- **Spindle**: spins the platters.

Disk address space = array of sectors. Operation = read/write N sectors starting at sector X.

### The 3 components of disk access time

> **Time = Seek + Rotation + Transfer**

| Component | Typical | What it is |
|---|---|---|
| **Seek time** | 4-9 ms | Move arm to correct track |
| **Rotational delay** | 4-8 ms (avg) | Wait for sector to rotate under head |
| **Transfer time** | 5 ms/MB (= 0.005 ms/KB) | Actually read the data |

### Performance implications (LIKELY EXAM QUESTION)
- Head in right place: read 1KB takes **0.005 ms**
- Head in wrong place: read 1KB takes **8-17 ms** (~3400x slower)
- **Lesson**: sequential access on a track = no seek + no rotate = fast. **Random access = brutal**.

### Rotational delay formulas (memorize)
```
Worst case rotation = full revolution = 60 / RPM seconds = 60000 / RPM ms
Average rotation    = half revolution = 60 / (2*RPM) sec = 30000 / RPM ms
Transfer rate       = track_size_MB * RPM / 60   MB/sec
```

Drill table (these are the values actually tested):
| RPM | Avg rotation | Worst case |
|---|---|---|
| 3000 | 10 ms | 20 ms |
| 6000 | 5 ms | 10 ms |
| 7200 | 4.16 ms | 8.33 ms |
| 10000 | 3 ms | 6 ms |
| 15000 | 2 ms | 4 ms |

### Disk geometry math (F15 Q1-Q4 pattern)
```
Heads          = surfaces (one head per surface)
Cylinder size  = bytes_per_track * surfaces
Total capacity = surfaces * tracks_per_surface * bytes_per_track
Sector size    = bytes_per_track / sectors_per_track
```

> **Note from pattern summary**: disk scheduling (FCFS, SSTF, SCAN, C-SCAN, SPTF) is also Tier 1 but is covered in the disk scheduling lecture, not here. Don't forget to drill it separately.

---

## 7. Solid-State Drives (SSDs)  `[TIER 1 - DRILL HEAVILY]`

> **Sample exam hits**:
> - SP22 Q49 (direct map @ 500 erases/sec -> ~500 random writes/sec)
> - SP22 Q50 (log @ 10000 page writes/sec -> 10000 random writes/sec)
> - S23 Q44 (same workload comparison)
> - Pattern summary calls this Tier 1

### Why SSDs exist
HDDs are mechanical and slow. SSDs use **NAND flash** (transistor-based, no moving parts). Persistent unlike DRAM.

### Cell types (density vs reliability tradeoff)
| Type | Bits/cell | Speed | Reliability | Density |
|---|---|---|---|---|
| SLC (Single) | 1 | Fastest | Most reliable | Lowest |
| MLC (Multi) | 2 | Slower | Less reliable | Higher |
| TLC (Triple) | 3 | Even slower | Even less | Even higher |
| QLC (Quad) | 4 | Slowest | Worst | Highest |

Levels are distinct **voltage levels** within one cell. More levels = harder to distinguish = less reliable.

### Three flash operations (memorize cost order!)
| Operation | Cost | What it does |
|---|---|---|
| **Read** a page | 25-75 us | Get contents of a page |
| **Program** (write) a page | 200-1400 us | Change selected 1s -> 0s |
| **Erase** a block | 1.5-4.5 ms | Reset all bits in block to 1 |

> **Critical asymmetry**: reads/writes are at **page** granularity. Erase is at **block** granularity. To write a single page, you must first erase the **entire** block.

### Pages and Blocks
- **Page** = a few KB (e.g., 4 KB). Smallest read/write unit.
- **Block** = many pages (e.g., 128 KB or 256 KB). Smallest erase unit.

> **Trap question**: "Read, Program, and Erase all operate on ~4 KB segments." FALSE. Only Read and Program are page-level. Erase is block-level.

### Wear-out
A block fails after ~1000 erases (slowest/highest density QLC) up to ~100,000 erases (fastest/lowest density SLC).

### Striping
Page addresses are striped across multiple flash chips (like RAID 0). Single request can hit multiple chips in parallel = natural load balancing.

### SSD Structure
```
[Host Interface] <-> [DRAM] <-> [Flash Controller] <-> [NAND Flash chips]
                                       ^
                              [Embedded processor + SRAM]
                              (runs the FTL firmware)
```

---

## 8. Flash Translation Layer (FTL)  `[TIER 1 - HIGHEST YIELD]`

> **Sample exam hits**:
> - SP22 Q49: direct-map random write throughput = erases per second
> - SP22 Q50: log-based random write throughput = page writes per second (~10000)
> - S23 Q44: same pattern
> - Pattern summary explicitly calls this out as Tier 1

The FTL is firmware inside the SSD. It exists because flash can't do in-place writes.

### FTL Goals
1. **Translate** logical block reads/writes -> physical page reads/erases/programs. Lets SSDs export the same simple block interface as HDDs.
2. **Reduce write amplification** (the extra copying caused by erase-before-write).
3. **Wear leveling**: spread writes evenly across all blocks so no single block dies first.

### Approach #1: Direct Mapping (BAD)
Logical page N maps to physical page N.

To write page N:
1. Read entire physical block into memory
2. Modify the relevant page in memory
3. Erase the entire block
4. Program the entire block back

**Problems**:
- **Write amplification**: writing 1 page = read+erase+write whole block. Slow.
- **Poor reliability**: hot logical blocks pound their physical block to death.
- **Data loss risk**: if power fails between erase and rewrite, you lose the whole block.

> **Exam math**: random write throughput on direct-map SSD = erases per second. If the SSD does 500 erases/sec, you get 500 random writes/sec. If 1000 erases/sec, you get 1000.

### Approach #2: Log-Based Mapping (GOOD - this is what real SSDs do)

Treat physical pages like an **append-only log**:
- Every write goes to the **end of the log** (the next free page).
- An in-memory **logical-to-physical map** tracks where each logical page lives.
- Old versions of overwritten pages become **garbage**.

#### Trace through example
Initial state: all blocks erased.

```
write(page=92, data=w0):
  -> erase block 0 (if needed), program physical page 0 with w0
  -> map: 92 -> 0
  -> log head moves to page 1

write(page=17, data=w1):
  -> program physical page 1 with w1
  -> map: 92 -> 0, 17 -> 1
  -> log head moves to page 2

... continue writing 33 -> 2, 68 -> 3 ...

write(page=92, data=w4):  // overwrite of page 92!
  -> program physical page 4 with w4 (NOT page 0)
  -> map updates: 92 -> 4
  -> physical page 0 is now GARBAGE
```

#### Advantages over direct mapping
- No expensive read-modify-write per write.
- **Natural wear leveling**: writes spread across pages even if logical writes have spatial locality.
- Better reliability.

> **Exam math**: random write throughput on log-based SSD = page program rate. If pages program at 100 us each, that's ~10000 page writes/sec.

### Garbage Collection (GC)
Eventually old pages take up too much space and the log fills up. GC reclaims space:
1. Pick a block with mostly garbage pages.
2. Read the still-valid pages from that block.
3. Write those valid pages to the **end of the log**.
4. Update the logical-to-physical map for moved pages.
5. Erase the now-empty block. It's free for reuse.

GC adds extra read+write traffic = **write amplification**.

### Overprovisioning
SSD exposes a **smaller logical address space** than the physical one. Hidden extra pages give GC room to work in the background, off the critical path of user writes.

### Wear Leveling for cold data
Even live (non-garbage) data that's never overwritten gets shuffled occasionally so that "cold" blocks get used too. Otherwise hot blocks would die while cold ones sit unused.

---

## 9. SSD vs HDD Throughput (from slides)  `[TIER 3 - reference]`

| Device | Random Read | Random Write | Seq Read | Seq Write |
|---|---|---|---|---|
| Samsung 960 Evo Plus SSD | 85.5 MB/s | 244 MB/s | 2664 MB/s | 2508 MB/s |
| Crucial BX100 SSD | 24.5 | 73.5 | 466 | 392 |
| Samsung 840 EVO SSD | 44.1 | 112 | 502 | 494 |
| Seagate Savio 15K.3 HDD | **2** | **2** | 223 | 223 |

**Takeaways**:
- HDDs choke on random I/O (~2 MB/s random vs 223 MB/s sequential).
- SSDs are much better at random I/O than HDDs.
- HDDs are ~**10x cheaper per bit**.
- For SSDs, sequential and random write throughput are closer because of the FTL's log-structured writes (random writes appear sequential to the flash).

---

## 10. Quiz Answers (from slide 48)

1. **"Interrupts generally provide greater CPU efficiency than polling"** -> TRUE (for slow devices; CPU can do other work).
2. **5" disk vs 2.5" disk - which has faster seek time?** -> The **2.5" disk**. Smaller platter = shorter distance for arm to travel.
3. **Disk at 3000 RPM (50 RPS), worst-case rotational delay?** -> **20 ms** (= 1/50 sec = full revolution).
4. **"With DMA, the main CPU transfers data from RAM to peripheral device"** -> **FALSE**. The whole point of DMA is the device transfers data without the CPU.

---

## 11. What to Drill (ranked by sample exam frequency)

1. **HDD time formula and RPM math** (Tier 1). Be able to compute average rotational delay from RPM, transfer time from track size + RPM, and total access time given seek + rotation + transfer in seconds. SP22 and S23 both hit this directly.
2. **SSD direct map vs log random write throughput** (Tier 1). Direct = erases/sec, Log = page writes/sec. SP22 Q49-Q50 nailed this exactly.
3. **Page vs block granularity** (Tier 1). Read/program = page. Erase = block. This T/F shows up as a trap.
4. **Erase >> Program > Read cost order** (Tier 1).
5. **DMA: device transfers, not CPU** (Tier 2). One-line gotcha.
6. **Polling latency vs interrupt latency** (Tier 2). Interrupts have HIGHER latency for fast devices, not lower.
7. **HDD geometry math** (Tier 2). F15 hit this hard, SP22/S23 less so.

## 12. What to Skim (low yield)

- NVMe SQ/CQ doorbell mechanics. Not seen in any sample.
- Driver class interface details. SP22 had two T/F that just need the gist.
- Throughput reference numbers. Just know SSD random vastly beats HDD random.

---

## 13. One-Page Cheat Sheet (for Midterm 3 sheet)

```
HDD: time = seek (4-9ms) + rotation (4-8ms) + transfer (0.005ms/KB)
     worst rotation = 60000/RPM ms, avg = 30000/RPM ms
     transfer rate = track_size_MB * RPM / 60 MB/s
     random ~3400x slower than sequential

SSD ops: read 25-75us | program 200-1400us | erase 1.5-4.5ms
Page = R/W unit (~4KB). Block = erase unit (~128-256KB).
Erase sets all bits to 1. Program clears bits to 0.

FTL goals: translate, reduce write amp, wear level
Direct map: random writes = erases/sec (BAD: ~500/sec, write amp, hot blocks die)
Log-based:  random writes = page writes/sec (GOOD: ~10000/sec)
GC: read live pages, rewrite to log, erase old block
Overprovisioning: hidden pages let GC work in background

Polling vs Interrupts: fast device -> polling, slow -> interrupts
Livelock: too many interrupts, no progress. Fix: drop+coalesce.
PIO: CPU moves data. DMA: device moves data (CPU free).
SQ/CQ doorbells = batched I/O queue mechanism.
```
