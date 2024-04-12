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
