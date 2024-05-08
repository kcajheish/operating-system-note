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
