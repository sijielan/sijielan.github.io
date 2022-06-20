# A small summary of Block Device Driver
### How to handle a Read/Write request in Linux kernel
1. A process issues a *read()* system call.
2. *read()* system call will activates a suitable function.
passing a file descriptor and corresponding offset inside file. VFS(virtual filesystem) is the upper layer of that handling architetcture.
3.	VFS determines if the requested data is already available or not. (data already in RAM or not?)
4. If the kernel must read the data from secondary storage(SSD or HDD), it need ot know the physical location of the data. Thus, kernel works on ***mapping layer***:
	1.	Mapping layer determines two things: **block size** of the file system, ***file block numbers*** of extent of requested data. (relative to the beginning of file)
	2.	Next, Mapping layer invokes a filesystem-specific function which access file's disk inode and determines the physical position of the requested data on disk in terms of *logical block numbers*.
5.	Kernel issue read operation on the block device, by using ***generic block layer***, in this layer, I/O operations response for transder requested data. Typically, _each I/O operation involves a group of blocks
that are adjacent on disk, thus generic block layer might operate multiple I/O operations_. **Each I/O operation is represented by a "block I/O" (bio) structure, which contains all information needed by lower components**.
6. In the lower layer, I/O scheduler would arrange all pending I/O requests according to predefined kernel pilicies. Key idea of this is to arrange data that closer to each other on disk together.
7. Last, ***block device driver*** take care of actual data transfer by sending suitable commandsto hardware interfaces of disk controller.


### Multiple components and layerso of this process

|            Components           | chunks length |                   Comments                  |
|:-------------------------------:|:-------------:|:-------------------------------------------:|
|     Block Device Controller     |    Sectors    |              typically 512Bytes             |
| VFS, mapping layer, filesystems |     Blocks    |                typically 4KiB               |
|          block devices          |    Segments   | a memory page or a portion of a memory page |
|           Disk caches           |     Pages     |             pages of page frame             |
|       Generic block layer       |   all above   |                     all                     |

table of different chunks lengths of different components.

### More detail of each kind of chunck
**Sectors**: each data transfer operation for a block device acts on a group of  adjacent bytes called a **sector**. Typically, the size of sector is 512 Bytes, although there are some larger sizes, like 1024
Bytes and 2048 Bytes, if a device use a large size of sector, low-level block device driver will do the corresponding conversions. Thus, a group of data in block device is identified on disk by its position: 
 **index of first 512-byte sector** and **its length as number of 512-byte sectors**. These information is stored in type *sector_t*.

**Blocks**: When kernel access a file content, it reads a block which containing the corresponding **disk Inode** of the file. In the block, it contains one or more sectors which are looked at as a single data unit.
In linux, the block size must be the a power of 2 and cannot be larger than a page frame. Also, it must be a multiple of the sector size, since block must contain **an integral number** of sectors.
Several partitions on the same disk could use different block sizes. 

*Block buffer*, is a RAM memory area used to store data in a block after reading the data from  the block in the device. Similarly, when doing write, it flush block buffer into the corresponding block.

Each buffer has a ""buffer head" descriptor of type *buffer_head*, which contains all information needed by kernel.

*b_page*: page descriptor address of the page frame,

**Segments**: *scatter-gather DMA transfer*, block device driver send following things to the disk controller:

1. The initial sector number and total number of sectors to be transferred .
2. A list of descriptors of memory areas, each of which consists of an address and length.

### 	The Generic Block Layer
1. Puts data in high memory, the page frames will be mapped in the kernel address space when CPU must use it, and unmapped right after.
	1. High memory: user space, low memory: kernel space. 
2. Implement, with "zero-copy" schema, where disk data is directly put in the User Mode address space without being copied to kernel memory first. Actually, buffer used by kernel for I/O transfer lies in a
 **page frame mapped in the User Mode address space of a process**.
	2. **Zero-copy:** Usually when a user space process has to execute system operations like reading or writing data from/to a device (i.e. a disk, a NIC, etc.) 
	through their high level software interfaces or like moving data from one device to another, etc., it has to perform one or more system calls that are then executed in kernel 
	space by the operating system.  *ref: https://developer.ibm.com/articles/j-zerocopy/*


### The Bio Structure
1. bio(block io), essentially include an identifier for a disk storage area-- initial sector number and the number 
of sectors included in the storage area--and one or more segments describing the memory areas involved in the I/O operation.

2. each **segment** in a bio is represented by a ***bio_vec*** data structure. ***bi_io_vec*** of the bio points to
 the first element of ***an array of bio_vec*** data structures, and ***bi_vcnt*** is used to store current number of elements in the array.

### Submitting a request
1. Executing **bio_alloc()** function to allocate a new **bio descriptor**. For the bio descriptor, kerbel will set following things:
	1. **bio_sector** field: initial sector number of data.
	2. **bi_size** field: number of sectors covering data.
	3. **bi_bdev** field: address of block device descriptor.
	4. **bi_io_vec** field: initial address of an array of **bio_vec** data structures, each of which describes a segment involved in the I/O operation.
	5. **bi_vcnt** field: total number of segments in the bio.
	6. **bi_rw** field: flag of operation. **WRITE(1)** or **READ(0)**.
	7. **bi_end_io** field: address of a completion procedure which is exxcuted whenever I/O operation on bio is finished.
