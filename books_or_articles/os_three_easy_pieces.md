# Operating Systems: Three Easy Pieces

## An Introduction to Operating Systems for Software Engineers
- Every line of code you write is a request, a demand made to the most powerful program on your computer:
  - the Operating System.
- You ask it to manage memory, schedule tasks, and store data, but do you understand how it says 'yes'?
  - This guide is for the software engineer who is no longer content with asking.
- It's for the engineer who wants to understand, control, and ultimately master the foundational layer upon which all modern software is built.
- An Operating System (OS) is the software that sits between your programs and the physical hardware.
  - Its two primary jobs are to act as a resource manager and to provide a virtual machine.
  - As a resource manager, it manages the computer's core components—the CPU, memory, and disk—ensuring they are used efficiently and fairly by all running applications.
  - As a virtual machine, it creates a simplified and powerful illusion that makes the complex hardware much easier for programs to use.
- The OS creates a grand illusion for two of the most critical resources:
  - CPU
    - It takes one physical CPU and makes it appear as if there are many.
    - This allows numerous programs to seem like they are all running at the same time.  
  - memory
    - It takes one physical memory and gives each program its own private version.
    - This makes each program think it has the entire memory to itself, safe from interference by others.
- These illusions are central to how modern computers work, and they are made possible by three core OS concepts: virtualization, concurrency, and persistence.

### 3 Easy Pieces

#### The First Piece: Virtualization - The Art of Illusion
- Virtualization is the operating system's trick of taking a single physical resource, like a CPU, and creating many virtual, easy-to-use versions of it.
- The OS transforms a physical resource into a more general and powerful virtual form, which is why the OS itself is sometimes called a virtual machine.
```
[ Physical CPU ]  --OS-->  [ Virtual CPU 1 ] [ Virtual CPU 2 ] [ Virtual CPU 3 ]
```
- **ELI5:** The OS is like a magician who makes it look like every program gets its own personal computer, even though they're all sharing one.

#### The Second Piece: Concurrency - The Challenge of Cooperation
- Concurrency addresses the complex problems that arise when many things try to happen at the same time.
- For example, if two threads try to update a shared counter simultaneously, the final value can be incorrect because the individual steps of reading, updating, and writing the value get interleaved in unexpected ways.
```
Thread A -> [ Shared Data ] <- Thread B   (Potential for chaos!)
```
- **ELI5:** The OS is like a traffic cop for programs, making sure they don't crash into each other when using the same resource.

#### The Third Piece: Persistence - The Memory That Lasts
- Persistence is the principle of storing data so that it survives even when the computer's power is turned off.
- The OS achieves this through its file system, which manages how data is stored on devices like hard disks.
- A program uses system calls like `open()`, `write()`, and `close()` to ask the OS to save its data permanently.
```
Program -> OS -> [ Disk ]   (Data is safe even if power goes out)
```
- **ELI5:** The OS saves your work to a digital filing cabinet (the disk) so you don't lose it when you turn off your computer.
- These three pillars—Virtualization, Concurrency, and Persistence—are not abstract ideals; they are achieved through specific components like processes, threads, and files.

### Key OS Concepts

#### Operating System (OS)
- Technical: The operating system (OS) is the software in charge of making sure the system operates correctly and efficiently in an easy-to-use manner, primarily by virtualizing resources and managing them.
- Layman: The OS is the main software that runs on a computer.
  - It manages all the hardware parts and provides common services for computer programs.
  - It's the foundation that all your other apps run on.
- ELI5: The OS is the boss of the computer that tells all the parts what to do.
- Example/Analogy: Windows, macOS, and Linux are all operating systems.
  - Think of the OS as the government of a city; it manages infrastructure (hardware) and provides services (APIs) to its citizens (programs).

#### Process
- Technical: A process is the abstraction provided by the OS of a running program.
  - It includes the program's machine state: its memory, the contents of its registers, and information about I/O resources like open files.
- Layman: A process is a program that is currently running.
  - When you double-click an application, the OS creates a process to execute it.
  - Each browser tab you have open might be its own process.
- ELI5: A process is a program that's doing its job right now.
- Example/Analogy: When you run a compiled program from your terminal, the OS creates a process to execute its instructions.
  - A process is like a chef actively cooking a dish in the kitchen; they have their own recipe (code), ingredients (data), and workspace (memory).

#### Address Space
- Technical: An address space is the abstraction for physical memory provided to a process.
  - It presents memory as a private, linear array of bytes, typically containing the program's code, a stack, and a heap (OSTEP, Section 13.3).
- Layman: This is the private memory view that the OS gives to each process.
  - The process thinks it has a huge, clean slate of memory all to itself, even though it's sharing the computer's physical memory with many other processes.
- ELI5: It's a program's own private playground of memory.
- Example/Analogy: Two processes can both access memory address 0x200000, but they will see different data because the OS maps their virtual addresses to different physical locations in the computer's RAM.

#### Thread
- Technical: A thread is a single stream of execution within a process.
  - Multiple threads can exist within the same process, sharing the same address space but having their own private set of registers and stack.
- Layman: Threads are like mini-programs running inside a main program.
  - They let a single application do multiple things at once, like a web browser loading a page (one thread) while you can still scroll (another thread).
- ELI5: Threads are different workers all working on the same big project.
- Example/Analogy: In the shared counter example, two threads are created within the same process.
  - They share the counter variable but race to update it, causing errors without proper coordination.

#### System Call
- Technical: A system call is the mechanism through which a user program requests a service from the OS kernel.
  - It involves a special trap instruction that transfers control to the OS and raises the hardware privilege level to kernel mode.
- Layman: This is how a regular program asks the OS to do something special that it's not allowed to do on its own, like reading a file from the disk or creating a new process.
  - It's the official way to talk to the OS kernel.
- ELI5: A system call is a program asking the OS for permission to do something important.
- Example/Analogy: When your program calls `open("/tmp/file", ...)` or “write(fd, ...)`, it is making a system call. It's like a citizen making a request to a government agency (the OS) to access a public record (a file).

#### File
- Technical: A file is a linear array of bytes that is managed by the file system, a part of the OS.
  - Each file has a low-level name (an inode number) and can have one or more human-readable names.
- Layman: A file is a container on your disk for storing information, like a document, a picture, or a program.
  - The OS provides a way to create, read, write, and delete them.
- ELI5: A file is a named box where you keep your digital stuff.
- Example/Analogy: A program can create a file named /tmp/file and write "hello world" into it using the open(), write(), and close() system calls.

#### Protection
- Technical: Protection is a key principle of OS design, centered on isolating processes from one another and from the OS itself.
  - This ensures that the behavior of one application, whether malicious or buggy, does not harm others or the system as a whole.
- Layman: Protection means the OS builds walls between different programs.
  - A crash in your web browser shouldn't crash your word processor or the entire computer.
  - Each program runs in its own secure sandbox.
- ELI5: The OS makes sure programs can't mess with each other.
- Example/Analogy: Memory protection ensures that a process trying to access memory outside of its own address space will be stopped by the OS. This prevents one buggy program from corrupting the data of another running program.

#### API vs. ABI
- Technical: An API (Application Programming Interface) defines the set of calls and conventions a programmer uses to interact with a library or OS.
  - An ABI (Application Binary Interface) specifies the low-level details for compiled code, such as calling conventions and system call numbers, ensuring binary compatibility.
- Layman: An API is like a menu at a restaurant; it tells you what you can order (what functions you can call). The ABI is like the kitchen's internal rules for how the cooks should prepare the order (how the compiled code actually makes the function call happen).
- ELI5: The API is the list of rules for programmers; the ABI is the list of rules for the computer.
- Example/Analogy: The POSIX standard defines an API for operating systems. You can write C code that calls fork() on both Linux and macOS because they follow this API. The code compiles and runs because the compiler and OS on each system agree on the ABI.

#### Kernel Mode vs. User Mode
- Technical: User mode is a restricted processor mode in which applications run; certain privileged instructions, like I/O requests, are forbidden.
  - Kernel mode is a privileged mode where the OS has full access to all hardware capabilities.
  - A trap instruction switches from user to kernel mode.
- Layman: Your programs run in "user mode," which is a safe, limited mode. The OS runs in "kernel mode," which is the all-powerful "god mode." Programs must ask the OS (via system calls) to switch to kernel mode to perform sensitive tasks.
- ELI5: User mode is for playing safely, kernel mode is for doing serious work.
- Example/Analogy: When your application needs to read a file, it traps into the kernel.
  - The hardware switches to kernel mode, the OS performs the privileged disk I/O, and then it returns to your program, switching back to user mode.

### How It All Works Together (A High-Level View)

#### The Lifecycle of a Process
- A process isn't always actively running on the CPU.
  - Think of processes as chefs in a kitchen.
  - A Running chef is actively chopping vegetables at the single cutting board (the CPU).
  - A Ready chef has all their ingredients and is waiting for the cutting board to be free.
  - A Blocked chef is waiting for an oven to preheat (an I/O operation) and can't do anything until it beeps.
- The OS keeps track of what each process is doing by assigning it one of these states:
  - Running: The process is currently executing instructions on a CPU.
  - Ready: The process is ready to run, but the OS has chosen another process to run for now.
    - It's waiting in a queue for its turn on the CPU.
  - Blocked: The process has performed an operation that makes it unable to proceed, such as requesting to read data from a disk.
    - It is waiting for an event (like the I/O completing) before it can become ready again.
```
Scheduled   +----------+
-------->   | Running  |
<--------   +----------+
Descheduled      |
                 | I/O request
                 v
+----------+   +---------+
|  Ready   | <-| Blocked |
+----------+   +---------+
                 I/O done
```

#### System Calls: Talking to the OS
- A system call is the primary way a user program asks the operating system to perform a privileged task on its behalf.
  - When a program makes a system call, the hardware traps into the kernel, switching from user mode to kernel mode, executes the request, and then returns control to the program.
  - Common system calls on UNIX-like systems include:
    - `open()`: Open or create a file.
    - `read()/write()`: Read data from or write data to a file.
    - `fork()`: Create a new process that is a copy of the current one.
    - `exec()`: Transform the current process into a new one by loading a new program.
    - `wait()`: Wait for a child process to terminate.
```
User Program        |    Kernel (OS)
--------------------|-----------------
1. Calls read()     |
2. Executes TRAP -> | 3. Kernel executes read()
                    | 4. Returns from TRAP
5. Continues      <-|
```

#### Scheduling: Deciding Who Runs
- With many processes in the "Ready" state, the OS needs a way to decide which one to run next.
  - This job belongs to the scheduler.
  - The scheduler maintains a ready queue of all processes waiting for CPU time.
  - A simple and fair scheduling policy is Round Robin (also called time-slicing), where each process gets to run for a short time slice before the OS switches to the next process in the queue.
  - This ensures that all processes make progress and the system feels responsive.
```
CPU: [ Process B ]

Ready Queue: [ Process C ] - [ Process A ] - [ Process D ]
```

#### Memory: The Address Space
- Each process has its own virtual address space, which is its private map of memory.
  - This illusion of private memory protects processes from one another and makes programming simpler.
  - A typical process address space has a standard layout:
    - `Code`: The executable instructions of the program, usually located at the bottom of the address space.
    - `Static Data`: Global variables and other static data.
    - `Heap`: Dynamically allocated memory, such as memory requested via malloc(). The heap typically grows upwards from the static data region.
    - `Stack`: Used for local variables, function parameters, and return addresses. The stack typically grows downwards from the top of the address space.
```
+-----------------+  <-- Top of Memory
      Stack      |
        |        |
        v        |
                 |
      (free)     |
                 |
        ^        |
        |        |
       Heap      |
+-----------------+
   Static Data   |
+-----------------+
      Code       |
+-----------------+  <-- Bottom of Memory (0x0)
```

### Why Software Engineers Should Care
- Understanding operating systems is not just an academic requirement; it's a practical necessity for writing high-quality software.
- The abstractions the OS provides are not magic—they come with performance costs and behavioral rules.
- Knowing these rules allows you to write code that is faster, safer, and more reliable.
- This section connects core OS concepts to the real-world consequences you'll face as a programmer.

#### Performance
- Every action the OS takes on behalf of your program has a cost.
- This overhead is often invisible, but it can significantly impact your application's performance, especially in time-sensitive code.
- **Context Switches**: Every time the OS switches from one process to another, it must save the state of the old process and load the state of the new one.
  - This operation is not free.
  - This is the tangible cost of the OS's virtualization magic; the illusion of infinite CPUs is paid for with the overhead of each switch.
  - Code that causes frequent context switches (e.g., by creating too many threads that are all competing for a lock) will suffer.
- **System Call Overhead**: Making a system call is significantly more expensive than a regular function call.
  - Each system call requires a trap into the kernel, a context switch from user to kernel mode, execution of the OS code, and another context switch back.
  - Placing system calls inside a tight, performance-critical loop can be a major performance bottleneck.
- **Cache Effects**: Modern CPUs rely on caches for fast memory access.
  - When a context switch occurs, the new process will likely find that the data it needs is not in the CPU cache.
  - It must then fetch data from the much slower main memory, a phenomenon known as a "cold cache," which hurts initial performance.
- **Blocking vs. Non-Blocking I/O**: A "blocking" system call, such as a standard read() from a network socket, will cause your entire process to freeze until the operation completes.
  - In applications that need to remain responsive (like a UI or a server handling multiple clients), using non-blocking I/O techniques is essential to allow the program to do other work while waiting.

#### Safety
- The OS provides fundamental safety guarantees that make programming far simpler and more reliable.
  - Without these, every application would have to worry about defending itself from every other program on the system.
- **Memory Protection**: The OS, with help from the hardware, ensures that each process can only access its own address space.
  - This means a bug in one program (like a wild pointer) cannot corrupt the memory of another program or the OS itself.
  - This isolation is a cornerstone of modern system stability.
- **Process Isolation**: The principle of protection extends beyond just memory.
  - The OS isolates processes from each other so that a crash in one application is contained.
  - A single buggy program will not bring down the entire system, a crucial feature for reliability.

#### Portability
- The OS provides a stable and standard set of Application Programming Interfaces (APIs), such as the POSIX standard on UNIX-like systems.
  - By writing your code to use these standard APIs, you ensure it is portable.
  - Your C program that opens, reads, and writes files can be compiled and run on different hardware architectures and on different OS variants (like Linux or macOS) with few or no changes, because the OS abstraction layer hides the hardware-specific details.

#### When It Goes Wrong
- Ignoring OS principles can lead to common and costly programming mistakes.
  - Here are a few examples:
    - Spawning an unbounded number of processes or threads in response to requests can quickly exhaust system resources like memory or process table slots, causing the system to become unresponsive.
    - Making a blocking system call on a critical, performance-sensitive thread (e.g., in the main event loop of a game or a high-frequency server) will cause the application to "freeze" or "hitch" unexpectedly.
    - Ignoring the return values from system calls like `open()` or `write()` can lead to silent failures.
      - Your program might think it successfully wrote data to a file, but if the disk was full, the operation failed and the data was lost.

### Mapping to High-Frequency Trading (HFT) / Low-Latency
- In fields like high-frequency trading, the operating system is not a helpful abstraction.
  - It is a hostile, unpredictable environment that must be tamed.
  - The goal is not just low latency, but deterministic latency.
  - Jitter—the variation in that latency—is the primary enemy.
  - In this world, the "invisible" overhead of the OS becomes a primary obstacle that must be actively managed and defeated.

#### How the OS Influences Latency and Jitter
- Every standard OS feature designed for fairness and general-purpose use can introduce unacceptable delays and unpredictability in an HFT environment.
  - **Context Switches**: A context switch is a massive, high-latency event.
    - When a critical trading application is switched out, it is paused for milliseconds—an eternity in HFT.
    - This is a primary source of unpredictable latency.
  - **Scheduler Policies**: The standard OS scheduler aims for fairness, not for the single-minded performance of one application.
    - The scheduler's decision to run a different process, even for a moment, is a major source of jitter for a low-latency application.
  - **Timer Ticks & Interrupts**: The OS relies on a periodic timer interrupt to regain control of the CPU.
    - Each of these "ticks," along with other hardware interrupts (e.g., from a network card), steals CPU cycles from the application at unpredictable moments, causing jitter.
  - **Page Faults**: Accessing a piece of memory that isn't currently in RAM triggers a page fault, requiring the OS to load it from disk.
    - This is a high-latency event that is completely unacceptable during trading operations.
  - **System Calls**: As discussed, the overhead of trapping into the kernel for a system call is significant.
    - In a tight latency budget, every single system call on the critical path is a liability.

#### A Practical HFT/Low-Latency Checklist
- For a developer beginning to explore low-latency programming, here are some fundamental OS-level techniques to control performance.
- Pin Hot Threads (CPU Affinity):
  - Why it matters: This prevents the OS scheduler from moving your critical thread to a different CPU core, which would destroy CPU cache locality and cause a performance hit.
  - How to test: On Linux, use the taskset command to bind a process to a specific CPU and measure the change in its performance or jitter.
- Reduce Context Switches:
  - Why it matters: This minimizes the time your process spends paused while the OS scheduler runs other tasks on your designated core.
  - How to test: On Linux, use pidstat -w [pid] to monitor the number of voluntary and involuntary context switches your process is experiencing.
- Warm Memory (Avoid Page Faults):
  - Why it matters: The goal is to pre-fault all critical memory paths at application startup so that no page faults occur during the trading day's critical execution path. This front-loads high-latency events to a non-critical time.
  - How to test: On Linux, run your application under perf stat -e page-faults ./my_app and observe the number of faults on the first run compared to subsequent runs.
- Minimize System Calls in Hot Loops:
  - Why it matters: Every system call is an expensive trip into the kernel that burns microseconds from your latency budget; they must be avoided on the critical execution path.
  - How to test: On Linux, use strace -c -p [pid]. On macOS, use dtruss -c -p [pid]. These tools attach to your running process and give a summary of system call counts and frequencies.
- Use Polling Instead of Interrupts:
  - Why it matters: For network I/O, waiting for a hardware interrupt from the network card introduces unpredictable latency; polling the card in a tight loop gives your program full control over when it checks for data.
  - How to test: This is an advanced programming pattern. To observe its importance, use mpstat -P ALL 1 on Linux during a network-heavy workload and note how kernel-time (%sys) on the core handling network interrupts can spike unpredictably, stealing cycles from your application.
- Watch NUMA Locality:
  - Why it matters: On machines with multiple CPU sockets, accessing memory attached to a different CPU is significantly slower than accessing local memory.
  - How to test: On Linux, use numactl --hardware to view the system's NUMA topology and use numactl to control process placement and measure performance differences.

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


