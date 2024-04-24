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
