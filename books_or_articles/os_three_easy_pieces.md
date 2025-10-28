# Operating Systems: Three Easy Pieces

## Introduction to Operating Systems
### The First Piece: Virtualization - The Art of Illusion

**What is Virtualization?**
- At its core, the operating system is a resource manager.
- It is in charge of the computer’s processor (CPU), memory, and disks, and its primary role is to manage these resources efficiently and fairly.
- The main way the OS achieves this is through virtualization.
- The OS takes a physical resource, like a processor or system memory, and transforms it into a more general, powerful, and easy-to-use virtual form of itself.
- This is why the operating system is sometimes referred to as a virtual machine:
  - it creates an illusion for programs, making them believe they have exclusive access to resources they are actually sharing with many other programs.

**Example: Virtualizing the CPU**
This is a simple program that does nothing more than loop and print a string.
```c
char *str = argv[1];
while (1) {
    Spin(1); // A function that loops for roughly one second
    printf("%s\n", str);
}
```
Running one instance of this program is straightforward. But what happens if we run four separate instances at the same time on a computer with only one processor?
```
prompt> ./cpu A & ; ./cpu B & ; ./cpu C & ; ./cpu D &
[1] 7353
[2] 7354
[3] 7355
[4] 7356
A
B
D
C
A
B
```
- This output reveals the magic. As the source text asks:
  - "Even though we have only one processor, somehow all four of these programs seem to be running at the same time! How does this magic happen?"
- This illusion is CPU virtualization.
  - The operating system, with help from the hardware, creates the appearance of a nearly infinite number of CPUs.
  - It achieves this by running one program for a short burst, then stopping it and running another.
  - This technique of rapidly switching between programs is called time sharing, and it is the foundation of modern multitasking.
  - This time sharing of the CPU is one of the OS's primary strategies; for other resources, like disk space, it uses a technique called space sharing, where the resource is divided among users.

**Example: Virtualizing Memory**
Consider a program that allocates some memory, prints the address of that memory, and then loops.
```c
int *p = malloc(sizeof(int)); // Allocate memory
printf("(%d) address of p: %08x\n",
        getpid(), (unsigned) p); // getpid() returns the unique process identifier
*p = 0;
while (1) {
    Spin(1); // A function that loops for roughly one second
    *p = *p + 1;
    printf("(%d) p: %d\n", getpid(), *p);
}
```
When we run two instances of this program simultaneously, we see something truly surprising in the output:
```
(24113) memory address of p: 00200000 (24114) memory address of p: 00200000
```
Both processes believe they have allocated memory at the exact same address (00200000).
Yet, they run without interfering with one another. How is this possible?
  - This is memory virtualization.
    - The address that each process "sees" is not a direct physical address.
    - Instead, the OS gives each process its own private virtual address space.
    - Each process can read and write to its own virtual addresses (like 00200000) as if it owned the machine.
    - Behind the scenes, the OS and hardware work together to map these virtual addresses to different, unique locations in the computer's actual physical memory.

**Why Virtualization is Fundamental**
- Virtualization provides a clean, powerful, and essential abstraction.
- It allows programs to run as if they each have their own dedicated computer, which dramatically simplifies the task of programming.
- While virtualization allows many programs to seem to run at once, managing their interactions when they try to work together introduces a new set of challenges, known as concurrency.

### The Second Piece: Concurrency - The Challenge of Cooperation

**What is Concurrency?**
- Concurrency refers to the host of problems that arise when multiple threads of execution attempt to access and update shared data at the same time.
- These problems first appeared inside the OS itself, but they are now common in modern multi-threaded applications.

**Example: A Race Condition**
- Consider a multi-threaded program that creates two threads.
- Each thread's job is to increment a shared counter variable a specified number of times.
- When we run this program with a small number of loops, it behaves as expected.
- But when we increase the number of loops substantially (e.g., to 100,000), something strange happens.
```
Loops per Thread
Expected Final Value
Actual Final Value (from text)
1,000
2,000
2,000
100,000
200,000
143012 (huh??)
```
- This bug is non-deterministic, meaning it doesn't happen the same way every time.
- The source text highlights this by running it again: 137298 // what the??.
- This unpredictability makes concurrency bugs notoriously difficult to find and fix.

**The Crux of the Problem: Non-Atomic Operations**
- This brings us to the crux of the problem:
  - How can we build correct concurrent programs when even simple C statements are not atomic?
- The concurrency bug happens because what looks like a single, simple operation in C is not a single, indivisible operation for the processor.
  - The line of C code counter = counter + 1; is not atomic.
  - Instead, the compiler translates this single C statement into a sequence of three machine instructions:
    1. Load the value of the counter from memory into a register.
    2. Increment the register.
    3. Store the new value from the register back into memory.
- Because these three steps are separate, an untimely interruption from the OS can cause a huge problem.
  - Imagine Thread 1 completes steps 1 and 2 (reading the counter's value of 50 and calculating 51 in its register).
  - Before it can perform step 3, the OS switches to Thread 2.
  - Thread 2 then runs all three steps: it reads the counter (still 50), increments it to 51, and writes 51 back to memory.
  - When the OS finally switches back to Thread 1, it resumes at step 3 and also writes its value—51—to memory.
- The result: two increments occurred, but the counter's final value is 51, not 52.
  - One of the updates was lost.
  - This scenario, where the outcome depends on the uncontrolled scheduling of threads, is called a race condition.

**Why Concurrency is Fundamental**
- To build correct concurrent programs, the OS must provide tools to manage this chaos.
- The piece of code that accesses the shared resource is called a critical section.
- When multiple threads enter a critical section at once, a race condition occurs, and the result of the program becomes indeterminate—meaning it is not the same from run to run.
- To ensure correctness, programmers must use OS-provided synchronization primitives, such as locks, to guarantee mutual exclusion.
- This ensures that only one thread can enter a critical section at a time, making the operations within it execute atomically.
- Just as the OS must manage concurrent access to shared memory, it must also manage long-term access to shared data on storage devices, a concept known as persistence.

### The Third Piece: Persistence - The Memory That Lasts
**What is Persistence?**
- System memory (like DRAM) is volatile; its contents are lost when the power goes out.
- For data to last, it must be stored on a persistent storage device, such as a hard disk or solid-state drive.
- The OS is responsible for managing these devices and providing a way for programs to store data for the long term.

**Example: Writing a File**
- This is a simple program that creates a file and writes "hello world" into it.
```c
int fd = open("/tmp/file", O_WRONLY | O_CREAT | O_TRUNC, S_IRWXU);
write(fd, "hello world\n", 13);
close(fd);
```
- To accomplish this task, the program makes three system calls: open(), write(), and close().
- By providing these calls, the OS is acting as a standard library for applications, giving programmers a simple, consistent API to interact with complex hardware.
- These particular calls are requests to the part of the OS called the file system, which is responsible for managing persistent data.

**The Crux of the Problem: Hiding Complexity**
- This brings us to the crux of the problem:
  - How can the file system store data persistently, with high performance and reliability, despite the complexities of hardware and the possibility of system failure?
- The file system provides a simple, clean interface, but it hides immense underlying complexity.
  - The source text highlights several key challenges the file system must handle:
    1. Device Complexity: The file system must deal with the intricate, low-level details of countless different storage devices, each with its own specific interface for reading and writing data.
    2. Performance: To be fast, the file system uses sophisticated techniques like caching frequently used data in memory and batching many small writes into a single, larger, more efficient write to the disk.
    3. Reliability: The file system must guarantee that data is not lost, even if the system crashes in the middle of a write operation.
        - It uses complex protocols like journaling or copy-on-write to ensure it can recover to a consistent state after a crash.

**Why Persistence is Fundamental**
- The file system is another powerful abstraction.
- It provides a simple, standard library of calls that allows programmers to save data for the long-term without having to worry about the messy, complex details of hardware interfaces or crash recovery protocols.
- By managing virtualization, concurrency, and persistence, the operating system provides the foundational illusions that make modern computing possible.

### Conclusion: The Three Pieces Together
1. Virtualization transforms physical hardware into more useful virtual resources, creating the illusion of a nearly infinite number of processors and private memory spaces.
2. Concurrency control provides the necessary tools, like locks, to manage the chaos of parallel execution and ensure correct operation on shared resources.
3. Persistence provides the abstractions of the file system, offering a simple and reliable way to store data for the long term.
- These three pieces form the conceptual core of what an operating system is and does.
- Understanding them provides a foundational and powerful way to reason about complex systems.
- They are the mechanisms that make our powerful computers usable.

## The Abstraction: The Process
### Overview
1. The Grand Illusion
    - **Problem OS solves:** Make one (or few) CPUs feel like many things run at once.
    - **Key abstraction:** **Process** = a **running** program with live state and resources.
    - **Program vs process (analogy):**
      - **Program:** recipe in a book (passive, on disk).
      - **Process:** a chef actively cooking (active, in memory).
      - Many chefs can cook the **same** recipe → many processes from one program.
    - **Two superpowers:**
      - **Time sharing (CPU virtualization):** take fast turns on the CPU to create the *appearance* of parallelism.
      - **Isolation (protection):** each process has a private address space; one crash doesn’t sink the system.
    - **Modes & safety:** **User mode** (apps) vs **Kernel mode** (OS). Sensitive actions require **system calls** that trap into the kernel.
    - **Kitchen analogy:** One cutting board (CPU), many cooks (processes), the head chef (OS) assigns turns and enforces boundaries.
    - **Layman recap:** A process is a program “brought to life.” The OS lets many live programs take turns on the CPU and keeps them from interfering.
    - **ELI5:** The rules of tic-tac-toe are the program; each separate game you play is a process.
    - |Program|Process|
      |-------|-------|
      |A passive entity|An active entity|
      |A file on disk|A program in execution|
      |Lifeless instructions|Has state: memory, etc.|
2. What Makes a Process (Machine State)
    - A process is defined by everything it can read/change while running:
      - **Memory (address space):** code, static data, heap, stack.
      - **CPU state:** general registers, flags, **program counter (PC)**, stack pointer.
      - **I/O state:** open files/sockets (e.g., stdin/stdout/stderr).
      - **Identity:** PID, parent PID, credentials.    
    - **Virtual memory & isolation (unique point):**
      - Each process sees its own **virtual** addresses (e.g., `0x200000`).
      - The OS/MMU maps those to **different physical** RAM for each process.
      - Same instruction in two processes can touch different physical memory safely.
    - **Layman recap:** Every chef has private ingredients and tools; the OS maps each chef’s counter to a different physical spot in the kitchen.
    - **ELI5:** Your toy box looks the same to you as your friend’s does to them, but the toys are in different rooms.
3. Lifecycle & States (State Machine)
    - Core states (simple model):
      - **Running:** executing on a CPU.
      - **Ready:** can run, waiting its turn (in the ready queue).
      - **Blocked/Waiting:** can’t run until an external event completes (e.g., I/O).
    - Extended states:
      - **New:** being created and set up.
      - **Terminated:** finished; OS is cleaning up.
    - **Typical transitions:**
      - Ready → **Running** (scheduler picks it).
      - Running → **Ready** (time slice expired; **preemption**).
      - Running → **Blocked** (e.g., I/O request).
      - Blocked → **Ready** (I/O completes; interrupt wakes it).
    - **Layman recap:** Cooking now (Running), waiting for stove (Ready), waiting for oven timer (Blocked).
    - **ELI5:** In a checkout line: your turn (Running), waiting in line (Ready), waiting for something else first (Blocked).
4. The PCB & Context Switch (The Sleight of Hand)
    - **PCB (Process Control Block):** the OS’s “passport/save file” for a process.
      - Fields include **PID**, **state**, **register snapshot** (PC, SP, GP regs), **memory map/page tables**, **open files**, **priority/scheduling data**, **parent**, and **kernel stack pointer**.
      - **Kernel stack (unique point):** Separate protected stack used **only** while in kernel mode (system calls/interrupts) to preserve kernel integrity.
    - **Context switch (mechanism):**
      1. OS regains control (timer interrupt or system call).
      2. Saves current registers to PCB(A).
      3. Chooses next process.
      4. Restores registers from PCB(B).
      5. Returns to user mode; B continues.
    
    - **Cost model already in your notes (illustrative):**
      - **System call:** ~**0.5 µs**.
      - **Context switch:** ~**5 µs**.
      - **Disk I/O wait:** >**1000 µs**.
    - **Hidden dominant cost:** **Cache pollution/thrashing.**
      - Switching to B fills L1 with B’s data/code, evicting A’s “hot” lines.
      - When switching back, A’s L1 is cold; refills from L2/L3/DRAM cause unpredictable latency.
    - **Layman recap:** A pit stop: time is spent swapping drivers and their seat setup; afterward, the new car’s quick-access memory isn’t warm yet.
    - **ELI5:** When a new kid uses the computer, your apps close and theirs open; when you come back, yours must reload.
5. Scheduling & Fairness
    - **Preemptive multitasking** (not cooperative): hardware **timer** fires every few ms; OS can forcibly preempt any process to ensure fairness.
    - **Round-robin:** ready queue + fixed **time slice/quantum**.
    - **Trade-off:** **Short quantum →** snappy feel but more switching overhead; **Long quantum →** better throughput but laggier feel.
    - **Tiny numeric example (from notes):**
      - If switch cost is “1 unit,” a 10-unit quantum wastes ~**9%** on switching; a 100-unit quantum wastes **<1%** but feels less responsive.
    - **Layman recap:** Like sharing a toy. Short turns = everyone tries sooner but you waste time passing the toy; long turns = more play per child, longer waits.
    - **ELI5:** A teacher calls on students one at a time for a fixed time.

6. Interleaving CPU & I/O (Worked Example)
    - **A (I/O-bound):** 10 ms CPU, then 10 ms disk I/O, repeat.
    - **B (CPU-bound):** 50 ms straight CPU.
    - **Naïve:** Run A alone → CPU idle during A’s I/O.
    - **Smart:** When A blocks for I/O, run B → **overlap** B’s compute with A’s wait.
    - Result: Higher utilization, shorter total finish time.
    - **Layman recap:** While pasta water boils (A waiting), chop veggies (B compute).
    - **ELI5:** While you wait for a download, play a game.
7. Core System Calls (Process Management)
    - **`fork()`**: Clone current process to create a child.  
      - Parent gets child **PID**; child gets **0**.
    - **`exec()`**: Replace current process image with a new program (same PID; never returns on success).
    - **`wait()`**: Parent waits for child to terminate and collects exit status.  
      - **Zombie**: child exited but parent hasn’t `wait()`ed yet; PCB persists with status until reaped.
    - **`exit()`**: Process terminates itself.
    - **Layman recap:** `fork` makes a twin; `exec` swaps your body for a new one; `wait` is a parent waiting for a child to finish.
    - **ELI5:** Copy yourself, put a different costume on, and your parent waits for you to come home.
8. Practical Extensions for Software Engineers
    - **Debugging “works on my machine”:** often **process state** differences:
      - Env vars, permissions, library versions, open files, **ulimit**s.
    - **Reliability via isolation:**
      - Split components into separate processes for **crash containment** and **graceful degradation**.
    - **Performance sensitivities:**
      - **Context switches:** too many processes/threads → overhead + cache thrash.
      - **Syscalls:** each is a kernel trap; batch I/O, avoid hot-path syscalls.
      - **Blocking I/O:** use threads, **async/non-blocking I/O**, or dedicated I/O processes.
    - **Concurrency models (design choice):**
      - **Multi-process:** strong isolation, higher memory/IPC cost.
      - **Multi-thread:** easy sharing, needs careful synchronization (locks).
      - **Async I/O/event loop:** great for I/O-bound single-threaded servers, but can get complex.
    - **Security & least privilege:**
      - Run as unprivileged users; apply resource limits to prevent runaway resource use.
    - **Tools (from notes):**
      - **Linux/macOS:** `strace` / `dtruss` for syscall traces.
      - **System monitors:** `top`, `htop`, Activity Monitor/Task Manager (states, CPU, memory, context-switch rates).
      - **Profilers:** identify hot paths & syscall hotspots.
    - **Quick checklist when an app stalls:**
      1. **State:** Running/Ready/Blocked/Zombie?
      2. **Blocked on I/O?** Check with `strace/dtruss`.
      3. **CPU-starved?** Competing priorities/ready queue congestion?
      4. **Memory thrash?** Page faults, excessive RSS growth?
      5. **Deadlock (threads)?** Lock waits?
      6. **Resource limits?** `ulimit`, file descriptors, mem caps.
9. Why This Matters for Quant Devs / HFT (Low-Latency)
    - **Goal:** ultra-low, **predictable** latency; **jitter** kills strategies.
    - **OS-induced unpredictability:**
      - **Scheduling latency:** preemption/migration at bad moments.
      - **Context switch cache effects:** cold L1 → ns–µs penalties.
    - **Typical mitigations (from notes):**
      1. **No blocking I/O** on hot path (async or offload).
      2. **Minimize syscalls** (kernel bypass: DPDK/Onload, mmap, batched I/O).
      3. **CPU pinning/affinity:** keep the critical process on one core; avoid migrations and keep caches hot.
      4. **Core isolation:** keep other tasks/interrupts off the dedicated core (e.g., isolcpus, tickless/nohz full).
      5. **NUMA awareness:** colocate CPU, memory, NIC.
      6. **Process isolation for risk:** separate feed/strategy/execution/risk processes; fast restarts; strict resource limits.
    - **Conceptual latency budget (illustrative from notes):**
      - **User compute:** target sub-µs; optimize code paths.
      - **Kernel time (syscalls):** aim **0** on hot path (bypass).
      - **Context switches:** aim **0** on hot path (pin & isolate).
      - **I/O waits:** aim **0** on hot path (async).
    - **Layman recap:** Keep the trader’s critical brain on one seat, don’t make it stand up, don’t make it ask the OS for favors, and don’t let anyone else bump its elbow.
    - **ELI5:** Give one kid the desk, stop other kids from poking him, and don’t make him go ask the teacher during the test.
    - |Source of Delay|Typical Cost|
      |---------------|------------|
      |User Computation|Varies|
      |Kernel System Call|`~0.5µs`|
      |Context Switch|`~5µs`|
      |Disk I/O Wait|`>1000µs`|
10. Historical Context & Policy
    - **Why processes were invented:** keep the expensive CPU busy while slow I/O happens → **multiprogramming** and **time sharing.**
    - **Cooperative → preemptive:** from “programs politely yield” to “OS forces fairness with timer interrupts.”
11. Open Questions / Future Directions
    - The **cache-pollution** cost of context switching is a fundamental bottleneck.
    - Could hardware evolve to:
      - Partition caches per process/core?
      - Preserve more state across switches?
      - Offload more virtualization to hardware?
    - Aim: **strong isolation** *without* steep switching penalties.
12. One-Page Recap
    - **Process:** running program with private memory and CPU state.
    - **Illusion:** time sharing makes one CPU look like many.
    - **Safety:** isolation via virtual memory + user/kernel modes.
    - **Mechanics:** PCB + context switch (fast, but cache-expensive).
    - **Scheduling:** preemptive, quantum trade-off: responsiveness vs throughput.
    - **I/O overlap:** run CPU-bound work while another waits on I/O.
    - **Engineer’s toolkit:** avoid hot-path syscalls, use async I/O, inspect with `strace`/`top`, enforce limits.
    - **HFT playbook:** pin, isolate, bypass kernel, be NUMA-aware, forbid blocking, monitor jitter.


