# What Every Programmer Should Know About Memory — Summary & Cheat Notes

- **Memory access is the bottleneck.** CPUs gained speed and cores faster than memory latency/bandwidth improved, so programs often stall waiting for data.
- **Caches bridge the gap—but need your help.** CPU caches can only shine if code uses cache-friendly data layouts and access patterns.
- **Storage vs memory diverged.** Storage slowness is largely hidden by OS/device caching; **main memory** bottlenecks required **hardware** innovations.
- **Paper’s focus:** How modern **memory subsystems** and especially **CPU caches** work, and what programmers should do to achieve optimal performance (Linux for OS-specifics).
- **No absolutes.** Real systems vary; expect “usually” rather than “always.”

## Key Points (with brief explanations)

1. **Modern performance limit = memory, not CPU**  
   CPUs outpaced DRAM; the processor is often idle waiting for data. Optimize **data movement**, not just instructions.

2. **Caches are essential but not automatic**  
   They reduce effective latency only when code **reuses nearby data** (spatial locality) and **reuses the same data soon** (temporal locality).

3. **Historical reason for caches**  
   Early machines were balanced; later subsystem optimization created a **CPU–memory speed gap**. Caches were introduced to mask DRAM latency.

4. **Storage vs memory solutions**  
   - **Storage:** OS page cache and drive caches (software-heavy).  
   - **Memory:** Needs **hardware**: faster/parallel **Random-access
memory (RAM)**, smarter **memory controllers**, **CPU caches**, and **Direct memory access (DMA)**.

5. **Core hardware levers you should know**  
   - **RAM design:** affects peak bandwidth/parallelism.  
   - **Memory controllers:** schedule DRAM access efficiently.  
   - **CPU caches:** multi-level, coherent, with lines/sets/policies.  
   - **DMA:** devices move data without CPU, affecting cache coherence.

6. **Scope and limits of the paper**  
   Commodity hardware; emphasizes **CPU caches** and their practical impact. OS specifics are **Linux-only**. Many topics are intentionally shallow to stay practical.

7. **Qualifiers matter**  
   Hardware diversity makes universal rules rare; always measure on your target system.

### Why This Matters (What you’ll gain)

- A mental model of the **memory hierarchy** (CPU ↔ caches ↔ controllers ↔ DRAM).
- Awareness that **data layout and access order** can dominate performance.
- A checklist mindset for writing **cache-cooperative code** (you’ll see the “how” in later sections of the paper).

#### Ultra-short One-Liners
- **Bottleneck:** *Data arrival beats instruction speed.*  
- **Caches:** *Fast only if you feed them right.*  
- **Storage vs Memory:** *Software hides disks; hardware helps RAM.*  
- **Focus:** *CPU caches + Linux details.*  
- **Rule of thumb:** *“Usually,” not “always.” Measure!*

#### Quick Checklist (use while coding)
- [ ] **Access data sequentially** where possible (spatial locality).  
- [ ] **Reuse hot data soon** (temporal locality).  
- [ ] **Pack/align structures** to reduce wasted cache lines.  
- [ ] **Batch work** to reduce random access and cache thrash.  
- [ ] **Measure** on your hardware; avoid one-size-fits-all assumptions.

## Commodity Hardware Today
- **Scale is horizontal.** Data centers favor many commodity servers over a few specialized machines due to cheap, fast networks. 
- **Typical (as of 2007) building block:** up to **4 sockets × quad-core × hyper-threading ≈ 64 virtual CPUs**; optimizations target this class. 
- **Classic PC/server layout:** **Northbridge/Southbridge**. CPUs talk to RAM and devices **through the Northbridge**; Southbridge handles I/O buses (PCI/PCI-E/SATA/USB). *(Fig. 2.1)*
- **Bottlenecks:** shared **Front-Side Bus (FSB)**, single/limited **memory channels**, and **Northbridge** contention (CPUs vs device **DMA**). 
- **Bandwidth evolution:** single memory bus → **dual channels (DDR2)** → more channels (e.g., **FB-DRAM**). **Interleaving** spreads accesses across channels. 
- **Architectural variants:**  
  1) **External memory controllers** beside Northbridge (more buses, more bandwidth/memory). *(Fig. 2.2)*  
  2) **Integrated memory controllers** in each CPU → **local RAM per CPU**, higher aggregate bandwidth but **NUMA** effects for remote RAM. *(Fig. 2.3)*
- **NUMA reality:** Accessing **remote** memory (via inter-CPU links) costs extra time (“**NUMA factors**”); topologies may have multiple “hops”/levels. 

### 1) Northbridge/Southbridge (Fig. 2.1)
**What it is:**  
- CPUs connect via **FSB** to the **Northbridge** (which includes the **memory controller**); **Southbridge** handles I/O (PCI/PCI-E, SATA, USB; legacy PATA/1394/serial/parallel). 

**Consequences / bottlenecks:**  
- **CPU↔CPU traffic** and **CPU↔RAM** share the **same FSB/Northbridge path**.  
- **Device DMA** now bypasses the CPU (good) but **competes** with CPUs for **Northbridge bandwidth** (contention).  
- **Older single-bus RAM** disallows parallel access; newer designs add **multiple channels**. 

### 2) Northbridge with External Memory Controllers (Fig. 2.2)
**What it is:**  
- Several **external memory controllers (MCs)** attach to Northbridge, each with its **own memory bus**. 

**Why it helps:**  
- **More buses ⇒ more total bandwidth** and **more addressable memory**.  
- **Concurrent** access across **different banks** reduces delay, especially with multiple CPUs.  
- Primary limit shifts to **Northbridge’s internal bandwidth** (noted as “phenomenal” for this Intel design). 

### 3) CPUs with Integrated Memory Controllers (Fig. 2.3)
**What it is:**  
- **Each CPU** has a **built-in memory controller** and **local RAM** (e.g., AMD Opteron; Intel later via CSI/Nehalem). 

**Why it helps:**  
- **As many memory banks as CPUs**; on 4-CPU machines, **aggregate memory bandwidth ≈ 4×** without a huge Northbridge. 

**Trade-off — Non Uniform Memory Access (NUMA) systems:**  
- Memory is **Non-Uniform**:  
  - **Local** RAM: usual speed.  
  - **Remote** RAM: must traverse **inter-CPU interconnects**; each hop adds cost (**“NUMA factor”**).  
  - Topologies can have **multiple levels/hops**; some systems group CPUs into **nodes** with cheap intra-node and expensive inter-node access. 

### 4) Memory Channels & Access Patterns
- **Bandwidth depends on channels**: old **single bus** vs **dual channel (DDR2)** vs **more channels** (e.g., **FB-DRAM**).  
- **Northbridge interleaves** accesses across channels.  
- **Concurrency** (many threads/CPUs/DMA) **and access pattern** (how you touch memory) **strongly affect latency/bandwidth**.

### “Think-Before-You-Code” Implications
- Expect **contention** when many cores/threads and devices **hit memory simultaneously**.  
- **Access pattern matters** (channel interleaving benefits sequential/structured layouts).  
- On **NUMA**, prefer **locality** (keep working threads/data near their CPU); avoid unnecessary **remote** accesses.

---

### RAM Types
- **Two main RAM types in one machine:** **SRAM** (fast, costly) + **DRAM** (dense, cheap). We use **SRAM for caches** and **DRAM for main memory** because of **cost and power**.
- **SRAM cell (6T):** flip-flop of cross-coupled inverters → **stable state** as long as power (Vdd) is present; **no refresh**, **near-instant read**, but **6 transistors per bit** and continuous power. *(Fig. 2.4)*
- **DRAM cell (1T+1C):** **capacitor** holds charge = bit; **destructive read**, must **refresh** (typically **every ~64 ms**); **slower** due to **RC charge/discharge** and **sense amplifier** steps, but **tiny area per bit** ⇒ far cheaper & denser. *(Fig. 2.5–2.6)*
- **Addressing DRAM:** cells are in **rows × columns**; controller uses **RAS** (row) then **CAS** (column) to cut pin count via **address multiplexing**. Reads/writes obey timing windows determined by RC physics. *(Fig. 2.7)*
- **Key consequences:** more pins/lines ⇒ cost; **results are not immediate**; access/refresh timings dominate performance. 

#### Static RAM (SRAM) — 6T cell
- **Structure:** 6 transistors (M₁–M₆); core is **two cross-coupled inverters** (M₁–M₄). *(Fig. 2.4)*
- **Operation:** Raise **WL** to read; **BL/BL̄** carry the bit immediately. To write, drive BL/BL̄, then raise WL.
- **Properties:**  
  - **Fast read**, **no refresh**; **rectangular (sharp) signals**.  
  - **Always powered** to keep state.  
  - **Area/cost high** (≈6 transistors/bit).

#### Dynamic RAM (DRAM) — 1T1C cell
- **Structure:** 1 **transistor** (M) + 1 **capacitor** (C). *(Fig. 2.5)*
- **Operation:** Raise **AL** to access; **DL** senses charge (read) or supplies charge (write).
- **Complications:**  
  - **Leakage**: tiny caps (femto-farad range) lose charge; **refresh ~ every 64 ms**; refresh blocks access.
  - **Destructive read**: read depletes C; must **restore** via sense amp feedback. 
  - **Slow edges**: charge/discharge follow **RC** curves (below) ⇒ extra delay and energy.
- **Advantage:** **Much smaller, simpler, denser** than SRAM ⇒ **dramatic cost difference** ⇒ chosen for main memory.

#### RC Timing (why DRAM is slower)
- **Equations (from text):**  
  - Charging: `Q_charge(t) = Q0 · (1 − e^(−t/RC))`  
  - Discharging: `Q_discharge(t) = Q0 · e^(−t/RC)`  
  - **Implication:** it takes several **RC** time constants for the sense amp to see a reliable 0/1; signals are not instantaneous. *(Fig. 2.6)*

#### How DRAM Is Addressed (RAS/CAS)
- **Problem:** Directly exposing all address lines is impractical (e.g., 1 Gbit ⇒ ~30 lines; huge demux & pin cost). 
- **Solution:** Arrange cells in **rows × columns**; send address in **two parts**:  
  1) **RAS** selects a **row**;  
  2) **CAS** selects a **column**;  
  **Multiplexing halves external address pins**; timing signals indicate RAS/CAS phases. *(Fig. 2.7)*
- **Timing matters:** specs define **how long after RAS/CAS data appears** and **how long data must be held** for writes (caps don’t fill/drain instantly). 

#### Practical Implications
- **Expect latency** from DRAM due to **RC + RAS/CAS + refresh**; organize software for **good locality** and **burst/row reuse** (later sections).
- **Results aren’t instant:** even after issuing a read, **data takes time** to be valid; writes also need **hold time**. 
- **Hardware cost drivers:** **address lines/pins** significantly affect controller/module cost. 

#### Tiny Glossary (from the excerpt)
- **WL / AL / BL / DL:** Word/Access/Data lines controlling reads/writes. 
- **Sense amplifier:** Circuit that decides 0 vs 1 from tiny charge differences and **restores** the cell after read.   
- **RAS / CAS:** **R**ow **A**ddress **S**trobe, **C**olumn **A**ddress **S**trobe; the two-phase addressing for DRAM. 
- **RC time constant:** Product of **resistance × capacitance**; sets the speed of charging/discharging. 

#### Conclusions
- Not all memory is SRAM for cost/power reasons.
- Cells must be selected and pins/lines drive cost across controller/board/module/chip.
- Read/write results take time to become valid due to physical RC limits and protocol timing.
- SRAM is used for caches (size-limited but fast); DRAM is used for main memory (large but slower).

---

### DRAM Access Technical Details

- **SDRAM/Double Data Rate (DDR) are clocked (synchronous).** A read = **RAS (row)** → **tRCD** → **CAS (column)** → **CL (CAS latency)** → **burst** on DQ. Precharge (**tRP**) and **tRAS** gate when the next row can start. *(Fig. 2.8–2.9)*
- **Throughput vs latency:** The bus can burst at high rates, but **gaps from tRCD/tRP/tRAS** slash sustained bandwidth unless accesses reuse open rows and burst multiple words.
- **Refresh matters:** DRAM must refresh about **every 64 ms**; with **8192 rows** that’s ~**7.8125 µs** between refreshes on average, blocking access to the refreshed row.
- **Generations:** **SDR → DDR1 (double data rate) → DDR2 (faster bus + 4-bit I/O buffer) → DDR3 (lower V, 8-bit I/O buffer)**. Names (PC/DDR-xxxx) reflect **effective** rates.
- **FB-DIMM (FB-DRAM):** Serial, full-duplex channel cuts pins (≈**69 vs 240**), allows **more channels & DIMMs**, and higher aggregate bandwidth; trade-offs: extra latency per hop and higher module power.
- **Other memory users:** **DMA** devices contend for memory/FSB bandwidth; NUMA layouts can isolate DMA vs compute memory. Integrated-graphics sharing system RAM hurts latency.

#### 1) SDRAM Read: Timing Anatomy (Fig. 2.8)
- **Row open:** Put **row address** on Address bus, **lower RAS**.  
- **tRCD:** Wait **RAS-to-CAS delay** cycles.  
- **Column select:** Put **column address**, **lower CAS**.  
- **CL:** Wait **CAS Latency** cycles, then **data burst** appears on **DQ** (1 word/cycle for Single Data Rate (SDR), 2 for DDR).  
- **Burst length:** Controller chooses 2/4/8 words to amortize setup; can issue **additional CAS** to the same open row (subject to **Command Rate** Tx).  
- **Key insight:** Row reuse (keeping it “open”) minimizes RAS work; random row switches amplify latency.

#### Timing Symbols (appear in specs like **w-x-y-z-T**, e.g., **2-3-2-8-T1**)
- **CL (w):** CAS Latency (cycles).  
- **tRCD (x):** RAS→CAS delay.  
- **tRP (y):** Row Precharge time (to close row before next open).  
- **tRAS (z):** Minimum **Active (row open) → Precharge** delay.  
- **T:** Command Rate (**T1** = new command each cycle; **T2** slower).

#### 2) Precharge & Next Activation (Fig. 2.9)
- After the burst, issue **Precharge** (often WE↓ + RAS↓ combination).  
- Must wait **tRP** before a **new RAS**. Also, **tRAS** may force additional delay if the prior row wasn’t open long enough.  
- **Illustrative occupancy:** Example shows **2 cycles of data** out of **7 total cycles** → the 800 MHz-effective × 8 B burst **6.4 GB/s** shrinks to ~**1.8 GB/s** (6.4 × 2/7). 
- **Takeaway:** Align access patterns to reduce precharge/activate churn (reuse rows, longer bursts).

#### 3) Refresh (Recharging)
- **Requirement:** Refresh **every ~64 ms**. For **8192 rows**, controller schedules refresh on average every **64 ms / 8192 ≈ 7.8125 µs**.  
- **Effect:** During refresh of a row, **no access** to that row; may cause visible stalls depending on timing.  
- **Programmer note:** You can’t control refresh, but remember it when interpreting sporadic long-tail latencies.

#### 4) Throughput Math & “Effective” Bus
- **Burst bandwidth formula:** `BW_theoretical = (bus_width_bytes) × (effective_bus_freq)`  
  - Example in text: **8 B × 800 MHz = 6.4 GB/s** (quad-pumped 200 MHz).
- **Sustained bandwidth** depends on **occupancy** (fraction of time the bus actually transfers data).  
  - Example: **2/7 occupancy ⇒ 6.4 × 2/7 ≈ 1.8 GB/s**.

#### 5) SDR → DDR1/DDR2/DDR3 (Figs. 2.10–2.13)
**Core ideas:**
- **SDR:** Array and bus at same freq **f**; one transfer per cycle. *(Fig. 2.10)*
- **DDR1:** **Double-pumped bus** (rise+fall) with 2-bit I/O buffer; latency unchanged, burst faster. *(Fig. 2.11)*
- **DDR2:** Bus at **2f**, 4-bit I/O buffer; array still at **f**. Names reflect 4× factor (quad-pumped). *(Fig. 2.12)*
- **DDR3:** Lower voltage (**~1.5 V vs 1.8 V** → ~30% power reduction by V²), bus faster, **8-bit I/O buffer**; initially higher CL but enables higher bandwidth; naming per Table 2.2. *(Fig. 2.13)*

**Naming examples from tables:**
- **DDR2**: e.g., **PC2-6400** (6,400 MB/s) a.k.a. **DDR2-800** (bus 400 MHz, effective 800 MHz).
- **DDR3**: e.g., **PC3-12800** a.k.a. **DDR3-1600** (bus 800 MHz, effective 1600 MHz).

**Practical limits (from text):**
- Many pins: ~**240 pins**/DIMM and signal-integrity constraints → typically **≤2 DIMMs/channel (DDR2)** and **often 1 DIMM/channel at high DDR3 rates**. This caps capacity per controller unless more channels/controllers/IMCs are used.

#### 6) FB-DRAM (Fully Buffered DIMM) Highlights
- **Serial, full-duplex differential links** instead of wide parallel bus.  
- **Pins:** ~**69** per module vs **240** (DDR2).  
- **Scalability:** **Up to 8 DIMMs/channel**; controllers can drive **more channels** (e.g., 6).  
- **Aggregate bandwidth:** Example table compares **~10 GB/s (DDR2)** vs **~40 GB/s (FB-DRAM)** for a controller configuration.  
- **Trade-offs:** Additional **per-DIMM hop latency** and **higher energy** for the serial chip; still advantageous for large-memory servers.

### Other Main-Memory Users (DMA)
- **DMA devices** (NICs, storage) read/write RAM directly, **competing** with CPUs for memory/FSB bandwidth → potential extra stalls.  
- **NUMA mitigation:** With per-CPU memory (Fig. 2.3), place compute on nodes away from heavy-DMA nodes; even attach Southbridges per node to distribute load.  
- **Integrated graphics using system RAM** adds heavy, frequent traffic and can increase latency; avoid for performance-critical systems.

#### Quick Math
- **Theoretical burst:** `8 B × 800 MHz = 6.4 GB/s` (quad-pumped 200 MHz bus).
- **Observed in example:** `6.4 × (2/7) ≈ 1.8 GB/s` when timing gaps dominate.
- **DDR3 power note:** `P ∝ V²` → from **1.8 V → 1.5 V** ≈ **(1.5² / 1.8²) ≈ 0.69** → ~**30% reduction**.

#### Practical Tips (implied by the section; details later in paper)
- **Batch & stream:** Prefer sequential, row-friendly bursts (fill cache lines; reuse open rows).  
- **Prefetch:** Use hardware/soft prefetch to overlap timing gaps and shift contention.
- **NUMA-aware placement:** Keep threads near their data; separate DMA-heavy devices from compute memory when possible. 
- **Avoid shared-RAM graphics** in performance builds.
