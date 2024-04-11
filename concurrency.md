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

## Event based concurrency

What is event loop
- Events are gathered and are executed in order using event handler.
    - By execute in order, application has the control over scheduling.
- Code
```
while (1) {
	events = getEvents();
	for (e in events)
		processEvent(e);
}
```

No lock is required
- A thread execute the handler that processes one event at a time. Because thread is not interrupted, concurrency bugs are avoided

Potential issue: main thread waits for IO to complete.
- Client reads file from disk
    - open/read system call are made
    - Only one thread is running in the server. So server just waits and doesn’t make any progress
- Note: it’s not the case for threaded concurrency because IO overlaps with other activities

Solution: Asynchronous IO
- Call of async IO returns right away and thread continue making progress
- In Mac
    - Use AIO control block for IO request and aio_read for issuing request
    - To know the status of IO request, call aio_error
    - If there are thousands of IO requests, calling aio_error for each request can be slow.
        - Use interrupt to avoid checking too much
            - When IO request is completed, UNIX sends a signal to application. Thread stop what it's doing and execute signal handler if that signal can be recognized.
```
struct aiocb {
	int aio_fildes; // File descriptor
	off_t aio_offset; // File offset
	volatile void *aio_buf; // Location of buffer
	size_t aio_nbytes; // Length of transfer
};

int aio_read(struct aiocb *aiocbp);

int aio_error(const struct aiocb *aiocbp);

void handle(int arg) {
    printf("stop wakin’ me up...\n");
}

int main(int argc, char *argv[]) {
    signal(SIGHUP, handle);
    while (1)
        ; // doin’ nothin’ except catchin’ some sigs
    return 0;
}
```

Con of event loop
- Still need locks when multi CPUs are used
- Can’t integrate well with paging
    - When page fault becomes prevalent, server performance drops
- Hard to maintain
    - Routine can be blocking/non-blocking. They need different styles of coding
- Async IO requires both select() and AIO for network IO and disk IO
