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

## Segmentation

To virtualize memory with base and bound register is wasteful. There are empty space between stack and heap.

Instead, we create logical segments for code, stack and heap. By specifying start and length of the segment, allocated space is not wasted

Segment is allocated when memory is used. Thus you see sparse **address space**
![alt text](image.png)

Segment consists of base and size which show physical memory that is allocated.
![alt text](image-1.png)

To access virtual address, 100, in code segment, add off set to the base and check it isn't out of bound
- e.g. 100 + 32K(base) = 32768(byte), assumes the start of virtual address of code is 0.

Segment violation/fault
- User process tries to access memory that is out of bound.

How to tell which segment we refer to?
- use top 2 bits of 14-bit virtual address
- e.g. 00 code, 01, heap, 10 stack

virtual address 4200
- Top two bits are 01. thus we use heap base and bound
- the next 12 bits are 104 and are offset.

![alt text](image-2.png)

code
- SEG_MASK = 0x3000
- 0x: hexadecimal format
- 3000: digits
    - convert to decimal
        - $3*16^3= 12288$

- SEG_SHIFT = 12
```
1 // get top 2 bits of 14-bit VA
2 Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT
3 // now get offset
4 Offset = VirtualAddress & OFFSET_MASK
5 if (Offset >= Bounds[Segment])
6   RaiseException(PROTECTION_FAULT)
7 else
8   PhysAddr = Base[Segment] + Offset
9   Register = AccessMemory(PhysAddr)
```

issue with top two bits approach
1. 3 out of 4 segments are used. Space is wasted
2. Each segment can't grow over a max size

Note that stack grows backward in physical memory. Thus, the offset is calculated differntly.
- e.g. hex 0x3C00
    - binary: 11 1100 0000 0000
        - 11: stack segment
        - 1100 0000 0000 -> 3072 byte = 3KB
        - b/c stack grows backword, we have to calculate negative offset
            - 3KB - 4 KB(size of a segment) = -1 KB
        - physical address = base + negative offset

To save more memory, share code segment between processes.
- By specifying protection bit, os determine whether a process can read/write/execute data in physical memory.
- The same physical memory can be mapped to many address space from different processes.

Coarse grain segment
- address space is chopped into large segments like stack, heap, code

Find grain segment
- more and smaller segments
- require segment table
- why?
    - to improve memory efficiency, we have to know break down of the memory segment.

issues with segmentation
1. In context switch, segment registers need to be saved and restored
2. When heap doesn't have free space, allocate free space for heap segment and update the segment size
3. Segment size varies. It's not easy to allocate new space without holes -> **external fragmentation**
![alt text](image-3.png)

Thus, **compaction** can copy data of process to continuous memory block.
- con:
    1. memory and computation extensive
    2. make segment hard to grow

Alternatively, use free-list management to track free space. Process can allocate memory from free list that fits their need
- e.g. best-fit, first-fit, buddy algorithm

## Free Space Management

malloc: given size, return pointer to the memory in heap with that size
- Note that once memory is allocated you can't relocate it. Thus compaction doesn't work.

free: given pointer, free memory in heap for that pointer.
- note that os has to keep track of size of the pointer somehow

Internal fragmentation: empty space in heap when allocated memory in heap is larger than usage.

Scenario
1. allocate memory larger than 10 bytes
2. allocate memory equal to 10 bytes
    - ![alt text](image-4.png)
    - ![alt text](image-5.png)
3. allocate memory smaller than 10 bytes, **split**
    - ![alt text](image-6.png)
4. (continue from 2)free 10 bytes and **coalesce** current and nearby free space into continuous free space
- ![alt text](image-7.png)
- ![alt text](image-8.png)

Each allocated memory has **header** along with it. It stores size of allocation which is used in free call.
```
typedef struct {
    int size;
    int magic;
} header_t;
```
![alt text](image-9.png)

given a pointer to the free space, calculate pointer to the header with pointer arithmetic
```
header_t *hptr = (header_t *) ptr - 1;
```

Use node as header to track list of free space
```
typedef struct __node_t {
    int size;
    struct __node_t *next;
} node_t;
```

call mmap to allocate free space in heap
- a single node for free space, $4096 - 8 = 4088 bytes$
- start at virtual address, 16KB
- ![alt text](image-10.png)

allocate 100 bytes
- this node is splitted into 108-byte and 3988-byte node
- pointer to 100-byte free space is returned
- header is moved to the start of free list
- ![alt text](image-11.png)

allocate two more 100 bytes space
- ![alt text](image-12.png)

free the middle allocated space
- move header to middle allocated node
- point next to the previous header
![alt text](image-13.png)

free the rest of allocated space
- move header around and build next pointer
- you can see internal fragmentation even though they form a continuous block.
![alt text](image-14.png)

When allocation library runs out of space in heap
1. make system call, sbrk, to grow heap size
2. return null as error

Every allocation policy has it strength and weakness. There is no best approach.
- best fit
    - pro: space is not wasted
    - con: cost of linear scan
- worst fit
    - pro: leave more large chunks in free list
    - con: cost of linear scan
- first fit
    - pro: fast scan in best case
    - con: scan is slow when free list has many small chunk at head
- next fit
    - pro: avoid splinter at the head
    - con: have to track two pointers

Segregated list is dedicated for popular size request. When free space in segregated list is running low, slab allocator ask for a slab of memory from general memory allocator.
- how much should you ask for slab depend on page size and number of objects allocated
- pro
    - can serve fix size request quickly
    - less fragmentation

Buddy allocation treats free space having size of $2^N$. Recursively split the space into half. When a buddy in one half is free and has enough space for the request, return pointer.
- pro: easy to coalesce since next buddy and current buddy differ only by one bit
![alt text](image-15.png)

other allocation technique that scales:
1. advance data structure
    - balanced binary tree
    - splay tree
    - partially ordered tree
2. runs on multiple processor

to see how real world allocator works: [Understanding glibc malloc](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/)

## Paging

Segmentation divides virtual memory into variable-sized chunk. It tracks memory usage better but introduce fragment.

Paging divides virtual memory into fixed-sized chunk and thus avoids fragment.

Page
- a fixed-sized unit in address space

Page frame
- a fixed-sized slot in physical memory that can be assigned a page

![alt text](image-16.png)*64 bytes address space in 128 bytes physical memory*
- Physical memory is divided into fixed-sized slots
- note that a page frame is reserved for OS
- Few page frames are not used
- The rest are assigned to pages in address space
    - Page are not arranged in order.

advantage of paging
- good abstraction
    - you don't have to worry about how stack/heap grow to implement your translation
- simple to grab free page from free list

page table
- store address translation per process
    - i.e. map virtual page number to page frame number

![alt text](image-17.png)*virtual address of 21*
- movl 21, %eax
    - copy value of address, 21, to general purpose register, %eax
- virtual address 21 refers to virtual page 1 with 5 bytes offset
    - binary of 21 = 010101
- Since each page is 16 byte, we need 4 bits for offset
- The size of address space is 64 byte. Since there are 4 pages intotal, we need top two bits for virtual page number
- generally, for address space with size N, we need $\log_2N$ bits for each address

![alt text](image-18.png)
virtual page number is translated to physical address before OS issues load to physical memory.

Page table
- map virtual address to physical address
- usually resides in virtual memory and sometimes swapped to disk

linear page table
- use array to store physical page frame
    - index is virtual page number

![alt text](image-19.png)*page table entry (PTE)*
- PTE resides in page table; size ~=4 bytes
- valid bit: mark unused space in address space as invalid
    - process traps to os when it tries to read invalid address
    - os won't allocate physical page frame for invalid address
        - to save memory
- protection bitst: whether page can be read/written/executed
- present bit: whether virtual page is in physical memory or disk
    - e.g. swap out rarely used page when address space outgrow physical memory
- dirty bit: whether page is modified since page is brought to memory
- reference bit: whether page is accessed
    - useful for page replacement

![alt text](image-20.png)*access memory with paging*
- memory address of page table is stored in register
- VPN_MASK: 110000
- SHIFT: 4
- for virtual address, 21, VPN = 1 and PTEAddr
- OFFSET_MASK: 1111
- PFN: physical frame number, which is extracted from PTE
- offset: number of bytes away from the base of page
- note that it is computation expensive to fetch PTE from memory

assembly code in x86
```
1024 movl $0x0,(%edi,%eax,4)
1028 incl %eax
1032 cmpl $0x03e8,%eax
1036 jne 1024
```
- copy 0 to (array base + 4 * array index)
    - each element in the array is integer with 4 bytes
- increase index by 1
- substract 1000 with index and set the process status flag accordingly
- jump to label(target address) if status is not equal

![alt text](image-21.png)*memory trace*
- size of address space = 64KB
- page size = 1KB
- code lives in the first page
- physical address of page table = 1KB
- array virtual address from 40000 to 44000
    - 1000 integer with 4-bytes in an array
    - virtual page from 39 to 42
- note that for each loop we have two fetches
    1. fetch PTE
    2. fetch element of an array

## Translation look aside buffer

Page table is stored in memory. If every instruction fetch and memory access needs to find page table, it’s too slow. To speed things up, we need support from hardware.

TLB(translation look aside buffer)
- It’s part of memory management unit
- Also called address translation cache
- OS looks up TLB before it look up page table in physical memory

TLB hit
- Virtual page number is stored in TLB and is found

TLB miss
- Virtual page number isn’t stored in TLB; thus OS looks up page table in physical memory for virtual address.
- After virtual address is found, physical address(physical frame number) is calculated.
- Then, TLB is updated with physical frame number and instruction is retried.
- When it happens, memory access becomes very slow because extra lookup in TLB is required.

WHO handles TLB miss
- Software (os)
    - Simple
        - Hardware raises exception when TLB miss. DO nothing else
        - OS take care of TLB miss handler of trap.
    - flexible
        - OS can uses any data structure to implement TLB without hardware changes

TLB content
- Fully associated
    - Hardware searched full TLB in parallel to find a match of virtual address.
- Format
    - VPN
    - PFN
    - Other bits
        - Valid
            - Translation is valid or not
        - Protection
            - Read/write/execute?
        - Dirty
        - Address space identifier

TLB issues
- Context switch happens when another process likes to run. Same virtual page number may differ in physical frame number for different process. Thus, TBL has to tell which process does this virtual page number belong to
- Sol
    - Flush
        - Turn valid bits to zero when hardware detects base register of a page table changes
            - con
                - Every context switch incurs TLB miss. The cost is high
    - Address space identifier
        - It’s used to identify which process is running
        - ASID of current process is stored in privilege register when context is switched
        - It resolved TLB miss since address translation of different process can be stored in TLB

Replacement policy
- When new entry is added to TLB, old one has to be replaced
- We choose policy to minimized TLB miss and to increase performance
- Policy
    - Random
    - LRU(least recently used)

Real TLB entry
- MIPS uses software managed TLB.
- MIPS supports 32-bit address space and 4KB(4k bytes) page size
    - Need 12 bits for offset because:
        - One Page size = 4,096 bytes
        - How many bits are needed to represent 4096?
            - 2^k = 4096; k = 12
        - The rest of 20 bits is for virtual page number
- Privileged instructions
    - TLBWI
        - Write to specific entry in TLB
    - TLBWR
        - Replace entry in TLB randomly
    - TLBP
        - Probe entry in TLB
    - TLBR
        - Read entry in TLB
- Structure
    - VPN
    - Not used slot
    - G
        - Whether address. can be shared
    - ASID
        - Address space identifier
    - PFN
        - Physical frame number
    - C
        - Coherence, how entry is written to TLB
    - D
        - Dirty, whether address is updated in TLB
    - V
        - Valid, whether address translation is valid or not
- Some entries in TLB are reserved for os kernel. A wired register is used to store how many such entries exist

##  Paging: Smaller Table

Page table grows quickly
- size
    - page: 4KB
    - 32-bit address space: $2^{32}$ byte
    - page table entry: 4 byte
- each process will have
    - number of pages: $2^{32} \over {2^{12}}$ = $2^{20}$
    - page table size: 2^20 * 4 bytes = 4 MB

Big page
- size of the page increases, number of pages decreases, and thus size of page table decreases.
- con:
    - internal fragmentation

hybrid approach: segmentation and paging
- intuition: we like to see best of both segmentation and paging
    - only save base and bound register to avoid waste
    - fix size allocation to avoid fragmentation
- pro: unused page won't take spaces in page table
- con:
    1. waste space when sparsely use in large space
    2. introduce external fragmentation since we segment page table

![alt text](image-22.png)
- page table for 16-KB address space
- Most of the pages are invalid(not used) but still have entries in the page table.

![alt text](image-23.png)
- Use top two bits(segment bits) to specify types of segment
    - e.g. 00 unused segment, 01 for code segment, 10 for heap segment, 11 for stack segment

```
SN = (VirtualAddress & SEG_MASK) >> SN_SHIFT
VPN = (VirtualAddress & VPN_MASK) >> VPN_SHIFT
AddressOfPTE = Base[SN] + (VPN * sizeof(PTE))
```
- Each process uses segment bits to find base and bound registers for code/heap/stack.
    - base and bound registers reference to linear page table in memory
    - process traps to os when process accesses memory out of bound
    - save and restore registers during context switch

![alt text](image-24.png)

Multi-level page
- Only save translation in page table when process access memory. This can save space for page table.
- structure
    - page directory
        - page directory entry(PDE): pointer to page of page table
            - valid bit: page of page table is allocated or not
            - PFN: location of physical memory
    - page table
        - page table entry(PTE): pointer to page of data you like to access
- pro
    - compact
        - it supports sparse address space
    - easy to manage memory
        - each page table fits within a page
- con
    - computation costs
        - issue two loads when address is not saved in TLB
    - complexity
        - traverse multiple levels to find the page table you need


![alt text](image-25.png)
page table index and page directory index
- assume
    - size of address space = 16 KB
    - page size = 64 bytes
    - PTE size = 4 bytes
- number of page entry = $size\ of\ address\ space\ \over page\ size$ = 256
- thus address is represented by 14 bits
    - 6 bit offset
    - 8 bit VPN
- page table size = PTE size * number of page entry = 1 KB
- number of pages for page table =  page table size/page size =  16 pages
    - each page holds 16 PTE
- we needs 16 page directory index
    - thus, 4 bits of VPN
- we need 16 page table index
    - thus 4 bits of VPN

to find address of page directory index
- PDEAddr = PageDirBase + (PDIndex * sizeof(PDE))

to find address of page table index
- (PDE.PFN << SHIFT) + (PTIndex * sizeof(PTE))
- assumes page size = 64 bytes, shift = $\log_2 64$ = 6

finally, to find physical address for the request
- PhysAddr = (PTE.PFN << SHIFT) + offset


Next we consider more than two levels
![alt text](image-26.png)
- assume
    - 4 byte PTE
    - 512 byte page
        - holds $2^7$ = 128 entries
    - 30 bit address space
- offset bit = $\log_2 512$ = 9
- VPN bit = 30 - 9 = 21
    - bit of page table index = 7
        - number of page table index for a page
            - = page size / PTE size
            - = $2^7$
            - = 128
    - page directory index(14 bit)
        - we can't fit $2^{14}$ PDE in a single page
        - thus split it into
            - 7 bit page directory index 0
            - 7 bit page directory index 1

code
```
1 VPN = (VirtualAddress & VPN_MASK) >> SHIFT
2 (Success, TlbEntry) = TLB_Lookup(VPN)
3 if (Success == True) // TLB Hit
4   if (CanAccess(TlbEntry.ProtectBits) == True)
5       Offset = VirtualAddress & OFFSET_MASK
6       PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
7       Register = AccessMemory(PhysAddr)
8   else
9       RaiseException(PROTECTION_FAULT)
10 else // TLB Miss
11  // first, get page directory entry
12  PDIndex = (VPN & PD_MASK) >> PD_SHIFT
13  PDEAddr = PDBR + (PDIndex * sizeof(PDE))
14  PDE = AccessMemory(PDEAddr)
15  if (PDE.Valid == False)
16      RaiseException(SEGMENTATION_FAULT)
17  else
18      // PDE is valid: now fetch PTE from page table
19      PTIndex = (VPN & PT_MASK) >> PT_SHIFT
20      PTEAddr = (PDE.PFN<<SHIFT) + (PTIndex*sizeof(PTE))
21      PTE = AccessMemory(PTEAddr)
22      if (PTE.Valid == False)
23          RaiseException(SEGMENTATION_FAULT)
24      else if (CanAccess(PTE.ProtectBits) == False)
25          RaiseException(PROTECTION_FAULT)
26      else
27          TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
28      RetryInstruction()
```

inverted page tables
- use hash table to store PTE
    - VPN: (PFN, process id)

swapping page table to disk
- Swap page table to disk during high memory pressure
    - works for system that place page table in kernel virtual memory

## Swapping

To run multiple processes concurrently, size of address space is larger than memory space. Thus, os stashes away some memory in hard disk drive.

illusion of large address space
- Program allocates memory as much as it need without worrying that address space is running out.

swap space
- reserved area in disk where OS read/write pages from/to
- note that binary code is not in swap space; they can be swapped too.

disk address
- address of a page in swap space

page fault
- when looks up a page table index and found that page is not in physical memory
    - present bit is not set to 1

page fault handler
- it is executed by os to service page fault
- steps
    - find PTE for disk address, issue the request to disk to load page into memory
    - page table is updated with new PFN for a PTE
    - OS fetch with given virtual address again
    - translation of address can't be found in TLB
    - lookup page table for the translation
    - update TLB with translation
    - find translation in TLB and fetch data

overlap of io with execution of another process
- IO is in flight during swapping, process is blocked, another process is run.

page replacement policy
- when memory is full and you like to page in, you page out few pages to make rooms in memory
- Speed of memory access is like disk if you kick the wrong pages

hardware control flow
- note that when page fault is raised, hardware traps invalid access, and os run trap handler to load page from swap space into physical page frame.
```
1 VPN = (VirtualAddress & VPN_MASK) >> SHIFT
2 (Success, TlbEntry) = TLB_Lookup(VPN)
3 if (Success == True) // TLB Hit
4   if (CanAccess(TlbEntry.ProtectBits) == True)
5       Offset = VirtualAddress & OFFSET_MASK
6       PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
7       Register = AccessMemory(PhysAddr)
8   else
9       RaiseException(PROTECTION_FAULT)
10 else // TLB Miss
11  PTEAddr = PTBR + (VPN * sizeof(PTE))
12  PTE = AccessMemory(PTEAddr)
13  if (PTE.Valid == False)
14      RaiseException(SEGMENTATION_FAULT)
15  else
16      if (CanAccess(PTE.ProtectBits) == False)
17          RaiseException(PROTECTION_FAULT)
18      else if (PTE.Present == True)
19          // assuming hardware-managed TLB
20          TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
21          RetryInstruction()
22      else if (PTE.Present == False)
23          RaiseException(PAGE_FAULT)
```

os control flow
```
1 PFN = FindFreePhysicalPage()
2 if (PFN == -1) // no free page found
3   PFN = EvictPage() // replacement algorithm
4 DiskRead(PTE.DiskAddr, PFN) // sleep (wait for I/O)
5 PTE.present = True // update page table:
6 PTE.PFN = PFN // (present/translation)
7 RetryInstruction() // retry instruction
```
swap daemon(page daemon)
- a background thread that pages out pages from memory when num of free pages < low watermark, and until num of free pages = high watermark

cluster pages
- write a group of pages to disk partition at once; thus increase performance by reducing disk seek and rotation.
- we need spaces for clustering; thus replacement is necessary to keep some of memory free.

## Policy

Replacement policy
- OS has to decided which page in physical memory to evict when OS is running out of memory.
    - It indicates that free page list doesn’t have any more free page.
- When choosing a policy, we likes to maximize cache hits and minimize cache miss

Cache management
- Average memory access time
    - $AMAT = T_M + P_m*T_D$
        - $T_M$: cost of  memory access
        - $P_m$: probability of cache miss
        - $T_D$: cost of disk access
    - Since the cost of disk access is so high, we likes to minimize $P_m$
        - Usually, $T_D$ in 10 ms and $T_M$ in 100 ns
- Cold start miss
    - When cache is empty, cache miss will happen definitely

Policy
- Optimal replacement policy
    - Furthest in the future
    - Con
        - You can’t predict the future as OS is for general purpose.
- FIFO
    - First in first out
    - Page at the tail of the queue is evicted
    - con:
        - Low cache hit rate
            -  Page is evicted based on whether it’s first brought into memory rather than how those page is used
- Random
    - Choose a random page in the cache
    - Cache hit rate depends on luck of the draw
- LRU
    - Least recently used
    - If a page has been accessed in the near past, it’s likely to access that page again in the future
- LFU
    - Least frequently used
    - Pages which aren’t accessed frequently should be evicted

Workload example
- When cache size is big enough to hold every page, which policy is chosen doesn’t matter.
- When workload is random, optimal strategy is the best
- When workload has locality(e.g. 80% of references targets 20% of pages), optimal and LRU work best
- When workload is sequential(e.g. access page 1, 2, 3,… 100 in order), random and optimal work best.
- Note than random doesn’t have weird corner case.

Historical algo
- Hardware can update time field to current in memory when memory is accessed. To replace least recently used page, simply scan through time field of all page and pick the oldest
    - con: cost is expensive. e.g. for 4 GB memory size and 4KB page size, there are million of pages which are a lot to scan even for modern CPU

Approximate algo
- Clock algorithm
    - Use clock hand to point to a page, check it reference bit. If $reference bit = 1$, page has been used recently. Set it to zero and move to the next bit until a zero reference bit is found and return that page.

Dirty Page
- Hardware set dirty bit(modified bit) when page is changed
- Eviction is first made to page that is clean.

Page selection: it determines when page is brought to memory
- Demanding page
    - Load page into memory when OS requires it
- Prefetching
    - Predict the use of page and load it into memory

How page is written to disk
- Clustering or grouping
    - Collect pending writes and write them to disk
        - It efficient because structure of disk lets large single write more efficient

Thrashing
- It happens when memory demand of the process is more than what is available
- Admission control
    - Only allows subset of process to subscribe to memory
- Out of memory killer
    - Daemon chooses a memory intensive process and kills it
