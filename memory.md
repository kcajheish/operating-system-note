# Memory


## Address space
In early day, memory of os is divided:
- os related, starting at 0k
- your program, starting at 64k

When switching among processes, it's not efficient to store memory of a process in the disk. It's better to leave them in memory.

Due to time sharing, cpu switch process. We like to avoid memory of one process being written by the other.

Address spce is an abstraction over physical memory.

There are three components in address space:
- code
    - code that is executed
    - start at 0kb
- heap
    - keep track of memory that user allocates
    - start at 1kb
    - grow downward when user calls malloc()
- stack
    - keep track of function calls, local variable,parameters, and returned value
    - start at bottom, 16kb
    - grow upward when user calls a procedure

In reality, physical memory is scattered and does not always start at zero.

Hence, OS virtualize memory. It provides an address space for multiple running process on a single physical memory.

To achieve this, we need:
- transparency
    - Each process use address space as if it has its own private memory. It shouldn't know how os multiplex memory to multiple processes.
- efficiency
    - Don't use extra space for virtualization
    - Implementation shoudn't slow down process
- protection(isolation)
    - Each process can't modify memory of other process or os.

## address translation

To virtualize CPU, limited direct execution let process run on hardware.

Address translation use hardware to redirect app memory reference to actual location of memory.

OS duties:
- controls how memory is used
- keeps track of which location is used
- manage memory


below is an example of assembler
- ebx is the address of x
- eax is a register.
```
128: movl 0x0(%ebx), %eax ;load 0+ebx into eax
132: addl $0x03, %eax ;add 3 to eax register
135: movl %eax, 0x0(%ebx) ;store eax back to mem
```

Address doesn't always start at zero. It may start with arbitrary place in physical memory.

base and bounds
- Needs base and bound register define where code, stack and heap should be loaded in physical memory
    - physical address = base + virtual address
- Bound(size of address space) is used to prevent process from accessing memory that belongs to other process
- Since memory is calculated at run time. It's also called dynamic relocation

Each CPU has a pair of base and bound register.

MMU(memory management unit) translates memory address from virtual to physical.

OS can check the free list for available physical memory.

When process is ended, OS cleanned up physical memory of that process and add those free space to free list.

If memory is out of bound, an exception is raised by os.

OS  can switch between kernel and user mode. CPU uses a single bit(process status word) to indicate which mode the process is in.

CPU MMU stores additional base and bound register so process virtual can be converted to physical one.

Base and bound registers can be modified in kernel mode.

Exception handler is called when process tries to access out of bound address and cpu raises exception.

When context is switched, base and bound registers are stored in PCB(process control block) in memory.

When process is descheduled by cpu, its base and bound registers can be moved.

