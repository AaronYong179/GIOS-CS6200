## Introduction and Overview

An operating system is in charge of managing the underlying hardware, as established in [[P1L2 Introduction to Operating Systems]]. However, we have only covered the CPU and memory thus far. In this lecture, I/O management will be investigated.

In order to manage I/O operations, OSes incorporate interfaces for different types of I/O devices. These interfaces determine the **protocol used for accessing I/O devices**. The OS also **provides certain handlers** (e.g., device drivers, interrupt handlers) for managing I/O devices. 

The combination of I/O protocols and handlers allow the OS to decouple I/O details from core processing. In other words, I/O hardware devices are abstracted (hidden) from calling applications via interfaces and provided handlers.
### I/O Devices

Examples of I/O devices include:
- keyboards (_input_)
- microphones (_input_)
- displays (_output_)
- speakers (_output_)
- mice (_input_)
- network interface cards (NIC) (_input and output_)
- flash card (_input and output_)
- hard disk drive (_input and output_)

![[Pasted image 20231021113846.png]]

As the above figure suggests, devices come in all shapes and sizes, with **a lot of variability** in their hardware architecture, functionality (that they provide) and interfaces (that are used to interact with them). However, they _must_ have some key features that enable their integration within a system.

Any device can be abstracted to have the following set of features:
![[Pasted image 20231021114353.png]]
- Any device will have a set of control registers that can be accessed by the CPU and permit CPU device interactions.
	- A command register is used by the CPU to control the device.
	- A data register is used by the CPU to control data transfer in and out of the device.
	- A status register is used by the CPU to find out exactly what is happening on the device.
- Internally, the device will incorporate all other device-specific logic.
	- The device will have a microcontroller, which is akin to the device's own CPU. The microcontroller will be the actual entity controlling the operations that occur on the device, which may be influenced by the actual CPU.
	- The device will also have its own memory.
	- Depending on the device, some hardware-specific chips might also be included. For instance, some devices may need specialised chips to convert analog signals to digital signals. Network devices may need special chips to interact with the physical network medium.
## Attaching a Device
### CPU/Device Interconnect

![[Pasted image 20231021115141.png]]

The figure above depicts how devices are connected with the rest of the CPU complex, via some CPU device interconnect.
- Specifically, devices are connected to the CPU complex via the **Peripheral Component Interconnect** (PCI) bus. PCI is one of the standard methods for connecting devices to the CPU.
- Notice that all devices interface with the rest of the system via some **controller**, which is typically integrated as a part of the device. These controllers determine the type of interconnect that a device can attach to. However, differences can be bridged with **bridging controllers**.

Modern platforms typically support PCIe (PCI express) -- it has more bandwidth, is faster, has lower latency, and supports more devices than its predecessors. However, for compatability, most platforms still support the older PCI-X (PCI extended), which extends the original PCI standard.

Note that the PCI bus is not the only possible interconnect. For example, the **SCSI bus** connects SCSI disks, and some **expansion bus** connects devices such as keyboards.
### Device Drivers

![[Pasted image 20231021133322.png]]

The OS supports devices via **device drivers**. Device drivers are **device-specific software components**. In other words, the manufacturer of a device is responsible for providing the necessary driver, and they are responsible for ensuring that the driver is available for all OSes where the device will be used. The OS needs to include a device driver for _every type of device within the system_. 

Device drivers are responsible for all aspects of device **management, access, and control**. These devices should manage:
- The logic that determines how requests are passed from higher level components to the device, or
- How these components react to errors or notifications from the device.

In sum, device drivers govern any device-specific configuration/operation details.

Of course, the OS provides a standardised interface for developers of device drivers. By placing the responsibility on the device manufacturers instead, the OS is allowed to be **device-independent**, and a greater **diversity of devices** can be supported.
#### Device Types

In order to deal with the great diversity of devices, devices are categorised into a few distinct device types. 
1. **Block devices** (e.g. disks)
	- These devices operate at the granularity of **blocks of data**. 
	- A key feature of block devices is their ability to access individual blocks. As an example, if there are ten blocks of data on a device, it is possible to directly access the ninth block.
2. **Character devices** (e.g. keyboards)
	- These devices deal with a serial sequence of characters and support a get/put interface.
3. **Network devices**
	- These devices are somewhere in between character and block devices. While they deliver more than a character at a time, they do not have a fixed block size. 
	- These devices handle a **stream of data chunks** that may potentially have different sizes.

The interface that a device exposes to the OS must be standardised based on the device type.
- All block devices must support reading and writing a block of data.
- All character devices must support some get/put operations for a character.

The OS's responsibility then, is to maintain some representation of devices on the platform. This is typically performed by the **file abstraction**, where each device is represented by a file. This allows the OS to interact with these devices using the mechanisms that are already in place for files.
- Note that since these "files" ultimately represent a device, certain operations such as `read` or `write` will be handled in a device-specific manner.
- On Unix-like systems, all devices appear as files under the `/dev` directory. As special files, they are treated under special filesystems, `tmpfs` and `devfs`.

_Quiz_
_The following Linux commands all perform the same operation on an I/O device (represented as a file). What operation do they perform?_

```
$ cp file > /dev/lp0
$ cat file > /dev/lp0
$ echo "Hello World" > /dev/lp0
```

These commands print some content to the `lp0` printer device. `lp` stands for "line printer", and the `0` refers to the first line printer identified by the Linux system. 

_Quiz_
_Linux supports a number of pseudo (virtual) devices that provide special functionality to a system. Given the following functions, name the pseudo-device that provides that functionality._

_accept and discard all output (no output produced)_: `/dev/null`
_produce a variable-length string of pseudo-random numbers_: `/dev/random`

_Quiz_
_Run the command ls -la /dev in a Linux environment. What are some of the device names that you see? Enter at least five device names._
- `hda`
- `sda` 
- `tty` (can be used to pipe output from one terminal to another)
- `null`
- `zero`
- `lp`
- `mem`
- `console`
## Communicating with a Device

### CPU/Device Interactions

##### CPU to Device

Device registers appear to the CPU as memory locations, with their own physical address. When the CPU writes to these memory addresses, the integrated PCI controller recognises that the memory access should be routed to the appropriate device.
- In other words, this necessitates a portion of physical memory to be dedicated for device interactions. This is also known as **memory-mapped I/O**.
- The portion of memory reserved for these interactions is controlled by the **Base Address Registers** (BAR). 
- The BARs are configured during boot in accordance with the PCI protocol.

The CPU may also access devices via special instructions. 
- For example, on x86 platforms, certain in/out instructions are specified for accessing devices. Each instruction must specify the intended device (i.e., the I/O port), as well as some value to be passed to the device. 
- This model is known as the **I/O Port Model**.
##### Device to CPU

Devices can generate **interrupts**, as seen in [[P2L4 Thread Design Considerations]].
- Interrupts can be triggered as soon as the device requires the CPU (e.g., has information for the CPU).
- However, interrupt handling costs CPU cycles, in addition to the need to set/reset interrupt masks. More indirectly, calling an interrupt handler may result in cache pollution. 

Alternatively, devices may be polled by the CPU. The CPU reads a device's status registers to determine if it has some response/data.
- The OS -- with polling -- is now able to determine the most convenient time to handle a device (e.g., when cache pollution effects are minimised).
- However, if polling is done with a significant delay in between polls, the device event might be handled with latency. On the other hand, if polling is done rapidly, this introduces CPU overheads. 
### Device Access Methods

#### Programmed I/O

With just basic support from an interconnect (e.g., PCI), a CPU can request an operation from an I/O device using **programmed I/O** (PIO). Note that this method _requires no extra hardware support_.

The CPU issues instructions by writing into the **command registers** of the device. The CPU controls data movement to/from the device by writing/reading the **data registers** for the device.

Consider the following scenario, where a CPU transmits a network packet via an NIC (network interface card).
- The CPU first writes to the command register of the NIC. This command instructs the device to perform a transmission using the data the CPU will provide.
- The CPU then copies the packet into the data registers, and this repeats until the entire packet is sent.
- For example, a 1500 byte packet being transmitted by an 8 byte data register will require 188 (1500/8) CPU accesses to the data register. In addition to the first CPU access to the command register, a total of 189 CPU accesses are required to transmit the packet.
#### Direct Memory Access

Alternatively, with hardware support in the form of a **DMA controller**, direct memory access can be performed.
- The CPU still writes commands directly into the command registers of the device. 
- However, data movement to/from the device is controlled by the DMA controller. Of course, the DMA controller will need to be configured accordingly.

Consider again the scenario where a CPU transmits a network packet via an NIC.
- The CPU first writes to the command register of the NIC. This command instructs the device to perform a transmission using the data the CPU will provide.
- The CPU also configures the DMA controller with the information regarding the **memory address** and **size of the buffer** holding the network packet.
- For example, a 1500 byte packet being transmitted would only require one CPU access to the command register of the NIC and another one DMA configuration operation.
- Note however that DMA configuration is not trivial and will take more CPU cycles than a memory access. For **smaller transfers, PIO is more efficient**.

As a final caveat, the data buffer must be in physical memory for DMA to take place. The DMA controller only has access to physical memory -- the memory regions involved in DMA must be pinned (i.e., non-swappable).

_Quiz_
_For a hypothetical system, assume the following:_
- _It costs 1 cycle to run a store instruction to a device register_
- _It costs 5 cycles to configure a DMA controller_
- _The PCI-bus is 8 bytes wide_
- _All devices support both DMA and PIO access_
_Which device access method is best for the following devices?_

_Keyboard_
> PIO is best. A keyboard will likely not transfer much data for each keystroke, hence PIO is cheaper.

_NIC_
> It depends on the size of the data packet to be transferred. If a larger size is encountered, then DMA is better.
### Device Access Flow

#### Typical Device Access

![[Pasted image 20231023205630.png]]

1. User
	- When a user process needs to perform an operation that requires a device, the process will make a system call specifying the appropriate operation.

2. Kernel/OS
	- The OS runs the in-kernel stack associated with the specific device. The kernel may also perform some data pre-processing. For example, the kernel may form a TCP/IP packet from an unformatted buffer.
	- The OS then invokes the appropriate device driver.

3. Driver
	- The device driver performs the necessary configuration of the device request. For example, an NIC device driver will write a record that configures the device to transmit a packet sent from the OS.
	- The device driver will then issue commands and/or send data using the appropriate PIO or DMA operations.
	- The drivers are responsible for ensuring that commands and data needed by the device are not overwritten or undelivered -- after all, they are the ones that understand the device registers/memory, as well as any other pending requests.
	- In short, the drivers perform the necessary **configuration and control**.

4. Device
	- Finally, once the device is properly configured, it will perform the actual request. For example, the NIC will transmit the packet onto the network.
	- Any results/events originating on the device will traverse the call chain in reverse.
#### OS Bypass

![[Pasted image 20231024100804.png]]

It is not always necessary to go through the kernel to access a device. Some devices may be directly accessible from user level -- this is known as an **operating system bypass**.
- Briefly put, any memory/registers assigned for device use is directly available to a user process.
- The OS is involved in making the device registers/memory are mapped to the user process. After the initial configuration stage, the OS is _mostly_ out of the way.

Of course, since the user process should not be able to breach the OS kernel layer, the device driver must sit at the user level. 
- This is roughly akin to some library that the user process uses in order to perform device-specific operations.
- These libraries are usually provided by the device manufacturers, very much like the kernel-level drivers.

When bypassing the OS, it is not completely true to claim that the OS is removed from the picture entirely. The OS still has to maintain **some coarse-grained control**. 
- For example, the OS should be able to enable/disable a device, modify permissions, or even add more processes that will use the device.
- Therefore, the device must have **enough registers**; some must be map-able -- by the OS -- to potentially multiple user processes (for actual device operations), while some must be retained for the OS to interact with the device (for configuration and control).

In the event that multiple processes are interacting with a device, the device **must figure out which process the data belongs to**.
- In short, the device must perform some protocol functionality in order to **demultiplex** (demux) different chunks of data belonging to different process.
- Take for example a network packet received by an NIC: this would mean that the device might have to peek inside the packet to see the receiving port, and the device should know which process is listening on that specific port.
- In a typical device access flow, the kernel performs the demultiplexing. In OS bypass, this responsibility belongs to the device.
#### Sync vs. Async Access

When an I/O request is made, the user process typically requires some form of response from the device, even if the response is merely an acknowledgement. The user thread behaves differently after an I/O request is issued depending on the type of request: synchronous or asynchronous.

![[Pasted image 20231024102634.png]]

For synchronous operations:
- The calling thread blocks. The OS kernel will place the thread on the wait queue associated with the device.
- Eventually, when the device responds, the thread will become runnable again.

For asynchronous operations:
- The thread is allowed to continue after it has issued its request. 
- At some later point in time, the user process may check if the response is available. Alternatively, the device or the OS can notify the user process that the operation has completed and any results are ready at some location.

## Block Devices

### Block Device Stack

Block devices (e.g., disks) are typically used for storage. The typical storage-related abstraction used by applications is the **file**. 
###### #DEFINE File 
>A file is a **logical storage unit**, mapped to some **underlying physical storage location**. In other words, higher level user processes do not think in terms of interactions with blocks of storage, rather the file **abstraction** is interacted with.

![[Pasted image 20231024103400.png]]

Below the file-based interface used by applications sits the kernel-level **filesystem**.
- The filesystem will receive read/write operations from the user process and it then has to determine _where_ the file is, _how_ to access it, _which_ portion to access, _what_ permission checks should be performed and so forth.
- Of course, the filesystem will ultimately have to initiate the actual access.

OSes allow for a filesystem to be modified, or to be completely replaced with a different filesystem altogether.
- This allows flexibility in how files are laid out in disk, how access operations are performed, etc.
- OSes therefore specify a filesystem interface both in terms of the **interface exposed to the user process**, as well as the interactions between a filesystem and the underlying device via its driver.
- For example, the POSIX API is used by applications to interact with the filesystem, including `read` and `write` system calls.

At the lowest level, different types of block devices can be used for physical storage (e.g., SCSI ID disks, hard disks, USB drives, SSDs).
- Interactions with the different block devices would require certain **protocol-specific API**. In other words, there will be differences among their APIs (e.g. error types).
- As such, the block device stack sees the inclusion of a **generic block layer**.

The generic block layer sits between the OS and the device drivers.
- This layer provides a standard for an OS to all types of block devices. The full device features are still available and accessible through the device driver, but are abstracted away from the filesystem.

_Quiz_
_In Linux, the `ioctl()` command can be used to manipulate devices. More specifically, this command allows the access of a device's registers. Complete the code snipped, using `ioctl`, to determine the size of a block device._

```c
// ...
int fd;
unsigned long numblocks = 0;

fd = open(argv[1], O_RDONLY);
ioctl(fd, BLKGETSIZE, &numblocks);
close(fd);
// ...
```

### Virtual Filesystem

#### Motivation

What if we want to make sure that a user application can seamlessly see files across multiple devices (e.g., HDD, SSD, USB Drive) as a single, coherent filesystem?

What if different types of devices work better with different filesystem implementations?

What if files are not even local to a machine, and are accessed over some network drive?

To resolve these questions, OSes such as Linux include a **virtual filesystem** (VFS) layer. This layer hides all details regarding the underlying filesystem from user-level applications. 

![[Pasted image 20231024153222.png]]

Applications may continue to interact with the VFS interface using the same POSIX API. On the other end, the VFS specifies a more detailed set of filesystem-related abstractions that every single underlying filesystem must implement.
#### VFS Abstractions
##### File

The file abstraction represents elements on which the VFS operates.
##### File Descriptor

The OS represents files as file descriptors. A file descriptor is an integer that is created **when a file is first opened**. 

There are many operations that are supported on files via the file descriptors, such as `read`, `write`, `sendfile`, `lock`, and `close`.
##### Inode

For each file, the VFS maintains a persistent representation of the files "indices" known as an **inode**. The inode (index node) basically maintains a list of all data blocks corresponding to the file. This is important as the file may not necessarily be stored contiguously on disk -- these block indices must therefore be tracked.

The inode also contains other relevant information for the file, such as the permissions associated, size, and other metadata. 
##### Dentry

We are used to the notion of files being organised within directories. However, from the VFS' perspective (and Unix-based systems in general), a directory is simply a file as well. These directory "files" contain information about the contain files as well as their inodes.

To help with the unique operations on directories, Linux maintains a data structure known as a **dentry** (directory entry). 
- Each dentry object corresponds to a single path component that is being traversed as we try to reach a particular file.
- For example, if a file within `/users/User` is accessed, the filesystem will create a dentry for `/`, `/users`, and `/users/User`.
- If another file within `/users/User` is required, there is no need to search and walk through the entire path length again. The filesystem maintains a **dentry cache** containing directories previously visited.
- Note that dentry objects are not persisted (soft state) -- they exist only in memory.
##### Superblock

This is akin to a map that the filesystem maintains so that it is able to figure out how it has organised persistent data elements (e.g., inodes, data blocks) on disk. In addition, each file system might maintain some additional metadata in the superblock; different file systems store different metadata.

Briefly put, the superblock abstraction provides information about how a particular filesystem is laid out on some storage device.
#### VFS Abstractions; Summary

The VFS data structures are **software entities**. They are created and maintained by the OS' filesystem component. Other than the dentries (which recall: are soft state), the remaining components will correspond to blocks that are actually present on disk. The figure below shows an arbitrary representation of this idea:

![[Pasted image 20231025135753.png]]
- Suppose that the blue and green blocks correspond to two files -- notice that they are not contiguously placed in disk.
- The inodes, represented in red, are also persisted within a block. In this case, two red inodes correspond to the blue and green files.
- To keep track of everything, the superblock is responsible for maintaining an overall map of a disks (and all disk blocks) on a particular device. This map keeps track of which blocks hold **data**, **inodes**, or are simply **free**. This map is used for both allocation and lookup.
#### VFS Abstractions; Case Study

In order to concretise the discussion thus far, the `ext2` filesystem will be examined. `ext2` stands for Extended Filesystem version 2, an older default filesystem on Linux until it was replaced by `ext4`. Note that `ext2` is **not Linux-specific**; it is available on other OSes as well.

A disk partition that is used as an `ext2` filesystem will look as follows:

![[Pasted image 20231025141856.png]]

The first block is not used by the operating system and is often used to contain code necessary for booting the computer. The rest of the partition is divided into groups -- the group sizes do not depend on physical characteristics of the disk (e.g., size of disk cylinders or sectors).

Each block group will be organised as shown in the bottom row:
1. The first block is the superblock, which contains information about the overall block group. For example, it will track the number of inodes, the number of disk blocks (within this block group), and the **start** of free block.

2. The overall state of the block group is further described by the **group descriptor**, which contains information about the _location_ of bitmaps, number of free nodes, and the total number of directories. 
	- The total number of directories is important as `ext2` tries to balance the overall allocation of directories and files across block groups.

3. Bitmaps are used to quickly find a free block or a free inode. Higher level allocators will be able to read bitmaps to quickly determine which blocks and inodes are free/in use.

4. The inodes are numbered from 1 to some maximum value. Each inode in `ext2` is a 128 B data structure that describes exactly one file. 
	- The inode will contain information such as file ownership, or other accounting information that system calls such as `stat` would return.
	- Of course, the inode will contain information on how to locate the actual data blocks.

5. Finally, the data blocks themselves are also contained within a block group.
	- A side note: directories are really just considered files, though dentries will be created higher up on the filesystem software stack. 
### Inodes

#### Basic Inodes

In this subsection, inodes will be considered in greater detail. After all, inodes play a key role in organising how files are stored on disk.

![[Pasted image 20231025143536.png]]

A file is said to be **uniquely identified by its inode**. The inode contains a list of all blocks (or their indices, technically) that correspond to the actual file. Recall that the inode also contains additional metadata information.

Consider the figure above, where a file has five blocks allocated to it. If more storage is needed, the filesystem simply allocates a free block, and updates the inode to contain a sixth entry, pointing to some newly allocated block (`30`, for example).

The upside of this simple approach is that it is **easy to perform both sequential and random accesses** to the file.
- Sequential access simply involves retrieving the next block index.
- Random access simply involves computing the necessary block reference based on the block sizes.

However, this approach severely limits the file sizes that can be maintained. For example, a 128 B inode containing 4 B block pointers can only ever address 32 blocks. If each block stores 1 kB worth of data, our overall file size limit is a mere 32 kB, which is much too restrictive.

#### Indirect Pointers

It is possible to overcome the file size limitation with **indirect pointers**.

![[Pasted image 20231025145034.png]]

Again, the inode maintains some metadata information, shown as the light blue rows near the top. We will mainly be focusing on the section with pointers to blocks.

The first section contains pointers that directly point to a block on disk that stores the file data. Using the same example as before, where each block pointer takes 4 B and the block size is 1 kB, the direct pointers will point to 1 kb per entry.

To extend the number of disk blocks that can be addressed via single inode element, while also keeping the overall inode size small, indirect pointers can be used.
- As their name suggests, instead of pointing to an actual data block, these pointers point to a block of more pointers.
- In the case of a single indirect pointer, it will point to a block of pointers that eventually point to their corresponding data blocks.
- Again, since each block size is 1 kB, and a pointer is 4 B large, a single indirect pointer can point to 256 kB worth of file content.
- A double indirect pointer can point to 64 MB (256 $\times$ 256 $\times$ 1 kB) worth of file content.

Indirect pointers therefore allow the use of relatively small inodes while still being able to address large files.

However, file access is slowed down. With direct pointers, two disk accesses are needed (at most) to get a block of content: one to access the inode, and another to access the data. With double indirect pointers, we effectively double this number: (i) inode access, (ii), pointer block (iii), another pointer block, and (iv) data.

_Quiz_
_An inode has the following structure. Each block pointer is 4 B._

![[Pasted image 20231025200942.png]]

_If a block on disk is 1 kB, what is the maximum file size that can be supported by this inode structure (nearest GB)?_
17 GB

_What is the maximum file size if a block on disk is 8 kB? (nearest TB)_
64 TB

_Use the formula:_
$$ \text{max}(\text{file size}) = N(\text{addressable blocks}) \times \text{size}(\text{block})$$
_The number of addressable blocks is basically the sum of all blocks addressable by direct pointers, as well as al the indirect pointers._
## Disk Access Optimisations

Filesystems use several techniques to try and minimise accesses to disk and to reduce file access overheads.
##### Buffer Caches

Similar to how hardware caches are used to hold recently used data (and to avoid accessing main memory), filesystems rely on **buffer caches**. Do note that these "caches" actually reside in main memory -- it is not as fast as hardware CPU caches, for example.

Content will be read and written to these buffer caches, and will be periodically **flushed to disk**. Filesystems support this operation via the `fsync` system call.
##### I/O Scheduling

The main goal of I/O scheduling is to **reduce disk head movement**, which is a slow operation. Simply put, I/O schedulers attempt to maximise **sequential accesses** over **random accesses**.

For example, if a disk head is at block 7, and two requests come in: (i) to write at block 25 and (ii) to write at block 17. The I/O scheduler will re-order these requests such that block 17 access takes place _before_ block 25.
##### Prefetching

There is often a lot of locality in how a file is accessed. Hence, cache hits can be increased by fetching nearby blocks during a request.

For example, if a read request comes in for block 17, a filesystem may also read blocks 18 and 19 into cache. This uses up more disk bandwidth (3X, in this example), though the idea is to reduce access latency by increasing cache hit rate.
##### Journaling

This serves more in tandem with I/O scheduling. With I/O scheduling, data is kept in memory while waiting for the I/O scheduler to interleave them in an appropriate manner. However, if the system crashes, these data blocks will be lost. 

In order to buffer against this potential loss of data, a quick update to some log is performed -- this is not writing data to disk, since we _still_ want the I/O scheduling to take place properly.
- This log will contain some description of the write that is waiting to take place.
- If the block, offset, and the value are specified, then an individual write is essentially describe. Note that this is an oversimplification for the purposes of discussion.
- The writes described within the log must be periodically applied to proper disk locations.



