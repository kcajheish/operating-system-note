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

## Redundant Array of Inexpensive Disks(RAIDs)

In RAIDs, multiple disks are connected in parallel to increase capacity, redundancy, and speed up IO time

RAIDs offers transparency
- RAIDs interface is like a single disk; it has excellent software compatibility and deployability.

mirrored RAIDs system
- a logical IO takes more than one physical IO to keep disk block redundant

structure
- connection
    - e.g. SATA, SCSI
- micro-controller
    - firmware: write data to buffer block or read data from DRAM

fault model
- fail stop
    - disk has two states: working or fail
- a controller hardware or software detects disk fault immediately

evaluation
- capacity
    - given N disk with B block, how many blocks can client effectively write to?
        - without mirrored: N*B blocks
        - mirrored: N*B/2 blocks
- reliability
    - how many disk faults system can take before disk stops/fails
- performance
    - depends on the workload

RAID level 0: striping
- An array of blocks are distributed to disks using round-robin
    - increase parallelism for IO requests
- stripe
    - blocks in the same rows
- chunk size
    - In a stripe, number of blocks a disk can have
    - large chunk
        - less parallelism
        - better positioning time, especially when the file fits in a single block
    - smaller chunk
        - more parallelism
        - poor positioning time

steady-state throughput
- bandwith of concurrent requests

single-request latency
- latency of single logical IO request
    - implies degree of parallelism for a logical IO

workload
- sequential
    - read continuous blocks
    - e.g. scan a key work in a large file
- random
    - e.g. read in database management system
- mix

S: bandwidth of sequential workload
R: bandwidth of random workload

To calculate bandwidth of a disk under sequential/random workload
$$
band\ width = {
    {amount\ of\ data\ transfer}
        \over {T_{rotation} + T_{seeking} + T_{transfer}
    }
}
$$

to calculate bandwidth of RAID 0
$$
    {band\ width} = SN or RN
$$

## File and directory

file
- inode number + readable name
- an abstraction over persistent storage
- bytes on the disk blocks; store your data

directory
- inode number + readable name
- bytes on the disk blocks; include a list of tuple (inode number, readable name) for file and directory

directory hierarchy
- root(/)
- subdirectory
- file
    - filename + extension

open system call
- param
    - filename
    - flag
        - permission
        - truncate
        - create
- return
    - file descriptor
        - a pointer to persistent storage; you can read & write to it
        - stored in a process struct
            - unique to a process

read
- params
    - file descriptor
    - buffer
    - size of buffer
- return
    - no. bytes read
        - stop when you can't read anything

write
- params
    - file descriptor
    - buffer
    - size of buffer
- return
    - no. bytes written

close
- params
    - file descriptor

how does cat command access file?
- e.g. cat foo.txt
- use strace to find out
    - it traces every system call

reserved file descriptor for a process
- 0: standard input
- 1: standard output(screen)
- 2: error

open file table
- per process
- each entry belongs to a unique file descriptor; contains
    - underlying file
    - offset
    - readable/writable

offset
- location where next read/write happnes
- updated for each read and write
- **lseek** moves the offset
```
int main(int argc, char *argv[]) {
    int fd = open("file.txt", O_RDONLY);
    assert(fd >= 0);
    int rc = fork();
    if (rc == 0) {
        rc = lseek(fd, 10, SEEK_SET);
        printf("child: offset %d\n", rc);
    } else if (rc > 0) {
        (void) wait(NULL);
        printf("parent: offset %d\n",
        (int) lseek(fd, 0, SEEK_CUR));
    }
    return 0;
}
```

file struct in xv6
```
struct file {
    int ref;
    char readable;
    char writable;
    struct inode *ip; // point to underlying file
    uint off;
};
```

open file table struct in xv6
```
struct {
    struct spinlock lock;
    struct file file[NFILE];
} ftable
```

opening the same file returns different entry
- e.g. offset is updated independently for each entry

parent/child(created by fork) shares the entry in open file table.
- reference count
    - increased by one for each share
- coordinate different process to work on the same task

dup(fd) -> fd
- return a different fd that points to same underlying file
- useful for redirection
```
int main(int argc, char *argv[]) {
    int fd = open("README", O_RDONLY);
    assert(fd >= 0);
    int fd2 = dup(fd);
    // now fd and fd2 can be used interchangeably
    return 0;
}
```

writes are buffered before they are persisted in storage.
- however, data are lost if machine crashes before data are persisted

fsync
- force all dirty written to the storage
- make sure to fsync both file and directory if file is created.
```
int fd = open("foo", O_CREAT|O_WRONLY|O_TRUNC,S_IRUSR|S_IWUSR);
assert(fd > -1);
int rc = write(fd, buffer, size);
assert(rc == size);
rc = fsync(fd);
assert(rc == 0);
```

rename(old_file_name, new_file_name)
- atomic
    - no in-between state of the file name
- used when issue mv or update file with editor
```
int fd = open("foo.txt.tmp", O_WRONLY|O_CREAT|O_TRUNC, S_IRUSR|S_IWUSR);
write(fd, buffer, size); // write out new version of file
fsync(fd);
close(fd);
rename("foo.txt.tmp", "foo.txt");
```

stat
- print out metadata of a file
    - metadata is stored in inode
        - inode is stored in persisten data structure and manipulated by file system
```
struct stat {
    dev_t st_dev; // ID of device containing file
    ino_t st_ino; // inode number
    mode_t st_mode; // protection
    nlink_t st_nlink; // number of hard links
    uid_t st_uid; // user ID of owner
    gid_t st_gid; // group ID of owner
    dev_t st_rdev; // device ID (if special file)
    off_t st_size; // total size, in bytes
    blksize_t st_blksize; // blocksize for filesystem I/O
    blkcnt_t st_blocks; // number of blocks allocated
    time_t st_atime; // time of last access
    time_t st_mtime; // time of last modification
    time_t st_ctime; // time of last status change
};
```

unlink
- remove file

mkdir
- make directory

ls
- read directory
```
int main(int argc, char *argv[]) {
    DIR *dp = opendir(".");
    assert(dp != NULL);
    struct dirent *d;
    while ((d = readdir(dp)) != NULL) {
        printf("%lu %s\n", (unsigned long) d->d_ino,d->d_name);
    }
    closedir(dp);
    return 0;
}

struct dirent {
    char d_name[256]; // filename
    ino_t d_ino; // inode number
    off_t d_off; // offset to the next dirent
    unsigned short d_reclen; // length of this record
    unsigned char d_type; // type of file
};
```

rmdir
- remove directory
    - folder must be emtpy

ln old_file_name new_file_name
- create a new link that refers to the file
    - having new readable name that is given to the same inode number
- you can't link a directory to avoid cycle

reference count
- track number of links to a inode
- if reference count = 0, file is removed(e.g. data/node blocks are free).

when file is created
1. inode is created
2. human readable name is assigned to that inode

when file is deleted
1. link to the inode is removed
2. inode is removed if reference count = 0

symbolic link
- ln -s file file2
    - create symbolic link file2
- a file that contains path
    - the longer the path, the larger the link is
- dangling reference
    - symbolic link doesn't point to any hard link
        - hardlink is removed while symbolic link has yet
```
> echo hello > file
> ln -s file file2
> cat file2
hello
> rm file
> cat file2
cat: file2: No such file or directory
```

permission bits specify permissions for groups
- identity
    - owner
    - group
    - other
- bit
    - readable, 4 bits
    - writable, 2 bits
    - excutable, 1 bit
- e.g. owner can read/write to the file
    - chmod 600 file_name

to have more granular contorl down to each user, use ACL

Different file systems can be glued to existing file system tree.
- implication
    - can swap or integrate various persistent device
- e.g. > mount -t ext3 /dev/sda1 /home/users
    - mount point: /home/users
    - device: /dev/sda1
    - file system type: ext3
    - result: the root of device becomes mountpoint

## File system implementation

The direct pointers in the inode refers to a data block that belongs the file.
    - Con:
        - For a large file, there are many pointers which outsize the inode block(~4KB)

Indirect pointer lets a inode holds more pointer. It points to a different block of pointers, each of which point to the data block.
- e.g. Assume block size = 4 KB, address size = 4 byte, for a inode with 12 pointers and 1 indrect pointers, we have
    - 12 + 4*1024 byte/4 byte = 12 + 1024 = 1036 pointers

Multi indirect pointer can grow the file size exponentially. Take a doulble pointer as an example.
- number of pointer: 1024*1024
- size: 1024*1024*4KB =~ 4GB

Why multi level design works?
- Most of the files are small. It makes senses to mix direct and indrect pointer.

Directory is treated as a file. It has a inode that is stored in inode table. Data block pointer by that inode stores a list of tuple of directory-name and inode number.

Another approach is to use linked list.
- A file allocation table stores key as address of data block and the value as address of next data block.
- Con
    - really store when you try to read the last data block or random access

vsfs(very simple file system)
- Data structure stores reference to metadata and actual data
- Use system call to read those data structure.
- Organization
    - How many inodes in a block?
        - disk was partitioned into 4 KB blocks.
        - Assume there are 64 blocks in total
            - 56 are data region
            - 5 are inode table
                - inode table is an array of inode.
        - Size of inode = 256 byte = 1/4 KB
        - Thus a block have $${4\ KB \over 1/4\ KB } = 16\ inode$$
        - We have 5 blocks. Thus in total we have 80 inode.
    - Whether blocks are allocated or not?
        - Use data structure to keep track of free blocks
            - free list
            - bitmap
    - A superblock stores information of the file system
        - e.g. number of inode data block, begin of inode ...etc
    - Three blocks are reserved for bitmap of inode and data.
    - Inode
        - index node
        - Assumes file system reads inode number of 32, which sector should be located by disk head?
            - Inode table address starts at 12 KB.
                - note that three blocks are already reserved
            - address of the inode 32
                $$12 + 1/4*32 = 20 KB$$
            - Since addressable sector on disk is 512 byte(1/2 KB), we have sector
                $${20 \over 1/2} = 40$$
- What does "mount the file system to the disk" mean?
    - file system uses a tree structure to locate address of file and directory.
    - By mounting the file system, we map that tree to each physical location on the disk.

To track data block of file and directorty, file system uses multilevel index
- **Direct pointers** are stored at inode
- **Indirect pointers** are stored at inode as well, but they point to data blocks that have pointer to other data block.
    - Indirect pointers make file system scalable.
        - Assume that
            - memory address ~4 byte(32 bit)
            - One level indirect pointer
        - Number of addres that a block includes
            $$(4 * 1024)/4\ = 1024$$
        - Those data blocks can have size of
            - 1024 * 4KB = 4056 ~= 4 GB
        - If level of indirect pointer is two, then we scale the size of the file system by 1000 times.

Directory
- It is also pointed by a inode and it data block includes
- a list of pair of files and directory
    - inode number, entry name
- .
    - current directory
- ..
    - parent directory
- If file/folder is deleted from the directory, empty space is left and can be reused.

Free space management
- Use bitmap to track inode/data block that is free. When a new file is created, system scan through bitmap and allocate blocks for the new file
- Pre allocation
    - Continuous data blocks are allocated for a file.
     This ensures that access to a file is fast.

To access a file, we have to traverse the path to find the inode of a file.
- To open /foo/bar,
1. start with root inode, access data block to read inode from entry of foo
2. read data block of foo, read inode from entry of bar and then load it into memory
3. allocate a file descriptor for this inode and in per-process open table
4. return file descriptor to the user

Open a file incrus lots of IO. This is costly.
- Each level of directories requires two IO
    1. read inode
    2. data block
- File system caches popular blocks to save IO.

Cache and buffering
- fixed size
    - static partition
    - around 10% of memory are reserved to cache address of blocks
- unified page cache
    - cache both file system and translation in virtual memory
    - pro
        - Reduce memory cost when file access is not frequent
- How cache affects write?
    - Multiple writes to the same block can be batch. This reduces # of IO down to one.
        - Those batch will be scheduled so that changes are made to the disk.
    - avoid waste
        - if you add a line and later remove that line, no IO is needed
- How long you should wait before you write a batch to disk?
    - Longer wait leads to good performance but reduce reliability. If system crushes, then we lose all writes in batch.

## Locality and Fast File System
eary file system is slow
- treat disk like random access
    - positioning cost for a single file access is large. Inode and data block are far away.
- fragmentation
    - A small file is removed and leaves empty slots in continuous allocated bocks. Subsequent write may spread out.
    - degragmentation
        - a process that put file inode and data blocks close together

disk aware
- consider disk physical structure when design allocation and file system structure

disk structure
- cylinder
- cylinder group
- track

block group
- a logical expression for cylinder group; each group have blocks that os can write to
- fast file access in the same group since cylinders in the same group are close.
- structure
    - super block
        - replica of super block is kept in each group for reliability purpose
    - inode bitmap
    - data bitmap
    - inode block
    - data block

placement heuristics
- put new directory in the gorup with the least directory and the most free inode
- put inode and data block of a file in the same group
- put file in the same directory in the same group.
- e.g. /a/c, /a/d, /a/e, /b/f
```
group inodes     data
    0 /--------- /---------
    1 acde------ accddee---
    2 bf-------- bff-------
    3 ---------- ----------
    4 ---------- ----------
    5 ---------- ----------
    6 ---------- ----------
    7 ---------- ----------
```
name based locality
- files in the same folder are often accessed together
- measure locality with the distance to least common ancestor folder

large file exception
- spread large file into multiple groups and reduce costs of seeking among groups
    - amortization: reduce overhead by doing more work
        - e.g. put first 12 blocks close to inode rather than just one block
    - maintains locality for the folder of large file

other stuff about FFS
- subblock has size ~512 byte < block size 4KB
    - Subblocks are first allocated for the small file; when file size > 4KB, copy bytes from sublock to blocks and erase bytes in subblocks
    - it avoids internal fragmentation
        - e.g. file size = 2KB, block size = 4KB, wasted space $2/4 = 50\%$
    - con: issue many IO
- buffering write to subblocks before issue an IO
- parameterization layout
    - next sector is placed far away from the current sector so that file system has enough time to issue next write.
    - con:
        - peak bandwidth is reduced by 50% since it takes at least two rotations to read every sectors on the same track
- track buffer
    - simply cache bytes on the track

## Crash Consistency: FSCK and Journaling

crash-consistency
- how to update persistent data structure when system crashes or power is lost
    - persistent data structure: directory, file, metadata

two methods that can tackle crash consistency
- journaling(write ahead logging)
- fsck(file system check)

An append the the file requires:
1. update bitmap
2. update inode
3. update data block

Before the write takes effects, write is cached in buffer before is written to disk. A crash leads to
1. bitmap is updated
    - space leaks
2. only data block is written
    - file system doesn't know the file exist
3. only update inode
    - reads garbage data
    - inconsistent with bitmap
4. bitmap and inode are both updated
    - read garbage data
5. inode and datablock are written
    - inconsistent with bitmap
6. bitmap and datablock are written
    - inconsistent with inode
    - don't have pointer to the datablock

File system checker(fsck)
- check the entire disk and fix/remove problematic block
- timing
    - at boot
    - run before file system is mounted
- what is checked
    - superblock
        - make sure data block doesn't outgrow
    - free blocks
        - build directory tree by scanning inode
        - compare it to bitmap and trust the result of inode
    - duplicate
        - whether two inodes point to the same block
    - inode link
        - establish new link count and compare it with number in inode
        - move dangling inode to lost+found directory
    - inode state
        - check type of the inode
            - e.g. file, directory, symbolic linke...etc
            - clear inode if types can't be fixed
    - bad pointer
        - point to invalid address that is outside of the partition
    - directory
        - no directory is linked more than once
            - each directory has one parent and doesn' have cycle
        - each inode in the directory is allcoated
- con
    - too slow, has to scan entire disk blocks for only few wrong blocks
    - can't fix all problems
        - e.g. inode points to garbage data
    - need intricate knowledge of file system

Since writes are committed one at a time, it is hard to move file system from one state to another atomically.

write ahead logging
- Take note what to do next before data is written to disk. If system crashes, revew notes and resume write
    - Update takes more time but recovery time is reduced

Journal partition
- In ext2 and ext3, a small part of memory is reserved for jounaling before block groups after super block.

Data journal
- journal write
    - TxB: transaction begein
        - includes info of block address and TID(transaction identifier)
    - physical content of I, B, Db
        - also called physical logging which is in contrast with logical logging
- journal commit
    - TxE: transaction ends
- checkpoint
    - Keep file system update to date with pending write in journal

Design consideration of data journal
- Journal is commited after journal write.
    - Commit min 512 bytes to disk; thus, TxE must be 512 bytes in size to ensure atomicity.
- You can't commit transaction at once
    - Internal scheduling in disk may commit parts of big write first. Thus, we lose atomicity.

ordered journaling
- Performance is better than data journaling
    - In data journal, write traffics are double and most IO cost is spent on data block
    - In ordered journaling, data block is only written once
- Steps
    - data write: user data is written to block group directly
    - journal metadata: write metadata to journal
    - journal commit
    - checkpoint metadata
    - free: mark transaction free in superblock of journal
- data write comes first to ensure crash recovery
    - object is created before pointer
        - This makes sure pointer never points to garbage data
- notes that for correctness, we make sure to journal metadata and write datablock before journal is commited.
- ticky case block reuse
    - case: delete files in a directory and open a new file; system crashes before checkpoint; for ordered journaling, replay every log.
        - folder data are written to new file data block -> weird state for that datablock
    - sol: add revoke type
        - don't replay that data that has revoked

timeline
- data journaling
    - transaction begin and content must be commited before transaction end is issued
    - data write is issued after transaction end is commited
- ordered journaling
    - data write must be commited before transaction end is issued

other write techniques to maintain consistency
- soft update
    - write to data block comes before write to inode
    - pro: inode never points to garbage
    - con: requires knowledge of file system data structure
- copy on write
    - place new update in unused blocks
    - pro: easy to keep file system consistent
- backpointer
    - each block has a backpointer to inode/direct block
    - if forward pointer points to current block, the file is consistent
- optimistic crash consistency
    - issue as many writes as possible; thus reduce the risk of in consistency

## Log Structured File System

As technology evolves, memory size increases and most of the read reqests can be served from memory. Thus, performance of the file system is mostly determined by how fast a file can be written.

Cost of writing is determined by
- speed of rotation and seeking
    - e.g. how fast a disk arm can move
- number of IO requires to make a file
- whether bytes of files are stored sequentially

LFS
- Write requests are batched in the segment. Write segment into next new blocks in disk after segment is full.
    - Segment is a chunk of data for write reqeusts
    - Write to disk is sequential
    - Segment holds large write requests so that they can be written continuously without disk spinning too much
- size of buffer
    - we like to amortize the cost of position overhead
    - to achieve max bandwidth, we need to transfer more bits
    $$
    \begin{align}
    T_{write} = T_{position} + T_{transfer} \\
    T_{write} = T_{position} + {D \over R_{peak}} \\
    {D \over F*R_{peak}} = T_{position} + {D \over R_{peak}} \\
    {D \over R_{peak}}({{1 \over F} - 1}) = T_{position}\\
    D = T_{position}R_{peak}{F \over {1-F}}
    \end{align}
    $$
- keep track of lastest oversion because the inode moves when file is overwritten in place
    - inode map
        - params
            - input: inode number
            - output: disk address of the lastest inode
        - array type
            - each entry 4 byte(size of the address)
        - kept in disk and place right next to the file block
            - can recover from crash
            - reduce cost of positioning during write
    - checkpoint region
        - why?
            - Inode map is spread across memory. Need a way to find latest inode map for a given inode number.
        - map inode number to address of inode map
            - this can be cached in memory to speed up read
        - content is updated for every 30 seconds
            - write performance is amortized
- directory
    - create a file in directory
        - new inode and data block for file & directory
        - create a inode map having address of both file and directory
    - read process
        - read address of directory inode from inode map
        - read inode and then data block of the directory
            - you will find (name, inode number) tuple of the files
        - retrieve inode number of the file you looks for and looks up inode address in inodemap
        - read file inode and data block
- LFS avoid recursive update problems
    - We update address in the imap/cross region and never expose new location to directory. Thus, we don't have to update anything in the parent inode/data block.


garbage
- old version of the inode/data block is not needed when new file content is created
    1. overwrite the file
    2. append to the file
- live: newest version of data block
- versioning file system
    - the old version of file blocks is kept
- garbage collection: clear old version of blocks
    - LFS cleaner
        - steps
            1. read in old segment
            2. kept the live blocks
            3. write live block into new segment
            4. free old segment
        - The process is also called compaction
        - why segment not individual block?
            - avoid having holes on the disk; they are hard to reuse for large write

block liveness
- segment summary block(SSB)
    - For each data block, store tuple (inode number, offset in a file)
    - It is stored at the head of the segment.
- to determine whether block is obselete
    1. Given block address, find current inode number and offset from SSB
    2. Look up inode address in inodemap for the inode number
    3. Look up start address and offset in that inode
    4. If the block is not within the range(start + offset), then that block is old
- improvement: keep track of version for blocks in imap
    - In the same inode, block that has lower version is old

Policy: timing of cleaning
- hot segment
    - blocks that are often overwritten
- cold segment
    - blocks that are seldom overwritten
- we like to amortize cost of cleaning
    - cost of cleaning is expensive
        1. prove liveness for each block
        2. copy live block to new segment
        3. free old segment
    - intuition
        - If update to the segment is frequent, having to clean the segment for each update doesn't make sense
    - implication
        - The more often the segment is updated, the longer you wait to clean

Crash recovery
- System may crash when data is writte to disk. We like to avoid partial update in this scenario
- log
    - stored in CR
    - structure
        - pointer to head and tail of checkpoint
        - pointer to current and next segment
- Crash during write to CR
    - take turns to record CR on both end of the disk
    - include timestamp at header and end for each CR update
        - if header and end timestamps are inconsistent, we know system crashes
        - If that is the case, look for CR on both end that has latest complete pair of timestamp and keep it
- Crash during write to segment
    - roll forward(?)
        - lookup last checkpoint in log in CR
        - check valid write in the next segment
        - restore data since last checkpoint

## Flash-based SSD

solid-state storage device
- like memory device which consists of transistor without mechanical moving part
- however, it doesn't lose data after power is lost


Flash is the technology of SSD.
- to write to a flash page, you erase a flash block.
    - often writes leads to wear out

A cell in the flash chip can preset bit
- e.g. 4 level of charge is trapped in the cell. we have 4 binary representation:
    1. 00
    2. 01
    3. 10
    4. 11

flash ship structure
- bank
    - block, ~256 KB
        - page, ~4 KB

flash operation
- read
    - microsecond duration, regardless of
        - position of the device
        - position of previous request\
        -> good random access performance
- erase
    - expensive, ms duration
    - have to erase entire block if you like to write a page
        - by setting every bit in page to 1
        - before write, block data is copied somewhere
- program
    - 100 microseconds
    - write the page by settings bit patterns in that page

page state
- erase -> erased
- program -> valid
- doing nothing -> invalid\
-> to program, page must be in erased state

reliability
- wear out
    - A block goes through high and low voltage when it is erased and programmed. After multiple cycles of this, extra charges start to accumulate in block, to the point where we can't tell whether it is high or low voltage
- life time cycle
    - def: number of program/erased before a block wears out
    - range: 10k ~ 100k P/E
- disturbance
    - def: bits are flipped in neighboring pages during program/read

side note
- backward compatibility is important since it ensures interoperability by having multiple system follows a interface
- however, you may want to rewrite the whole thing when the interface can't be used for technology in the next generation

SSD structure
- flash chips
- memory
    - cahcing and buffering
    - mapping table
- control logic
    - orchestrate devices in SSD
    - flash translation layer
        - turns logical block(client read/erase/program) to low level commands on physical block & page
- consideration
    - performance
        - paralle chips
        - write amplification
            - def:
            $$
                write\ traffics\ issued\ to\ the\ chip\ by\ FTL(bytes) \over{
                    write\ traffics\ issued\ to\ the\ FTL\ by\ client(bytes)
                }
            $$
    - reliability
        - wear leveling
            - def: spread writes evenly across all page so that all pages wear off at the same time
        - sequential programming
            - def: erase/write from low page to high page in a block.
                - it minimizes disturbance

Directed Mapped
- Each logical page number maps to a physical page
    - implication: a write to the logical page requires
        1. erase all pages in the block
        2. write new content to the pages\
    - bad performance
        - cost ${\propto}$ number of pages in a block
        - large write amplications
    - bad reliability
        - hot page wears out quickly
            - e.g. file metadata

Log Structured FTL
- logging
    - next write happens at the next free page
- mapping table
    - logical block -> physical page
        - note that logical block here has different meaning from the one in directed mapped approach
- example
    - config
        - read/write with size of 4KB
        - 16 KB block
        - 4 KB page
        - 4 pages/block
        - initial state of page = invalid
    - write a1 to (001)
        - pick a free physical page, 0, for logical block 001
        - erase physical page, 0
        - write to first page in the block with a1
        - record (001, physical page 0) in mapping table
            - for lookup during read
    - pro
        - reduce write amplification; increase performance
            - erase block once in a while
        - wear leveling
            - write evenly to the page
    - con
        - too much garbage(old version of data) collection increases write amplification

Garbage collection
- reclaim dead block
    - garbage block: block that have outdated physical page
    - steps
        1. read live pages from block
            - lookup mapping table for pages with latest content
        2. log live pages them
        3. reclaim the whole block
    - con
        - high cost of reading and rewriting live pages
    - optimization
        - reclaim block with full dead pages
        - overprovision
            - increase device capacity so that we have more room for write and cleaning can be done in free time

Block Level FTL
- physical pages are partitioned into chuncks
    - logical block
        - chunk number + offset to the target page
    - mapping tables
        - chunk number: start page of the chunk
- pro
    - requires less memory space to store address, compared to page level mapping
- con
    - small write is expensive
        - each write a page needs
            1. copy pages in the chunk
            2. reclaim pages in the chunk
            3. write pages to new chunk with new content

Hybrid Mapping
- structure
    - log table
        - logical block: physical page number
    - data table
        - chunk number: physical block number
- goal: reduce traffics during small write
- case
    1. switch merge
        - overwrite data in a physical block
            - write data in new block
            - every page is written into log table
            - detect old block is dead
            - erase old block
            - point chunk number point to new block in data table and remove pointers in log table
    2. partial merge
        - overwrite partial data in few pages in a block
            - write partial data in new block
            - written pages are stored in log table
    3. full merge
        - turn every log-mapped page into block-mapped page
            - read every block for each page in log table
            - write pages in block to a new block and update data table
            - clear pages in log table

Page Mapping + Caching
- only translation of hot page is stored in the memory to save memory space
- evict
    - working set can't be accommodated in memory
        - working set: all possible translations
    - throw away old mapping and load new mapping
- dirty page
    - map in memory is updated but disk isn't

Write leveling
- Long structured FTL, writes are spread across pages and thus each page has the same erase/program cycle
- Detect long live block and rewrite them
    - Garbage collection reclaims dead pages; however, long live page is never reclaimed and thus break the leveling

Compare SSD with HDD(hard disk drive)
- SSD is similar to DRAM in that
    1. fast random read/write, ~100 MB/s vs 1 MB/s for HDD
    2. fast sequential read/write, ~400 MB/s vs 200 MB/s for HDD\
- thus:
    1. random IO cost is cheaper in SDD
    2. sequential IO cost is comparable to HDD
- use case
    1. hybrid of SSD + HDD
        - hot data in SSD
        - cold data in HDD
    2. use HDD for archive purpose
    3. use SSD if you care performance of random access

## Data integrity and protection

fail stop model
- disk works or fails completely
    - e.g. raid disk

latent sector error
- disk head crash on surface; you can't read bits from sector

error correcting code
- detect whether sector is damaged or not

block corruption
- data is corrupted when it pass faulty buss or firmware write the data in one block but client wants the other block
- silent fault; you can't tell whether it goes wrong

fail-partial model
- disk returns error or wrong content sometimes when client read/write from/to it

To build a reliable storage system, detect and recover form latent sector error and block corruption

to handle LSE, recover through redundancy mechanism
- e.g. second parity group to backup data
    - con: more cost for space

to detect corruption, use **checksum**
- given a chunk of data(block ~4KB), compute summary bytes(4~8 bytes) and store summary with the chunk data
- before return chunk data to client, compute the summary and compare it with stored summary

checksum function
- xor
    - Among two bits, either one holds 1 bit exclusively
        - e.g.
            - 1 xor 1 = 0
            - 0 xor 1 = 1
            - 1 xor 0 = 1
            - 0 xor 0 = 0
    - if we flip two bits, the result is the same
- 2’s-complement addition
    - steps
        - choose number of bits to present the number; add extra bit for negative sign
        - to add a negative number, flip all bits and plus one
    - overflow happens when adding two positive or negative number that is out of range
        - out of range: $number > 2^{number\ of\ bits\ that\ presents\ number}$
    - [src](https://www.youtube.com/watch?v=ydboHy_yNts&ab_channel=MITOpenCourseWare)
- cyclic redundancy check(crc)
    - divide a block, D, with value, k; the remainder is the checksum
        - D: number that represent the bits in block
        - k: any chosen value
- Fletcher
    - a block D is divided into byte
        - e.g. d1, d2, d3... dn(we have n division for n bytes)
    - calculate s1, s2 recursively
        - s1 = (s1 + di) mod 255
        - s2 = (s2 + s1) mod 255
collision
- two different content have the same checksum
- a good checksum function minimizes collision

checksum layout
1. spare extra space for checksum in each sector
    - pro: low IO cost
    - con: only work for certain disk
2. a checksum sector followed by n data block
    - pro: work for all disks
    - con: high IO cost

When $C_s(D) != C_c(D)$, there is corruption because data changes since the time it is stored
- $C_s(D)$: stored checksum
- $C_c(D)$: computed checksum
- D: block

When there is corruption in block, look for backup data or return error to the client

misdirected write
- Data are written to the wrong block or disk
- resolve: add physical identifier(physical ID)
    - include disk and block number
    - although they are redundant, they are helpful for error detection and recovery

lost write
- client is informed completion of write but the write never reaches to the disk surface
- e.g. Zettabyte File System (ZFS)
- sol
    1. copy checksum in file inode and indirect block
        - write is lost if checksum in inode ~= checksum in data block.
    2. read after write(write verify)
        - con: double IO cost of write

disk scrubbing
- periodically scan every block, whether checksum is valid
    - reduce the chance that all copy data are corrupted
    - cold blocks are rarely accessed and we don't know whether they are corrupted or not

overhead due to checksum
- disk stores checksum and thus user data has less space
- load stored cheksum and computed checksum into memory
    - however, it is short-live and impact is minimal
- CPU has to compute and compare checksum when data is loaded/stored.
    - mitigation: before data & stored checksum are transfered from kernel page to user buffer, compare the checksum
- extra IO is required when 1. scrubbing 2. checksum and data are not stored in the same block
