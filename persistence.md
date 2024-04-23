## I/O devices

System architecture
- Memory bus
    - connects CPU to RAM
- PCI
    - connects graphics & high performance IO device
- Peripheral bus, such as as SATA, USB, SCSI
    - connects slow IO device such as disk, mice, keyboard
- Note: an IO chip may connect to CPU with DMI interface. This offers higher performance.

Why system designer doesn’t make everything fast?
- Physics and cost
- Memory bus is close to the CPU, and peripheral bus is furthest from CPU

Internal structure of IO device
- Chips
- RAM
- firmware(software that runs in hardware)

Device protocol
- It regulates how data can be transfer to device
- Register
    - Command
    - Status
    - Data
- How os to interact with device
    - Polling the device
        - basically keep checking whether status register == busy
    - Pass data to data register
        - Usually needs multiple write since device block(e.g. 4KB) can be large
    - Set the command that is executed by the device
        - OS will keep polling after command is set
- Why is it not efficient?
    - Polling pauses the process and CPU is waiting.
- Example
```
While (STATUS == BUSY) ; // wait until device is not busy
Write data to DATA register
Write command to COMMAND register (starts the device and executes the command)
While (STATUS == BUSY) ; // wait until device is done with your request
```
- Use interrupt to overlap IO with other computation task
    - Process issues disk IO requests
    - CPU pauses the process and start another process
    - When IO request is completed, device interrupts to stop current process
    - Interrupt handler is executed, and waiting process restarts.
- Lots of interrupts lead to high cost of context switch
    - Use hybrid of polling and interrupt to save cost
- Use poll when device is performant

Data movement with DMA
- Programmed IO(PIO) transfer data from memory to a device before each word is written in device.
- Use DMA(direct memory access) instead of CPU.
- When DMA finishes transferring the data, it sends an interrupt to OS.

How does os actually sends the data to device?
- IO instruction
    - e.g. OS uses in/out instruction with parameters of register and port of device.
    - It’s a privileged instruction. OS has to trust user program which is a bad idea.
- Memory mapped I/O
    - Hardware provides device registers.
    - OS issues load&store to that register.
    - Hardware transfers data from register to device.

Device driver
- a software that encapsulates details of how os issue read/write request
- File system doesn’t know detail of how specific device should be read/written. Rather, it use a generic block read/write interface.
- You can use specific block interface to read/write to the device. It helps support low-level storage management
- Con
    - SCSI device has detail error reporting but high level abstraction hides most of them.


## hard disk

Hard disk drive
- Persistent data storage which is managed by file system
- Consist of an array of sectors from 0 to n-1, each with 512 bytes
    - You can think of them as address space of drive
- Each write to a sector is atomic
- Accessing continuous blocks is faster than accessing random blocks
- To read/write to the disk, induce magnetic change to the disk head

Structure
- Platter
    - Plates with two side, each side coated with magnetic material
- Spindle
    - spin the platter
- Track
    - Circular area on the platter that stores data in unit of block
- Disk head + disk arm

Seek phase
- Acceleration
- Coasting
- Deceleration
- Settling

Track skew
- Allows disk head to read continuously even when the sectors are distributed across track boundaries

Write mechanism
- Write back
    - Write is confirmed as long as data is written to cache
    - Pro: fast IO
    - Con: error prone
- Write through
    - Write is confirmed when data is written to disk
        - Pro: Correct
        - Con: slow IO

IO time
- $T_IO = T_seek + T_rotation + T_transfer$
- $R_IO = Size_transfer / T_IO$
- Most disks are optimized for sequential read; random read will be slow
- In the market, people pursue either
    - Performance, rate of transfer
    - Capacity, cost per byte

Disk scheduling
- OS decides order of IO requests
- type
    - Shortest job first
    - Shortest seek time first
    - Nearest block first

Scan
- Why?
    - We like to avoid starvation. If we favor disk request that reads nearby track, then the request that reads far away track will be starved
    - These types of scan algorithm are also called elevator algorithm.
- Sweep
    - Disk head is positioned from inner to outer or from outer to inner.
- F-Scan
    - If tracks are already read in a sweep, then further requests that read those tracks will be stored in a queue and will be serve in the next sweep
- C-Scan
    - Circular scan
    - In a sweep, disk head is first positioned in outer track and the move toward inner track.
- Shortest positioning time first
    - Schedule disk requests based on both seek and rotational costs
    - Often scheduling is implemented in the drive because drive itself has knowledge of position of disk head and layout of the track.

IO merging
- Work conserve
    - OS issues the disk request as long as there is one.
    - Disk is busy most of the time
- Non work conserve
    - Or anticipatory disk schedule
    - Group disk requests that read nearby block

