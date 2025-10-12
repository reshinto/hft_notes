# What Every Programmer Should Know About Memory

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
- Once subsystems were optimized independently, **memory and storage lagged**, creating bottlenecks. OSs hid many **storage** delays with software caching, but **main memory** bottlenecks require **hardware** solutions: faster/parallel **RAM**, better **memory controllers**, **CPU caches**, and **Direct memory access (DMA)**.
- This paper (focused on **commodity hardware** and **Linux**) explains how modern memory subsystems and caches work and what programmers should do to use them effectively—while noting that real machines vary, so statements are often qualified as “usually.”

**Key Points**
1. **From balance to bottlenecks**  
   Subsystem-specific optimization left **memory & storage** improving more slowly than CPUs → system performance became **memory-bound**.

2. **Storage vs memory mitigation**  
   - **Storage:** Largely masked by **OS page cache** and device caches (software-centric).  
   - **Memory:** Needs **hardware** changes (RAM design/parallelism, controllers, caches, DMA).

3. **Focus of the paper**  
   Practical guidance on **CPU caches** and **memory controllers**, with **DMA** folded into the big picture, using **commodity hardware** examples.

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

2. **Section 3 — CPU Caches (essential)**  
   Detailed **cache behavior** with graphs (lines, sets, associativity, misses, prefetchers). *Core to understanding performance.*

3. **Section 4 — Virtual Memory (essential)**  
   How **address translation**, **pages/ TLBs**, and protection work; prerequisite for later topics.

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
- Practical upshot: SRAM is used in **CPU caches** (small, fast, directly addressed), while **DRAM** is used for **main memory**.
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
  **CPU caches/on-die SRAM** are **directly addressed** and kept small; **SRAM max speed** depends on design effort and can be **slightly slower than the core** up to **10–100× slower**. *(§2.1.4)*

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

## CPU Caches
