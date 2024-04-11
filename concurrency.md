## Concurrency

Thread
- Multi thread program has multiple PC(program counter)
- Threads of a process share the same address space
- Context is switched when CPU stops thread A from running and start thread B. Use thread control block(TCB) to save/restore state.
- In a multi-threaded process, each thread has its own stack(namely, **thread local storage**) in address space.

Why thread
- Parallelism
    - Turn a single threaded program into multiple thread when multiple CPUs are available.
    - Tasks are processed in shorter time
- Avoid blocking due to slow IO
    - Thread enables overlap of IO with other activities within program

Thread vs Process
- Multi-threaded program and multiprogramming both speed thing up.
- Thread is useful when you like to share data among them.
- Process is preferred when tasks are independent of each other.

Why concurrency is a bad thing
- When two threads try to update same shared variable, the result is not deterministic

Uncontrolled scheduling
- e.g. Before thread A updates value in memory, timer interrupts the thread A, save the state of thread A in TCB, and then restore thread B from TCB.
- Race condition(data race)
    - Result depends on timing of interrupt
    - Nondeterministic
- Critical section
    - It’s code that accesses shared variable and thus must not be executed concurrently
- Mutual exclusion
    - To ensure one thread can enter critical section

Tool you should know
- gdb, a debugger
- valgrind/purify, memory profiler
- objdump, turn executable into assembler

Atomic
- All actions succeed or none of them works. There is no in-between state
- Two scenarios require atomicity
    - Multiple threads update a shared variable
    - One thread performs IO task to load data from disk to memory; another waits until IO thread completes its task

Transaction
- Group a sequence of actions to ensure atomic result

How does program ask hardware for atomicity in critical section?
- Synchronization primitives


