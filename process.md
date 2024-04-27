## Limited direct execution

Time sharing: cpu executes a task for a while and then the other

Direct execution: process of how process is created by CPU and executes code.

User mode
- Prevent code from issuing IO

Kernel
- Code can run any privileged instruction

System call
- Empowered by hardware
- Allows user code makes privileged instruction
    - Like IO request, create a new process, share data…etc

Trap
- Change from user mode to kernel mode for a process
- Privileged instruction can be issued

Return from Trap
- Change from kernel mode to user mode

Trap table
- Avoid user code to issue a privileged instruction directly
- It’s created in kernel mode when OS is booted up.
- Make hardware remember which trap handler executes when event occurs

Two phase in limited direct execution
- At boot time
- When running a process

Kernel stack
- Per process
- Store program counter in stack when process moves from user code to kernel
- Counter is piped from stack when return from trap

System call number
- User code stores system call number in register
- Trap handler uses that number to issue system call
- Indirection enables protection

Switch between process
- Issue: when process is running on CPU, OS is not. How can OS let another process to run?

Cooperative approach: let process itself transfer control of CPU
- Some system calls use yield
    - Yield is called in system call and will let process transfer control of CPU to OS
- Illegal access leads to trap
- Con
    - Process may refuse to make mistake or give back control

Timer interrupt
- Timer interrupts a process every few ms; OS runs interrupt handler and regains control of CPU.
    - Timer is started when OS boot up.
    - Interrupt handler is also remembered by OS during boot up.
- Hardware saves state of program in the kernel stack.

Scheduler
- After timer interrupts or process traps to OS, OS regains control of CPU. A scheduler would decide whether to switch to another process.

Context switch
- When scheduler decides to switch, context switch stores register of current process and resume registers of another by posing another kernel stack. When process returns from trap, another process can resume.

## CPU Scheduling

We start with simple case with following workload assumption:
1. Each job runs for the same amount of time.
2. All jobs arrive at the same time.
3. Once started, each job runs to completion.
4. All jobs only use the CPU (i.e., they perform no I/O)
5. The run-time of each job is known.

Need scheduling metric to know the success of policy
- turnaround time
    - $T_{turnaround} = T_{completion} - T_{arrival}$
    - It's a performance metric
    - vs Fairness
        - To achieve best performance, we sometimes need to not run a few jobs and hence decrease fairness.
- response time
    - interative performance
    - $T_{response} = T_{first_run} = T_{arrival}$

Policies
- FIFO
    - Start the jobs in the order of arrival.
    - If jobs can be finished in different duration, we have a problem
        - Turnaround time is worse when heavy jobs run before light job.
- SJF(shortest job first)
    - It's a optimal algorithm for jobs with different workload.
    - It's a nonpreemptive scheduler.
    - If jobs can arrive at different time, the issue
        - The heavy jobs arrive earlier than light jobs and hence large turnaround time.
- STCF(shortest time to complete first)
    - Preemptive scheduler
    - When new jobs arrives, we pick job the can shortest time left
    - Con
        - Bad response time because jobs that run latter has to wait for previous job.
- Round Robin
    - Time slice
        - Multiple of timer interrupt
        - A duration in which CPU dedicates to a job.
    - Pro
        - Better response time because jobs take turn for every time slice.
    - Con
        - Bad turnaround time due to
            1. the cost of context switch
                - Make sure that the time slice is long enough to amortize the cost of switching.
            2. Jobs are stretched into multiple time slice rather than get it done as soon as possible.

Incorporate IO
- Treat each CPU burst as a job. This allows interative process performs IO while CPU intensive jobs run.
    - Next subjobs is submitted only when IO is completed

The common tradeoff
- To have fairness, turnaround time is bad.
- To achieve performance, resopnse time is bad

What is the cost of context switch?
- State of process has to be flushed and restored in CPU cache and TLB...etc.

## Scheduling: Proportional Share

proportional Share
- Each process has a certain percentage of CPU

ticket & lotter
- ticket currency
    - Each user has multiple jobs and assigns ticket number to those jobs. But for os to schedule them, those ticket number should be changed from a local to global currency to make a fair comparison.
- ticket transfer
    - A process temporarily hands over tickets to other process
        - It's useful when client process has to wait for result from server process.
- ticket inflation
    - When processes trust each other, they can raise/lower the ticket based on their need for CPU resource
- implementation
    - process list
        - Each proces holds a number of ticket
        - Sort by number of ticket in descending order
    - Random generator draws the ticket
    - Add process ticket to counter

lottery fairness
- F: time to complete the first job/ time to complete the second job
- best fairness F=1, where each process takes longer time slices

Stride Scheduling
- Deterministic fair share scheduler
- $stride = constant/number of ticket$
- OS pick the process with smallest pass from the queue, switch to that process, and then update the pass value with stride value.
```
curr = remove_min(queue); // pick client with min pass
schedule(curr); // run for quantum
curr->pass += curr->stride; // update pass using stride
insert(queue, curr); // return curr to queue
```
- vs lottery
    - Pro: Stride scheduling gets the proportion right at the end of scheduling cycle
    - Con: New process monopolize resource when it participates in scheduling half way.

completely fair scheduler
- vruntime
    - how long has each process executed
    - accumulated physical time when process runs
- sched_latency
    - define target latency for all processes
        - e.g. for 5 processes, divide sched_latency by 5, then each process has time share of $sched_latency/5$
- min_granularity
    - a threshold for min time slice a process can have
    - why? When number of concurrent process is large, each process has really small time slice. This leads to lots of context switch in given latency. So we need to set a threshold for it.

Weighting(niceness)
- nice level is from -20 to 19 and default is 0
    - if the process is too nice, it has low priority
- $time\_slice_k = {weight_k \over \sum_{i=0}^{n-1}w^i} * sched\_latency$
- to take priority into account in vruntime
    - $vruntime_i = vruntime_i + { w_0 \over w_i * runtime}$


Using Red Black Tree
- use red black tree to keep track of the runtime of process; those process that waits for IO are put to sleep
- Tree are kept balanced to ensure logn search and insertion. This is faster than linear search in a list.

Deal with IO and sleeping process
- when a process waits for IO, it's put to sleep. After sleeping process wakes up, other processes already make progress(a lot of runtime). Thus, process which wakes up will monopolize CPU because of smallest runtime.
    - sol: update the runtime with smaless run time found in the red black tree.

## Multiprocessor Scheduling

Multicore processor
- Multiple CPU are packed into a chip
- Application should run in parallel with threads to leverage multiple CPU.

Hardware cache
- When data is first loaded into memory, they are copied to cache as well
- e.g. CPU cache

Locality
- How os populate data in cache
    - Temporal locality
        - If you use it now, you need it later
            - e.g. function call in a loop
    - Spatial locality
        - If you use data from x, you need data from neighbor of x as weel
            - e.g. accessing an array through a loop

Cache coherence
- how much is actual data deviated in CPU caches
- why?
    - When multiple processes are running on different CPU and accessing the same memory address, CPU cache may have stale data because some processes may update the data in memory in runtime.
- bus snooping
    - A bus connects CPU cache to memory. When value of memory is updated, CPU caches are noticed and either invalidate or update the data in its own cache.


Even with coherence protocol programs need multual exclusion primitive to update data
- consider below example:
    - Two threads both read the head node at the same time
    - We can add lock to resolve it at the cost of decreasing performance
```
typedef struct __Node_t {
    int value;
    struct __Node_t *next;
} Node_t;

int List_Pop() {
    Node_t *tmp = head; // remember old head
    int value = head->value; // ... and its value
    head = head->next; // advance to next
    free(tmp); // free old head
    return value; // return value @head
1}
```

Why should scheduler consider cache affinity?
- States are remembered in cache so a process can be resumed faster. Scheduler should consider running a process on the CPU that the process used to.

Simple queue multi processor scheduling
- con
    - scalability
        - multiple CPU contends for the same lock
    - cache affinity
        - Once job is put to queue, it can be resumed by a different CPU and hence cpu cache has to be populated again.
- sol
    - Some processes have affinity to certain CPU while others migrate from CPU to CPU to balance the load.
        - con: hard to implement

multiple queue multi processor scheduling
- Each CPU has a queue. The queue is not shared between CPU. Thus we avoid synchronization problem and have better cache affinity.
- con:
    - A cpu may be lucky to have easiest jobs. Thus, cpu finishes them quickly and have the remaining jobs monopolize the cpu. This is unfair
- sol
    - Transfer the job to the CPU that has fewer job. Thus, load is balanced and fairness is achieved.
    - a. work stealing
        - con: large overhead due to constantly peeking
    - b. keep switching jobs
