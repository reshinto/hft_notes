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
    - Core problem OS solves: Make one CPU feel like many things run at once.
    - Why it seems impossible: Music, browser tabs, chat apps all "together."
    - Key insight: The OS uses the abstraction of a process.
      - The Process
        - A running program
        - The OS creates this abstraction to manage everything a program is doing
        - Analogy: A process is a cook following a recipe.
          - The OS is the head chef telling each cook when to work.
    - A program is passive: a file of instructions on disk (like a recipe book on a shelf).
    - A process is active: the program running with state and resources (like a chef cooking from the book).
    - Multiple processes from one program: Two chefs can cook the same recipe; two independent browser windows are two processes.
    - Sandbox / private universe: Each process gets its own “virtual CPU” and private space so they don’t step on each other.
    - Two superpowers of this abstraction:
    - Resource sharing: Fair turns with the single CPU.
    - Isolation: One app crashing doesn’t take down the whole machine.
    - Kitchen analogy: One cutting board (CPU), many cooks (processes), one head chef (OS) fairly coordinates turns so work gets done without chaos.
    - Set-up for next topics: Fair sharing and scheduling are the next pieces.
    - |Program|Process|
      |-------|-------|
      |A passive entity|An active entity|
      |A file on disk|A program in execution|
      |Lifeless instructions|Has state: memory, etc.|
2. A process’s Life
    - Life cycle is structured, not random. Five main states:
      - `New`: OS is setting the process up.
      - `Ready`: Prepared to run, waiting its turn (sprinter in the blocks).
      - `Running`: Currently using the CPU.
      - `Waiting`: Blocked on slow operations (e.g., disk I/O), so it doesn’t hog the CPU.
      - `Terminated`: Finished; OS cleans up its resources.
    - State “dance” example (email app):
      - Writing an email, the email app is in the `Running` state.
      - Attach a large file (slow disk read)
        - the OS is smart by moving the email to the `Waiting` state.
      - While waiting, another process (e.g., music player) runs.
      - Once data is ready, email app returns to `Ready` state, then gets scheduled to `Running` state again.
      - Result: Continuous, fast transitions make the computer feel responsive.
3. The OS’s Rolodex
    - How the OS keeps track: A Process Control Block (PCB) per process.
      - Think of it as a high-fidelity bookmark:
        - if you're reading a book and someone interrupts you, you put a bookmark in.
      - Saves the exact instruction position (program counter).
      - Captures CPU registers (“scratch pad notes”).
      - Stores unique ID and memory information (and other essentials).
    - Context switch (the sleight of hand):
      - OS saves current process state into its PCB.
      - OS loads the next process’s PCB and resumes it exactly where it left off.
      - This is the `trick` behind multitasking.
    - Speed and cost:
      - Context switches can be a few microseconds.
      - A blink is ~100,000 microseconds; thousands of switches can occur in one blink.
      - Not free: There’s overhead; small but real, and sometimes critical.
4. Juggling for fairness
    - The scheduler, a component of the OS, decides who runs next
    - Can think of the OS is a super strict but fair parent
    - Main technique: Time sharing with fixed time slices (quanta).
    - Example: “Process A, you get 10 ms, go.”
      - When Time’s up → context switch → “Process B, you’re up.”
    - Fast repetition creates the illusion of parallelism on one CPU.
    - Classic trade-off (responsiveness vs throughput):
      - Short time slices: System feels very responsive; more context-switch overhead.
      - Long time slices: Higher overall work done (throughput); system may feel sluggish.
      - OS design balancing act: Pick time slices to feel fast and accomplish a lot over time.
5. From Code to career
    - Why developers should care:
      - “It works on my machine” often comes from differences in process state:
        - Different memory contents, environment variables, open files, etc.
      - Thinking of a process as a self-contained universe clarifies debugging.
    - Debugging with process states:
      - App “hangs” — is it Waiting (I/O or network blocked)?
      - Or Ready but starved for CPU time?
      - This mindset turns you into a performance detective rather than guessing.
    - Helpful tools (examples):
      - Activity Monitor on macOS.
      - strace on Linux.
      - These let you “look inside” a process’s world to see what it’s doing.
6. The nanosecond game
    |Source of Delay|Typical Cost|
    |---------------|------------|
    |User Computation|Varies|
    |Kernel System Call|`~0.5µs`|
    |Context Switch|`~5µs`|
    |Disk I/O Wait|`>1000µs`|
    - Context where the stakes are extreme: High-Frequency Trading (HFT).
    - Nanoseconds matter.
    - A ~5-microsecond context switch is not “tiny” here; it’s unacceptable jitter.
    - Unpredictable delays can cost millions.
    - How they respond:
      - Minimize Syscalls: avoid kernel calls on critical paths
        - Avoid OS help: Write code that rarely needs the OS, because system calls and switches add delay.
      - Avoid Switches: use CPU pinning to prevent interruptions
        - CPU pinning: Bind a critical process to a specific CPU core; tell the OS not to move it.
      - Isolate Processes: run risk-management code separately.
      - Manage Resources: pre-configure file and memory limits.
