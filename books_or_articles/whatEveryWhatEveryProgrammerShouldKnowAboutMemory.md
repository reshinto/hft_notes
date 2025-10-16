# What Every Programmer Should Know About Memory

## Table of Contents
- [Abstract](#abstract)
- [1 Introduction](#1-introduction)
- [2 Commodity Hardware Today](#2-commodity-hardware-today)
- [3 CPU Caches](#3-cpu-caches)
- [4 Virtual Memory](#4-virtual-memory)
- [5 NUMA Support](#5-numa-support)
- [6 What Programmers Can Do](#6-what-programmers-can-do)

## Abstract
- Modern CPUs keep getting **faster** and add **more cores**, but most programs are limited by how **quickly data can be fetched from memory**.
- Hardware helps with techniques like **CPU caches**, yet these mechanisms don’t deliver peak performance **without cache-friendly code**.
- Many programmers don’t fully understand **how memory subsystems are built or what they cost in latency/bandwidth**, so this paper explains **why caches exist, how they work,** and **what coding practices** make programs run optimally on **commodity hardware**.

**Key Points**
- **Memory, not compute, is the bottleneck.**  
  CPU speeds/cores have outpaced DRAM latency improvements; software often stalls waiting for data.
- **Caches are essential but need cooperation.**  
  Caches only shine when programs exhibit **locality** (access nearby data and reuse it soon).
- **Programmer knowledge gap.**  
  Misunderstanding of **hierarchy, timing, and costs** (latency, bandwidth, contention) leads to slow code.
- **Paper’s promise.**  
  Teaches **structure of modern memory subsystems**, **cache design/behavior**, and **practical techniques** to exploit them.

**Quick Checklist for Cache-Friendly Code**
- [ ] Process data in **contiguous chunks** (iterate in memory order).
- [ ] **Reuse** recently loaded data before evicting it.
- [ ] **Batch** operations to reduce cache thrash and misses.
- [ ] Choose **data layouts** that match access patterns (SoA vs AoS).
- [ ] Measure with **profilers** to confirm cache misses/stalls are reduced.

## 1 Introduction
- Early computers were **balanced**: CPU, memory, storage, and networking improved together.
- Once subsystems were optimized independently, **memory and storage lagged**, creating bottlenecks. OSs hid many **storage** delays with software caching, but **main memory** bottlenecks require **hardware** solutions: faster/parallel **RAM**, better **memory controllers**, ****, and **Direct memory access (DMA)**.
- This paper (focused on **commodity hardware** and **Linux**) explains how modern memory subsystems and caches work and what programmers should do to use them effectively—while noting that real machines vary, so statements are often qualified as “usually.”

**Key Points**
1. **From balance to bottlenecks**  
   Subsystem-specific optimization left **memory & storage** improving more slowly than CPUs → system performance became **memory-bound**.

2. **Storage vs memory mitigation**  
   - **Storage:** Largely masked by **OS page cache** and device caches (software-centric).  
   - **Memory:** Needs **hardware** changes (RAM design/parallelism, controllers, caches, DMA).

3. **Focus of the paper**  
   Practical guidance on **** and **memory controllers**, with **DMA** folded into the big picture, using **commodity hardware** examples.

4. **Scope limits**  
   - **Platform:** **Linux-only** for OS specifics.  
   - **Coverage:** Not exhaustive; explains enough detail for performance goals and points to deeper references.

5. **Expect qualifiers**  
   Real hardware has many variants; the paper uses **“usually”** because **absolute rules are rare**.

**What to Watch For in Later Sections**
- How **cache hierarchies** interact with **access patterns**.  
- How **controller timing** and **parallel channels** shape throughput.  
- Programmer tactics to **reduce misses**, **improve locality**, and **avoid bandwidth contention** (incl. **DMA**).

### Document Structure
- The paper targets **software developers**.
- Before giving coding advice, it lays **groundwork**: RAM basics, **CPU cache** behavior, **virtual memory**, and **NUMA**.
- The **central section (6)** synthesizes these pieces into **practical performance guidelines**.
- Tooling (Section 7) helps find real bottlenecks, and Section 8 surveys **near-future** memory/caching technology.

**Section-by-Section Overview**
1. **Section 2 — RAM (technical background)**  
   Deep dive into **DRAM/SRAM** and access timing. *Nice to know; can be skipped initially.* The paper adds **back-references** where needed.

2. **Section 3 —  (essential)**  
   Detailed **cache behavior** with graphs (lines, sets, associativity, misses, prefetchers). *Core to understanding performance.*

3. **Section 4 — Virtual Memory (essential)**  
   How **address translation**, **pages/ Translation Look-Aside Buffer (TLB)s**, and protection work; prerequisite for later topics.

4. **Section 5 — NUMA (detailed)**  
   How **non-uniform memory latency** arises and what it means for **thread/data placement**.

5. **Section 6 — Practical Advice (central chapter)**  
   Brings Sections 2–5 together into **concrete coding guidelines** for different hardware scenarios.

6. **Section 7 — Tools**  
   Profilers and measurement tools to **locate cache/memory bottlenecks** in real codebases.

7. **Section 8 — Outlook**  
   Short overview of **emerging / desirable technologies** on the horizon.

**Suggested Reading Order (Developer-friendly)**
1 → **6 →** 3 → 4 → 5 → **(dip into 2 as needed)** → 7 → 8

[back to top](#table-of-contents)

## 2 Commodity Hardware Today
- **Scale out > scale up.** Cheap, fast networks make **many commodity machines** more cost-effective than a few specialized boxes.  
- **Typical data-center box (per text, 2007):** up to **4 sockets × quad-core with Hyper-Threading**, stated as **“up to 64 virtual processors.”**  
  - *Note:* 4×4×2 = **32** logical CPUs by simple multiplication; the excerpt still quotes **64**. I cannot confirm the discrepancy.
- **PC/server chipset model:** **Northbridge/Southbridge.** CPUs connect via **FSB** to Northbridge (**memory controller** lives here in classic designs); Southbridge hosts **I/O buses** (PCI/PCI-E, SATA, USB, etc.).  
- **Immediate consequences:** everything **to RAM** and **CPU↔CPU** traffic goes **through Northbridge**; **RAM is single-ported**; device I/O to RAM transits **Northbridge** → **contention**.
- **Bottlenecks & fixes:**  
  - **DMA** offloads CPU but **competes** for Northbridge bandwidth.  
  - **Channels** (dual-channel DDR2, etc.) increase RAM bandwidth; **access patterns** and **concurrency** strongly affect delays.  
  - Alternatives: **external memory controllers** (more buses) or **integrated memory controllers (IMC)** inside CPUs → **local memory per CPU** (NUMA).

1. Why commodity hardware matters
   - Specialized hardware is shrinking in relevance; most scale is **horizontal** with commodity parts due to **cheap, fast networking**.  
   - Optimizations in this paper target the **commodity “sweet spot.”**

2. Classic Northbridge/Southbridge design (Fig. 2.1)
   - **FSB → Northbridge (memory controller)** → RAM  
   - **Northbridge ↔ Southbridge** → PCI/PCI-E/SATA/USB/…  
   - **Implications:**  
     - **CPU↔CPU** messages and **CPU↔RAM** share the **FSB/Northbridge**.  
     - **Single-ported RAM** limits parallelism.  
     - **Device DMA** reduces CPU work yet **contends** with CPUs for **Northbridge/RAM** bandwidth.

3. Bandwidth & access patterns
   - Old systems: **single memory bus** (no parallel RAM access).  
   - Newer: **dual or more channels**; Northbridge **interleaves** across channels.  
   - **Concurrency** (hyper-threads/cores/DMA) increases wait times; **access pattern** (sequential vs scattered) strongly affects effective bandwidth.

4. Northbridge + external memory controllers (Fig. 2.2)
   - Attaching **multiple external controllers** ⇒ **multiple memory buses**, **more total bandwidth/capacity**.  
   - Limiting factor shifts to **Northbridge’s internal bandwidth**.

5. Integrated memory controllers (IMC) and NUMA (Fig. 2.3)
   - **Per-CPU memory controllers** with **local DIMMs** ⇒ aggregate bandwidth scales with CPU count.  
   - **Trade-off:** memory becomes **NUMA**:  
     - **Local** memory = fast; **remote** memory = extra **interconnect hops** (higher latency, “NUMA factors”).  
     - Complex topologies can have **multiple hop levels**; some systems group CPUs into **nodes** with cheaper intra-node access.

6. Programmer takeaways (from this section)
   - Expect **contention** on classic NB/SB designs, especially under **DMA** load.  
   - On **multi-channel** memory, **layout & stride** matter—sequential, channel-friendly patterns perform better.  
   - On **NUMA**, performance depends on **placing threads near their data**; recognize when you’re on a NUMA box and plan accordingly.

---

### 2.1 RAM Types
- The paper narrows focus to **modern RAM** and, in particular, why machines use both **SRAM** and **DRAM**.
- Although **SRAM** offers the **same functionality** as DRAM and is **much faster**, systems don’t use it for all memory because **cost to produce** and **cost to operate** (e.g., power/area) are significantly higher.
- To explain the performance-visible differences, the paper first sketches **bit-cell implementations** at the **logic-signal level**—enough detail for software developers, without deep hardware design minutiae.

**Key Points**
- **Scope:** Ignore legacy RAM; focus on **current types** whose behavior affects **kernel/app performance**.  
- **Why two kinds in one machine?**  
  - **SRAM:** faster, functionally equivalent; **too costly** (manufacturing + usage/power/area).  
  - **DRAM:** slower but **far denser and cheaper**, so it’s used for **main memory**.  
- **Developer-oriented depth:** The section uses **logic-level signals** (not circuit-design depth) to connect **hardware behavior → software-visible costs** (latency, timing, refresh, etc.).  
- **What comes next:** A minimal, targeted look at **SRAM vs DRAM bit cells** to ground later performance guidance.

#### 2.1.1 Static RAM (SRAM)
- A **6T SRAM cell** stores one bit using **two cross-coupled inverters** (transistors **M1–M4**) that form a stable latch (0 or 1) as long as **Vdd** power is present.
- A **word line (WL)** enables access:
   - on read, the bit appears **immediately** on the complementary **bit lines (BL/BL̄)**;
   - on write, external drivers first set **BL/BL̄** to the new value and then raise **WL** to overwrite the cell (the external drivers are stronger than the latch).
- **No refresh is required**, but each bit consumes **six transistors** and **constant power**, which makes SRAM fast but area- and power-expensive.

**Key Points**
- **Latch architecture (M1–M4):**  
  Two cross-coupled inverters create **bistability** (holds 0 or 1) as long as **Vdd** is applied → inherent **data stability** without refresh.

- **Access control (WL + BL/BL̄):**  
  - **Read:** Raise **WL** → cell’s state is driven **immediately** onto **BL/BL̄** (signals are sharp/rectangular like normal CMOS gates).  
  - **Write:** Drive **BL/BL̄** with desired value, then raise **WL** → stronger external drivers **flip** the latch.

- **Performance & power profile:**  
  - **Very fast reads** (no sensing/restore step like DRAM).  
  - **No refresh**, but **continuous power** is needed to maintain the state.

- **Cost/area:**  
  **6 transistors per bit** (4T variants exist but with disadvantages) → **large area** and **higher cost** vs DRAM; hence SRAM is used for **caches**, not bulk main memory.

- **Not in scope:**  
  Slower, lower-power SRAM variants exist with simpler interfaces than DRAM, but they’re **not relevant** here because this section focuses on **fast RAM**.

**Practical Implications (for programmers)**
- **Caches use SRAM** to deliver **very low access latency** without refresh overhead.  
- Expect caches to be **small and power-constrained** (6T cost), reinforcing the need for **cache-friendly access patterns** in software.


#### 2.1.2 Dynamic RAM (DRAM)
- A **DRAM cell** stores one bit using **one transistor (M)** and **one capacitor (C)**.
- Raising the **access line (AL)** connects the cell to the **data line (DL)**:
   - on **read**, the sense amplifier decides 0/1 from the capacitor’s charge;
   - on **write**, DL is driven and AL is held long enough to **charge or drain** the capacitor.
- Because the cell’s tiny charge **leaks**, DRAM must be **periodically refreshed (≈ every 64 ms)** and **reads are destructive**, requiring a restore.
- Charging/discharging follow **RC time constants**, so signals aren’t instantaneous—this **latency** and **refresh interference** make DRAM **much slower** than SRAM, but **far denser and cheaper**, which is why main memory uses DRAM.

**Key Points**
- **Structure (1T1C):** One **transistor + capacitor** ⇒ **very small area** per bit.  
- **Read:** **AL↑** connects the cell; depending on stored charge, current may or may not flow on **DL**. A **sense amplifier** is required to decide 0/1 across a range of partial charges.  
- **Write:** Drive **DL** to the desired value, then **AL↑** for long enough to **charge/drain** the capacitor.  
- **Destructive read:** Reading **depletes** the capacitor; the sense amp’s output is **fed back** to **restore** the charge (extra **time** and **energy**).  
- **Leakage ⇒ Refresh:** Tiny capacitors (femto-farad range or lower) **lose charge** and must be **refreshed ~every 64 ms**. During a refresh, **no access** to that row is possible; for some workloads, this overhead can **stall up to ~50%** of memory accesses (as reported in the cited study within the paper).  
- **RC-limited speed:** Charging/discharging follow
  Signals are **not rectangular**; the cell needs **time** before data is usable, which **severely limits DRAM speed** compared to SRAM.  
- **Why DRAM wins for main memory:** **Cost and density** outweigh speed: DRAM’s simple, regular structure **packs many more cells per die** than SRAM, despite the latency/refresh downsides.

**Programmer Implications**
- Expect **higher base latency** than SRAM due to **RC + sensing + restore + refresh**.  
- Performance is sensitive to **when** you access (refresh windows) and **how** you access (patterns that align with later DRAM protocol details).

#### 2.1.3 DRAM Access
- Programs issue **virtual addresses** that the CPU translates to **physical addresses**;
   - the **memory controller** then selects a DRAM chip and, within it, a **row** and **column**.
- To avoid impossible pin counts and huge demultiplexers
   - e.g., naïvely addressing **4 GB** would need **2³²** lines; a **1 Gbit** chip ~**30** address lines
   - DRAM arrays are organized as **rows × columns** and addressed in **two phases**: **RAS** (row address strobe) then **CAS** (column address strobe).
- Data appears on the bus only after specified **timing delays** because cells are tiny capacitors whose signals need amplification and time; writes also require a hold time.
- To further cut pin counts, DRAM **multiplexes** the address: the controller sends **row** bits first, then **column** bits on the *same* pins, trading pins for protocol complexity and timing constraints.

**Key Points**

1. Addressing without exploding pins
   - Directly selecting every cell from the controller would require **one select line per cell** (impractical).  
   - Example scales in the text:  
     - **4 GB** addressed directly ⇒ **2³²** lines from controller (impractical).  
     - **1 Gbit** chip ⇒ **~30** address lines and **2³⁰** selects → demultiplexer becomes huge/slow/area-heavy.

2. Array organization + RAS/CAS
   - Cells arranged in **rows × columns**.  
   - **RAS:** Row bits (e.g., `a0–a1` in the example) go through a **row demux** to latch a whole row.  
   - **CAS:** Column bits (e.g., `a2–a3`) select **one column** via a multiplexer; the selected bit(s) appear on the **data pin(s)**.  
   - Many chips operate **in parallel** to deliver a bus-width chunk (e.g., 64 bits).

3. Read/write flow (timing matters)
   - **Read:** Controller sends row, then column; **data becomes valid only after specified delays** because cell signals are **weak** (need **sense amps**) and capacitors don’t discharge instantly.  
   - **Write:** Controller places the new value on the **data bus**, selects row/column; the cell requires a **minimum data-valid window** to **charge/drain** its capacitor.

4. Why address multiplexing?
   - **Pins are precious.** Exporting ~30 address pins per DIMM *and* replicating for multiple modules would explode controller pin counts.  
   - **Multiplexing halves external address pins:** send **row** first (stays active), then **column** on the **same lines**, plus control lines indicating RAS/CAS phases.  
   - **Trade-off:** Fewer pins, but **extra protocol steps and timing constraints** that affect performance.

**Quick Numbers**
- **4 GB direct addressing** ⇒ **2³²** address lines (impractical).  
- **1 Gbit** chip ⇒ **~30** address lines, **2³⁰** selects → demux area/timing becomes prohibitive.

**Practical Implications (for programmers)**
- Expect **non-zero fixed latency** between issuing a request and data becoming valid (row/column selection + sensing).  
- **Access patterns** that align with rows (and later, pages/bursts) reduce overhead; random jumps incur repeated RAS/CAS/timing costs.  
- Controller/DIMM **timing parameters** (discussed next in §2.2) directly shape **sustained bandwidth** vs **burst peaks**.

#### 2.1.4 Conclusions
- Not all memory can be **SRAM** because of **cost/power/area**.
- **DRAM** cells must be **individually selected** and accessed through limited **address lines**, whose count drives the **cost/complexity** of controllers, boards, modules, and chips.
- **Reads/writes aren’t instantaneous**—DRAM physics and protocol introduce **non-zero latency**.
- Practical upshot: SRAM is used in **** (small, fast, directly addressed), while **DRAM** is used for **main memory**.
- SRAM speed depends on design effort and can range from **near-core speed** to **1–2 orders of magnitude slower** than the CPU core.

**Key Points**
- **Why not all SRAM?**  
  **Cost & power/area** make SRAM impractical for bulk memory; DRAM wins on **density**. *(§2.1.4)*

- **Cells are individually addressed.**  
  DRAM needs **row/column selection**; this addressing machinery is fundamental to how memory is used. *(§2.1.4)*

- **Pins/lines drive cost.**  
  The **number of address lines** directly impacts the cost/complexity of the **memory controller**, **motherboard routing**, **DRAM module**, and **chip**. *(§2.1.4)*

- **Latency is inherent.**  
  It **takes time** before read/write results are valid due to DRAM cell physics and protocol timing. *(§2.1.4)*

- **Where SRAM is used (and how fast).**  
  **/on-die SRAM** are **directly addressed** and kept small; **SRAM max speed** depends on design effort and can be **slightly slower than the core** up to **10–100× slower**. *(§2.1.4)*

**Practical Implications (for developers)**
- Expect **non-zero DRAM latency** even for simple accesses.  
- Design for **cache efficiency** (since SRAM is limited and precious).  
- Remember **addressing costs** exist beyond just “bytes/sec”: layout and access patterns interact with controller/addressing realities.

---

### 2.2 DRAM Access Technical Details
- Modern **SDRAM/DDR** operates **synchronously** to a controller-provided clock.
- Vendors often quote an **“effective” frequency** (double/quad-pumped transfers per cycle) that inflates the raw clock.
- Each transfer moves **64 bits (8 B)**, so the **peak (burst) bandwidth** equals `8 B × effective bus frequency`.
- Real programs rarely reach this peak because the **DRAM protocol** has **downtime** (row/column selection, latencies, precharge/refresh), which you must understand and minimize to get good sustained throughput.

**Key Points**
- **Synchronous operation:** SDRAM/DDR is timed to a clock; **DDR** transfers data **on both edges** (and later generations add further pumping), raising *effective* rate without raising the array frequency.
- **Marketing vs physics:** A “quad-pumped **200 MHz** bus” is marketed as **800 MHz effective**; the cells do **not** run at 800 MHz—only the transfer scheme appears that fast.
- **Peak bandwidth formula:**  
  `8 Bytes × effective bus frequency`
  Example from the text: **8 B × 800 MHz = 6.4 GB/s** (burst).
- **Burst ≠ sustained:** Protocol gaps (address multiplexing, activation, CAS latency, precharge, refresh) create idle periods where the bus **cannot** transmit data, cutting sustained bandwidth.

#### 2.2.1 Read Access Protocol
- A DRAM **read** proceeds in timed phases: the controller places a **row address** and asserts **RAS**, waits **tRCD** cycles, then places the **column address** and asserts **CAS**. After **CAS Latency (CL)** cycles, data appears on the bus in a **burst** (e.g., 2/4/8 words).
- Controllers may issue **additional CAS** operations to the same **open row** (no new RAS), improving throughput.
- The **Command Rate** (**T1/T2**) limits how frequently commands can be accepted. **DDR** doubles words per cycle (rise+fall edges), shortening **transfer time** for a burst but **not** the access **latency** (tRCD/CL remain).

**Key Points**
- **tRCD (RAS-to-CAS Delay):** Cycles between row activate (RAS) and column select (CAS).
- **CL (CAS Latency):** Cycles from CAS to the **first data word** on the bus (can be integer or half: e.g., **CL=2.5**).
- **Burst Length (BL):** Number of words transferred after the first data appears (commonly **2/4/8**); helps fill cache lines without another RAS.
- **Open-page access:** Keep the **row open** and issue **new CAS** for adjacent columns to avoid extra RAS/precharge. Always-open can backfire with real workloads.
- **Command Rate (T):** **T1** = can accept a new command **every cycle**; **T2** = every **other** cycle.
- **SDR vs DDR:** **SDR** outputs **1 word/cycle**; **DDR** outputs **2 words/cycle**. DDR **reduces burst time**, but **doesn’t reduce** intrinsic **latency** (tRCD/CL).

**Why This Matters (performance intuition)**
- **Latency pieces add up:** `access time ≈ tRCD + CL (+ precharge/activate if row miss)`.
- **Throughput loves bursts:** Longer **BL** and **row reuse** (open-page) amortize setup costs.
- **DDR helps bandwidth, not latency:** Faster **drain** of the burst, same **time-to-first-byte**.

#### 2.2.2 Precharge and Activation
- After a burst read finishes for one row, the DRAM **must precharge** (close) that row before a **new RAS** can open another row.
- Precharge can’t start until the current **data burst** completes; then you must wait **tRP** cycles before issuing the next RAS.
- Additionally, a row must remain active for at least **tRAS** cycles from its RAS before it’s legal to precharge.
- These gaps mean the **data bus can sit idle**: in the example, only **2 cycles of data in 7 total cycles** are achieved, reducing a theoretical **6.4 GB/s** link to about **1.8 GB/s** sustained.
- Understanding **tRCD/CL/tRP/tRAS** (and command rate **T1/T2**) is key to interpreting performance and configuring memory timings.

**Key Points**
- **Precharge requirement:**  
  You cannot issue a new **RAS** (activate another row) until the current row is **precharged** (closed). Precharge often uses a control encoding (e.g., **WE↓ + RAS↓**).

- **Ordering constraints:**  
  1. **Burst must finish** before precharge can be issued.  
  2. After precharge: wait **tRP** cycles before next RAS.  
  3. **tRAS**: minimum time a row must stay active after RAS **before** it may be precharged. If `tRCD + CL + transfer_time` is shorter than **tRAS**, you must **delay** precharge further.

- **Command rate (T):**  
  DRAM can accept a new command **every cycle (T1)** or **every other cycle (T2)**; **T1** parts are typical of higher-performance DIMMs.

- **Throughput impact (example in the text):**  
  With a short burst (2 words on SDR, or think 4 words on DDR) and the needed tRP/tRAS spacing, the **bus is active only 2/7 cycles**.

**Timing Shorthand (common “w-x-y-z-T”)**
| Field | Meaning                         |
|------:|----------------------------------|
| **w** | **CL** — CAS Latency            |
| **x** | **tRCD** — RAS-to-CAS Delay     |
| **y** | **tRP** — Row Precharge Time    |
| **z** | **tRAS** — Active→Precharge Min |
| **T** | Command Rate (**T1** or **T2**) |

Example: **2-3-2-8-T1** ⇒ CL=2, tRCD=3, tRP=2, tRAS=8, Command Rate=T1.

**Why This Matters**
- Short, scattered accesses that **switch rows often** suffer from **precharge/activate gaps**.  
- **Longer bursts** and **row reuse** (open-page hits) amortize setup costs and **raise sustained bandwidth**.  
- Knowing your DIMM timings (w-x-y-z-T) helps **interpret measurements** and, where safe, **tune BIOS/UEFI** settings (with the usual stability caveats noted in the text).

#### 2.2.3 Recharging
- **DRAM cells leak charge** and must be **refreshed** on a schedule (typically every **64 ms**).
- Refresh is **not fully transparent**: when a **row** is being refreshed, **no access** to that row is possible, and real systems can see **noticeable stalls** depending on how refresh is organized and scheduled by the **memory controller**.
- The DRAM module **tracks the last refreshed row** and auto-increments on each refresh request.
- Programmers cannot control refresh timing, but should **account for its effects** when interpreting latency and bandwidth measurements.

**Key Points**
- **Why refresh?**  
  DRAM stores bits in capacitors that **leak**, so cells must be **periodically recharged**.
- **Not fully hidden:**  
  During a **row refresh**, that row is **inaccessible**; critical accesses landing on a refreshing row can **stall** the CPU.
- **Refresh schedule & organization matter:**  
  A cited study found refresh **organization can affect performance dramatically**; the **memory controller** decides when to issue refreshes.
- **Who does what:**  
  - **Controller:** Issues refresh commands at appropriate intervals.  
  - **DRAM module:** Keeps the **address of the last refreshed row** and **auto-increments** for the next refresh.
- **Programmer guidance:**  
  You usually **cannot influence** refresh timing directly; **remember it** as a source of **outliers** in timing data.

**Quick Math (from the text’s example)**
- **JEDEC refresh window:** every **64 ms** per cell.  
- **If the array has 8,192 rows**, average interval between refresh commands:
  - Convert 64 ms → **64,000 µs**  
  - Compute **64,000 µs ÷ 8,192 = 7.8125 µs**  
  ⇒ Controller issues a refresh **about every 7.8125 µs** on average (can **queue** refreshes, so the **max** interval may be higher).

#### 2.2.4 Memory Types
- **SDR** DRAM moves one data item per cycle and runs the **array** and **bus** at the same frequency—raising throughput by raising frequency (and power).
- **DDR1** doubles transfers per cycle (rise+fall) via a **2-bit I/O buffer** without speeding the array.
- **DDR2** doubles the **bus** frequency (quad-pumped effective rate) and uses a **4-bit I/O buffer**; marketing names report **effective** rates (e.g., DDR2-800 = PC2-6400).
- **DDR3** lowers voltage (~1.8 V → ~1.5 V), cuts power (∝ V²), runs the array at **¼ bus** with an **8-bit I/O buffer**, initially with slightly higher CL but enabling higher bandwidth.
- Parallel wide buses hit pin/signal-integrity limits (e.g., ~240 pins/DIMM, ≤2 DIMMs/channel on DDR2; often 1 at high DDR3 speeds), capping capacity.
- **FB-DIMM (FB-DRAM)** solves scaling with a **serial, full-duplex differential** link (~69 pins), allowing **more DIMMs/channel** and **more channels**, boosting aggregate bandwidth at the cost of **extra latency per hop** and **higher module power**.

**Key Points**
**SDR vs DDR1**
- **SDR:** Array and bus at same **f**; 1 transfer/cycle. Raising **f** boosts bandwidth but **costs power** (and often voltage).  
- **DDR1:** 2 transfers/cycle (**double-pumped**) using a **2-bit I/O buffer**; **array freq unchanged**. Naming switched from raw MHz (e.g., **PC100**) to **sustained byte rate**:  
  - Example: `100 MHz × 64 bit × 2 = 1600 MB/s` → **PC1600** (factor-of-two improvement).

**DDR2**
- **Bus frequency doubled** (quad-pumped effective rate) with a **4-bit I/O buffer**; the array still runs slower.  
- **Marketing names** reflect **effective** bus rates (below). Also note the “FSB frequency” is often the **effective** (inflated) number (e.g., 133 MHz array / 266 MHz bus → “FSB 533 MHz”).

   **DDR2 naming (from Table 2.1):**
   
   | Array Freq | Bus Freq | Data Rate | PC2 Name | DDR2 Name |
   |---:|---:|---:|:--|:--|
   | 133 MHz | 266 MHz | 4,256 MB/s | **PC2-4200** | **DDR2-533** |
   | 166 MHz | 333 MHz | 5,312 MB/s | **PC2-5300** | **DDR2-667** |
   | 200 MHz | 400 MHz | 6,400 MB/s | **PC2-6400** | **DDR2-800** |
   | 250 MHz | 500 MHz | 8,000 MB/s | **PC2-8000** | **DDR2-1000** |
   | 266 MHz | 533 MHz | 8,512 MB/s | **PC2-8500** | **DDR2-1066** |

**DDR3**
- **Voltage** drops from ~**1.8 V** (DDR2) to ~**1.5 V** (DDR3) → **~30% power reduction** from voltage alone (power ∝ V²), plus die/electrical advances can **halve power at same freq**.  
- **Array at ¼ bus**, using an **8-bit I/O buffer**; higher top **bandwidth**, initially **higher CL** until the tech matures.  
- At high transfer rates (e.g., **≥ 1600 Mb/s**), **DIMMs/channel may be limited to one**, constraining capacity.

   **DDR3 naming (from Table 2.2):**
   
   | Array Freq | Bus Freq | Data Rate | PC3 Name | DDR3 Name |
   |---:|---:|---:|:--|:--|
   | 100 MHz | 400 MHz | 6,400 MB/s | **PC3-6400** | **DDR3-800** |
   | 133 MHz | 533 MHz | 8,512 MB/s | **PC3-8500** | **DDR3-1066** |
   | 166 MHz | 667 MHz | 10,667 MB/s | **PC3-10667** | **DDR3-1333** |
   | 200 MHz | 800 MHz | 12,800 MB/s | **PC3-12800** | **DDR3-1600** |
   | 233 MHz | 933 MHz | 14,933 MB/s | **PC3-14900** | **DDR3-1866** |
   
   - **Reminder:** Vendors often quote **effective** (edge-counted) bus/“FSB” speeds.

**Why Wide Parallel DDR Buses Don’t Scale Easily**
- **Pin count:** ~**240 pins**/DIMM (data + addresses) → routing with matched lengths is hard.  
- **Signal integrity:** Daisy-chaining degrades signals; DDR2 often **≤ 2 DIMMs/channel**, DDR3 **~1 at high speeds**.  
- Practically caps consumer boards to **~4 DIMMs total**, limiting capacity unless more channels/controllers are added.

**FB-DIMM (FB-DRAM)**
- Replaces wide parallel bus with **serial, full-duplex, differential** links (like SATA/PCIe ideas), using the **same DRAM chips** as DDR2/DDR3 on the module.  
- **Pins:** ~**69** per module (vs **240**), easier routing, **up to 8 DIMMs/channel**, and **more channels** per controller (e.g., **6**).

   **Comparison (from the text’s summary):**
   
   | Feature | DDR2 | FB-DRAM |
   |:--|--:|--:|
   | Pins / DIMM | **240** | **69** |
   | Channels / Ctrl | **2** | **6** |
   | DIMMs / Channel | **2** | **8** |
   | Max Memory (example) | **16 GB** | **192 GB** |
   | Throughput (example) | **~10 GB/s** | **~40 GB/s** |

- **Trade-offs:**  
  - **Latency**: Each DIMM hop on the serial chain adds a (small) delay.  
  - **Power**: The high-speed serial chip on the module consumes **significant energy**.  
  - **Upside**: For large memory systems, FB-DIMM can **out-scale** DDR2/DDR3 using commodity parts.

**Practical Implications**
- **Bandwidth vs latency trade-off:** Later DDR generations deliver **more bandwidth**—not necessarily **lower first-byte latency**.  
- **Capacity planning:** Wide-bus DIMM limits can constrain max RAM; **IMCs** and **FB-DIMM** address scaling differently (with their own trade-offs).  
- **When tuning:** Know your **effective rates** and **timings**; don’t mistake marketing MHz for **sustained** throughput.

#### 2.2.5 Conclusions
- DRAM access is inherently **slower** than CPU/register/cache access, so even tiny **memory-bus stalls** expand into many **CPU cycles** of idle time.
- When accesses are **sequential and row-friendly**, DRAM can deliver **high sustained bandwidth** (e.g., DDR2-800, dual channel ≈ **12.8 GB/s**).
- But **non-contiguous** patterns force **precharge + new RAS** operations, creating gaps and lowering throughput.
- **Hardware/software prefetching** overlaps these timings and shifts reads earlier so they don’t collide with writes, reducing contention before data is needed.

**Key Points**
1. **CPU vs DRAM speed gap matters**  
   Example ratio: **11:1**—each **1 memory-bus cycle stall** ≈ **11 CPU cycles stalled** on a 2.933 GHz Core 2 with a quad-pumped 1066 MT/s FSB.  
   - **Math:** Base bus clock ≈ 1066 / 4 = **266 MHz**. Ratio ≈ **2.933 GHz / 0.266 GHz ≈ 11.0**.

2. **Sustained bandwidth can be high (when sequential)**  
   Entire **rows** can stream with **no stalls**, keeping the bus at **100% occupancy** (ideal case).

3. **Throughput example (from text)**  
   **DDR2-800, dual channel** → **12.8 GB/s** peak sustained under ideal streaming:  
   - **Math:** 800 MT/s × **8 B/transfer** × **2 channels** = **12.8 GB/s**.

4. **Random/scattered access hurts**  
   **Precharge** (close row) + **RAS** (open new row) insert **idle cycles**, reducing sustained bandwidth.

5. **Prefetching reduces stalls & contention**  
   **HW/SW prefetch** overlaps activation/latency, **pulls reads forward**, and avoids read/write bunching right when data is needed.

---

### 2.3 Other Main-Memory Users (DMA)
- Beyond CPUs, **high-performance devices** (NICs, storage controllers, etc.) access RAM via **DMA**, bypassing the CPU but **competing for memory/FSB bandwidth** through the **Northbridge/Southbridge**.
- Heavy DMA can **stall the CPU** while it waits on memory.
- On **NUMA** systems, smart topology (e.g., using memory on nodes *without* heavy DMA, or giving each node its own Southbridge) can reduce contention.
- Systems that **share system RAM for graphics** (no dedicated VRAM) generate frequent memory traffic and add latency; they’re **poor choices for performance-critical workloads**.

**Key Points**
- **DMA = direct RAM access by devices**  
  Frees CPU cycles, but **consumes the same memory/FSB bandwidth** the CPU needs → potential **stalls** under load.
- **Everything funnels through bridges in classic designs**  
  Even non-DMA buses (e.g., **USB**) still traverse **Southbridge → Northbridge → FSB**, using shared bandwidth.
- **NUMA can isolate contention**  
  With an architecture like Fig. 2.3, place **compute on nodes** whose **local memory** and I/O paths **aren’t saturated** by DMA; attaching **one Southbridge per node** can distribute traffic.
- **Integrated graphics using system RAM**  
  No dedicated VRAM → frequent reads/writes to **main memory**; since system RAM is **single-ported**, display refresh and GPU activity **compete** with CPU accesses, increasing **latency**.

**Quick Math Check (from the text’s example)**
   **Display traffic (1024×768 @ 16 bpp, 60 Hz):**
   - Pixels/frame: `1024 × 768 = 786,432`
   - Bytes/pixel: `16 bpp = 2 bytes`
   - Bytes/frame: `786,432 × 2 = 1,572,864`
   - Bytes/second: `1,572,864 × 60 = 94,371,840 B/s ≈ 94 MB/s (decimal) ≈ 90 MiB/s`  
   This aligns with the quoted **≈94 MB/s** and shows why shared-RAM graphics can materially load the memory subsystem.

**Practical Tips**
- On NUMA, **pin** DMA-heavy devices and their buffers to **specific nodes**; pin compute elsewhere.  
- **Batch I/O** and **align buffers** to be cache-/channel-friendly; avoid unnecessary cache pollution by DMA (later sections discuss APIs/techniques).  
- For performance builds, prefer **dedicated VRAM** GPUs over shared-memory graphics.

[back to top](#table-of-contents)

## 3 CPU Caches
- CPU cores have sped up far faster than DRAM. Making DRAM as fast as cores is **technically possible but economically infeasible**, so systems pair a **small amount of fast SRAM** (as **CPU caches**) with large, cheap **DRAM**. Mapping SRAM as a special OS-managed address range would drown in **software overhead and portability issues**.
- Instead, processors **transparently cache** recently/nearby data, exploiting **temporal** and **spatial locality**.
- Because caches are much smaller than main memory (often ~**1/1000** the size), performance depends on **smart replacement and prefetching**—and on programmers writing **cache-friendly code**.

**Key Points**
- **The widening gap:**  
  1990s onward: **CPU frequency ↑↑** while **DRAM/FSB lagged** for cost/power reasons → **memory stalls dominate**.
- **Why not “all SRAM”?**  
  SRAM could match core speeds but is **orders of magnitude more expensive** than DRAM for capacity; practical systems need **lots of RAM**.
- **OS-managed fast region? Not viable.**  
  Exposes **complex mapping**, **per-process arbitration**, **portability** issues; the **overhead erases benefits**.
- **Transparent caches exploit locality:**  
  - **Temporal locality:** recently used data/code is likely reused soon.  
  - **Spatial locality:** nearby data/code (same line/region) often accessed together.
- **Caches are tiny vs DRAM:**  
  Example rule of thumb: ~**1/1000** (e.g., **4 MB cache vs 4 GB RAM**). If the **working set** fits the cache, great; if not, rely on **replacement** and **prefetching**.
- **Prefetching & replacement matter:**  
  Good **hardware/software prefetch**, plus smart **replacement**, can make a small cache **act larger** by hiding DRAM latency.

**Worked Example (from the text, with explicit math)**
**Assumptions:** DRAM access = **200 cycles**, cache hit = **15 cycles**.  
**Use case:** **100** unique data items, each used **100** times.

- **Without cache:** `100 × 100 × 200 = 2,000,000 cycles`.
- **With cache (all data fits):**  
  For each item: 1 miss (**200**) + 99 hits (**99 × 15 = 1,485**) = **1,685 cycles**.  
  Total: `100 × 1,685 = 168,500 cycles`.

**Practical Tips**
- **Process in cache-sized tiles**; iterate in **memory order**.  
- **Choose data layouts** that match access patterns (SoA vs AoS).  
- **Use prefetching** (compiler intrinsics or built-in HW) to hide latency.  
- **Measure cache misses/stalls** to confirm improvements.

---

### 3.1 CPU Caches in the Big Picture
- Modern CPUs interpose **fast SRAM caches** between cores and main memory: **all loads/stores go through the cache** over a fast core↔cache link.
- Early designs had a **single cache**, but practice showed that **separating instruction (L1i) and data (L1d)** caches works better, and that **additional cache levels (L2/L3)** are needed as the **core↔DRAM gap** widened.
- With **multi-core** and **multi-thread (SMT/Hyper-Threading)** CPUs, **threads on the same core share that core’s L1**, cores **share higher-level caches**, and **different sockets do not share caches**.
- Exact cache interfaces vary by CPU and may not require data to pass through every higher level; those design choices are **invisible to programmers**.

**Key Points**
- **Minimum cache configuration (Fig. 3.1)**  
  The CPU core is **not directly connected to DRAM**; **every load/store** traverses the cache over a **special, fast link**. The cache and DRAM sit on the system/FSB side for access to the rest of the machine. *(user-provided text)*
- **Split L1 caches: instruction vs data**  
  Although machines are “von Neumann” in memory model, **separate L1i and L1d** improve performance because **code and data working sets are largely independent**. Bonus: caching **decoded instructions** can speed re-execution after branch mispredictions/flushes. *(user-provided text)*
- **Deeper hierarchies as the gap grows (Fig. 3.2)**  
  A single small/fast L1 couldn’t bridge the core↔DRAM gap, so **bigger, slower L2 (and often L3)** caches were added. **Three levels** are common; future designs may add more as core counts rise. *(user-provided text)*
- **Data flow isn’t strictly linear through all levels**  
  The diagram is **schematic**: data **need not** pass through every higher level on the way to memory; **designers have latitude** in cache interfaces. **Programmers do not see these details directly.** *(user-provided text)*
- **Cores vs threads; who shares what? (Fig. 3.3)**  
  - **Threads** (SMT) on the **same core** share **that core’s L1** (the core’s resources are largely shared apart from some registers).  
  - **Each core** has **its own L1i/L1d**.  
  - **All cores on a socket** typically **share higher-level caches** (e.g., L3).  
  - **Different sockets (processors)** **do not share caches**.  
  *(user-provided text)*

---

### 3.2 Cache Operation at High Level
- By default, **all CPU loads/stores go through caches**; only special regions or instructions bypass them. Caches store data in **cache lines** (typically **64 B**) and identify them with **tags** derived from the (virtual or physical) address.
- A line is fetched on first use; subsequent hits are served quickly.
- **Writes** normally require the line in cache (read-for-ownership), creating **dirty** lines that must be written back.
- On modern multicore systems, caches are kept **coherent** (e.g., MESI-style) via **snooping** and invalidation / cache-to-cache transfers.
- Implementations differ (e.g., **inclusive** vs **exclusive** hierarchies), but all aim to minimize expensive trips to DRAM.
- The real cost varies by level (e.g., Pentium M: **L1≈3 cy**, **L2≈14 cy**, **DRAM≈240 cy**) and can sometimes be hidden by **pipelining** and **write buffering**.
- Measured behavior shows **plateaus** as working sets overflow L1 then L2—often causing **order-of-magnitude** slowdowns—hence the importance of cache-friendly code.

**Key points**
1. What’s cached and how it’s identified
    - **Default cacheability:** All normal memory is cacheable (OS may mark exceptions; programmers can use special instructions to bypass caches—discussed later in §6).
    - **Cache lines (spatial locality):** Caches fetch **lines** (not single words). Early lines were 32 B; **64 B is common now**. A 64-bit bus needs **8 transfers** per 64-B line; DDR moves these efficiently.
    - **Address split (offset / set / tag):** For a 64-B line, **offset O = 6 bits** (byte-in-line). Let **S** be the **set index** bits (there are **2^S sets**). The **tag T = address_bits − S − O** identifies which memory line occupies a slot in that set.  
  *(Address used for the tag can be **virtual or physical**, depending on the implementation.)*

2. Reads and writes
    - **Read:** If not present, the **entire line** is loaded into **L1d**; the lower **O** bits select the byte(s) within the line.
    - **Write (read-for-ownership):** CPUs still **load the line first** (can’t hold partial lines) unless using special mechanisms (e.g., **write-combining**, §6.1). A modified line becomes **dirty** until written back; on writeback, the **dirty flag clears**.

3. Evictions & hierarchy policy
    - **Eviction chain:** Making room in **L1d** may push the victim **down** (L2 → L3 → memory). Each hop costs more.
    - **Exclusive vs inclusive:**
      - **Exclusive (e.g., many AMD/VIA):** L1 and L2 **do not duplicate** contents; evicting L1 **pushes into L2**. Pros: better **effective capacity**; loading a new line may **touch only L1**. Cons: evictions can be **more expensive**.
      - **Inclusive (e.g., many Intel):** **L1 ⊆ L2**; evicting L1 is **cheap** (the line is also in L2). Cons: some **duplication** reduces effective capacity, mitigated if L2 is large.
    - **Controller freedom (within the memory model):** CPUs may **proactively write back** dirty lines during low bus activity; many internal policies are hidden behind the architectural memory model.

4. Coherence across CPUs/cores (SMP)
    - **Goal:** All processors observe a **uniform memory view**.
    - **Mechanism:** **Snoop** and react to others’ read/write intents for a given line:
      - If another CPU **writes** a line you hold **clean**, your copy is **invalidated**.
      - If you hold a **dirty** line and another CPU needs it, your cache **supplies it directly** (cache-to-cache), often with memory updated by the controller.
    - **Simple invariants (MESI-style behavior; §3.3.4 later):**
      - A **dirty** line exists in **only one** cache.
      - **Clean** copies may exist in **many** caches.

5. Latency/costs (Pentium M example)

    | Destination     | Cycles (approx.) |
    |-----------------|------------------|
    | **Register**    | ≤ 1              |
    | **L1d**         | ~ 3              |
    | **L2**          | ~ 14             |
    | **Main memory** | ~ 240            |

    - **Wire delay dominates** much of on-die L2 latency; larger caches tend to have **worse wire delays** (process shrinks help).

6. Hiding some of the cost
    - **Pipelining & early issue:** Loads started **early in the pipeline** can overlap other work; **L1 hits** (and sometimes **L2**) can be largely hidden.
    - **Limits:** If the **address is late** (depends on prior ops) or resources are busy, you **pay** the latency.
    - **Writes:** CPUs need not stall for the store to **complete in memory**; **store buffers / shadow regs** preserve ordering per the memory model while allowing progress.

7. What measurements reveal (Fig. 3.4 narrative)
    - Random access over a growing **working set** shows **three plateaus** (L1-fit, then L2-fit, then DRAM-bound).  
  From the observed steps, you can infer cache sizes (e.g., **L1 ≈ 2¹³ B**, **L2 ≈ 2²⁰ B** in the cited example).  
    - When the working set exceeds L2, average cost can jump to **hundreds of cycles**; modifying data further adds **writeback** traffic.

---

### 3.3 CPU Cache Implementation Details
- Because **every memory location is potentially cacheable** but caches are **tiny relative to DRAM** (often about **1:1000** in size), many distinct memory locations will **compete for the same limited cache slots** when a program’s **working set** is large.
- This fundamental pressure is the root cause of **evictions and misses** and motivates the cache organization/mapping policies discussed next.

**Key points**
- **Tiny cache vs huge memory.**  
  It’s normal for cache capacity to be roughly **~1/1000** of main memory, so **far more lines exist in DRAM than can fit in cache** at once. *(user-provided text)*
- **Working-set competition.**  
  If your **working set > cache**, many addresses will **contend** for the same few cache slots, triggering **frequent replacements**. *(user-provided text)*
- **Why this matters.**  
  The contention translates to **higher miss rates** and **more trips to DRAM**, which are orders of magnitude slower than L1/L2 hits.
- **What comes next.**  
  This pressure is exactly why caches use **line-based fills**, **sets**, and **placement/replacement policies**—topics typically introduced immediately after this setup.

**Quick, concrete intuition (derived example)**
- Not from the text—just to visualize the scale.
- **4 GB DRAM, 4 MB cache ⇒ ~1:1000.**  
- With **64 B cache lines**:  
  - DRAM has `4 GB / 64 B ≈ 67,108,864` distinct lines.  
  - Cache holds `4 MB / 64 B = 65,536` lines.  
  ⇒ Roughly **1,024 DRAM lines per cache line slot** *on average* competing for residency.

#### 3.3.1 Associativity
- A cache must be able to hold a copy of **any** memory line, but it’s tiny vs DRAM (often ~**1:1000**), so many lines **compete** for the same slots.
- A **fully associative** cache lets any line go anywhere (great flexibility) but is **hardware-expensive** (comparators for every entry → impractical for L1/L2/L3).
- A **direct-mapped** cache is **fast/simple** (one slot per set) but suffers **conflict misses** when many addresses map to the same slot.
- **Set-associative** caches are the compromise:
  - each set has multiple “ways” (e.g., 2-, 4-, 8-way…), comparing all tags in a set **in parallel**.
- Empirically, going from direct-mapped to low associativity (e.g., **2-way**) can cut misses significantly (e.g., **~44%** for an 8 MB L2 in the cited gcc run), with **diminishing returns** beyond ~**8-way** in single-threaded cases;
  - shared caches (SMT/multicore) **effectively reduce** associativity, so higher associativity or extra levels (shared **L3**) help.
- Access order matters: **sequential** traversals benefit much more than **random** traversals.

**Key Points**
- Fully Associative
  - **Definition:** Any memory line can occupy **any** cache line; **S = 0** (no set index), tag = address bits above the line offset.
  - **Hardware cost:** Needs a **comparator per cache line** to check all tags in parallel; for a 4 MB L2 with 64 B lines:  
    - Entries = `4 MiB / 64 B = 65,536` comparators → **impractical** for large caches.
  - **Reality:** Used only for **very small** structures (e.g., some **TLBs**) with **dozens** of entries.

- Direct-Mapped
  - **Definition:** Each address maps to **exactly one** line in the cache.  
  - **Indexing:** Example 4 MB/64 B → 65,536 entries → use **bits 6..21** as the index (offset **O=6** for 64 B lines).  
  - **Pros:** **Fast**, **simple**, **one comparator**.  
  - **Cons:** Prone to **conflict misses** if many hot addresses share the same index bits.

- Set-Associative (the usual choice)
  - **Definition:** The cache is split into **sets**; each set holds **N ways** (N cached lines for the same set index). For a request, all **N tags** are compared **in parallel** within the indexed set.
  - **Example (from text):** 4 MB L2, 64 B lines, **8-way** associativity:  
    - Total entries: `4 MiB / 64 B = 65,536`  
    - Sets: `65,536 / 8 = 8,192` → **S = log₂(8192) = 13** set-index bits  
    - Offset: **O = log₂(64) = 6**  
    - **Tag bits T** (for a 32-bit address): `32 − S − O = 32 − 13 − 6 = 13`
  - **Hardware:** Only **N comparators** (one per way) per access; scales well as cache capacity grows by adding **columns** (entries), not more **rows** (ways), unless associativity is increased.

**Performance Findings (from the provided text)**
- **Associativity reduces misses (up to a point).**  
  In the gcc experiment (Table 3.1 / Fig. 3.8), for an **8 MB** L2 with 32 B lines, moving from **direct-mapped → 2-way** cut L2 misses by **~44%**. Gains beyond **~8-way** are typically **small** in **single-threaded** workloads.
- **Shared caches & SMT/Multicore.**  
  With **hyper-threading** (threads share L1) and **multi-core** (often shared L2/L3), effective associativity per thread **drops** (roughly halves with two threads, quarters with four), so **higher associativity** or adding a **shared L3** helps avoid conflict.
- **Cache size vs working set.**  
  Benefits scale with **working set** size and **address distribution**. In the cited run (peak WS ≈ **5.6 MiB**), a **16 MiB** cache still shows some conflict effects (hence some benefit from associativity), but going to **32 MiB** would likely offer **negligible** additional benefit for *that* workload. As workloads **grow**, larger caches **regain** value.
- **Access order matters (Fig. 3.9).**  
  Two tests:  
  1. **Sequential** (follow pointers in **memory order**) → much better cache behavior.  
  2. **Random** (pointer-chasing in **random order**) → many misses; associativity helps, but access locality dominates.

**Rules of Thumb (per the text)**
- **Large caches:** Fully associative is **impractical**; **set-associative** is standard.  
- **Associativity gains:** Biggest jump is **DM → 2-way**; **diminishing returns** beyond **~8-way** (single-threaded).  
- **Shared caches:** Effective associativity **drops** with **more threads/cores** sharing → consider **higher associativity** or a **shared L3**.

**Practical Implications (for your code)**
- **Order your accesses** (make them sequential/blocked) to avoid conflict-heavy patterns.  
- If random/pointer-chasing is unavoidable, **increase locality** (e.g., **pool/arena allocations** to pack related nodes) and consider **software prefetching**.  
- On SMT/multicore, **avoid false sharing**; separate hot data per thread to prevent extra conflicts in shared caches.

#### 3.3.2 Measurements of Cache Effects
- All figures are produced by a synthetic benchmark that builds a **circular singly-linked list** over an array of fixed-size elements.
- The list order is either **sequential (memory order)** or **random**.
- The program always advances via the **pointer field** (even for sequential layouts), so each step is a **pointer chase**.
- The element includes a configurable **payload (`pad`)** to set the element size and thus the **working set** and **cache-line footprint**.
- Runs use **read-only** or **read-modify** variants to expose the cost of **write-allocate** and **writeback**.
- A working set declared as **2^N bytes** contains exactly `2^N / sizeof(struct l)` elements; with **NPAD=7**, `sizeof(struct l)` is **32 B** on 32-bit and **64 B** on 64-bit systems.

```c
struct l {
  struct l *n;  // next pointer (always used to advance)
  long int pad[NPAD];  // payload to control element size
};
```

**How the workloads are constructed**
-	Topology: array of struct l elements; the n pointers are wired into a circular list.
-	Traversal order:
  -	Sequential: the n pointers are arranged so visiting n follows physical memory order (lower part of Fig. 3.9).
  -	Random: the n pointers form a random permutation (upper part of Fig. 3.9).
-	Operation type: either read-only (touch data) or read-modify (write), to surface dirty-line and writeback costs.
-	Pointer chasing always: even in the sequential case the benchmark dereferences the pointer, rather than indexing i+1. This:
  -	Forces a true data dependency between steps.
  -	Reduces artificial compute overlap and makes latency/miss effects visible.
  -	Keeps the control flow identical across sequential vs random configurations.

**Working set sizing & element count**
- Definition: Working set of `2^N` bytes `⇒` number of elements
- Examples:
  - `32-bit, NPAD=7 → sizeof=32 B`
    - `2^20 B (1 MiB) ⇒ 2^20 / 32 = 32,768 elements`
  - `64-bit (LP64), NPAD=7 → sizeof=64 B`
    - `2^20 B ⇒ 2^20 / 64 = 16,384 elements`

**Why these choices expose cache behavior**
- Cache-line footprint: with a 64-B line:
  - 32-bit/sizeof=32 B ⇒ 2 elements/line → sequential walk often reuses the same line (better locality).
  - 64-bit/sizeof=64 B ⇒ 1 element/line → sequential walk still benefits from streaming prefetch, but no intra-line reuse.
- Sequential vs random:
  - Sequential: maximizes spatial locality and enables hardware prefetch → higher sustained bandwidth, fewer activations.
  - Random: defeats prefetch & locality → higher miss and activate/precharge frequency; stresses associativity.
- Read vs write:
  - Read-only shows hit/miss latency and bandwidth ceilings.
  - Writes add write-allocate (fetch on store), create dirty lines, and incur writeback on eviction → higher effective cost.
- Scaling working set: sweeping 2^N bytes reveals plateaus/steps (L1-fit → L2-fit → DRAM-bound), letting you infer cache sizes.

##### Single Threaded Sequential Access
- A dense, sequential walk over a circular list shows three clear speed plateaus that map to **L1d**, **L2**, and **DRAM** behavior.
- On this P4: **L1d hits ≈ 4 cycles**; with an effective **hardware prefetcher**, even L2/DRAM fetches can be kept to **≈ 9 cycles** in the tight sequential case.
- But once the **stride grows** (larger `struct l`), prefetch **can’t keep up** (and can’t cross pages), so costs jump toward **L2 ≈ 28 cycles** and beyond.
- Separating each element onto its **own page** exposes **TLB capacity** limits (≈ **64 entries** here), which produce a dramatic spike once the working set exceeds the TLB reach.
- **Writes** add **write-allocate + writeback** traffic; when eviction requires writes, **FSB bandwidth is effectively halved**, doubling observed cycles.
- Finally, **bigger last-level caches** keep you longer on the “fast plateau,” translating into **substantial real speedups** for sequential-friendly workloads.

**Key Points**
- Three plateaus = L1d, L2, DRAM (Fig. 3.10)
  - Plateaus observed at working sets:
    - **≤ 2¹⁴ B** (fit L1d ≈ **16 KB**): ~**4 cycles** per element.
    - **2¹⁵–2²⁰ B** (fit L2 ≈ **1 MB**): ~**9 cycles** per element (prefetch hides much of L2 latency).
    - **≥ 2²¹ B** (beyond L2): still ~**9 cycles** if prefetch remains effective.
  - Why **9 cycles** (not the raw L2/DRAM latencies)?  
    **Hardware prefetch** streams the next line; when you arrive, it’s **already in flight/arrived**, so the visible stall is much smaller than nominal L2/DRAM access.

- When stride grows, prefetch falls behind (Fig. 3.11)
  - Increasing `NPAD` increases **distance** between successive `n` pointers (e.g., 0, 56, 120, 248 B).
  - **L1d-fit region:** all curves similar (no prefetch needed).
  - **L2-fit region:** large-stride runs cluster near **~28 cycles** ⇒ **prefetch disabled/ineffective**; every iteration stalls on L2.
  - **Beyond L2:** curves diverge strongly; larger strides hurt more because:
    - **Prefetcher does not cross page boundaries**, so many opportunities are lost as stride grows.
    - Each step has **less time** to overlap memory before the next dependent pointer chase.

- TLB pressure is real (Fig. 3.12)
  - Two setups with `NPAD=7` (64-B elements):
    1. **Packed**: 1 cache line per element; **new page every 64 elements**.
    2. **One element per page**: each iteration hits a **different page**.
  - Result: the per-page layout shows a **huge spike** when working set reaches **2¹³ B** (which is **128** elements × 64 B; i.e., **128 pages**).  
    - The spike marks **TLB overflow**; deduced **TLB ≈ 64 entries** on this machine.
  - Takeaway: **Address translation cost adds** to L2/DRAM cost; with larger elements, **more TLB misses per unit work**.

- Writes double the pain when bandwidth saturates (Fig. 3.13)
  - Baseline **Follow** (read-only) vs **Inc** (modify current) vs **Addnext0** (read next, then modify current).
  - Surprise: **Addnext0** can be **faster** than **Inc** (for L2-fit sets) because reading from the “next” element acts like a **forced prefetch**—the next node is **already hot** when you advance.
  - But once DRAM is active, **write traffic matters**: modified lines must be **written back** on eviction ⇒ **available FSB bandwidth halves** ⇒ observed plateau roughly **doubles** (e.g., ~**28** vs ~**14** cycles in the comparison cited).

- Why bigger last-level caches help (Fig. 3.14)
  - Compared three CPUs:
    - P4: **32 KB L1d + 1 MB L2**
    - P4: **16 KB L1d + 512 KB L2 + 2 MB L3**
    - Core2: **32 KB L1d + 4 MB L2**
  - Larger **LLC** keeps the run longer on the **lower plateau**, **doubling** throughput in the 2²⁰-byte region in one P4 comparison. The Core2 with **4 MB L2** is better still.
  - If your workload can be **tiled to fit** the LLC, the payoff can be **dramatic**.

**Quick Numbers (from the narrative)**
- **L1d hits:** ~**4 cycles** (P4).
- **L2-fit, effective prefetch:** ~**9 cycles** per element (sequential stride).
- **L2 when prefetch can’t keep up (large stride):** ~**28 cycles**.
- **DRAM raw latency:** **200+ cycles** (hidden well only with streaming prefetch).
- **TLB capacity (here):** ≈ **64 entries** (spike at 2¹³ B with 1 element/page).
- **Writeback effect:** dirty evictions **halve effective bandwidth**, **roughly doubling** observed cycles.

**Practical Tips (you can apply)**
- **Keep strides small** and **walk in memory order**; where possible, **tile** to your LLC.
- Use **contiguous allocations** (arenas/pools) to avoid unnecessary **page crossings**; consider **huge pages** if appropriate to extend TLB reach.
- For one-pass writes, use **streaming stores / write-combining** (when safe) to avoid read-for-ownership churn.
- When you must modify, **batch** and **tile** so that dirty lines are evicted less often.
- If you control hardware choice, **favor larger LLCs** for scan-heavy or tile-friendly workloads.

#### Single Threaded Random Access
- Sequential scans let hardware **prefetch** hide most of the **L2/DRAM latency**, yielding low, flat "plateaus."
- With **random** pointer-chasing, prefetching can’t predict the next line, so the processor suffers **soaring L2 miss rates**, **more L2 traffic per step**, and **TLB thrashing**—pushing per-element cost well beyond the nominal 200–300-cycle DRAM latency (measured **450+ cycles**).
- Constraining randomness to **blocks of pages** lowers TLB misses and significantly improves performance.

**Key Points**
- **Prefetch only helps predictable streams.**  
  With contiguous strides, lines are pulled into **L2→L1d** early, keeping per-element time ≈ **9 cycles** even when data resides beyond L2.
- **Random access breaks prefetch → costs skyrocket.**  
  In the randomized list, per-element time rises to **450+ cycles** for large working sets—worse than raw DRAM latency—because speculative prefetches often **fetch the wrong lines** and add contention.
- **No “plateaus” for random; cost keeps rising.**  
  Unlike sequential curves (clear L1/L2/DRAM steps), the random curve **doesn’t flatten**; it climbs as the working set grows.
- **Why it rises: L2 miss rate → 100% & more L2 traffic per step.**  
  From Table **3.2**:
  - At **2²² B**, sequential: **~1.78%** L2 misses and **~21.9** L2 uses/iter; random: **~50.0%** misses and **~174.0** L2 uses/iter.  
  - As the set grows (2²⁹ B), random misses reach **~57.8%** and L2 uses/iter explode (**~141k**), reflecting many failed cache reuses and poor overlap.
- **TLB misses are a second big driver.**  
  Random pointer-chasing touches many pages in short succession; the small, very-fast **TLB** overflows, adding translation latency before each cache/memory access.
- **Evidence via “blocked randomization.”**  
  Keeping randomness **within page blocks** (e.g., randomize 60 pages at a time) caps the number of **concurrent TLB entries**, improving throughput; as block size grows → performance approaches the “one big randomization” curve. This shows a **large share** of the random-access penalty comes from **TLB pressure** (reported gains **up to 38%** when reducing TLB misses).
- **NPAD = 7 context.**  
  With **64-B elements** (NPAD=7 on 64-bit), each step typically uses a new cache line, and prefetchers (which **do not cross page boundaries**) have little room to help in random order.

**Selected numbers (from Table 3.2)**

| Working set | Access | L2 miss rate | L2 uses / iteration |
|---|---|---:|---:|
| 2²⁰ B | Sequential | **0.94%** | **5.5** |
| 2²⁰ B | Random     | **13.42%** | **34.4** |
| 2²² B | Sequential | **1.78%** | **21.9** |
| 2²² B | Random     | **50.03%** | **174.0** |
| 2²⁹ B | Sequential | **4.67%** | **2,889.2** |
| 2²⁹ B | Random     | **57.84%** | **141,238.5** |

- **Trend:** As the working set doubles, **random** access more than doubles the per-element time, driven by both **more L2 activity** and **higher miss rates**.

**Rules of thumb**
- Expect **flat plateaus** only for **predictable** access; **random** will **climb**.
- When random is unavoidable, **bound the randomness** (operate in **page-sized blocks**) to keep the **TLB hot**.
- Big working sets + random order ⇒ **miss rate → 100%**, **cycles/element explode**.

**Practical implications**
- **Reshape work** to maximize **predictable** access (stream/tiles) whenever possible.  
- If you must be random, **batch within page blocks** (or per-NUMA node) to reduce **TLB churn**.  
- Keep element/page mapping friendly (e.g., avoid scattering each node to a distinct page unless that’s the point of the test).

#### 3.3.3 Write Behavior
- **Write-through**: every store also writes main memory → simple coherence but **heavy bandwidth** use.
- **Write-back**: stores mark the cache line **dirty**; memory updated **later** on eviction or opportunistically → **higher performance**, but requires **coherence machinery** so other CPUs see the latest data.
- **Write-combining (WC)**: hardware **merges multiple writes** to a line before sending to the device/bus → great for **framebuffers / device RAM**.
- **Uncacheable**: used for **MMIO / special addresses**; **never cache** (device state can change outside CPU).
- On x86, the kernel selects the policy for ranges (e.g., via **MTRRs**); user code sees coherence **transparently** (kernel may still need **explicit flushes**).

**Key Points**
1. Coherent caches: two main policies
    - **Write-through**
      - **What happens:** CPU updates the cache **and** main memory **immediately** on every write.
      - **Why it’s simple:** Memory and cache are always in sync → other CPUs/devices can safely read memory.
      - **Trade-off:** Generates **lots of traffic** on the FSB/memory bus, even for **short-lived** temporaries.
    - **Write-back**
      - **What happens:** CPU updates the cache line and sets a **dirty bit**; memory is updated **later** (on eviction or when the CPU finds idle bus time).
      - **Why it’s fast:** Avoids unnecessary memory writes; can **opportunistically** push lines when the bus is free.
      - **Coherence wrinkle:** If **another CPU** needs data that’s **dirty** in this CPU’s cache, it **must get it from that cache**, not memory → requires a **coherence protocol** (introduced next in the paper).
    - **Programmer view:** Both policies are **transparent** to user-level code; the kernel sometimes must **flush caches** explicitly.

2. Special policies for device / non-RAM regions
    - **Write-combining (WC)**
      - **Use case:** Device-mapped RAM (e.g., **graphics framebuffers**).
      - **What it does:** **Combines multiple stores** to a cache line before writing out, so you amortize bus costs across a whole line (ideal for **contiguous pixel rows**).
      - **Benefit:** Significantly **higher throughput** to device memory vs. writing each store immediately.
    - **Uncacheable**
      - **Use case:** **MMIO** or special hard-wired addresses (e.g., a board LED, PCIe BARs).
      - **Why:** Device state can **change without the CPU**; caching would show **stale data** or delay **observability** (e.g., LED toggles).
    - **x86 configuration note:** The kernel picks the type per address range (e.g., with **MTRRs**) and can also choose between **write-through** and **write-back** for RAM regions.

**When policies shine (and when they hurt)**

| Policy            | Shines when…                                           | Hurts when…                                         |
|------------------|---------------------------------------------------------|-----------------------------------------------------|
| Write-through     | Simple coherence, small shared regions                  | Frequent updates → **excessive bus traffic**        |
| Write-back        | Many **temporary** or **hot** updates to same lines     | Requires robust **multi-CPU coherence** handling    |
| Write-combining   | Streaming, **contiguous** device writes (framebuffers)  | Sparse / random stores (less to combine)            |
| Uncacheable       | **MMIO**, state external to CPU                         | Any workload that expects caching for performance   |

**Practical takeaways**
- Expect **write-back** for normal RAM on modern CPUs (better performance).
- Use **WC** mappings for device buffers you **stream-write**.
- Keep **MMIO** regions **uncached**; read/write them as specified by the device.
- In multi-CPU contexts, remember: a line **dirty** in one cache must be **sourced from that CPU**, not memory (the paper’s next section covers the protocol).

#### 3.3.4 Multi-Processor Support
- In multi-CPU/multi-core systems, caches must appear **coherent** without exposing that complexity to user code.
- Directly peeking into another core’s cache is impractical, so processors use the **MESI** protocol and **snoop** the bus: when a core needs a line that’s **dirty** elsewhere, that line is **transferred** (and memory updated accordingly).
- The most expensive traffic involves **Request-For-Ownership (RFO)** messages (e.g., on thread migration or true sharing).
- Coherence latencies depend on system fan-out and interconnect delays, and overall throughput is further limited by **shared buses** and **finite memory-module bandwidth**, even on NUMA.
- Hence, performance hinges on **minimizing cross-core touches** of the same cache lines and avoiding unnecessary coherence traffic.

**MESI in one table**

| State | Meaning | Local Read | Local Write | Other CPU Read | Other CPU Write (RFO) | Notes |
|---|---|---|---|---|---|---|
| **M**odified | Dirty line; only copy | Hit; stay **M** | Hit; stay **M** | Supplying core **sends data**; both → **S** and memory updated | Supplying core **sends data**; local → **I** | Dirty implies uniqueness |
| **E**xclusive | Clean; only copy | Hit; stay **E** | **Silent** → **M** (no bus traffic) | → **S** (second reader) | → **I** (loses ownership) | E→M cheaper than S→M |
| **S**hared | Clean; shared | Hit; stay **S** | **RFO** → **M** (others invalidated) | Still **S** | → **I** | Fallback when exclusivity unknown |
| **I**nvalid | Not present | Miss | Miss | — | — | Initial state |

- **Key point:** **E**xclusive avoids a bus transaction on the first local write (**E→M**). **S→M** needs an **RFO** and invalidations.

**What triggers costly traffic?**
- **RFOs (Request-For-Ownership):**
  - **Thread migration:** the new CPU must acquire the lines the old CPU had modified.
  - **True sharing:** two CPUs write the **same cache line**.  
- **Dirty-line reads:** a reader must fetch the **up-to-date** line from the owning CPU (not memory).  
- **Protocol latency scaling:** a MESI transition waits until **all CPUs** had a chance to observe/respond; the **slowest path** (bus collisions, NUMA hop delay, traffic) bounds the transition time.

**Hardware limits beyond coherence**
- **Shared FSB / interconnect:** If one CPU can saturate the bus, **multiple CPUs divide** that bandwidth.
- **Memory-module contention:** Even with multiple controllers, **concurrent access to the same module** limits throughput.
- **NUMA reality:** Local memory is fast, but synchronization and shared regions **cross nodes**, re-introducing latency and bandwidth limits.

**Practical implications (actionable, per the excerpt)**
- **Minimize cross-core access** to the **same cache lines**; fewer RFOs ⇒ less bus traffic.
- **Avoid unnecessary sharing** for synchronization; still required at times, but keep it **infrequent**.
- **Be aware of migration costs:** moving a thread to another CPU can force a **wave of RFOs** to rebuild its working set.
- **Design for locality:** prefer structures and work partitioning that **keep data on one CPU/node** when possible.

#### Multi Threaded Access
- When **multiple threads** run on a machine where CPUs/cores **share the same memory buses**, performance can **drop even without any writes**.
- Prefetch traffic from each thread contends for the **shared bus** (CPU↔memory controller) and the **memory-module bus**.
- Once writes enter the picture, **write-back** traffic piles on, further saturating bandwidth.
- Coherence (RFOs) becomes a tax when threads truly share the **same cache lines** or when threads migrate across CPUs.
- Net effect:
  - scaling is **near-linear** only while the working set fits in **last-level cache**;
  - once it spills to DRAM, speedups **collapse**.

**What the graphs demonstrate**
- **Setup:** up to **4 threads** on a **4-processor** machine; fastest per-thread time is reported (total runtime is **even higher**).
- All processors share **one bus** to the memory controller and **one** to memory modules.
1. Sequential **read-only**, 128-byte entries
    - **No writes, no sync**, shared lines allowed.
    - Still see **~18% slowdown** with **2 threads** and **~34%** with **4 threads** (fastest thread).  
      *Cause:* pure **bandwidth contention** (prefetch traffic) on shared buses once the working set exceeds LLC.
2. Sequential **Increment** (writes), 128-byte entries
    - **Y-axis log scale** hides the pain: ~**18%** penalty with **2 threads**, and a striking **~93%** penalty with **4 threads**.  
      *Cause:* **prefetch + write-back** traffic saturates the bus; L1d benefits vanish as soon as >1 thread runs (even with tiny working sets).
3. Random access (**Add-next-last**)
    - Worst case: **~1,500 cycles/element** in the extreme; adding threads **worsens** contention.
4. Achieved speedups at largest working set (from your table)

    | #Threads | Seq Read | Seq Inc | Rand Add |
    |---:|---:|---:|---:|
    | 2 | **1.69×** | **1.69×** | **1.54×** |
    | 4 | **2.98×** | **2.07×** | **1.65×** |

    - Interpretation: With 4 threads and writes, speedup collapses to **~2.07×**; with random access it’s only **~1.65×**.
5. Why speedup collapses beyond LLC
    - Within **L2/L3**, curves approach **~2×** and **~4×** for 2/4 threads.  
    - Once the working set **exceeds L3**, speedups **crash**—2 and 4 threads converge to **similar** (poor) throughput due to **DRAM bandwidth + coherence** costs.

**What actually slows you down (cause→effect)**
- **Shared buses** (CPU↔MC + MC↔DIMMs): prefetch and write-back traffic **compete** → slower even for read-only threads.
- **Write-back** with multiple writers: each thread’s modified lines must be **written** (or owned) → heavy bus usage.
- **Coherence RFOs**: required when two CPUs/threads **write the same cache line** or after **thread migration**; they serialize ownership.
- **Random access**: prefetching becomes ineffective, **TLB/caching** help little, so DRAM latency dominates → **hundreds to ~1,500** cycles/element.

**Practical guidance**
1. **Scale while hot data fits in LLC.** Expect near-linear gains only while the working set resides in L3/L2; beyond that, **DRAM bandwidth** caps speedup.
2. **Partition data per thread** to avoid **touching the same cache lines** (reduces RFOs and bus invalidations).
3. **Prefer sequential access** and **batch writes** to maximize effective bandwidth; random patterns kill prefetching.
4. **Minimize thread migration** of hot workers (migration forces **ownership moves** / cache warm-up on the new CPU).
5. **Be cautious with 3–4+ threads** on a shared-bus box: for large working sets or random access, the **incremental gain past 2 threads is tiny**.

#### Special Case: Hyper-Threads
- - **Two hardware threads share one core.** They time-slice and **share almost all core resources** (execution units, caches, queues); they only have separate architectural **register sets**.
- **Goal:** keep the core busy by running the second thread **while the first is stalled**, typically on **cache misses / memory latency**.

**Key Points**
- **Benefit requires overlap.** SMT helps **only if** the second thread can run **during stalls** (e.g., cache misses) of the first.
- **Caches are shared.** Per-thread **effective cache capacity drops**, so **cache hit rates can fall**—sometimes enough to **negate any SMT gain**.
- **OS scheduling matters.** If the OS treats sibling hyper-threads like separate CPUs and runs **unrelated code** on them, you may see **more cache contention** and **higher miss rates**.

**Simple performance model**
- Let:
  - `N` = total instructions
  - `Fmem` = fraction of instructions that access memory
  - `Ghit` = cache hit rate
  - `Tproc` = cycles per (non-memory) instruction
  - `Tcache` = cycles for a cache hit
  - `Tmiss` = cycles for a cache miss  
- **Idea:** Execution time grows as cache misses grow; SMT must **reduce total wall time** by overlapping misses of one thread with useful work from the other.
- The text’s analysis (for a P4-class HT core) derives **minimum cache-hit rates** needed for SMT to **not slow each thread by ≥50%** (i.e., so two threads together can still beat one).

**Concrete thresholds from the text (same machine assumptions)**
- If **single-thread hit rate** is **< ~55%**, SMT **almost always helps** (core is often stalled; plenty to overlap).
- At **60% single-thread hit rate**, the dual-thread run needs only **~10%** hit rate per thread to break even → **usually feasible**.
- At **95% single-thread hit rate**, the dual-thread run needs **~80%** per-thread hit rate → **hard** once two threads **halve effective cache** and **interfere**.
- **Bottom line:** SMT is **most useful** for **memory-latency-bound** code with **modest hit rates** and **predictable overlap**. It’s **least useful** for already **cache-friendly** code.

**When to **enable/use** Hyper-Threads**
- Your workload is **memory-latency-bound** (low–moderate `Ghit`) and shows **front-end / backend stalls**.
- Two cooperating threads can **share code/data paths** to **reuse** cache lines (see text’s later suggestion about tightly-coupled collaboration).
- You can **pin** related threads to the **same core** and **minimize migration**, avoiding ownership transfers of hot lines.

**When to **avoid/disable** Hyper-Threads**
- Workloads are already **cache-efficient** (high `Ghit`); SMT likely **reduces** per-thread cache hit rates below the threshold.
- The OS scheduler regularly **co-schedules unrelated processes** on the same core (cache thrash risk).
- You observe **regressions** in throughput/latency after enabling SMT, especially under **mixed workloads**.

**Do / Don’t checklist**
**Do**
- Measure **cache-miss rates** (L1/L2/LLC) and **stall cycles** before/after SMT.
- **Pin** cooperating threads on the **same physical core** (and keep them there) if they **share working sets**.
- Structure work so the second thread can **run during stalls** of the first (e.g., overlap I/O or independent compute).
**Don’t**
- Co-schedule two **unrelated, cache-hungry** threads on the same core.
- Assume SMT is a free 2× speedup—it **rarely is**; expect gains only when **miss overlap** is large.
- Ignore **migration**: moving a hot thread between cores triggers **ownership moves** and **re-warm costs**.

#### 3.3.5 Other Details
- **Two address spaces exist:** **virtual** (what code uses) and **physical** (what DRAM sees).  
- **Cache lookups are on a critical path.** Doing the lookup with a **virtual tag** lets the CPU start earlier in the pipeline (before the MMU translation finishes).  
- **Design split (common today):**
  - **L1 caches (L1i/L1d):** **virtually tagged** (fast, tiny, can be flushed if page tables change).
  - **L2/L3 (and beyond):** **physically tagged** (translation finishes in time; flushing is costly due to size/latency).

**Why not use only physical tags?**
- Virtual addresses are **not unique**: same VA can map to different PAs over time or across processes.  
- But waiting for VA→PA translation before an L1 lookup would **expose latency**; hence **virtual tags in L1** to **hide** it.

**Practical consequences**
- **L1 maintenance:** When page tables change, L1 may need **partial/targeted flush** (or, worst case, full flush). Hardware often provides instructions to invalidate a **range**.
- **Lower levels stay physical:** Avoid frequent flushes on big caches; the MMU translation is **ready** by then.
- **Set conflicts:** All caches suffer if too many hot lines map to the **same set**. With physically tagged caches you **cannot** steer mapping from user space.
- **Synonyms/aliases matter:** Mapping the **same physical page** at **multiple virtual addresses** **within one process** can create coherency/duplication issues in virtually tagged L1. **Avoid if possible.**
- **Replacement policy:** Generally **LRU (or approximations)**. As associativity grows (with more cores/sharing), true LRU gets **expensive**; implementations may change. Little to tune from user space.
- **Virtualization wrinkle:** Under a hypervisor, the **VMM controls physical memory**; even the OS cannot fully predict PA placement/mapping diversity.

**What you should do (actionable)**
- **Don’t create synonyms:** Avoid mapping the **same PA** to **multiple VAs** inside a process.  
- **Use pages fully:** Pack data so each **touched page** carries useful work (better TLB/cache behavior).  
- **Prefer larger pages where it helps:** Bigger pages can **reduce TLB pressure** and **diversify PA mapping**, mitigating set hot-spots (details in the virtual memory section).  
- **Don’t overthink replacement:** You can’t pick LRU vs others; instead **shape access patterns** (locality, streaming, blocking) to be cache-friendly.

**Quick pitfalls**
- **Synonyming the same buffer** at two VAs → L1 incoherence hazards.  
- **Assuming OS can “color” pages reliably** under virtualization → may not hold; measure, don’t assume.

---

### 3.4 Instruction Cache
- **Code is cached too (L1i), but is usually less troublesome than data caching (L1d).**  
  Reasons given:
  - Executed code size correlates with **fixed problem complexity**.
  - **Compilers** generate instructions and typically follow good codegen rules.
  - **Control flow is more predictable** than data access; CPUs exploit this with **branch prediction** and **prefetching**.
  - **Instruction streams have strong spatial & temporal locality**.
- **Deep pipelines make fetch/branch behavior critical.**  
  - Modern CPUs execute instructions in **stages** (decode → prepare operands → execute).  
  - **Mispredicted branches** or **late instruction fetches** **stall** the pipeline and take time to refill.
- **Branch prediction & (for CISC) decode costs drive L1i design.**  
  - On x86/x86-64, decoding can be expensive; the text notes that some designs **cache decoded instructions in L1i (“trace cache”)** to skip early pipeline steps on a hit.  
  - **Lower levels (L2+) are unified** (code + data) and **store raw bytes**, not decoded instructions.
- **Practical guidance (most done by the compiler):**
  1. **Keep code small** when possible (exceptions: e.g., software pipelining may justify larger code).
  2. **Aid prefetching** via **code layout** and, where appropriate, **explicit prefetch hints**.

**Key Points**
- **Instruction caches are generally “easier” than data caches.**  
  Code size and flow are more deterministic → **better locality and predictability**, so hardware prefetchers and predictors work well.
- **Pipeline depth amplifies fetch/branch penalties.**  
  Longer pipelines (historically >20 stages) mean a **bigger recovery cost** on mispredicts or slow fetches.
- **Trace cache (decoded I-cache) concept.**  
  For CISC (x86/x86-64), caching **decoded uops** in L1i can **skip decode** on hits and **reduce stalls** after mispredictions.
- **Unified L2/L3 caches hold code and data together.**  
  These levels **cache instruction bytes**, not decoded uops; they feed the L1i/L1d.
- **Best results come from good codegen and layout.**  
  Compilers already optimize for **compactness**, **layout**, and **predictable control flow**; programmers mainly **choose flags/techniques** that enable these.

**Minimal To-Do for Programmers**
- **Enable/choose compiler options** that favor **code size** and **hot-path layout** (e.g., function reordering, hot/cold splitting).
- **Structure control flow** (reduce unpredictable branches, hoist invariant branches, use likely/unlikely hints if available).
- **Measure** with your workload; ensure any manual unrolling/pipelining **doesn’t blow I-cache** and degrade performance.

#### 3.4.1 Self Modifying Code
- **What & why (then vs now):** In early systems with extremely tight memory, programs sometimes **modified their own code** to save space.
  - Today SMC appears rarely—mostly for niche performance hacks or in **security exploits**.
- **Why to avoid:** SMC **breaks I-cache assumptions** and **hurts performance/correctness**:
  - Decoded-instruction caches (**trace caches**) can’t be used on modified code.
  - If an instruction already **entered the pipeline** and is changed, the CPU may **flush and restart**, sometimes discarding a large amount of internal state.
  - **L1i uses a simplified SI (Shared/Invalid) coherency model**, not full MESI, assuming code pages are immutable; when modifications are detected the CPU must make **pessimistic, costly invalidations**.
- **If you must:** Prefer **separate functions** or alternative designs. If SMC is unavoidable, **bypass the cache on writes** (see section 6.1 in the paper) to reduce I/D-cache interference.
- **How it’s surfaced on Linux:** Normal toolchains **write-protect code pages**; creating writable code sections requires deliberate link-time changes.
  - Modern Intel x86/x86-64 CPUs expose **performance counters** that can **count SMC events**, making SMC usage detectable.

**Key Points**
- **Historical artifact:** SMC arose from **severe memory scarcity**; not a generally valid optimization on modern hardware.
- **Trace cache invalidation:** Because trace caches store **decoded instructions**, modifying code invalidates those entries → **lost decode work** and more front-end stalls.
- **Pipeline disruption:** If an instruction gets modified **after fetch/decode**, the CPU may need to **flush the pipeline** and **rebuild state**, wasting cycles.
- **Instruction-cache coherency model:** L1i commonly uses **SI** rather than **MESI** (based on the near-universal assumption that code is immutable).
  - When SMC is observed, the CPU must **assume worst-case** and perform **expensive invalidations**.
- **Safer alternatives:** Prefer **static code paths** (e.g., function variants) over mutating code bytes at runtime.
  - If absolutely necessary, use **uncached/write-combining** stores for the modifications to **limit D↔I cache hazards** (paper’s §6.1).
- **Detection on Linux:** Writable code needs **non-default linking tricks**; Intel CPUs provide **counters to identify SMC**, aiding observability.

---

### 3.5 Cache Miss Factors
**Why cache misses hurt**
- **Latency gap:** Your text gives a Pentium-M example:  
  Registers ≤ **1** cycle → **L1 ≈ 3** cycles → **L2 ≈ 14** cycles → **DRAM ≈ 240** cycles.  
  Under random access, observed costs can climb to **~450+ cycles** due to added effects (e.g., TLB misses, precharge/activation, limited prefetch), far beyond “nominal” DRAM latency.
- **Bandwidth contention:** When data isn’t sequential, DRAM protocols add **tRCD / CL / tRP / tRAS** gaps; the data bus sits idle between bursts. Concurrent cores/threads and DMA also contend for memory bus bandwidth in the commodity Northbridge/Southbridge designs described earlier.
- **Write traffic doubles pressure:** Modifying data forces **write-backs** of dirty lines; in streaming updates this can effectively **halve usable bandwidth** (read + write sharing the bus).
- **TLB effects:** Page-walks add large, **additive** costs before any cache/DRAM access; with large working sets and poor locality, **TLB misses** amplify latency.
- **Multicore coherency:** MESI/RFO traffic (e.g., two CPUs wanting the **same cache line**) adds stalls and bus traffic; shared buses saturate quickly when several threads prefetch/write concurrently.

**Cost model (as given)**
- A simple execution-time model in the text (single cache level for intuition):
  - **Texe ≈ N · [ Tproc + Fmem · ( Ghit·Tcache + (1−Ghit)·Tmiss ) ]**  
    Where:  
    *N* = instructions; *Fmem* = fraction that access memory; *Ghit* = cache hit rate; *Tcache/Tmiss* = hit/miss costs.
- Implication: even small drops in **Ghit** (hit rate) can dominate runtime because **Tmiss ≫ Tcache**.

**What you can do**
1. Shape access patterns (enable prefetch, fill bursts)
    - **Make accesses predictable & sequential** so hardware prefetch can overlap latency. The text shows sequential scans staying near **~9 cycles/elem** (prefetch hides L2/DRAM), versus **~450+ cycles** for random.
    - **Operate in cache-sized tiles/blocks** so the *working set* fits into L1/L2/L3 while you reuse it (spatial & temporal locality).
    - **Batch loads/stores** to fill whole 64-byte cache lines and DRAM bursts; avoid fine-grained, sparse touches.
2. Reduce misses before they happen
    - **Data layout:** arrange arrays-of-structures vs. structures-of-arrays to minimize useless bytes fetched per line (paper discusses line size = **64 B** typical).
    - **Align data** to cache-line boundaries to avoid crossing lines/pages mid-access.
    - **Avoid pointer-chasing/randomization** in hot loops where feasible; if you must, confine randomness to **local blocks** to limit TLB churn (the text shows big wins when randomization is limited to page blocks).
3. Mind the TLB
    - **Use larger pages when meaningful** so fewer distinct pages are touched during a hot loop; the text explicitly advises using page sizes "as large as meaningful".
    - **Process memory in page-friendly chunks** to keep the TLB’s small, fast cache effective.
4. Writes & bandwidth
    - **Prefer read-mostly sharing; avoid multi-writer sharing of the same cache line.** MESI/RFO turn concurrent writes into expensive ownership transfers.
    - **Stream writes contiguously** so write-backs are efficient and combine well; avoid ping-ponging the same line across cores.
    - For device memory regions, the text notes **write-combining** can help (special policy for non-RAM regions).
5. Multicore/NUMA awareness
    - **Minimize cross-core sharing of hot lines** (each share can trigger invalidations/RFO).
    - **Place data close to the threads that use it** (NUMA locality) and pin threads appropriately to avoid remote memory penalties described in the NUMA section.
    - Expect **sub-linear scaling** once working sets exceed the last-level cache; the figures in the text show speedups collapsing when memory bandwidth saturates.
6. Prefetch consciously
    - **Let hardware prefetch do its job** by writing loops with simple, forward strides.  
    - Where applicable, use **software prefetch** to overlap latency further; schedule prefetch far enough ahead but not across page faults.
7. Measure, don’t guess
    - The paper’s section 7 introduces tools; modern x86 expose **performance counters** for L1/L2 misses, TLB misses, MESI events, and even self-modifying code. Use them to confirm whether you’re bandwidth- or latency-bound.

#### 3.5.1 Cache and Memory Bandwidth
- **Method:** A microbenchmark uses SSE (`movaps`) to *load/store 16 B at a time* while sweeping working-set sizes from **1 KB → 512 MB**, measuring sustained **bytes per CPU cycle** for **read**, **write**, and **copy**.
- **Intel Netburst (P4, 64-bit):**  
  - **Reads:** At L1d sizes, **~16 B/cycle** (one 16 B load per cycle). Past L1d, drop to **<6 B/cycle**; a step at **2¹⁸ B** shows **DTLB** exhaustion; with sequential access & prefetch the FSB streams at **~5.3 B/cycle**. Prefetched data is **not** propagated into L1d.  
  - **Writes/Copy:** Never exceed **~4 B/cycle** even for tiny sets → indicative of **L1d write-through** (limited by L2). Once past L2, **~0.5 B/cycle** (≈10× slower than reads).  
- **Netburst with Hyper-Threading (2 HT threads):**  
  - Both threads share cache/bandwidth; each effectively gets **~½** of the resources; no practical gain because both are memory-wait limited → **worst-case for HT** here.  
- **Intel Core 2 (dual-core, shared L2 ~4× P4’s L2):**  
  - **Reads:** Stay near **~16 B/cycle** across much larger sets; drop appears after **2²⁰ B** (DTLB). This implies **prefetch into L1d** is effective.  
  - **Writes/Copy:** Within L1d, near **~16 B/cycle** → **write-back**, not write-through. Past L1d/L2, write speeds fall; beyond L2, **reads vs writes differ by ~20×**. Still overall **better than Netburst**.  
- **Core 2 with 2 threads (one per core, shared L2):**  
  - **Reads:** Similar to single-thread (minor jitter).  
  - **Writes/Copy (data fits in L1d):** Performance collapses to **main-memory-like** levels because both threads write the **same locations**, triggering **RFO**/coherency traffic; after L1d evicts to **shared L2**, speed improves (served from L2).  
  - **Asymptote:** For large sets, each thread ≈ **½** of single-thread bandwidth due to shared FSB.  
- **AMD Opteron fam 10h (L1d 64 KB, L2 512 KB, shared L3 2 MB):**  
  - **Single-thread:**  
    - **Reads:** Can exceed **32 B/cycle** in L1d (processor can retire **2 loads/cycle**). The read curve **flattens quickly** to **~2.3 B/cycle** (prefetch ineffective here).  
    - **Writes:** ~**18.7 B/cycle** in L1d; then ~**6 B/cycle** (L2), **2.8 B/cycle** (L3), **0.5 B/cycle** (beyond L3).  
    - **Copy:** Bounded by min(read, write); initially read-limited, later write-limited.  
- **Opteron with 2 threads:**  
  - **Reads:** Largely unaffected (per-thread L1d/L2 do their jobs; L3 not heavily prefetched).  
  - **Writes:** **Shared data must traverse L3**; sharing is **inefficient**—even when the working set fits in L3, costs are higher than an L3 hit. Compared with Core 2’s shared L2 behavior, the Opteron’s write-sharing range is smaller and slower.  

**Key Points**
- **Reads can be very fast if prefetching is effective.**  
  Core 2 sustained ~**16 B/cycle** across large ranges because hardware prefetch **streams into L1d**; Netburst’s read stream topped out around **~5.3 B/cycle** with prefetched data **not** landing in L1d.  
- **Write policy matters (write-through vs write-back).**  
  Netburst’s L1d behavior indicates **write-through**, capping writes at **~4 B/cycle** and tanking to **~0.5 B/cycle** beyond L2; Core 2’s **write-back** achieves near-optimal **~16 B/cycle** inside L1d.  
- **DTLB capacity creates visible cliffs.**  
  Steps around **2¹⁸ B** (Netburst) and **2²⁰ B** (Core 2) correspond to **DTLB exhaustion**, adding per-page overhead even in sequential scans.  
- **Write-sharing across cores cripples throughput.**  
  Two cores writing the **same cache lines** trigger **RFO/coherency**; on Core 2, L1d-fitting shared writes behave like main-memory speed until lines reach shared L2; on Opteron, shared writes funnel through **L3** and remain costly.  
- **Architecture details dominate ceilings.**  
  Opteron shows **>32 B/cycle** reads in L1d (2 loads/cycle), **~18.7 B/cycle** writes in L1d, but poorer read prefetching beyond L1; Core 2 favors strong read prefetch and write-back; Netburst is limited by write-through and weaker prefetch integration.  

**Verification & Caveats**
- These are **microbenchmark limits** under **idealized loops** (no compute/use of loaded data for read tests). Real applications will attain **lower** throughput.  
- Values like **16 B/cycle**, **5.3 B/cycle**, **0.5 B/cycle**, **>32 B/cycle**, **18.7 B/cycle**, and DTLB thresholds (**2¹⁸/2²⁰ B**) are **from your provided figures/text** and are **architecture-specific**.  

#### 3.5.2 Critical Word Load
- DRAM sends cache lines in **64-bit chunks**; a **64–128 B** line takes **8–16 transfers** (“burst mode”).
- On a miss, if the needed word is late in the line, you wait longer: each 64-bit beat arrives **~4+ cycles** after the previous.
- Controllers use **Critical Word First & Early Restart**: fetch the needed word first so the CPU can resume while the rest streams in.
- This can’t help during **speculative prefetch** (no “critical” word yet), so layout still matters.
- Measured effect here: with the pointer in the **last** word vs **first**, slowdown is ~**0.7%** once you exceed L2; sequential access suffers slightly more (prefetch interference).

**What’s happening**
- **Burst fill:** Cache lines are filled in-order by default (8 or 16 x 64-bit beats).  
- **Miss penalty depends on offset:** If your critical word is beat #8 (or #16), you wait ~7 (or ~15) beats × **4+ cycles** each after the first beat arrives.
- **Critical Word First / Early Restart:** The memory controller can request the **critical beat first**, returning it early so execution resumes while the remainder fills in the background.
- **Prefetch caveat:** During hardware prefetches, the controller doesn’t know which word is critical; if a demand load arrives mid-prefetch, you may still wait for the original order.

**Practical takeaways**
- **Put what you touch first, first.**  
  Place the most-used pointer/field at the **start** of a 64 B cache line (e.g., structure field order, padding).
- **Align & pack thoughtfully.**  
  Cache-line align arrays of structs; avoid pushing the hot field to the tail with accidental padding.
- **Pointer-chasing layouts:**  
  For linked structures where `next` is read immediately, keep `next` in the **first 8 bytes**.
- **Beware prefetch overlap:**  
  In tight sequential scans where HW prefetch is active, the ~**0.7%** penalty for “hot field last” can show up more.
- **It’s a small %… that compounds.**  
  ~1% on a single path can add up across hot loops, multiple levels, and NUMA hops.

#### 3.5.3 Cache Placement
**What shares what?**
- **Hyper-threads (SMT):** share *everything but registers* → **L1i/L1d are shared**.
- **Cores on the same CPU:**
  - *Early multi-core:* no shared caches beyond each core’s L1.
  - *Intel Core 2 (dual-core):* **shared L2** across both cores.  
    *Quad-core variants:* two independent pairs, each pair with its own L2; **no L3**.
  - *AMD Family 10h:* **private L2 per core + shared L3** (unified).
 
**Pros & cons of sharing**
- **No shared L2/L3:** good when working sets don’t overlap; less contention.  
  Downsides: duplicated common code/data (e.g., libc hot paths) wastes cache.
- **Shared higher-level cache (e.g., Intel shared L2, AMD shared L3):**
  - Big win if threads **share/overlap data** → effectively larger pool.
  - But “smart” partitioning can mis-evict and **shrink effective per-core cache** when workloads compete.

**A telling experiment (shared L2 under pressure)**
- One core continuously writes a fixed **2 MiB** region (½ of a 4 MiB L2).  
- Second core varies its working-set size and reads/writes another region (no sharing).
- **Observation:** performance degrades **before** the 1 MiB mark per core; with a writer in the background, the measured core behaves as if it has **≈1 MiB** effective L2, not 2 MiB.  
  - Pure **eviction pressure** from the sibling core; no RFO traffic (no shared lines).

**Quad-core note (older Core 2 quads)**
- Implemented as **two dual-core pairs**: each pair shares an L2; inter-pair traffic goes over the **FSB** (no special fast path).  
- Net: little advantage over two dual-socket dual-cores for cache-sharing behavior.

**Where this is going**
- More layers: **private L2 + shared L3** is a practical balance.  
- As cores per socket rise, **cache size & associativity must scale** with sharers; L3 is slower, but that’s acceptable if L2 hits remain high.

**Practical affinity hints (architecture-aware)**
- **Heavy sharing/overlap:** place threads on cores that **share L2/L3** to increase effective cache footprint.
- **Independent, bandwidth-heavy or write-heavy:** **separate** onto cores that **don’t share** the next cache level (different L2; if possible, different L3/socket) to avoid thrashing.
- **SMT caution:** avoid putting two **memory-bound** threads on the **same core** (they fight for L1/L2 and ports).
- **Mixed workloads:** pair a memory-bound thread with a compute-bound thread on the **same shared cache** to exploit complementary use.
- **Know your topology:** query it (e.g., Linux `/sys/devices/system/cpu/`, `lscpu`, `hwloc`) and set **CPU affinity** accordingly.

**Quick checklist**
- Do my threads **share data?** → co-locate on **shared L2/L3**.
- Do they **stream/write a lot** with little sharing? → **isolate** across caches/sockets.
- Am I using **hyper-threads** for memory-bound code? → usually **avoid**; prefer real cores.
- Is effective cache smaller than advertised? → expect **contention** on shared caches; size for **effective**, not theoretical, capacity.

#### 3.5.4 Front Side Bus (FSB) Influence
- The Front-Side Bus (FSB) is the bandwidth bottleneck once your working set exceeds cache.
- In tests on two otherwise identical Core 2 systems—one with **DDR2-667** and one with **DDR2-800** (≈+20% clock)—the memory-bound workload achieved up to **~18.2%** speedup at large working sets.
- Faster RAM/FSB matters far less when the working set fits in cache (e.g., 4 MB L2).

**Why it Matters**
- **Memory-bound phases** saturate the bus → higher FSB/DRAM data rate = higher sustained throughput.
- **Cache-resident phases** hide DRAM latency → FSB gains don’t translate to runtime wins.

**When Faster FSB Helps Most***
- Working set **≫ last-level cache** (L2/L3).
- Multiple processes/threads all **stress memory** concurrently.
- Patterns with **high read/write bandwidth** (streaming copies, stencil updates, large scans).

**When It Helps Least**
- Hot data **fits in L2/L3**.
- Workloads are **compute-bound** or have excellent locality.
- Tight loops dominated by **ALU/FPU** with few cache misses.

**Practical Guidance**
- If memory-bound, **buy bandwidth**: higher-speed DIMMs, **more channels**, and larger **LLC** (L3/L2) if available.
- Keep buses uncongested in software:
  - Improve **spatial/temporal locality**.
  - Use **prefetch** sensibly; batch writes; reduce cross-core **cache ping-pong**.
- Verify platform support:
  - **CPU**, **northbridge/motherboard**, and **DIMMs** must all support the higher effective FSB.
  - Mind timings (**CL/tRCD/tRP**) and population rules (channels/slots).

**Example Result (Representative)**

| System | DRAM Speed | Effective FSB | Observed Gain (large WS) |
|---|---|---|---|
| Core 2 A | DDR2-667 | 667 MHz | baseline |
| Core 2 B | DDR2-800 | 800 MHz | **~+18.2%** |

- **Takeaway:** For large, memory-bound workloads, faster FSB/DRAM yields near-proportional speedups; otherwise, gains are minimal. Always check motherboard support and overall memory topology before upgrading.

[back to top](#table-of-contents)

## 4 Virtual Memory
- Virtual memory (VM) gives each process its own address space; this section focuses on **how** it’s implemented and what it **costs**.
- The OS builds and maintains **page tables**; the CPU’s **MMU** performs per-access translation from **virtual** to **physical** addresses.
- Virtual addresses are **32-bit** on 32-bit systems and **64-bit** on 64-bit systems.
- On x86/x86-64, **segmentation** can add a fixed offset to logical addresses, but it’s trivial for performance discussions here.

**Key Points**
- **Division of labor**
  - **OS:** populates page-table data structures.
  - **MMU/CPU:** executes the translation on every memory reference.
- **Address types**
  - **Virtual address:** what programs use.
  - **Physical address:** where data actually resides in RAM.
- **Bit widths**
  - 32-bit systems → 32-bit virtual addresses.
  - 64-bit systems → 64-bit virtual addresses (architecture may not use the full 64 bits).
- **Segmentation on x86/x86-64**
  - Acts as an additional indirection by adding a **segment base** to a logical address.
  - Considered **trivial/ignorable** here with respect to VM performance.

---

### 4.1 Simplest Address Translation
- The MMU translates **virtual addresses** to **physical addresses** on a **page-by-page** basis.
- A virtual address is split into fields that **index page tables** and an **offset** within the page.
- In the simplest (single-level) model, a **Page Directory** maps virtual pages to **physical page frames**; the **offset bits** are carried through unchanged.
- Example (x86, 4 MB pages): **10-bit directory index** + **22-bit page offset** → physical address = **directory entry base (10 b)** ∥ **offset (22 b)**.

**Key Points*
- **Remap granularity:** per page; multiple directory entries may point to the **same physical page**.
- **VA split (4 MB pages on x86, 32-bit VA):**
  - `DirIndex`: top **10 bits** → select one of **1024** Page Directory entries.
  - `Offset`: low **22 bits** → byte offset within the 4 MB page.
- **Page Directory in RAM:** OS allocates **contiguous physical memory** for it and stores its **base address** in a special CPU register.
- **Directory entry contents:** physical page **base address bits** (for 4 MB pages: **10 bits**) **+** **access/permission flags**.
- **Physical address construction:** concatenate the **directory entry’s base** with the **offset**.

**Address Fields (ASCII)**
Virtual Address (32-bit)

|10 bits | 22 bits|
|---|---|
|Page-Directory Index (PDI)|Page Offset within 4 MiB|
|`VA[31:22]`|`VA[21:0]`|

**Example**
- `VA = 0x12345678`
  - `PDI = 0x48` (`VA >> 22`)
  - `Offset = 0x00345678` (`VA & 0x3FFFFF`)
- Suppose `PD[0x48].Base = 0x0C000000` (4 MiB aligned)
- **Result:** `PA = 0x0C000000 | 0x00345678 = 0x0C345678`

**Notes**
- Real systems usually use multi-level paging with **4 KiB** pages; the **PS=1** large-page mode simplifies to one PD lookup.
- L1 caches often tag by **virtual** address for speed; L2/L3 by **physical**.
- TLB caches VA→PA translations to avoid repeating the walk.

---

### 4.2 Multi-Level Page Tables
- With 4 KiB pages the page-offset is **12 bits**, leaving too many upper bits to index a single, flat page table (it would be ~**4 MiB** per process).
- Modern CPUs therefore use **multi-level page tables** (often **4 levels**): each VA is split into several **directory indexes** plus a **page offset**.
- The MMU **walks** the levels (L4→L3→L2→L1) following pointers until the **L1 entry** yields the physical page frame; the **offset** supplies the low PA bits.
- Because unused VA regions have no tables allocated, the structure is **sparse** and memory-efficient.

**Key Points**
- **Why multi-level?**  
  - 4 KiB pages ⇒ **12-bit offset**; a single-level directory for all other bits would be enormous (e.g., 2²⁰ entries × 4 B = **4 MiB** per process).
  - Multi-level trees allocate tables **only where needed**.
- **Typical 4-level split (x86-64, 4 KiB pages, 48-bit canonical VA):**  
  - **PML4** index: 9 bits  
  - **PDPT** index: 9 bits  
  - **PD** index: 9 bits  
  - **PT** index: 9 bits  
  - **Offset**: 12 bits  
    *(9+9+9+9+12 = 48)*
- **Coverage & table counts (4 KiB pages, 512 entries per table):**  
  - 1 **PT** (L1) maps **512 × 4 KiB = 2 MiB**.  
  - To map **2 MiB** contiguously: **1 PT + 1 PD + 1 PDPT + 1 PML4** (total **4** tables).  
  - To map **1 GiB** contiguously: **512 PTs** (for 512 × 2 MiB) + **1** each of PD, PDPT, PML4.
- **Walk mechanics (hardware “page walk”):**  
  1. Load **PML4 base** from a CPU register (e.g., CR3-derived).  
  2. Use VA’s PML4 bits to index PML4 ⇒ get **PDPT base**.  
  3. Index PDPT ⇒ get **PD base**.  
  4. Index PD ⇒ get **PT base**.  
  5. Index PT ⇒ get **page frame base**.  
  6. **PA = frame_base | VA.offset**.  
      *(Some ISAs do this in hardware; others may involve the OS.)*
- **Sparsity & layout effects:**  
  - If a directory entry is **empty**, lower levels **don’t exist**.  
  - Real processes often need multiple higher-level tables: stacks and heaps are far apart; **ASLR** spreads code/DSOs/heap/stack, increasing the number of allocated tables.
- **Performance considerations:**  
  - **TLB** caches VA→PA translations; a TLB hit avoids a multi-memory-read walk.  
  - **ASLR** improves security but can **reduce contiguity**, growing table count and TLB pressure. If performance trumps security, ASLR can be reduced/disabled so DSOs load contiguously (trade-off!).

**Minimal table footprint examples**
- **2 MiB mapping:** 1×PML4 + 1×PDPT + 1×PD + 1×PT = **4 pages** of metadata.
- **1 GiB mapping:** 1×PML4 + 1×PDPT + 1×PD + **512×PT** = **515 pages** of metadata.

**Notes**
- Larger pages (e.g., 2 MiB “huge pages”) can reduce walk depth/pressure, but 4 KiB remains common for flexibility.
- Flags (Present, R/W, U/S, NX, cacheability) live in directory/table entries at each level.

---

### 4.3 Optimizing Page Table Access
- Walking multi-level page tables for every virtual→physical address translation is far too slow (each access could require multiple serialized memory reads).
- CPUs therefore cache **the final translation** (physical page frame + attributes) in a **Translation Look-Aside Buffer (TLB)** keyed by the virtual page number.
- On a hit, the MMU forms the physical address by appending the original page **offset**—fast enough to feed the pipeline.
- Misses trigger a **page walk**, which is expensive; hence TLB size, associativity, and locality matter a lot.

**Key Points**
- **Why not just cache page-table entries in L1D?**  
  A 4-level walk needs up to **4 dependent memory reads** per access. Even if they all hit L1D, you’d burn double-digit cycles and saturate cache bandwidth.
- **What the TLB caches:**  
  The **physical page number (frame)** plus permissions/flags, keyed by the **virtual page number** (the VA with the offset bits removed). **Offset** bits bypass the TLB and are spliced on after the hit.
- **Hit vs. miss cost:**  
  - **Hit:** address formed in a few cycles; often fully overlapped with the pipeline.  
  - **Miss:** hardware (or OS, depending on ISA) performs a **page walk** (multi-level), which is **slow** and competes for cache/bandwidth.
- **Associativity & levels:**  
  - **L1 TLBs** are tiny and very fast; historically **fully associative** (now often set-associative as sizes grow).  
  - **L2 TLB** (if present) is **larger, slower, unified** (covers both code and data).  
  - Separate **ITLB** for instructions and **DTLB** for data at L1.
- **Prefetching & the TLB:**  
  Hardware data prefetchers **must not** trigger page walks (they can’t safely fault), so they **do not prefetch TLB entries**. If you need it, use **explicit prefetch-with-NT/TA** style instructions that are allowed to populate translations (ISA/CPU-dependent).
- **Pipeline implications:**  
  TLB lookups sit on the critical path to L2 (when L2 is PA-tagged). Misses bubble through the pipeline; long page walks are hard to hide.
- **Locality matters (again):**  
  - **Spatial locality:** reusing addresses within the same page reuses a single TLB entry.  
  - **Working set of pages** should fit the TLB to avoid thrashing (just like cache sets/lines).

**Practical Tips**
- **Cluster hot data/code** into fewer pages; fewer distinct pages ⇒ fewer TLB entries needed.  
- Consider **larger pages** (e.g., 2 MiB/1 GiB) for hot, contiguous regions to cut TLB pressure (trade-offs: fragmentation, permissions granularity).  
- **Avoid needless VA aliasing** of the same physical memory inside one process (can multiply TLB demand).  
- When prefetching, prefer **documented prefetch ops** that can legally help translations (where available).

**Common Pitfalls**
- Assuming hardware prefetchers “warm” the TLB—**they don’t**.  
- Large, sparse heaps / heavy ASLR spreads increase the number of active pages ⇒ **TLB miss spikes**.  
- Ignoring instruction-side pressure: hot code scattered over many pages can thrash the **ITLB**, too.

**Terminology**
- **TLB:** Cache of VA→PA translations.  
- **ITLB/DTLB:** Instruction/Data TLB (L1).  
- **L2 TLB:** Larger, unified TLB (if present).  
- **VPN/PFN:** Virtual/Physical Page Number.  
- **Page walk:** Reading page-table levels to resolve a translation.

#### 4.3.1 Caveats Of Using a TLB
- The **TLB is per-core and shared** by all threads/processes running on that core.
- Because translations depend on the **current page-table tree**, cached TLB entries may become invalid across context switches.
- Two strategies exist:
  1. **Flush the TLB** on switch, or
  2. **Extend TLB tags** with an identifier for the address space (ASID/PCID-like), so entries remain valid across switches.
- **Flushing is simple but costly** (destroys many still-useful entries).
  - **Tagging preserves translations**, reducing misses—especially when entering/exiting kernel/VMM frequently or switching between threads of the same process.

**Key Points**
- **Why flushes hurt**
  - Full TLB flush on kernel/VMM entry/exit or process switch discards many entries that will be needed immediately again.
  - Example scale: a Core 2 with ~**128 ITLB** and **256 DTLB** entries loses >100/200 entries per flush.
- **Selective invalidation helps**
  - OS can invalidate **only the affected address ranges** (e.g., after `munmap`) instead of flushing everything.
- **Extended TLB tags (address-space IDs)**
  - Tag each TLB entry with a **unique identifier for the page-table tree**.
  - Benefits:
    - **Kernel/VMM entries** stay hot across frequent transitions.
    - **Thread switches within the same process** need **no TLB flush**.
    - **Process switches** can avoid full flush if tags distinguish address spaces.
  - **Tag space is limited** → identifiers may be **reused**, requiring **partial flush** of entries carrying the reused ID.
- **Virtualization angle**
  - AMD’s Pacifica introduced a **1-bit ASID** to distinguish **VMM vs. guest** address spaces, avoiding needless flushes when entering/leaving the VMM.
  - More tag bits in the future → fewer forced partial flushes.
- **Why TLBs aren’t (much) bigger**
  - If frequent flushes wipe entries, **very large TLBs show diminishing returns** (they’re often not resident long enough to matter).

**Practical implications**
- Systems **without extended tags**: expect higher TLB miss rates around syscalls, interrupts, VMM exits/entries, and process switches.
- With **ASID-like tags**: fewer stalls re-filling translations; better syscall-heavy and virtualization performance.

**OS/VM design notes**
- Prefer **overlayed/consistent kernel mappings** to reduce churn.
- Use **targeted TLB shootdowns/invalidation** for changed regions, not global flushes.
- In virtualized setups, leverage **distinct IDs** for guest vs. hypervisor to avoid thrashing.

**Programmer takeaways**
- Fewer distinct hot pages → less TLB pressure after switches.
- Minimize unnecessary address-space changes that force invalidations.

#### 4.3.2 Influencing TLB Performance
- Larger pages reduce TLB pressure (fewer translations and misses) and can shrink page-walk cost by shortening page-table depth, but they require physically contiguous memory and can waste RAM via alignment/fragmentation.
- On Linux, huge pages typically need pre-reservation (e.g., `hugetlbfs`).
- Raising the *minimum* page size is constrained by binary/ABI alignment (e.g., ELF `p_align`).
- You can also lower TLB pressure by colocating concurrently used data onto fewer pages.

**Key Points**
- **Bigger pages → fewer TLB entries**
  - Fewer translations; better TLB hit rate.
  - Larger **offset** eats more VA bits → shallower page walks on TLB miss.
- **Common sizes & coexistence**
  - x86/x86-64: base **4 KB** plus **2 MB** / **4 MB** options.
  - Other ISAs (e.g., IA-64/PowerPC) support larger base sizes (e.g., **64 KB**).
  - Multiple sizes can be **used concurrently**.
- **Costs of large pages**
  - Must be **physically contiguous**; allocator must find big runs (e.g., 2 MB = 512×4 KB).
  - **Fragmentation** over uptime makes late allocation hard.
  - **Internal fragmentation**: average ~½ page wasted per mapping due to page-aligned regions.
- **Linux practice: huge pages**
  - Reserve at boot or early via **`hugetlbfs`** (fixed pool).
  - Ties down RAM; expanding pool may require **reboot**.
  - Suits steady, RAM-rich, perf-critical apps (e.g., **databases**).
- **Raising the *minimum* page size**
  - All mappings must honor the larger granularity; not always feasible.
  - **Binary layout constraints** (e.g., ELF `p_align`) cap usable page sizes (e.g., 2 MB on x86-64 example).
- **Layout to reduce TLB pressure**
  - **Group hot data/code** that’s used together onto as **few pages** as possible.

**When to use huge pages**
- Large, long-lived heaps; working sets ≫ TLB reach; predictable allocation; uptime where boot-time reservation is acceptable.

**Operational tips**
- Pre-reserve ample huge pages; monitor usage to avoid starvation.
- Prefer **transparent huge pages** where safe, but validate latency/fragmentation impacts.
- For binaries, verify **ELF program header `p_align`** vs. intended page size.

**Rule of thumb**
- If TLB misses or page-walks show up in profiles, try **bigger pages** and/or **better colocation** before deeper surgery.

---

### 4.4 Impact Of Virtualization
- Full OS virtualization adds another layer of address translation and bookkeeping, which makes cache and TLB misses costlier than on bare metal.
- Classic Xen uses a separate VMM with shadow page tables (expensive), while hardware assists (Intel EPT / AMD NPT) replace shadowing with nested page tables and let the TLB cache final translations.
- Tags like ASID/VPID reduce some TLB flushes but not all.
- KVM integrates the VMM into the Linux kernel, reusing a single, mature memory manager instead of duplicating it.
- Even with assists, virtualization overhead remains—so reducing misses and mapping churn pays extra dividends.

**Key Points**
- **Containers vs. VMs**
  - Containers/jails share one OS → no extra memory-translation layer.
  - VMs (Xen/KVM) run independent OS images → add a second layer.
- **Xen model (Dom0/DomU + VMM)**
  - VMM controls physical memory; Dom0 manages most devices.
  - Guest page-table updates trap to VMM to update **shadow page tables** → high overhead in time and memory.
- **Hardware assists**
  - **Intel EPT / AMD NPT**: guest page tables map GVA→“host virtual”, VMM tables map to HPA; removes shadowing and most traps.
  - TLB caches **final** (nested) translation results.
  - **ASID/VPID** avoid full TLB flush on VM entry/exit (good), but are typically per-domain, not per-process → process switches in guests can still flush.
- **KVM approach**
  - No separate VMM: Linux kernel is the hypervisor; user-space `kvm` process controls a guest.
  - Reuses Linux’s memory manager → less duplication, fewer bugs, simpler NUMA/memory discovery.
- **Performance implications**
  - Cache (I/D) and **TLB** misses cost even more under virtualization.
  - EPT/NPT narrows but does not eliminate the gap to bare metal.

**Practical guidance**
- Use **huge pages/THP** in host and guests to cut TLB pressure.
- Minimize page-table churn: batch (un)maps, reduce frequent `mmap/munmap/brk` changes.
- Keep hot working sets compact; colocate hot data/code; reduce context switches.
- Be **NUMA-aware**: align vCPUs, guest memory, and devices to nodes.
- Pin vCPUs sensibly; avoid overcommit that thrashes TLBs/caches.
- Prefer paravirtual I/O to reduce VM exits.

**Observe & tune**
- Track VM-exit causes, EPT/NPT violations, TLB miss rates, page-fault sources.
- Validate with microbenchmarks (latency of loads/stores, page-walk counters) and end-to-end workloads.

**Caveats**
- ASID/VPID scope limits per-process TLB retention inside a guest.
- Shared last-level `cache / memory` bandwidth contention still apply in multi-VM hosts.
- Even with assists, nested translations add latency—optimize like it matters (because it does).

[back to top](#table-of-contents)

## 5 NUMA Support
- On Non Uniform Memory Access (NUMA) machines, memory access latency/bandwidth depend on which CPU socket (“node”) touches which region of physical RAM.
- Local-node accesses are faster; remote-node accesses traverse an interconnect and are slower.
- To get good performance, both the OS and applications must place threads and data with locality in mind.
- Linux exposes NUMA topology and provides schedulers, memory policies, and APIs/tools to control placement and measure effects.

**Key Points**
- **NUMA Basics**
  - System is split into **nodes** (CPU cores + memory controller + RAM).  
  - **Local** access: lowest latency, highest bandwidth; **remote** access: costlier (the “NUMA factor”).  
  - Interconnects (e.g., QPI/UPI/Infinity Fabric/mesh) add hops → more latency, contention.
- **Why it Matters**
  - Remote DRAM reads/writes and cross-node cache-line ownership (coherency traffic) inflate stalls.  
  - Bandwidth is finite per node/channel; concurrent nodes can throttle each other.
- **Linux Support (what you can use)**
  - **Discovery**: `lscpu --extended`, `numactl --hardware`, `lstopo`, `/sys/devices/system/node/`.
  - **Scheduling & AutoNUMA**: kernel can migrate pages toward where threads run; tunable via `/proc/sys/kernel/numa_balancing`.
  - **Placement Policies** (per-thread/region):
    - **first-touch** (default), **preferred**, **bind** (strict node set), **interleave** (spread across nodes).
    - Syscalls/APIs: `mbind(2)`, `set_mempolicy(2)`, `get_mempolicy(2)`, `move_pages(2)`; **libnuma** helpers like `numa_alloc_onnode`, `numa_set_membind`.
  - **Affinity**: pin threads/IRQs/devices to nodes with `taskset`, `numactl`, cpusets cgroup, `/proc/irq/*/smp_affinity`.
  - **Huge pages**: THP or per-node hugepages reduce TLB pressure; allocate per node.
  - **I/O NUMA**: PCIe devices have a node; place I/O threads on that node to avoid remote traffic.
  - **Observability**: `numastat`, `/proc/<pid>/numa_maps`, `perf`/PMU counters, `hwloc-ps`.
- **App-Level Practices**
  - **First-touch initialize** data from the thread/node that will use it.  
  - **Shard by node**: partition data & queues so threads mostly touch local memory.  
  - **Avoid cross-node sharing** (esp. writes); replicate read-mostly data per node.  
  - **Pin wisely**: keep producer/consumer pairs on the same node; align NIC/NVMe handlers with their device node.  
  - **Use huge pages** for large working sets; batch cross-node communication.

**Gotchas**
- BIOS “memory interleaving” can mask NUMA—disable if you need locality.  
- Memory pressure can override policies → unexpected remote allocations.  
- AutoNUMA/page migration helps or hurts depending on access stability—measure.  
- vNUMA in VMs: align vCPUs and guest memory to host nodes; avoid cross-socket placement.

**Quick Checklist**
1. Inspect topology (`numactl -H`, `lstopo`).  
2. Pin threads to nodes/cores; place memory with `numactl`/libnuma.  
3. Initialize on the node that will use the data (first-touch).  
4. Keep I/O and its workers on the same node.  
5. Validate with `numastat`, `/proc/*/numa_maps`, and `perf` before/after changes.

---

### 5.1 NUMA Hardware
- Non-Uniform Memory Access (NUMA) systems make some memory “closer” (lower latency, higher bandwidth) to a given CPU than other memory.
- Small NUMA factors appear in simple multi-socket designs with local DRAM per socket; large shared-memory machines use more elaborate interconnects where topology and contention can raise the NUMA factor.
- OSes and applications must account for this to avoid bottlenecks.

**Key Points**
- **Why NUMA exists**
  - A single shared memory controller (e.g., classic Northbridge) becomes a bottleneck as CPU counts grow.
  - DRAM is (effectively) single-ported; multi-port RAM is costly → architects distribute memory per CPU/socket and connect sockets.
- **Simple NUMA (e.g., per-socket local memory)**
  - Accessing local memory is cheaper than remote; NUMA factor is relatively low.
  - Still requires software awareness to keep threads/data local.
- **Interconnects & Topologies**
  - **HyperTransport/other links** connect sockets so non-local memory is reachable.
  - **Hypercube topology**: with **C** links per node, you can directly build up to **2^C** nodes with **diameter = C** (minimal maximum hop count).
    - Early Opterons: **3 HT links** per CPU; one often used for the Southbridge → practical **C=2** hypercube.
    - Next gens with **4 links** enable **C=3** hypercubes.
  - **Crossbars/fabrics** (e.g., Newisys Horus) allow larger systems but add latency and can raise NUMA factors; benefits taper beyond a size.
- **Large Shared-Memory Machines**
  - **IBM x445-style**: connect multiple commodity servers to appear as one shared-memory system; interconnect introduces a notable NUMA factor that software must handle.
  - **SGI Altix (NUMAlink)**: high-bandwidth/low-latency fabric for HPC; still expensive, and with many CPUs the effective NUMA factor becomes **dynamic** (contention-dependent).
- **What is *not* NUMA**
  - Clusters of commodity boxes connected over high-speed networks lack a shared address space → not NUMA (use message passing instead).

**Design/Scaling Implications**
  - More nodes/links reduce graph diameter but increase coherence traffic and contention risk.
  - Even with fast fabrics, remote access and cross-socket cache ownership are costly; locality remains king.
  - Software strategies (thread/data placement, first-touch, per-node sharding) are essential to keep NUMA factors low in practice.

---

### 5.2 OS Support for NUMA
- Operating systems must respect memory locality on NUMA machines: keeping a process’s pages near the CPU(s) running it reduces latency and bus contention.
- Because real workloads vary and memory pressure can be uneven across nodes, OSes often default to **striping (interleaving)** allocations across nodes to balance capacity and keep migration flexible—trading some peak performance for predictability.
- Linux lets each process override this default with NUMA-aware policies.

**Key Points**
- **Local-first is ideal (but tricky):** If a task runs on CPU node *N*, its code/data should come from node *N*’s RAM to avoid remote-access penalties.
- **Shared DSOs (e.g., libc):** Text pages are typically single-copy system-wide; with many CPUs, most will read them remotely. In theory the OS could **mirror** hot DSOs per node, but this is hard and usually only partially supported.
- **Avoid thread/process migration:** Moving a runnable entity to another node loses cache warmth and, on NUMA, raises memory access costs if its pages remain behind.
- **When migration is unavoidable:** Two paths:
  - **Temporal hope:** Move the task and plan to move it back when pressure subsides.
  - **Page migration:** Move its memory to the new node. This is expensive (copying lots of pages, brief stops for correctness) and should be used sparingly.
- **Uneven memory demand is normal:** If every allocation were strictly local, nodes hosting big jobs would exhaust local RAM while others sit idle.
- **Striped (interleaved) default:** To utilize total system RAM and make migrations “cost-neutral” on average, OSes commonly interleave pages across nodes. With **small NUMA factors** this is acceptable but still not optimal for performance-sensitive code.
- **Linux policy controls:** Linux allows per-process selection of NUMA allocation policies (and inheritance to children), so performance-critical apps can opt out of striping and favor locality.

**Practical guidance**
- Pin threads near their data; avoid migrations that cross nodes.
- Use local allocation for long-lived, node-bound workloads; prefer interleave when capacity balance and mobility matter more.
- Reserve page migration for long-lived retargeting; it’s costly and briefly stops the task.
- Don’t rely on DSO mirroring being automatic or comprehensive; design for locality regardless.

---

### 5.3 Published Information
- Linux exposes detailed cache and topology info in **sysfs** so you can discover which caches are shared, how cores are grouped, and how NUMA nodes map to CPUs and relative memory costs.
- Reading these files lets you bind threads wisely and pick memory policies that match the hardware (e.g., avoid cross-node hot data).

**Key Points**
- **Cache topology**
  - Path: `/sys/devices/system/cpu/cpu*/cache/index*/{type,level,shared_cpu_map}`
  - **Core 2 QX6700 example:** Each core has private **L1i** and **L1d** (shared map = one bit). **L2** is **shared by core pairs** (shared map shows two bits per pair).
  - **Dual-core Opteron (4 sockets) example:** Each core has private **L1i/L1d/L2** (shared maps are single-bit; no shared L2).
- **CPU/package/core layout**
  - Path: `/sys/devices/system/cpu/cpu*/topology/`
  - Useful files: `physical_package_id`, `core_id`, `thread_siblings`, `core_siblings`.
  - Reading these shows: number of sockets, cores per socket, and whether **SMT** is present (multiple bits in `thread_siblings` ⇒ hyper-threads).
- **NUMA nodes and distances**
  - Path: `/sys/devices/system/node/node*/{cpumap,distance}`
  - `cpumap` lists CPUs in each node; `distance` gives **relative memory access cost** per node (e.g., `10` local vs `20` remote).
  - In the Opteron example: 4 nodes (one per socket), uniform remote cost (all remotes = 20), implying equal penalty to any other node.

**Practical usage**
- Pin threads so that hot data stays within the **shared cache group** (e.g., same L2 pair on Core 2).
- On NUMA, pair `taskset`/`sched_setaffinity` with `numactl --membind/--interleave` based on `cpumap` and `distance`.
- Detect SMT via `thread_siblings`; avoid co-locating memory-intensive threads on the same SMT pair unless it helps latency hiding.
- Automate discovery: parse `shared_cpu_map` to build cache-sharing groups; parse `node*/cpumap` to build NUMA sets and weight by `distance`.

---

### 5.4 Remote Access Costs
- **NUMA distance matters.** Remote memory accesses (extra hops) measurably slow both reads and writes; topology (who’s 0/1/2 hops away) drives real performance.
- **You can *see* placement and distances in Linux.
  - ** `/sys/devices/system/node/node*/{cpumap,distance}` exposes node→CPU maps and relative access costs; `/proc/<pid>/numa_maps` shows where a process’s pages actually live.
- **Measured penalties:** Reads across nodes are slower (e.g., ~20% slower for 1-hop in practice), writes degrade further and are topology-sensitive.
  - Keeping threads and memory co-located pays off.

**Key Points**
- **AMD 4-socket NUMA costs (per AMD doc [1], Figure 5.3):**
  - 2-hop **reads** ≈ **+30%** vs 0-hop reads.
  - 2-hop **writes** often quoted two ways:
    - ≈ **+32%** vs 0-hop writes and **+17%** vs 1-hop writes.
    - Also cited as **+49%** vs 0-hop **reads** (different baseline; keep baselines straight).
  - Takeaway: more hops ⇒ higher write penalty than read penalty.
- **Topology matters:**
  - With **4 coherent HyperTransport links**, a 4-socket box can have **diameter 1** (every node 1 hop).  
  - With **8 sockets**, hypercube diameter becomes **3** → more 2-/3-hop traffic and higher penalties.
- **What Linux shows you:**
  - `/sys/devices/system/node/node*/distance` → matrix of relative costs (e.g., `10` local, `20` remote).
  - `/proc/<pid>/numa_maps` → page counts per node for each mapping (`N0…N3`), so you can confirm placement (e.g., code pages on N1/N2, dirty/heap on N3 ⇒ process likely ran on node 3).
- **Measured reality (Figure 5.4 style test):**
  - **Reads:** ~**20% slower** for 1-hop remote vs local across working-set sizes.
  - **Writes & copies:** ~**20% slower** while data fits in caches; for large working sets, **remote ≈ local** (DRAM latency dominates, interconnect keeps up).

**What to do in practice**
- **Pin threads** to the node that owns their hot data; or **bind memory** to the thread’s node (`numactl --cpunodebind=… --membind=…`).
- **Audit placement:** read `/proc/<pid>/numa_maps` during load to verify `N*` distributions align with CPU affinity.
- **Avoid 2-hop paths** for write-heavy shared data structures; shard or replicate read-only pages near consumers.
- When balancing, remember **write traffic** is more topology-sensitive than reads—prefer moving compute to data over moving data to compute.

**Quick checklist**
- [ ] Check `node*/distance` to grok penalties.
- [ ] Group cooperating threads within the same node.
- [ ] Place write-hot memory local to its writers.
- [ ] Verify with `numa_maps`; adjust with `numactl` or mbind/policy APIs.

[back to top](#table-of-contents)

## 6 What Programmers Can Do
- There are numerous ways programmers can affect performance—especially via **memory behavior**.
- The upcoming material will take a **bottom-up path**: from **physical RAM & L1 caches** through higher cache levels, up to **OS-level memory controls**.
- Focus stays on **memory-related optimizations**, since they often dominate runtime.

**Key Points**
- **Programmer impact is large**: small choices in data layout and access patterns can help—or hurt—significantly.
- **Scope**: only **memory-centric** opportunities (not general CPU or I/O), but spanning hardware to OS interfaces.
- **Structure**: begin with **low-level access (RAM/L1)** → **cache usage/prefetching** → **TLB/page size** → **NUMA placement** → **allocator/OS policies**.
- **Goal**: convert theory (DRAM, caches, TLBs, NUMA) into **practical techniques** that raise sustained throughput and lower latency.

**Mindset**
- treat performance as **end-to-end memory pipelines**, not isolated tricks.

**What to watch**
- cache miss rates, TLB misses, bandwidth vs. latency, NUMA locality, write-back traffic.

---

### 6.1 Bypassing the Cache
- **Non-temporal (streaming) stores/loads** let you move data without polluting caches when you won’t reuse it soon.
- **Streaming stores** bypass read-for-ownership and rely on **write-combining** to emit full cache lines directly to memory.
- In a 3000×3000 matrix init:
  - **Row-major (sequential) writes**: ~0.048 s with normal or non-temporal stores (write-combining hides the cost).
  - **Column-major (strided/random) writes**: normal ~0.127 s vs. non-temporal ~0.160 s (no combining → slower).
- Newer CPUs add **streaming loads** (e.g., `movntdqa`) to read without filling caches, useful for one-pass reads.

**Key Points**
- **When to use streaming stores**
  - Data is **produced once** and **not reused soon** (e.g., large zero/init buffers, final output frames, logging buffers).
  - Replace `_mm_store_*` with `_mm_stream_*` variants (e.g., `_mm_stream_si128`, `_mm_stream_ps`).
- **Alignment & grouping matter**
  - Align destinations to **16 B (SSE)**; issue stores **back-to-back for the same cache line** so write-combining can fire.
  - Example fills one 64 B line with four 16 B `_mm_stream_si128` stores.
- **Ordering/fences**
  - Non-temporal writes use **relaxed ordering**; insert an **SFENCE** (`_mm_sfence()`) before visibility assumptions/timing.
- **Sequential vs. strided**
  - **Sequential** NT stores can match cached throughput (controller combines writes → no RFO).
  - **Strided/random** NT stores often lose: each touch may act like an isolated write (no combining, more DRAM activates).
- **Don’t use NT when you will reuse soon**
  - E.g., inner kernels of **matrix multiply** reuse outputs quickly—keep normal cached stores.
- **Streaming loads (SSE4.1)**
  - `_mm_stream_load_si128` (`movntdqa`) pulls a line into a small **streaming-load buffer**, not L1/L2.
  - Read **all 16 B chunks of a line** before moving to the next; limited buffers → interleave at most a few streams.
- **Other arches**
  - PowerPC `dcbz` zeros a cache line w/o read-for-ownership (still allocates in cache; only writes zeros).

**Practical checklist**
- [ ] Confirm **no near-term reuse** of written data.
- [ ] Ensure **16 B alignment**; batch **4×16 B** to fill each 64 B line.
- [ ] Use **row-major (sequential) traversal** or **tile** to keep accesses combining-friendly.
- [ ] End sequences with **`_mm_sfence()`** when you need write completion.
- [ ] Avoid mixing temporal & non-temporal stores to the **same line** within a short window.

**Typical use cases**
- Large `memset`/buffer clears, one-pass encoders/decoders, streaming I/O staging buffers, framebuffers written once.

**Caveats**
- If a line is **already resident** in cache, some CPUs may still interact with it; aim for **cold destinations**.
- Strided column writes: prefer **block-transpose** or **tiling** so inner loops are contiguous.
- Measure: benefits depend on **cache hierarchy**, **write-combiner size**, and **memory controller** behavior.

---

### 6.2 Cache Access
- Biggest wins come from optimizing **L1 cache** behavior first.
- Core tactics: **improve spatial & temporal locality** and **align code/data** to cache-line boundaries.
- Keep hot working sets small, contiguous, and repeatedly reused before eviction.

**Key Points**
- **Spatial locality**
  - Use contiguous arrays; iterate in memory order.
  - Prefer **SoA** (structure-of-arrays) over **AoS** for vectorizable loops.
- **Temporal locality**
  - Reuse data soon after load; hoist invariant/reused values outside inner loops.
  - Tile/block loops so hot tiles fit in L1 (and L2 next).
- **Working-set blocking**
  - Choose tile sizes so: `(reads+writes) × element_size × live_arrays ≲ L1 size` (rule of thumb).
  - Multi-level block: first for L1, then for L2.
- **Alignment**
  - Align hot arrays/structs to **cache-line size (typically 64 B)**.
  - Keep frequently-touched fields at the **start** of a line; avoid crossing lines.
- **Avoid conflict misses**
  - Be wary of power-of-two strides; add padding to break unlucky set mapping.
  - Pad arrays/rows so `stride % (cache_size/associativity)` isn’t constant.
- **Prefetching**
  - Hardware prefetch helps simple strides; for deeper loops, use `_mm_prefetch(ptr+ahead, _MM_HINT_T0)` sparingly.
  - Use **NTA** prefetch only for one-time streams.
- **Writes**
  - Minimize write-allocate traffic: write full lines (memset/memcpy), batch stores, or use **streaming stores** for non-temporal data.
  - Avoid read-modify-write on cold destinations; accumulate in registers then store.
- **Branches & I-cache**
  - Make hot loops straight-line when practical; reduce hard-to-predict branches.
  - Keep hot code compact to fit L1I; inline only when it shrinks/helps the hot path.
- **Concurrency**
  - Prevent **false sharing**: place per-thread hot counters/state on separate lines (`alignas(64)`/padding).
  - Pin related threads to cores that share higher-level caches (and keep data close).
- **Measure & iterate**
  - Use PMCs (e.g., `L1-dcache-load-misses`, `cycles`) and tools (perf, Cachegrind) to guide changes.
  - Validate with A/B runs; cache work is workload- and CPU-specific.

**Quick checklist**
- [ ] Loop in memory order; tile to L1/L2.
- [ ] Align hot arrays/structs to 64 B; avoid crossing lines.
- [ ] Break power-of-two strides with padding.
- [ ] Prefetch modestly; verify with PMCs.
- [ ] Use streaming stores for one-pass outputs.
- [ ] Eliminate false sharing (`alignas(64)` per-thread data).
- [ ] Keep hot code small; reduce unpredictable branches.
- [ ] Re-measure after each change.

**Rules of thumb**
- Default cache line: 64 B; design data & strides around that granularity.
- Start with tiling and alignment; only add prefetch/streaming stores after profiling shows need.

#### 6.2.1 Optimizing Level 1 Data Cache Access
- Goal: make **L1** usage as effective as possible—iterate sequentially, reuse loaded lines before eviction, and align data/code.
- Matrix-mul case study: fix the non-sequential access to `mul2` via **transpose** or **cache-line–sized blocking (tiling)**; add SIMD after memory layout is fixed.
- Data layout matters: compress/pack structs, place “critical” fields first, align to cache-line size; avoid unaligned and conflict-miss patterns (power-of-two strides).
- Results (N=1000 doubles, Core 2 @ 2.66 GHz):
  - Original: **16.77e9 cycles (100%)**
  - Transposed: **3.92e9 (23.4%)**
  - Sub-matrix (cache-line tiling): **2.90e9 (17.3%)**
  - +SIMD (SSE2): **1.59e9 (9.47%)** ≈ **3.35 GFLOPS** (from 318 MFLOPS)

**Key Points**
- **Sequential access pays**  
  - Original triple loop walks `mul2` by columns → cache-line thrash.  
  - Transpose `mul2` (or logically traverse as if transposed) so both inputs are read row-major.  
  - Hardware prefetch then fills ahead effectively.
- **Cache-line–sized blocking (tiling)**
  - Let `CLS` be L1 line size (e.g., 64 B). For doubles, `SM = CLS / sizeof(double)`.  
  - Block i/j/k by `SM`, compute **SM×SM** tiles so each loaded line is fully consumed before eviction.  
  - Use pointer temporaries (`rres`, `rmul1`, `rmul2`) to avoid costly re-indexing/aliasing.
- **Vectorization after memory is fixed**
  - SSE2 can process 2 doubles/inst; apply once access is cache-friendly for an extra ~7% rel. win (vs. original).
- **Struct layout & packing**
  - Large structs that span multiple lines hurt when only a few fields are hot.  
  - Use tools like `pahole` to spot **holes** and cross–cache-line fields; reorder to:  
    1. fill alignment holes
    2. co-locate fields used together
    3. place likely **critical field first**.  
  - Consider **splitting** cold fields (e.g., buyer info) from hot fields (price/paid) to reduce bandwidth.
- **Alignment is non-negotiable**
  - Align hot objects/arrays to **cache-line** boundaries (`__attribute__((aligned(64)))`, `posix_memalign`).  
  - Unaligned accesses can cause **2× line fetches**; measured slowdowns: up to **~300%** (sequential in-L2), still **20–30%** on large working sets.
- **Stack alignment & ABI**
  - Respect platform stack alignment (e.g., x86-64 requires 16 B for SSE args).  
  - Where legal (e.g., 32-bit x86 without SSE needs), `-mpreferred-stack-boundary=2` can trim alignment overhead; not generally OK on x86-64.
- **Avoid conflict misses (set associativity)**
  - L1 (virtually indexed) has limited associativity (e.g., 8-way, 32 KiB).  
  - Power-of-two strides (e.g., every 4096 B) can map many hot lines to the **same set** → thrash (~10 cycles/elt vs ~3).  
  - Fix: add **padding**, vary strides, or rebase arrays to de-phase set mappings.
- **Banking (AMD note)**
  - AMD L1 is banked; two loads/cycle only if requests hit different banks (or same bank/same index).  
  - Group variables used together to improve banking parallelism.

**Practical tips**
- Detect cache line size at build (getconf LEVEL1_DCACHE_LINESIZE) and set CLS.
- Tile loops so each loaded line is fully consumed before eviction.
- Align arrays/objects to 64 B; avoid unaligned SIMD accesses.
- Repack structs; split hot/cold; put critical field first.
- Break power-of-two strides with padding/rebase.
- Add SIMD after fixing memory access; measure with PMCs.

#### 6.2.2 Optimizing Level 1 Instruction Cache Access
- Instruction-cache (L1) performance hinges on **smaller, straighter code** and **smart layout** so the front-end predicts, fetches, and decodes efficiently.
- Use compiler knobs to **balance size vs speed** (e.g., `-Os` vs `-O2/-O3`), **constrain inlining**, and **reorder cold blocks** out of the hot path.
- Align **functions** and **non-fallthrough blocks** when helpful; be careful aligning **loops**, which can add jumps/no-ops and backfire.

**Key Points**
- **Keep code compact unless proven otherwise**
  - `-Os` disables size-inflating opts; often faster if L1 pressure dominates.
  - Big wins from inlining happen only if they enable further scalar/SIMD opts; otherwise they bloat I-cache.
- **Control inlining deliberately**
  - Limit global inlining with `-finline-limit=<N>`.
  - Force when beneficial: `__attribute__((always_inline))`.
  - Prevent when it hurts locality or duplicates hot code: `__attribute__((noinline))`.
- **Make the hot path linear**
  - Reorder blocks so the likely branch is fallthrough; push rare paths away.
  - Use branch hints + block reordering:
    ```c
    #define likely(x)   __builtin_expect(!!(x), 1)
    #define unlikely(x) __builtin_expect(!!(x), 0)

    if (likely(cond)) { hot(); } else { cold(); }
    ```
  - Even better: **PGO** (profile-guided optimization) to auto-reorder based on real runs.
- **Small, tight loops help front-end**
  - Short inner loops can benefit from front-end loop streaming (e.g., LSD on Core 2) and stay resident in decode queues.
- **Align where it pays**
  - Cheapest wins: align **function entries** and **non-fallthrough targets** (fewer I-cache line splits, better decode).
  - Be cautious aligning **loop headers**; it may inject padding or an extra jump on entry.
  - GCC knobs:
    - `-falign-functions=<N>`
    - `-falign-jumps=<N>`
    - `-falign-loops=<N>`
    - (Avoid `-falign-labels` in general.)
  - Use modest `N` values; alignment is to the next power of two ≥ `N`, so large `N` wastes space.
- **When to inline vs not (rule of thumb)**
  - Inline if: single call site, enables constant-prop/loop-invariant/SIMD, or removes abstraction overhead in a hot loop.
  - Don’t inline if: many call sites in hot code, rare path, or it duplicates large cold pieces across the binary.

**Practical GCC flag recipes**
- Size-biased default: `-Os -freorder-blocks -fno-align-labels`
- Speed-biased but I-cache aware: `-O2 -freorder-blocks -finline-limit=80 -falign-functions=16 -falign-jumps=8 -falign-loops=8`
- Add **PGO** for layout: build with `-fprofile-generate`, run workloads, rebuild with `-fprofile-use`.

**Diagnostics**
- Inspect size & layout: objdump -d --visualize-jumps, nm -S, size.
- Measure: front-end stalls, I-cache misses, branch miss rates (e.g., perf stat -e cycles, instructions, branch-misses,icache.misses).
**Common pitfalls**
-	Over-unrolling/inlining that pushes hot code beyond L1 → more stalls.
-	Aligning every loop/label → extra padding/jumps, worse than the fix.
-	Relying solely on __builtin_expect without validating with profiles.

#### 6.2.3 Optimizing Level 2 and Higher Cache Access
- Last-level caches (LLC: L2/L3/…) are costly to miss and are often **shared** across cores/threads, so your effective cache is smaller than the headline size.
- For data reused more than once, **tile** work so the active working set fits **inside the LLC share** you actually get; otherwise prefetching can’t hide the latency.
- **Hardcode L1 line size** for fine-grain tiling, but **detect LLC size at runtime** (it varies a lot across machines).
- On Linux, read `/sys/devices/system/cpu/cpu*/cache/index*/{level,size,shared_cpu_map}` to compute a **safe per-thread LLC budget**.
- Pick block sizes from that budget (and still leave headroom for code, libraries, stacks, and other threads).

**Key Points**
- **Two LLC realities**
  - Misses at LLC ⇒ go to DRAM: very expensive.
  - LLC is **shared** (often per-socket): your usable capacity ≈ `LLC_size / (#CPUs sharing it)`.
- **When to care**
  - If each data element is touched only once, caching won’t help much; don’t force it.
  - If elements are **reused**, organize computation so that the reused subset fits in your **per-thread LLC budget**.
- **Runtime sizing (Linux)**
  - Find the highest `level` under `/sys/devices/system/cpu/cpu0/cache/index*/`.
  - Read its `size` (e.g., `"8192K"`) and its `shared_cpu_map` (hex bitmask of logical CPUs).
  - **Effective LLC share** (safe lower bound):  
    `llc_share_bytes = size_bytes / popcount(shared_cpu_map)`
  - Optionally scale by a safety factor `f∈(0,1)` to leave headroom (e.g., `f = 0.6`).
- **Choosing a tile (example: 3-matrix working set)**
  - For double-precision GEMM block of size `T×T`, three tiles live concurrently (A, B, C):  
    `bytes ≈ 3 * T^2 * 8`.
  - Require `3*T^2*8 ≤ f * llc_share_bytes`.  
    So `T ≤ floor( sqrt( (f * llc_share_bytes) / 24 ) )`.
  - Then **align T** to your L1 blocking size (e.g., multiple of `SM = L1_line_bytes / sizeof(double)`).
- **Why not just assume the biggest cache?**
  - LLC sizes differ by multiples (×2…×8+). Hardcoding penalizes most machines.
  - L1 line size changes little; LLC sizes change a lot ⇒ detect at runtime.

**Practical tips**
- If you pin threads so fewer share the LLC than the bitmap implies, you may choose a larger tile (with care).
- Libraries called inside the kernel loop (BLAS, RNG, I/O buffers) also consume cache; keep your f conservative.
- NUMA: the location of the data vs. the socket that owns the LLC matters—pair tiling with NUMA locality.
- Measure: confirm with hardware counters (LLC references/misses, DRAM bandwidth) and adjust T if miss rate rises.

#### 6.2.4 Optimizing TLB Usage
- There are **two TLB optimization levers**:
  1. **Use fewer pages at a time** to cut TLB misses; and  
  2. **Use fewer upper-level page-table directories** to make TLB lookups cheaper (less page-table memory, better cache hit rates).
- Page faults are rarer but **far costlier** than TLB misses; over long runs, frequent TLB misses can still dominate.
- **ASLR** spreads mappings across the address space (more directories). Only disable it if **maximum performance** is critical.
- If you must place mappings explicitly (e.g., `mmap(MAP_FIXED)`), choose addresses with **page-directory boundaries** in mind.

**Key Points**
- **Reduce pages in use concurrently**
  - Group hot code/data so that, at any moment, **fewer distinct pages** are touched.
  - This differs from page-fault tuning: it’s not just “avoid faults once,” it’s **keep the active TLB working set small** continuously.
- **Reduce page-table directory pressure**
  - Widely scattered virtual regions force **more levels/directories** to be consulted on a miss/walk.
  - Prefer **contiguous** virtual ranges when possible so fewer directories cover your working set.
- **ASLR trade-off**
  - Randomized placement increases address dispersion (more directories).  
  - Turning ASLR off can help in **rare, performance-critical** cases; otherwise, keep it on.
- **Fixed mappings caution**
  - Using `MAP_FIXED` is risky. If you must, **align/choose** the target address to avoid straddling a **last-level page-directory** boundary.

**When to act**
- If profiling shows high ITLB/DTLB miss rates during steady-state hot loops, prioritize **page coalescing** in those regions.

**Kernel note**
- A kernel could avoid mappings that cross directory boundaries—this would **slightly constrain ASLR** but is not generally relied upon.

**Practical workflow**
- Optimize for **TLB locality** (tight page footprint), then consider directory locality; only as a last resort consider **disabling ASLR** during benchmarking.

---

### 6.3 Prefetching
- **Goal of prefetching:** hide memory access latency by bringing data closer to the CPU before it’s needed.
- **Pipelines & OoO help only for cache hits;** they can’t mask main-memory latency without unrealistically long queues.
- **More cores ≠ solution** on non-OoO CPUs unless workloads are well-parallelized.
- Two approaches: **hardware prefetching** (automatic, pattern-triggered) and **software prefetching** (explicit hints from code).

**Key Points**
- **Latency gap:** CPU speeds outpace DRAM; without help, cache misses stall execution.
- **Pipeline limits:** Out-of-order execution overlaps work but mainly benefits **cache-resident** data.
- **Core-scaling caveat:** Adding cores to hide stalls only works if the application is **parallel**.
- **Prefetch types:**
  - **Hardware prefetching:** CPU detects predictable access patterns and fetches ahead.
  - **Software prefetching:** Programmer inserts prefetch hints to stage data early.

**When it helps:**
- Sequential or predictable strides benefit most; irregular/random patterns benefit less.

**Trade-offs:**
- Over-aggressive prefetching can waste bandwidth and pollute caches; distance/timing matters.

**Next steps:**
- Choose hardware defaults first; add software prefetches only where profiling shows sustained, predictable misses.

#### 6.3.1 Hardware Prefetching
- **Hardware prefetchers** kick in only after they see **≥2 cache misses** that form a recognizable pattern (adjacent lines or fixed **stride**).
- Prefetch logic exists per cache level (L1/L1, L2+), and **L2+/LLC prefetchers are shared** across cores/threads with a limited number of **streams**.
- **Prefetchers cannot cross page boundaries**, so you’ll still see misses at page edges unless software intervenes.
- They track fairly small windows (≈ **hundreds of bytes**) and **don’t learn non-linear**/irregular patterns.
- If harmful, hardware prefetching can be **tuned/disabled** (e.g., via MSRs on x86), but that’s a coarse, system-wide lever.

**Key Points**
- **Trigger rule:** Typically needs two+ sequential/strided **misses** before starting a stream; avoids spurious prefetch on random accesses.
- **Stride support:** Modern designs recognize fixed-stride streams (not just adjacent lines).
- **Stream budget:** ~**8–16 streams** tracked at higher levels; shared caches mean streams are **contended** across cores/SMT threads.
- **Per-level units:** L1 has its own prefetcher(s); L2/LLC prefetchers serve all sharers of that cache level.
- **Page boundary limit:** **Never crosses pages** to avoid unintended page faults/OS work; expect a miss at each boundary unless you prefetch or touch the next page explicitly.
- **Pattern window:** Recognition typically limited to a **small address window** (≈ up to ~512 B); beyond that, patterns aren’t pursued.
- **No non-linear learning:** Irregular or data-dependent jumps are treated as random → no prefetch benefit.
- **When it backfires:** Can **waste bandwidth** and **evict useful lines** if mis-triggered or if too many streams are active.
- **Mitigations:** 
  - Reshape data/code to make hot paths **sequential & in-page**.
  - Use **software prefetch** right before **page boundaries**.
  - Consider **larger pages** to reduce boundary frequency.
  - As a last resort, **disable** (entirely or “adjacent line” only) via **IA32_MISC_ENABLE** bits on Intel (privileged, kernel-level).
 
**Practical tips**
- Tile/stripe hot loops so working sets **advance linearly within a page**; avoid page-crossing in the inner loop.
- Align arrays/tiles so inner strides **match cache-line size** and minimize TLB churn near boundaries.
- For known strides, insert software prefetch with a **prefetch distance** that hides memory latency without polluting caches.
- Use **huge pages** (when feasible) to reduce boundary hits and TLB pressure.

**Diagnostics**
- Profile: LLC/TLB misses, prefetch request/use rates, bandwidth saturation, and eviction spikes when enabling/disabling prefetchers.
- Watch for **shared-LLC contention**: multiple threads can exhaust stream slots and degrade each other.

**Edge knobs**
- x86 `ud2` is sometimes used to **stop instruction-side speculation** past indirect branches—very niche; not a general data-prefetch fix.
- System-wide prefetch disable is **blunt**: validate per-workload; other apps may regress.

#### 6.3.2 Software Prefetching
- **Software prefetching** lets you explicitly pull data into cache (or stream it) when hardware prefetchers can’t help—especially across **page boundaries** or for irregular strides.
- On x86/x86-64 use `_mm_prefetch(ptr, hint)` with hints **T0/T1/T2/NTA** to target cache levels or mark data as **non-temporal**.
- Proper **prefetch distance** matters: prefetch far enough ahead to cover latency, but not so far that you evict what you still need.
- In a randomized pointer-chase with ~160 cycles of work per node (NPAD=31), prefetching ~**5 nodes ahead** cut miss cost by **~8%** once the working set exceeded L2.
- AMD Family 10h adds **`prefetchw`** (brings line in **M** state) which helps when writes follow; misuse hurts.

**Key Points**
- **API & Hints**
  - `_mm_prefetch(void* p, _MM_HINT_T0|T1|T2|NTA)` (in `<xmmintrin.h>`).
  - **T0**: bring to L1 (inclusive caches) / lowest level (exclusive). Use when you’ll consume **immediately**.
  - **T1/T2**: target higher levels to stage data without crowding L1 when the **immediate working set** is large.
  - **NTA**: non-temporal; try to **avoid polluting** lower caches and write back directly on eviction—great for single-use streams.
- **Safety/behavior**
  - On x86, prefetch to **invalid pointers is ignored** by hardware (no fault), but pointless prefetches still waste bandwidth and can evict useful lines.
- **Tuning distance**
  - Pick distance so: `work_per_iter_cycles × distance ≥ mem_latency_cycles (+ cache fill)`.
  - For the example (~160 cyc/iter, two cache lines per node), **distance ≈ 5** worked well.
- **Measured effect**
  - No win while the working set **fits in L2**; once it exceeds L2, software prefetching saved **~50–60 cycles/iter (~8%)**.
- **Write-biased prefetch (AMD)**
  - `prefetchw` (Fam 10h): fetches to L1 and sets state **M**; speeds up subsequent writes, but is **harmful** if no write occurs.
- **Compiler help**
  - `-fprefetch-loop-arrays` (GCC) can auto-insert prefetches for array loops; results vary—**profile** before keeping it.
- **Instrumentation**
  - Use PMU counters: hardware/software prefetches, **used** software prefetches, L1/L2/LLC misses, bandwidth, evictions.

**Best practices**
- Keep inner loops **in-page**; prefetch right before **page boundaries** (hardware can’t cross pages).
- Use **T0** for near use, **T1/T2** when staging, **NTA** for streaming data you won’t revisit.
- Consider **huge pages** to reduce boundary hits & TLB pressure (when practical).
- Don’t flood L1: if the immediate working set > L1, avoid T0 everywhere.
- Insert prefetches **sparingly** and near the producer of the address (avoid long dependency chains).

**Tuning workflow**
1. Locate hot misses (LLC/TLB) with PMU.
2. Add one prefetch at a time; choose hint & distance.
3. Re-measure: misses ↓, used-prefetch ↑, runtime ↓; if not, adjust or remove.

**Caveats**
- Over-prefetching can steal bandwidth and evict hot lines.
- Shared caches mean prefetch streams are contended across cores/SMT.
- prefetchw is AMD-specific (no standard Intel intrinsic).

#### 6.3.3 Special Kind of Prefetch: Speculation
- **Out-of-order (OOO) execution** can overlap independent ops, but it won’t hoist a **load above a potentially-aliasing store** because the addresses might match.
- **Speculative/advanced loads** (e.g., IA-64 `ld8.a` … `ld8.c.clr`) let you *start* a load early, then **validate or re-load** later, hiding cache/memory latency without breaking correctness.
- Compared to prefetching, speculative loads deliver a **register value** (ready for compute) instead of merely warming caches, and can reduce sensitivity to cache eviction.

**Key Points**
- **Why the plain load can’t move:** a preceding store (`st8 [r4]`) might target the same address as the load (`ld8 [r8]`). Until proven otherwise, the CPU must respect the **store→load ordering** to preserve semantics.
- **Speculative-load pair (IA-64 example):**
  - `ld8.a r6=[r8]` — begin an *advanced/speculative* load.
  - … do unrelated work to create a latency-hiding gap …
  - `ld8.c.clr r6=[r8]` — *confirm/clear*: if no conflict with the earlier store, keep the value; if there was a conflict, **re-load** so the result matches a normal `ld8`.
- **When it helps:**
  - Long-latency loads (L2/LLC/DRAM misses).
  - Enough independent instructions between `ld*.a` and `ld*.c` to **cover latency**.
  - Low alias probability between the store and the load.
- **Costs/risks:**
  - Extra instructions (issue bandwidth, register pressure).
  - **Mispredicted** speculation triggers a re-load (lost work).
  - Don’t use on `volatile`/MMIO; still follow the memory model.
- **Relation to prefetching:**
  - Prefetch: moves data toward caches (**no register result**), good cross-arch.
  - Speculative load: produces a **ready value** (and, as used here, doesn’t rely on the cache line remaining resident), so compute can proceed immediately after the check.
- **Portability reality:**
  - IA-64 exposes these ops; most mainstream ISAs rely on hardware OOO + **software prefetch** instead.
  - Portable wins: expose ILP, hoist loads safely, reduce aliasing (e.g., `restrict`), and prefetch where it pays.

**Practical tuning**
- Aim for a gap ≈ **(expected miss latency) / (avg work per cycle)** between `ld*.a` and `ld*.c`.
- Unroll/software-pipeline loops to create independent work between the two.
- Keep live values in registers to avoid spills; watch register pressure.

**Help the reordering logic (even without ISA support)**
- Use `restrict`/non-aliasing pointers so compilers/CPUs can hoist loads across unrelated stores.
- Separate data that may alias; align data; avoid store→load hazards on the same line when possible (store-to-load forwarding pitfalls).
**Measure**
- Track load-miss latency, replay/reload events, machine clears, useful prefetches, and overall IPC to see if speculation actually hid latency.

#### 6.3.4 Helper Threads
- **Problem:** Manual software prefetching complicates loops (look-ahead logic, tuning distance, more instructions, bigger L1i footprint, limited outstanding prefetches).
- **Alternative:** Run a **helper prefetch thread** that stays ahead of the main worker, touching data to pull it into a **shared cache** (ideally on the **sibling hyper-thread** of the same core).
- **Results:** If the working set fits in L2, the helper **hurts** (≈10–60% overhead). Once the working set exceeds L2, the helper **helps** (≈25% faster) by overlapping memory latency with computation.
- **Mechanics:** Pin threads to sibling HTs / shared caches, pace the helper so it doesn’t evict useful lines, and synchronize lightly (e.g., futex/pthreads). Use topology info (libNUMA or `/sys`) and cache size (`sysconf(_SC_LEVEL2_CACHE_SIZE)`).

**Key Points**
- **Why helper threads can win**
  - Offloads look-ahead logic; **no loop contortions**.
  - Real loads (not just prefetch hints) populate L2/L1 and **fill TLBs**.
  - Main thread spends more time on ALU work while helper soaks **DRAM/LLC latency**.
- **When they help**
  - Working set **exceeds L2/LLC**, memory latency dominates, and main thread has **~100–200+ cycles** of work per element.
  - HT or cores that **share a cache** (L2/L3). If no shared cache, benefit shrinks.
  - NUMA-local memory; otherwise remote access can erase gains.
- **When they hurt**
  - Working set **fits in L2** → helper just competes for decode/ports and pollutes caches.
  - Helper runs **too far ahead** → evicts prefetched lines before use.
- **Tuning**
  - **Distance** ≈ (miss latency cycles ÷ work/elem cycles) × (elems per cache line).
  - **Throttle** with a bounded gap (ring buffer semantics) to avoid cache pollution.
  - Use **L2 (or LLC) size** as an upper bound on the “in-flight” footprint.
- **Placement & affinity**
  - Pin main and helper to **sibling hyper-threads** (or cores sharing L2/L3).
  - Discover siblings with **libNUMA** `NUMA_cpu_level_mask(..., level=1)` or `/sys/devices/system/cpu/*/topology/thread_siblings`.
- **Caveats**
  - Watch **bandwidth**: two threads can saturate memory/FSB and stall both.
  - Limit **false sharing** on control variables (pad to cache line).
  - Prefetcher cannot cross pages, but the helper’s real loads **will** fault/touch pages—consider `mlock` or warmup if page faults are costly.

**Heuristics that work well**
- Start with look-ahead ≈ (LLC miss latency / cycles per iter); clamp so the total touched bytes ≤ ~½ of shared cache.
- Adapt: sample stall cycles or LLC-miss rate; nudge look-ahead up/down.

**Diagnostics to validate**
- LLC-load-misses, cycles-stalled-frontend/backend, mem-loads, mem-stores, prefetches, dtlb_load_misses.
- Confirm threads share a cache (drop in misses, improved IPC).

**If libNUMA isn’t available**
- Read /sys/devices/system/cpu/cpu*/topology/{thread_siblings,core_siblings} to pair HTs / cores that share caches.

**NUMA considerations**
- Allocate data on the same node as the HT pair (first-touch or mbind/numa_alloc_onnode), or you’ll chase remote memory penalties.

#### 6.3.5 Direct Cache Access
- **Problem:** DMA-based I/O (e.g., NICs) writes packet data straight to RAM, so when the CPU is notified it **misses in cache** reading the header—adding hundreds of cycles per packet.
- **Idea (DCA):** NIC + chipset tag the DMA with a **Direct Cache Access (DCA)** hint so the destination CPU **pre-populates its cache** with the packet header while the DMA completes.
- **Result:** When the interrupt/notification fires, the OS’s header reads are likely **cache hits**, cutting per-packet latency and improving throughput—especially at high packet rates.

**Key Points**
- **Where it happens:** NIC issues DMA → Northbridge/memory controller forwards data on the FSB **with a DCA hint** → CPU snoops bus and **loads header into lowest cache** (hint is advisory; CPU may ignore).
- **What’s cached:** Primarily the **packet header** (immediately consumed for routing/classification). Payload is typically handled later.
- **Why it helps:** Removes a “cold read” at packet arrival time; saving **hundreds of cycles** per packet can compound massively at tens of thousands of packets/sec.
- **Prereqs:** Tight **platform cooperation** (NIC, chipset, CPU) that supports the DCA hinting path; OS routes the interrupt to the **intended CPU** so the warmed cache is actually used.
- **Scope:** Transparent to application code; benefit shows up in **kernel fast-path** of RX processing.

**Simplified flow**
1. NIC starts DMA of packet → marks transaction as **DCA**.
2. Memory controller completes DMA to RAM **and** forwards header on bus with DCA flag.
3. Target CPU **snoops & pulls** header into L1/L2 (as policy allows).
4. NIC notifies CPU; OS inspects header → **cache hit**.

**Operational notes**
- DCA is a **hint**, not a guarantee; CPUs can choose to ignore it.
- Gains are most visible on **latency-sensitive** and **high-pps** workloads.
- Platform selection matters: without coordinated NIC/chipset/CPU support, **no DCA benefit**.

---

### 6.4 Multi-Thread Optimizations
- Multi-threaded programs face three cache-related challenges: **concurrency (shared address space interactions), atomicity (correct coordination on shared data), and bandwidth (limited shared memory/IO paths).**
- Best case is **disjoint working sets** per thread; shared data introduces **coherency/locking overhead** and can **saturate memory bandwidth**, hurting scaling.
- Multi-process scenarios see similar effects, but fewer optimizations apply since processes don’t naturally share memory structures.

**Key Points**
- **Concurrency**
  - Threads share one virtual address space → easy sharing, but also **cache interference** and **false sharing** risk.
  - Ideal: partition data so **each thread mostly touches distinct cache lines**; overlap only at infrequent sync points.
- **Atomicity**
  - Shared updates require synchronization (locks/atomics), which trigger cache-coherency traffic (e.g., **RFO/MESI transitions**) and stall others.
  - Contended atomics/locks serialize progress and can thrash “hot” cache lines, especially across cores/sockets.
- **Bandwidth**
  - Memory controllers, interconnects, and last-level caches are **shared resources**; multiple threads can **saturate them**.
  - Even read-only streams can contend; writes are worse (dirty evictions, writebacks). Scaling flattens once **LLC or memory BW** becomes the bottleneck.

**Practical implications**
- **Partition**: sharding, per-thread chunks, and work stealing with coarse granularity.
- **Avoid false sharing**: pad/align hot fields to cache-line size; group read-mostly data.
- **Choose sync wisely**: prefer per-thread accumulators + periodic reduction; use atomics only on truly hot paths; avoid highly contended locks.
- **NUMA/placement**: pin threads to cores, colocate their memory, minimize cross-node traffic.
- **Throughput vs. latency**: batch updates to reduce coherence traffic; use non-temporal stores for one-shot output streams.
- **Measure**: track LLC misses, coherency events, lock/atomic contention, and memory BW to find the scaling limit.

#### 6.4.1 Concurrency Optimizations
- **False sharing** happens when multiple threads write to different variables that live on the **same cache line**, triggering heavy coherency traffic (RFO/MESI transitions) and destroying scalability.
- Benchmarks show extreme slowdowns when threads pound a single shared line (e.g., +390%, +734%, +1,147% time vs. separate lines on a 4-way P4 SMP). On a single-socket quad-core (shared L2 pairs), the penalty may appear small—but multi-socket systems expose the true cost.
- Mitigate by **separating read-only/read-mostly vs. writable data**, **padding/aligning hot writable fields to cache-line boundaries**, and using **thread-local storage (TLS)** when each thread uses its own copy.

**Key Points**
- **Why it hurts**
  - Writes require ownership: the line must be in **E/M state** in one core; competing writers force **RFO** handoffs each increment → serialized progress across cores/sockets.
  - Even if threads touch different variables, sharing one line turns them into an implicit hotspot.
- **When it’s visible**
  - Multi-processor (multi-socket) systems: dramatic collapse in scaling.
  - Single-socket, shared-LLC machines: impact can look minor for simple tests, but will reappear with more sockets or different access patterns.
- **Data placement strategies**
  1. **Read-only / init-once data** → keep together (e.g., `.rodata`, `.data.rel.ro`)—shared **S state** is cheap.
  2. **Read-mostly** vs. **read-write** → separate sections to avoid mixing with highly written fields.
  3. **Hot writable vars** → place on their **own cache line** (pad + align to `CLSIZE`), ideally grouped if they’re updated together.
  4. **Per-thread private state** → make it **TLS** (`__thread`) to eliminate inter-thread line sharing.
- **TLS trade-offs**
  - Pros: no false sharing by design; simple to use.
  - Cons: setup cost per thread; slightly higher addressing cost; **memory overhead** (TLS blocks per DSO may allocate as a group).
- **Atomic update patterns (GCC builtins)**
  1. `__sync_add_and_fetch(&var, 1)` — add, returns **new** value.
  2. `__sync_fetch_and_add(&var, 1)` — add, returns **old** value.
  3. CAS loop (`__sync_bool_compare_and_swap`) — flexible, can fold custom logic but higher contention cost.
  - All still contend if `var` shares a cache line across threads; padding/partitioning still matters.

**Practical recipes**
- **Pad + align** a hot writer:
  ```c
  #define CLSIZE 64
  struct {
    struct { int bar; int xyzzy; } hot;
    char pad[CLSIZE - sizeof(struct { int bar; int xyzzy; })];
  } rwstruct __attribute__((aligned(CLSIZE))) = { .hot = {2,4} };
  // use rwstruct.hot.bar / rwstruct.hot.xyzzy
  ```
- **Thread-local copy** to avoid sharing:
  ```c
  __thread int counter = 0;  // each thread has its own instance
  ```
- **Separate mostly-const data** via sections:
  ```c
  int foo = 1;
  int baz = 3;
  int bar   __attribute__((section(".data.ro"))) = 2; // read-mostly
  int xyzzy __attribute__((section(".data.ro"))) = 4;
  ```

**Checklist**
- Identify hot writes; ensure each thread updates **distinct cache lines**.
- Group constants and read-mostly fields away from hot writers.
- Prefer per-thread accumulators + periodic merge over shared counters.
- Test on **true SMP (multi-socket)** hardware; microbenchmarks on a single socket can hide the issue.

#### 6.4.2 Atomicity Optimizations
- Concurrent writes to the **same memory location** are undefined unless you use **atomic operations**; otherwise increments can be lost due to independent cache reads/writebacks.
- Atomic ops come in flavors (bit test, **CAS**, **LL/SC**, **atomic arithmetic**). Many ISAs expose only CAS or LL/SC; **x86/x86-64 also provide fast atomic arithmetic** (e.g., atomic add) that can outperform a CAS loop.
- CAS loops are portable but often **much slower under contention** (extra loads, conditional store, retries, and more MESI/RFO traffic).
- On x86/x86-64, atomicity is enabled via the **`lock` prefix**; you can **dynamically skip** it when running single-threaded to avoid the atomic cost.

**Key Points**
- **Why non-atomic increments lose updates**
  - Two threads read old value from a shared line in **S** state → both add → whichever writes last wins → one increment is lost.
- **Atomic choices**
  - **Bit test/set**, **CAS** (compare-and-swap), **LL/SC** (load-linked/store-conditional), **atomic arithmetic** (`add`, `and`, etc. on memory; x86/x86-64).
  - ISAs provide **either CAS or LL/SC** (functionally equivalent); **x86 also has atomic arithmetic** and it’s usually cheaper than a CAS loop.
- **CAS vs atomic add (x86/x86-64, 4 threads, 1M increments)**
  - | Method                 | Time |
    |------------------------|------|
    | Exchange-Add (fetch-add old) | 0.23s |
    | Add-Fetch (return new)       | 0.21s |
    | **CAS loop**                  | **0.73s** |
  - CAS is slower due to **two memory ops**, conditional update, **retries**, and extra **RFO**/state transitions across cores.
- **MESI/RFO perspective**
  - Atomic arithmetic lets the core **hold ownership** and perform the read-modify-write as a unit → at most **one RFO** per iteration.
  - CAS loops ping-pong the line and **fail** under overlap → repeated retries amplify traffic.
- **Don’t universalize on CAS**
  - Expose atomics at an abstraction level that can map to **native atomic arithmetic** where available; use CAS only when needed (e.g., conditional update semantics).
- **Skip atomic costs when safe (x86)**
  - Same instruction can be atomic or not; use a branch to **jump over the `lock` prefix** when only one thread is active:
    ```asm
    cmpl $0, multiple_threads
    je 1f
    lock
    1: add $1, some_var
    ```
  - Worth it when single-threaded is common; atomic ops can cost ~100–200 cycles.

**Portable increment patterns (GCC builtins)**
1. **Add & get new**
    ```c
    for (int i = 0; i < N; ++i)
     __sync_add_and_fetch(&var, 1);
    ```
2. **Add & get old**
    ```c
    for (int i = 0; i < N; ++i)
     __sync_fetch_and_add(&var, 1);
    ```
3. **CAS loop (portable but slower under contention)**
    ```c
    for (int i = 0; i < N; ++i) {
     int v, n;
     do { v = var; n = v + 1; }
     while (!__sync_bool_compare_and_swap(&var, v, n));
    }
    ```

**Contention & layout still matter**
- Even perfect atomics **fight** if many threads update the **same cache line**. Prefer **sharded counters** (per-thread/per-core) + periodic combine, and **pad/align** to cache lines to avoid false sharing.

**Rule of thumb**
- Use **native atomic arithmetic** when available for plain RMW ops; reserve **CAS/LLSC** for condition-dependent updates. Combine with good **data layout** to minimize coherence traffic.

#### 6.4.3 Bandwidth Considerations
- Even without cache-line contention, many-threaded programs can be bottlenecked by **memory bandwidth** shared across cores, hyper-threads, sockets, and the northbridge/FSB.
- Use **hardware performance counters** to detect bus contention; if utilization is already high, adding threads won’t scale.
- Improve by **placing threads** to share cache when they share data (reduce duplicate memory traffic), or to separate caches when data sets are independent. Enforce with **thread affinity**.

**Key Points**
- **Bandwidth limits**  
  - Core frequencies far exceed DRAM bandwidth; dividing that bandwidth among many cores/threads makes stalls likely for large working sets.
  - Counters (e.g., Core 2: `NUS_BNR_DRV`, bus utilization, RFO counts) reveal cycles waiting for the bus; values near saturation ⇒ poor scalability headroom.
- **Scheduling for data locality**  
  - Bad: two threads on different cache domains **reading the same data** ⇒ memory must serve it twice.  
  - Better: co-locate such threads on cores **sharing a last-level cache** (or same socket) so the data is fetched **once** and reused.
- **Scheduling for isolation**  
  - If threads touch **distinct data sets**, pin them to **different cache domains** to avoid evicting each other and halving effective cache capacity/bandwidth.
- **Thread affinity APIs (Linux)**  
  - Process-level: `sched_setaffinity(pid, size, cpu_set_t*)`, `sched_getaffinity(...)`.
  - Thread-level (pthreads):  
    - runtime: `pthread_setaffinity_np(th, size, cpu_set_t*)`, `pthread_getaffinity_np(...)`  
    - at creation: `pthread_attr_setaffinity_np(attr, size, cpu_set_t*)`, `pthread_attr_getaffinity_np(...)`
  - CPU set helpers: `CPU_ZERO`, `CPU_SET`, `CPU_CLR`, `CPU_ISSET`, `CPU_COUNT` and `_S` variants for dynamic sizes.
  - Dynamic allocation: `CPU_ALLOC_SIZE(n)`, `CPU_ALLOC(n)`, `CPU_FREE(set)`.
  - Current CPU (best-effort): `sched_getcpu()` (may change immediately after call; use for hints, not invariants).
- **NUMA tie-in**  
  - Treat caches as a small “NUMA” layer: choose affinities that match **data placement**. Use NUMA libraries (/sys, libnuma) to compute masks for shared vs. isolated domains.

**Practical workflow**
1. Measure bus/FSB (or memory controller) pressure with perf counters.
2. If near saturation, co-locate threads that share data; separate those that don’t.
3. Enforce with affinity; verify with counters (lower RFOs/loads, fewer stall cycles).
4. On NUMA, also bind memory (first-touch, mbind) to the socket you pin to.

**Caveats**
- Affinity masks are inherited and may change (admin tools, CPU hotplug). Always re-check if critical.
- Over-constraining affinity can leave CPUs idle while others are oversubscribed—monitor and adjust.

---

### 6.5 NUMA Programming
- NUMA changes memory **access cost by page location**, so performance now depends on *where* pages live, not just *whether* they’re cached.  
- These ideas extend to **cache hierarchies** (LLC sharing vs. not), so thread placement and memory placement must be coordinated.  
- On Linux, practical NUMA control is via **policies and bindings**: `set_mempolicy`, `mbind`, `get_mempolicy`, plus page migration (`migrate_pages`, `move_pages`).  
- `libnuma` (numactl) mainly wraps syscalls; detailed topology still comes from **sysfs**. Understanding **memory policies** is the next step.

**Key Points**
- **Why NUMA matters**
  - Uniform memory: all pages “equal” → optimize page faults and cache use.
  - NUMA: **per-node latency/bandwidth** differ; page locality and thread placement directly affect runtime.
  - With many-core CPUs, even single-socket systems behave “NUMA-like” at the **cache** level (shared vs. private LLC).
- **Unifying caches & NUMA**
  - Threads on cores that **share an LLC** can reuse lines cheaply; separate-LLC placement can double memory traffic.
  - Treat cache groups like mini NUMA nodes when deciding **affinity**.
- **Linux tooling overview**
  - Topology & distances: `/sys/devices/system/cpu/**/cache` and `/sys/devices/system/node/`.
  - `libnuma` is a **convenience layer**; it does not expose all topology details.
  - Core syscalls:
    - `mbind` — bind a specific VA range to nodes/policy.
    - `set_mempolicy` / `get_mempolicy` — set/get **default** policy for future allocations.
    - `migrate_pages` — move *all* pages of a PID between node sets.
    - `move_pages` — move or query **selected** pages’ node locations.
- **Design implications**
  - First-touch allocation + **thread affinity** gives good defaults; refine with `mbind`/`move_pages` for hot pages.
  - Interleave (striping) can help balance, but **local bind** wins for latency-sensitive hot sets.
  - Always measure; bus/memory counters reveal when bandwidth or cross-node traffic dominates.

**Typical workflows**
- *Local-hotset apps:* pin worker to node X → `set_mempolicy(MPOL_BIND, {X})` → allocate → run.
- *Throughput apps with many producers/consumers:* consider `MPOL_INTERLEAVE` across active nodes, but keep communicating threads on the **same socket/LLC**.
- *After-the-fact correction:* sample hot pages, then `move_pages` them to the worker’s node.

**Placement + affinity**
- Use `pthread_setaffinity_np`/`sched_setaffinity` so threads stay where their pages are.
- On start-up, allocate per-thread buffers **after** pinning to ensure first-touch lands on the right node.

**When to use each policy (preview)**
- `MPOL_BIND`: lowest latency for node-local hot data.
- `MPOL_INTERLEAVE`: even bandwidth use across nodes; good for streaming/throughput.
- `MPOL_PREFERRED`: soft “try local node first,” fallback elsewhere.

**Caveats**
- ASLR & allocator behavior can scatter heaps; pin threads and **touch** memory intentionally.
- Over-constraining affinity can starve CPUs; monitor and adjust.
- Page migration has cost; amortize by moving **long-lived hot** pages, not short-lived ones.

#### 6.5.1 Memory Policy
- Linux NUMA **memory policies** let unmodified programs behave reasonably on NUMA by steering where pages are allocated.
- Policies are **inherited by child processes**, so tools like `numactl` can launch a program with a chosen policy.
- The kernel picks a policy via a **hierarchy**: VMA policy (and a special shared-memory policy) → task policy → system default.
- **System default**: allocate “local” to the requester; for multithreaded processes the “local” node is the process’s **home node** (the node that first ran the process), unless you override it.

**Key Points**
- **Policies**
  - `MPOL_BIND`: allocate **only** from the specified node set; if none available, allocation **fails**.
  - `MPOL_PREFERRED`: try the given nodes **first**; if unavailable, **fallback** to others.
  - `MPOL_INTERLEAVE`: spread allocations **evenly** across the specified nodes  
    - For VMA-based policies: node chosen by **offset** within the region.  
    - For task-based policies: node chosen by a **free-running counter**.
  - `MPOL_DEFAULT`: defer to the next level in the hierarchy (ultimately system default).
- **Hierarchy & scope**
  - **VMA policy** (per-mapping) overrides task policy; **shared memory** uses a specialized VMA policy class.
  - If no VMA policy: use the **task policy**; if none: use the **system default**.
- **Operational notes**
  - Policies are **per-address-range** (VMA) or **per-task**; both can coexist.
  - Using `MPOL_BIND` can cause **allocation failures** on pressure; `PREFERRED` is safer when capacity varies.
  - `INTERLEAVE` improves bandwidth balance but may increase **latency** vs strict locality.

**Practical usage**
- Launch with a policy:
  - Bind: `numactl --membind=0,1 --cpunodebind=0,1 ./app`
  - Preferred: `numactl --preferred=2 ./app`
  - Interleave: `numactl --interleave=0-3 ./app`
- Programmatic control:
  - Set default for future allocations: `set_mempolicy(...)`
  - Bind a specific region: `mbind(addr, len, MPOL_*, nodemask, ...)`

**Choosing a policy (rules of thumb)**
- **Latency-sensitive hot data** with pinned threads → `MPOL_BIND` to the thread’s node(s).
- **Throughput/streaming** workloads spanning nodes → `MPOL_INTERLEAVE`.
- **Mixed/variable** footprints → `MPOL_PREFERRED` to nudge locality without hard failures.

**Remember**
- Policies are inherited by children; combine with **CPU affinity** so first touch lands on the intended node.
- Review shared memory segments separately—they use their **own policy class**.

#### 6.5.2 Specifying Policies
- `set_mempolicy(mode, nodemask, maxnode)` sets the **task (thread)–level NUMA memory policy** for **future** page allocations only.
- It **does not migrate** already-allocated pages; first touch after the call determines placement under the new policy.
- `mode` is one of `MPOL_BIND`, `MPOL_PREFERRED`, `MPOL_INTERLEAVE`, or `MPOL_DEFAULT`; `nodemask` and `maxnode` define eligible nodes.

**Key Points**
- **Scope**
  - Affects **only the calling thread** (kernel “task”), not the whole process.
  - VMA- and shared-memory policies can override task policy (policy hierarchy).
- **Nodemask & maxnode**
  - `nodemask` is a bitset of NUMA nodes; `maxnode` is the bit **count** the kernel will consider (typically **highest-node-id + 1**).
  - Bits ≥ `maxnode` are ignored; out-of-range or empty masks (when required) → `EINVAL`.
- **Modes**
  - `MPOL_BIND`: allocate **only** from `nodemask`; if none available, allocation **fails**.
  - `MPOL_PREFERRED`:
    - `nodemask == NULL` → prefer **local** node.
    - Otherwise: prefer the **lowest-numbered set bit** in `nodemask`, fall back elsewhere if needed.
  - `MPOL_INTERLEAVE`: round-robin across nodes in `nodemask` (by VMA offset or a per-task counter).
  - `MPOL_DEFAULT`: ignore `nodemask`; revert to default policy (usually **local/first-touch**).
- **Allocation vs reservation**
  - `mmap` reserves address space; **physical pages are allocated on first access**.
  - If policy changes between page touches, a single mapping can span **multiple nodes**.
- **No retroactive effect**
  - Existing pages stay put. To change placement, use `mbind(...)` (per-range) or `move_pages(...)` / `migrate_pages(...)`.
- **Return & errors**
  - Returns `0` on success, `-1` with `errno` (e.g., `EINVAL` for bad mode/mask, `EFAULT` for bad pointer).

**Workflow tips**
- Combine with **CPU/thread affinity** so the thread touches pages on the intended node.
- After changing policy, **touch** (read/write) the mapping to allocate under the new policy.
- For portable sizing, query node count (e.g., via libnuma) and size `nodemask` accordingly.

#### 6.5.3 Swapping and Policies
- When Linux swaps out pages, it **forgets their NUMA node**. On page-in, placement is recomputed using the **current policy** (e.g., first-touch/`set_mempolicy`), so a page may return on a **different node**.
- Pages can also move because of **kernel migration** (balancing/space pressure) or **other processes** calling `mbind(...)` on shared mappings.
- Therefore, any “page → node” mapping you observe is **ephemeral**. Treat it as a **hint**, and query again when you need accurate info with `get_mempolicy(...)` (or `move_pages(...)` for per-page location).

**Key Points**
- **Swap behavior**
  - Clean pages are dropped; dirty pages are written to swap. On fault-in, Linux **does not restore** the original node; it chooses a node per **current policy** and availability.
- **Why mappings change**
  - **Swapping**: node tag is lost; re-fault can land elsewhere.
  - **Shared pages**: another process can relocate with `mbind(...)`.
  - **Kernel auto-migration**: may move pages if a node is full or to rebalance.
- **Implications**
  - Don’t cache page→node mappings in user space; they **age quickly**.
  - Performance can drift as hot data “comes back” on a different node.
  - For stable placement, combine **thread affinity + mempolicy + pre-fault**; optionally **pin** memory to avoid swap.
- **How to observe (just-in-time)**
  - `get_mempolicy(...)`:
    - With `MPOL_F_ADDR|MPOL_F_NODE`, returns the node **for a given address** (current or would-be under policy).
    - Without `MPOL_F_ADDR`, reports the **calling thread’s** default policy/mask.
  - `move_pages(...)` with `nodes=NULL` queries the **actual node** for many pages at once.
  - `/proc/<pid>/numa_maps` is useful, but still only a **snapshot**.
- **How to stabilize (when needed)**
  - Use `mlock/mlockall` to avoid swapping critical buffers.
  - Choose a policy (`MPOL_BIND`, `MPOL_PREFERRED`, `MPOL_INTERLEAVE`) via `set_mempolicy(...)` / `mbind(...)`.
  - **Pre-fault** pages (touch them) after setting policy and **pin the thread** to the target CPU/node.
  - For shared, read-mostly mappings, consider **multiple local copies** per node when memory allows.

**Operational tips**
- If node stability is crucial: mlockall(MCL_CURRENT|MCL_FUTURE), set policy via set_mempolicy/mbind, touch pages to fault them on the intended node, and keep threads affined.
- Expect that once memory pressure triggers swap or balancing, associations can change—re-query before making placement-sensitive decisions.
  
#### 6.5.4 Virtual Memory Area (VMA) Policy
- `mbind(start, len, mode, nodemask, maxnode, flags)` sets a **VMA-level NUMA policy** for the address range `[start, start+len)`. `start` must be **page-aligned**; `len` is rounded up to page size.
- By default (`flags=0`), it **only sets policy for future faults**—it does **not move existing pages**.
- Optional flags:
  - `MPOL_MF_STRICT`: fail if any page lies outside `nodemask` (and, with move flags, fail if any such page cannot be moved).
  - `MPOL_MF_MOVE`: **migrate pages owned by this process’s page tables** into `nodemask`.
  - `MPOL_MF_MOVEALL` (privileged): **migrate all pages** in range (even shared with other processes) into `nodemask`.

**Key Points**
- **Policy vs. placement**
  - `mbind` sets the **VMA policy** (where *future* faults place pages). It does not retroactively move pages unless `MOVE`/`MOVEALL` is set.
  - Use `set_mempolicy(...)` for a **thread’s default** future allocations; use `mbind(...)` to **pin a region’s policy**.
- **Alignment & sizing**
  - `start` must be **page-aligned**; `len` is **rounded up** to page size.
  - Best used **before touching** memory (e.g., after `mmap` and before first access), unless migrating with flags.
- **Flags behavior**
  - `STRlCT` alone: just **validate**; does **not** move pages.
  - `MOVE`: best-effort migrate pages **exclusively referenced** by this process.
  - `MOVEALL`: also migrates **shared** pages; requires capability (e.g., `CAP_SYS_NICE`/`CAP_NICE`) and can impact other processes.
- **Typical flow**
  - Reserve: `mmap(..., PROT_READ|PROT_WRITE, MAP_ANON|MAP_PRIVATE, -1, 0)`.
  - Bind policy: `mbind(p, len, MPOL_BIND, nodemask, maxnode, 0)`.
  - **Then touch** pages to fault them on desired node(s).  
    - If pages already faulted, optionally add `MPOL_MF_MOVE` to migrate.
- **Choosing `mode`**
  - `MPOL_BIND`: allocate **only** from `nodemask` (fail if not possible).
  - `MPOL_PREFERRED`: try `nodemask` (or lowest-set node); fall back elsewhere.
  - `MPOL_INTERLEAVE`: distribute pages across `nodemask` (by offset/counter).
  - `MPOL_DEFAULT`: revert to system/task default.
- **Caveats**
  - `MOVE`/`MOVEALL` available since Linux **2.6.16**.
  - Migration may **pause** the process and can **fail** if pages are pinned/locked/shared in disallowed ways.
  - Swapping and kernel auto-balancing may later **relocate** pages; consider `mlock/mlockall` for critical data.

**Practical tips**
- Combine mbind with thread affinity so the CPU and memory node match.
- For interleave (MPOL_INTERLEAVE), bind before touching to get even spread.
- For latency-sensitive regions, consider mlock and pre-fault to avoid swap-induced relocations.
- Verify placement with move_pages(pid=0, ...) or /proc/<pid>/numa_maps (snapshot only).

#### 6.5.5 Querying Node Information
- `get_mempolicy(int *policy, const unsigned long *nmask, unsigned long maxnode, void *addr, int flags)` queries **NUMA policy/state** for an address.
- With `flags == 0`, it returns the **governing policy** for `addr`: VMA policy if present, else task policy, else system default. Node set (if relevant) is returned via `nmask`.
- `MPOL_F_NODE`: if governing policy is `MPOL_INTERLEAVE`, `*policy` becomes the **index of the next node** that will receive an allocation for `addr`.
- `MPOL_F_ADDR`: `*policy` becomes the **index of the node that currently backs the page** containing `addr`.

**Key Points**
- **Policy resolution order**: VMA (if any) → task policy → system default.
- **Interleave insight** (`MPOL_F_NODE`): lets you anticipate **where the *next* page** will fault in—useful to align upcoming thread placement.
- **Actual placement** (`MPOL_F_ADDR`): tells you **where the *current* page lives**—useful for migration decisions or picking the most local worker thread.
- **Volatility**:
  - **CPU/node of a thread** can change due to scheduling.
  - **Page placement** is comparatively stable, but can change via swap, kernel rebalancing, or explicit migration; treat queried results as **snapshots**, not invariants.
- **libNUMA conveniences**:
  - `NUMA_mem_get_node_idx(addr)` → node index for one address.
  - `NUMA_mem_get_node_mask(addr, size, ..., dest)` → nodes covering a range (by policy/placement).
  - `NUMA_cpu_to_memnode(cpuset → memnodeset)` maps CPUs to their local memory nodes.
  - `NUMA_memnode_to_cpu(memnodeset → cpuset)` maps memory nodes back to local CPUs.
- **Typical uses**:
  - Pick **thread affinity** to match the node holding the data.
  - Decide **whether/where to migrate pages** before a phase change.
  - Combine with `mbind/set_mempolicy` to control future faults.

**Practical tips**
- Use MPOL_F_ADDR to pick the closest worker to the data; or migrate pages first, then bind worker threads.
- Use MPOL_F_NODE during interleaved allocations to pre-place threads on upcoming nodes.
- Re-query when making scheduling/migration decisions; don’t cache results long-term.
- Pair with mbind/set_mempolicy and thread affinity for end-to-end control of data & compute locality.

#### 6.5.6 CPU and Node Sets
- **CPU sets** let admins partition CPUs *and* memory nodes to constrain processes—useful for SMP/NUMA without changing application code.
- The interface is a **pseudo-filesystem** (traditionally mounted at `/dev/cpuset`); each directory is a cpuset with tunable files.
- Root cpuset initially contains **all CPUs and all memory nodes**; child cpusets inherit then refine those masks.
- A process in a cpuset has its **CPU affinity and NUMA memory policies masked** by that cpuset’s `cpus` and `mems`.
- Assignment is simple: **echo PIDs into `tasks`** of the target cpuset; restrictions are mostly invisible unless a mask becomes empty.

**Key Points**
- **Mount & layout**
  - Mount: `mount -t cpuset none /dev/cpuset` (mount point must exist).
  - Root directory exposes: `cpus` (allowed CPUs), `mems` (allowed memory nodes), `tasks` (member PIDs).
- **Create & configure cpusets**
  - Make a subdirectory → a **new cpuset** inheriting parent settings.
  - Write CPU list to `cpus` and memory nodes to `mems` to narrow the set.
- **Process assignment**
  - Add a process by writing its PID to the cpuset’s `tasks` file.
  - From then on, its **schedulable CPUs = affinity ∩ cpuset.cpus** and its **NUMA node choices = mempolicy ∩ cpuset.mems**.
- **Behavior & errors**
  - Generally **no errors**: masking happens transparently.
  - Errors surface if masking yields an **empty set** (no CPUs or no mem nodes).
- **Scope & controls**
  - Designed for **admin control** when code/changes aren’t possible.
  - Additional knobs exist (e.g., exclusivity, memory-pressure behavior) in cpuset files.
- **Terminology**
  - “CPU sets” here are **not** the `cpu_set_t` type used by `sched_setaffinity`; same words, different mechanisms.

**Practical tips**
- Use cpusets to isolate noisy workloads or keep NUMA-sensitive jobs near their memory.
- Remember masking: even if an app calls sched_setaffinity/mbind, it can’t escape its cpuset’s cpus/mems.
- Keep masks non-empty before moving tasks; empty masks will cause failures.
- Group related processes into the same cpuset to reduce cross-socket memory traffic on NUMA.

**Troubleshooting**
- If a task won’t run on certain CPUs or faults allocating memory on certain nodes, check the effective masks in its cpuset.
- Review additional files in the cpuset directory for exclusivity and memory pressure behaviors (see Documentation/cpusets.txt in the kernel tree).

#### 6.5.7 Explicit NUMA Optimizations
- When many threads across NUMA nodes need the same data, **locality rules alone won’t fix contention**.
- For **read-only data**, the best fix is **per-node replication** so each node reads its local copy.
- For **writable data**, use **per-node accumulation** (reduce/merge later) or, if necessary, **page migration**—but only when the write intensity amortizes the copy cost.

**Key Points**
- **Read-only: replicate**
  - Keep a per-node pointer table; lazily allocate a local copy on first use.
  - If a thread migrates to a different node later, it can keep working (may briefly hit remote memory until it refreshes the pointer).
- **Writable: avoid shared hot pages**
  - Prefer **node-local buffers** to accumulate results, then **combine** them (periodically or at the end).
  - If direct shared updates are required and are **bursty/non-concurrent across nodes**, consider **migrating pages** to the active node—only if the migration cost is amortized.
- **Restricting threads to the memory’s node** underuses the machine; favor strategies that **use all nodes** while keeping memory access local.
- **Thread moves happen**
  - Schedulers try to keep threads near their data, but moves can happen; code should tolerate **temporary remote hits**.

**Writable patterns**
- Per-node reduce:
  - node_buf[node] += partial; (local)
  - Periodic/global merge step to combine all node_buf[].
- Page migration:
  - Use mbind(..., MPOL_BIND|MPOL_MF_MOVE) or move_pages() only when writes are heavy and mostly from one node for a while.

**Practical tips**
- Batch work so each thread stays on a node long enough to benefit from locality.
- For read-mostly structures that change rarely, version and refresh per-node copies on update.
- Measure before migrating: migration is a full page copy; ensure enough subsequent local accesses.

**Failure modes to watch**
- Replicated data that becomes stale (add clear refresh/invalidations).
- Migrating pages too aggressively → thrashing.
- Over-serializing merges; make reductions hierarchical or asynchronous.

#### 6.5.8 Utilizing All Bandwidth
- When caches don’t help (streaming/one-time writes), **remote-node DRAM writes can be as fast as local** on some systems, while also **freeing local-memory bandwidth**.
- This can improve total throughput **if** the interconnect and the remote node’s memory controller have spare bandwidth and you **avoid write-allocate** penalties.
- The win is **workload- and platform-dependent**; validate with targeted measurements before adopting.

**Key Points**
- **When it helps**
  - Access pattern is **streaming/non-temporal** (little to no reuse; caches ineffective).
  - Local memory controller is the bottleneck; inter-socket link(s) + remote controller have headroom.
  - You can **pin threads** and **bind pages** so the write stream targets the intended remote node.
- **How to do it (Linux/NUMA)**
  - **Bind the target buffer** to a remote node (`mbind`/`set_mempolicy` with `MPOL_BIND`).
  - **Pin the writing thread** away from that node (so traffic goes over the interconnect).
  - Use **non-temporal stores** (SSE/AVX `movnt*`) to avoid write-allocate/RFO and cache pollution; ensure **64-byte alignment** and **full-line writes**; end a batch with `sfence`.
  - On AMD (Fam 10h+), consider **`prefetchw`** to obtain M-state earlier (only if you will write).
- **When it hurts**
  - Any **reuse** of written data soon after (you’ll pay remote-read latency).
  - Remote node is busy (you’ll **steal** its DRAM bandwidth).
  - Small, irregular, or misaligned writes (you’ll trigger write-allocate and coherence traffic).
  - High NUMA factors / saturated interconnect: gains vanish or go negative.

**Validation checklist**
- Measure write bandwidth and FSB/UPI/HT link utilization with perf/PMCs.
- Compare local vs. remote streaming write rates with and without non-temporal stores.
- Confirm minimal later reads from the remote-written region (or plan a staged handoff/merge).
- Watch remote node for headroom (its own workloads, LLC miss rate, DRAM BW).

**Operational tips**
- Batch writes into large, aligned ranges to maximize write-combining.
- Separate hot read-mostly structures from one-time write streams.
- If remote data will be consumed later on that remote node, you may net a double win (write offload now, local read later).

[back to top](#table-of-contents)
