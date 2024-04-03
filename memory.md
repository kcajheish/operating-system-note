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
