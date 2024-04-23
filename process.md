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
