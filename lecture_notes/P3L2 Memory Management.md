## Introduction and Overview

Recall from [[P1L2 Introduction to Operating Systems]] that the OS manages access to physical resources on behalf of executing processes. In this section, memory (or DRAM) will be considered. 

Also recall from [[P2L1 Processes and Process Management]] that processes are represented as a range of virtual addresses. The OS maintains a way to translate these virtual addresses (that are not distinct between processes) into physical addresses that **must be distinct** between processes. 
> Of course, this does not account shared memory for inter-process communication for the sake of discussion.

The range of virtual addresses presented to the process is not representative of the actual amount of physical memory present -- in fact, it is likely that the virtual address range overstates the amount of physical memory available. The OS must therefore **allocate** physical memory accordingly and **arbitrate** how memory is being accessed.

1. Allocation
	- The OS must incorporate certain mechanisms and data structures so that it can track free or used memory. 
	- Since physical memory is smaller than the range of virtual memory addresses, it is likely that some needed content is instead found in some secondary storage (e.g., disk). The OS then has to be able to decide on how to **replace** contents currently in physical memory with requested memory residing in disk.
2.  Arbitration
	- The OS must be able to quickly interpret a memory access. This involves translating a virtual address into a physical address, and then validating it to verify that a legal access occurred. 
	- Modern OSes rely on a combination of hardware support and software implementations to accomplish this
### Pages and Segments, Briefly

Pages and segments will be covered in much greater detail in its own subsection below, though these terms must be first introduced in order to make sense of the next subsection, "Hardware Support".
#### Pages, Briefly

The virtual address space is divided into fixed-sized segments called **pages**. Physical memory is also divided into **page frames** of the **same size**. The OS maps pages onto page frames (allocation), and keeps track of such allocations using a **page table** (arbitration). This is also known as **page-based memory management**.

Paging is the dominant memory management mechanism used in modern OSes.
#### Segments, Briefly

Fixed-size pages are not used. Rather, flexibly-sized segments are used and mapped to physical memory (allocation). Such allocations are tracked using **segment registers** (arbitration) that are typically supported on modern hardware. This is known as **segment-based memory management**.
## Hardware Support

A quick aside: memory management is rarely performed by only the OS. In fact, hardware support enables memory management to be performed easier, faster, and be more reliable.

Every CPU package is equipped with a **memory management unit** (MMU). 
- The CPU issues virtual addresses to the MMU, and the MMU is responsible for translating those into physical addresses. 
- The MMU may also generate a **fault**. This might occur if illegal access occurred (memory requested not allocated) or perhaps some issues with permission (e.g., writing to a write-protected location).
- A fault may also occur if the page requested is not currently in memory and must be fetched from disk. This will be considered in greater detail in the later subsections.

Hardware can further support memory management by assigning designated registers to aid with address translation.
- For example, registers can point to the currently active page table (paging)
- Alternatively, registers can point to the the base address, size of segment, and total number of segments (segmentation)

Since memory address translation must occur on every memory reference, most MMUs incorporate a small cache of virtual-to-physical translations. This is termed the **Translation Lookaside Buffer** (TLB). 

Clearly then, the **hardware is the entity responsible** for performing the virtual-to-physical address translations. The OS simply maintain certain data structures to aid this process (e.g., page tables). 
- It follows that the hardware dictates the supported memory management mode -- page-based or segment-based.
	- This should be obvious, since the hardware provides the relevant registers.
- The hardware also possibly dictates the format of virtual/physical addresses, since it will eventually need to understand both.

This is not to say that all of memory management is the sole responsibility of hardware. 
- Software implementations are allowed to dictate how memory is allocated, or which replacement policies to follow.
- Since this is an OS course, discussions will mainly focus on _software aspects_ of memory management.
## Pages

### Page Tables

Page tables are analogous to "translation tables", wherein they are used to convert virtual memory addresses to physical memory addresses. 

The OS creates a page table for every process. In other words, whenever a context switch occurs, page tables must be saved and restored. 
- Also recall that the hardware is roped in to assist with memory management. 
- This would mean that that the hardware maintains some register that points to the currently active page table. On x86 platforms, this register is termed the **CR3 register**.
- During a context switch, this register (CR3, for instance), must be updated as well.

The virtual memory **pages** and physical memory **page frames** are of the same size. 
- This allows translation of addresses *within* a page to be less complex. 
- Since the virtual page size corresponds to the physical page frame, any virtual address within that particular page can be found by simply computing the starting virtual address **added with some offset**. 
- As a result, the actual page table itself does not have to store all virtual addresses, rather only virtual addresses that mark the start of a page.
- More concretely, the virtual address itself actually consists of a **virtual page number** and an **offset**. The virtual page number can be translated to a **physical frame number**, which can then be summed with the offset to produce the actual physical address. 

![[Pasted image 20231012202508.png]]

Tracing an example:
1. Suppose an array is first initialised. Memory is allocated within the virtual address space of the process.
2. If the array is accessed, the OS first realises that there is no physical memory that corresponds to the virtual address. It will then proceed to take a free physical page and update the page table accordingly.

Notice then that the OS only allocates physical memory when it required (i.e., when the process is trying to access it). This is known as **allocation on first touch**. This allows physical memory to only ever be allocated when absolutely required.

In the same manner, if a process has not used a certain memory page for a long time, it is possible that the pages will be **reclaimed**. The contents will be swapped out to disk, and the OS may place some other content into the same physical page.

Clearly, the page table must be able to track if the mappings are valid or otherwise. 
### Page Table Entries

#### Flags/Bits

Page table entries (alongside the virtual page number and offset) will therefore also contain a set of bits that provide more information regarding the validity of access. 
- For example, a **valid bit** will be set to `1` if the page is within physical memory, and `0` otherwise. This is also known as the **present bit**.
- This set of bits is to be used by the MMU. If the MMU notices that the valid bit is set to `0` while a memory access is occurring, it will raise a fault and trap to the OS. 
- The OS will then decide if access should be permitted -- this naturally means that the corresponding page should be brought back into physical memory. 
- A direct effect of this would be that the page table will have to be updated with the new mapping. 

> A quick intermission. These bits are used by both the OS and the hardware, hence they should both understand what each of the bits mean.

Most hardware support a **dirty bit** that is set whenever a page is written to. 
- This is particularly useful in file systems, where files are cached in memory. 
- This dirty bit will then detect which files have been written to (which one changed?) and therefore will need to be updated on disk.

It is also useful to keep track of an **access bit**.
- This is quite simply used to track if a page has been accessed, be it for reading or for writing. 

Other protection bits are also maintained, which should be rather familiar:
- R, W, and X bits for read, write, and executable permissions respectively

![[Pasted image 20231012204852.png]]

The figure above shows a blueprint for a page table entry on Pentium x86 systems.
1. P = present (valid bit)
2. D = dirty 
3. A = accessed
4. R/W = single permission bit. `0` refers to read-only while `1` refers to both read and write permissions.
5. U/S = defines if the page can be accessed only through supervisor mode (`1`) or on user mode (`0`).
6. The other bits maintain caching related info (e.g., caching disabled)
7. There are some parts that are unused in the event there are some other bits that prove useful in the future (future-proofing).
#### Page Faults

The bits allow the MMU to establish the validity of memory access, alongside its translation responsibility. As briefly alluded to above, when the MMU decides that a memory access is invalid, it will generate a fault and trap into the OS kernel. The **page fault handler** is invoked.
- If the page is simply not in physical memory, then the OS can simply handle this fault as described above.
- If a permission protection was violated (as informed by the page table entry flags), then `SIGSEGV` might be generated.

On x86 platforms, the error code is generated from the flags in the page table entry and the faulting address is stored in the **CR2 register**.
### Page Table Size

1. The size of a page table would be equal to the size of a single page table entry, multiplied by the number of entries within the table.
$$ \text{size}(\text{page table}) = N(\text{entry}) \times \text{size}(\text{entry}) $$
2. The number of entries within a page table is equal to the number of virtual page numbers that exist in a virtual address space.
$$ N(\text{entry}) = N(\text{virtual page numbers}) $$
3. The number of virtual page numbers is equal to the total number of addressable memory (i.e. all possible addresses) divided by the size of each page. 
$$ N(\text{virtual page numbers}) = N(\text{possible addresses}) / \text{size}(\text{page}) $$
4. The number of virtual page numbers _as well as_ the size of each page table entry depends on the architecture used. 
$$ \text{size}(\text{entry}) = \text{bits}(\text{architecture}) $$
$$ N(\text{possible addresses}) = 2^{\text{bits}(architecture)} $$

Taking all four formulas together, the total size of a page table is therefore:
$$ \text{size}(\text{page table}) = 
\frac{2^{\text{bits}(\text{architecture})}}{\text{size}(\text{page})} \times \text{bits}(\text{architecture})$$

The size of a page is **commonly defined as 4 kB** (or $2^{12}$ bytes).
> 4 kB is equivalent to 4096 (or $2^{12}$) bytes

Suppose we have a 32-bit architecture. The total size of the page table is $2^{22}$, or 4 MB.
$$ \text{size}(\text{page table}) = \frac{2^{32}}{2^{12}} \times 4 $$
> 32 bits is equivalent to 4 bytes (8 bits = 1 byte)

Suppose we have a 64-bit architecture instead. The total size of the page table comes up to 32 PB, an obviously unrealistic size to use, especially considering that _each process has its own page table_.
$$ \text{size}(\text{page table}) = \frac{2^{64}}{2^{12}} \times 8 $$

Now recall that a process rarely ever uses the entire address space. In other words, it is very unlikely that all possible virtual addresses are used. 
- For example, on 32-bit architecture, it is unlikely that all possible virtual addresses (4 GB worth) will be used.
- However, the page table assumes an entry **per VPN**, regardless of whether the corresponding virtual memory is needed or not. 
- In subsequent sections, we will examine how the page table size can be greatly reduced with some careful design.
#### Common Page Sizes

In the examples above, a page size was assumed to be 4 kB. This corresponds to an offset of 12 bits.
> With 12 bits, there are $2^{12}$ possible combinations -- the offset can effectively "refer" to $2^{12}$ different positions. 
> Therefore, with a single page frame number and the $2^{12}$ different possible values of the offset, the page size is also $2^{12} = 4096$ bytes, or 4 kB.

In practice, different systems support a multitude of different page sizes. For example, Solaris 10 on SPARC machines support 8 kB, 4 MB, and 2 GB page sizes. Linux on x86 platforms support page sizes of 4 kB, 2 MB, as well as 1 GB.

| | regular | large | huge |
| --- | --- | --- | --- |
| page size | 4 kB | 2 MB | 1 GB |
| offset bits | 12 | 21 | 30 |
| reduction factor | NA | x512 | x1024 |

One benefit of using larger page sizes would be that more physical addresses can be mapped to one page table entry. 
- This results in **fewer page table entries**, which directly leads to **smaller page tables**. For a page size of 1 GB, for example, the page table size is reduced by 1024 times compared to that of a 4 kB page size.  
- The reduction of page table entries also means that more TLB hits are likely to occur.

However, large page sizes will lead to **internal fragmentation**.
- If a large memory page is not densely populated, there will be wasted, unused gaps within allocated memory.

_Quiz_
_On a 12-bit architecture, what are the number of entries in the page table if the page size is 32 bytes? What about 512 bytes? (Assume single-level page table)_

32 byte page size = 128 entries
512 byte page size = 8 entries
### Page Table Design

#### Hierarchical Page Tables

##### Basic

The issue with size outlined in the previous section is not really an issue in modern implementations, as we have moved away from "flat" page tables, to a more **hierarchical, multi-level structure**.

Consider the following two-level page table:

![[Pasted image 20231016211050.png]]

The outer level is also referred to as a **page table directory**. Its elements are not direct pointers to page frames in memory, rather pointers to page tables within the inner layer.

The inner layer contains multiple page tables that point to page frames in physical memory.
- Entries of these page tables contain the page frame number and the set of bits that determine validity of memory access.
- These page tables **only exist** for virtual memory regions that are valid. 
- The figure above depicts gaps between page tables within the inner layer -- these correspond to gaps in the virtual address space.

If a process requests more memory via `malloc`, the OS will potentially create another inner page table for the process and at the same time create a new entry within the page table directory. 

In order to access the correct address within physical memory, there is now a need to index into the outer page table as well as the inner page table. Finally, some offset is added to obtain the correct physical memory address. The following figure shows this concept:

![[Pasted image 20231016211830.png]]

The virtual address (also called logical address, as depicted) now consists of three parts. The first index $p_1$ is used to index into the outer page table. The second index $p_2$ is used to index into the inner page table. The inner page table will produce the **physical frame number** and this must be added with the offset $d$ to compute the physical address.

Suppose that in this implementation, 12 bits are used for storing the index $p_1$, while 10 bits are used for index $p_2$ and the offset $d$. 

| $p_1$ | $p_2$ | $d$ |
| ---- | ---- | ---- |
| 12 | 10 | 10 |

Every single internal page table can therefore address $2^{10} \times 2^{10}$ addresses, which comes up to 1 MB. 
> Since there are 10 bits used for $p_2$, this would mean that there are $2^{10}$ possible combinations of bits. 
> There are also 10 bits used for the offset $d$, which means that the total possible addresses would be $2^{10}$ from above multiplied by $2^{10}$ from the possible values of $d$ (this is the page size).

This also means that every gap of 1 MB can see the omission of the corresponding page table, allowing one to reduce the overall size of the page tables.
> I rationalise this as follows: each page table **must** handle all the possible addresses that it is allowed to. However, if there is a gap of 1 MB, that page table need not exist in the first place.

##### Extending Levels

Clearly, from the example given above, hierarchical page tables reduce the space requirements for a page table. This scheme can be extended to use additional layers. 
- For example, a can be added at the front, which consists of pointers to page table directories.
- Another layer can still be added to the front, which would be a map of pointers to page table directories.
- This technique is **especially important on 64-bit architectures**. 
	- As we have seen, the space requirements for a page table is much larger on 64-bit architectures.
	- At the same time, virtual addresses are more sparse, which would require the ability ignore gaps.

Adding more levels would result in smaller internal page tables or directories. As a result, it is more likely that the gaps within the virtual address spaces will match that granularity, and space can be saved.

However, more levels would also mean more memory accesses required for translation. **Translation latency** is said to be increased.

_Quiz_
_A process with 12 bit addresses has an address space where only the first 2 kB and the last 1 kB are allocated and used.
How many total entries are there in a single-level page table that uses Address Format 1? How many total entries are there in a 2-level page table that uses Address Format 2?_

_Address Format 1_

| $p$ | $d$ |
| --- | --- |
| 6 | 6 |

_Address Format 2_

| $p_1$ | $p_2$ | $d$ |
| --- | --- | --- | 
| 2 | 4 | 6 |

The total entries using Address Format 1 would be the total number of pages, $2^6$.


The total entries using Address Format 2, each page table would only have $2^4$ pages. The granularity of the inner page tables are 1 kB:
$$ 2^4 \times 2^6 = 1024 $$
Therefore the answer is $3 \times 2^4$ since only 3 kB of the process' virtual address space is being used. 
##### Speeding Up Translation; TLB

In the simple single-level page table design, a memory reference will actually require _two_ memory references:
- One to access the page table entry, and
- One more to access physical memory

In the four-level page table design, a memory reference will require _five_ memory references:
- Four to access page table entries to finally produce a physical frame number (PFN), and
- One more to access physical memory

The standard method to speed-up this process would be to use a **page table cache**. 
###### #DEFINE Translation Lookaside Buffer (TLB)
> On most architectures, the MMU integrates a **hardware cache for address translations**. This cache is known as the Translation Lookaside Buffer.

On each address translation, the TLB is first referenced. If a hit is encountered, then the multiple memory references can be skipped. Otherwise, the full lookup will still have to be performed.
- In addition to address translation, the TLB entries must also contain all necessary protection/validity bits to ensure that memory access is allowed.
- The TLB does not have to necessarily cache a large amount of translations, as memory references usually have high temporal and spatial locality.

On modern x86 platforms (core i7), there are multiple TLBs:
- Per core, there is a 64-entry data TLB as well as a 128-entry instruction TLB.
- There is also a second-level 512-entry TLB that is shared amongst all cores.
- These were deemed sufficient to address the typical memory access needs of processes today.
#### Inverted Page Tables

##### Basic

In the page table implementations encountered thus far, each process has its virtual address space -- the sum of all virtual addresses greatly outweigh the actual amount of physical memory available. It might make more sense to have a virtual memory representation that is closer to the order of physical memory. 

###### #DEFINE Inverted Page Tables
> Inverted page tables are managed on a **system-wide basis and not on a per-process basis**. Each entry in the inverted page table points to a frame in physical memory.

Logical addresses take on a slightly different form. These addresses now contain an additional **process ID** field, $pid$, alongside the first part of the virtual address, $p$, and the offset, $d$.

![[Pasted image 20231017185340.png]]

When the appropriate $pid$ and $p$ are found within the page table, the physical frame number will be combined with the offset $d$ to give the corresponding physical memory address.
- However, note that the page table entries correspond to physical memory addresses, and these physical addresses can be arbitrarily assigned to any process. 
- Therefore, the page table entries themselves do not contain any meaningful groupings or patterns.
- The table is **not ordered** and this necessitates a linear search for a $pid$ and $p$ match within the page table. 
- Of course, some caching with the TLB might speed up this process, but there will eventually be a need to perform a linear search regardless. 
#### Hashing Page Tables

These are page tables in their own right, though note that inverted page tables are often supplemented with hashing page tables to speed up the linear search mentioned above.

![[Pasted image 20231017190159.png]]

The hashing page table (as its name likely implies), computes some hash value from the first part of the virtual address. The hash value will map to some linked list of possible matches. Once the actual match is found, the physical address is computed in a similar manner as the other page tables.

The hashing into a smaller search space (just some linked list of possible matches) results in a speed up of search time. 
## Segmentation

Virtual to physical memory mappings can also be maintained using segments. The address space is divided into components of **arbitrary size** here, and these components usually correspond to some logically meaningful section of the address space (e.g., code, heap, stack, etc.).

The logical address in segmented memory includes a **segment selector** and an offset. The selector maps to some physical address and translation is performed with a **descriptor table**. The physical address is combined with the offset to describe the linear address of a memory location.

![[Pasted image 20231017201704.png]]

In its purest form, a segment could be with a contiguous portion of physical memory. In this case, the segment size is exactly equal to the segment base added with the limit register.

###### #DEFINE Base Register
> The base register specifies the **smallest legal physical memory address**.

###### #DEFINE Limit Register
> The limit register defines the **size (range) of legal physical memory addresses**. For example, if the base register is 1000 and the limit register is 200, the program is only allowed to access addresses from 1000 to 1200.

In practice, **segmentation and paging are used together**. The linear address produced by segment lookup is then passed to the paging unit to give a physical address.

Note that the type of address translation possible on a specific platform depends on the hardware. 
- Intel x86_32 platforms support segmentation and paging. Linux on these platforms allows up to 8000 segments per process and another 8000 global segments
- Intel x86_64 platforms support segmentation and paging for backwards compatability, though the default mode is paging.
## Memory Allocation

So far, we have discussed how the OS controls access to memory, though we have glossed over how memory is even allocated in the first place. This would be the responsibility of the **memory allocator** within the **memory management subsystem** of an OS.

In brief, memory allocation must determine the virtual address to physical address mapping to use. Subsequent memory access will simply follow address translation as discussed in the prior sections.

Memory allocators can be divided into (i) kernel-level allocators as well as (ii) user-level allocators.
1. Kernel-level allocators
	- Responsible for allocating pages for kernel use and also for certain **static** states of processes when they are created (e.g., code, stack, data). 
	- The kernel-level allocators are also responsible for keeping track of the free memory that exists in the system.
2. User-level allocators
	- Used for dynamic process state; the heap, for example. 
	- The basic syscalls are `malloc` and `free`. Once the kernel allocates memory through a `malloc` call, the kernel is no longer involved in management of that particular memory. 
	- There are a number of different user-level allocators, `dlmalloc`, `jemalloc`, `hoard`, `tcmalloc`, etc. that differ on certain criteria (e.g., cache efficiency). This course will not go through these details.

In the subsections below, the basic mechanisms used in the kernel-level allocators will be covered. Some design ideas are also used in the some user-level allocators outlined above.
### Memory Allocation Challenges

Before delving into kernel-level memory allocators, consider the following problem.
Suppose there are 16 page frames to allocate. Requests for page frames come in the order `alloc(2)`, `alloc(4)`, `alloc(4)`, and finally `alloc(4)`. Let each distinct colours represent the different allocations associated with each `alloc` call.

![[Pasted image 20231018093801.png]]

Suppose that the two page frames initially allocated were released by calling `free(2)`. At the same time, a new request for four page frames arrives, `alloc(4)`.

![[Pasted image 20231018093933.png]]

There are four available page frames, though the allocator cannot fulfil the `alloc(4)` request as these four pages are not contiguous. This example illustrates a problem termed **external fragmentation**.

###### #DEFINE External Fragmentation
> An event where non-contiguous holes of free memory are available, but requests for large contiguous blocks of memory cannot be satisfied.

Consider the following allocation strategy instead, where each `alloc` call is allowed four page frames, regardless of whether or not it occupies the page frames fully. In this case, the same order happens: `alloc(2)`, `alloc(4)`, `alloc(4)`, and finally `alloc(4)`.

![[Pasted image 20231018094237.png]]

When the two page frames are freed with `free(2)`, there is now contiguous space for the next `alloc(4)` call to be handled successfully.

![[Pasted image 20231018094424.png]]

Notice that in this strategy, the allocator is allowed **aggregates/coalesces** free areas into one larger free area. In this manner, the allocator is more likely to allocate for future requests. Basically, an allocator must be able to:
- Limit the extent of external fragmentation and to
- Allow for quick coalescing of free areas.
### Linux Kernel Allocators

The Linux kernel relies on two main allocators: the buddy allocator and the slab allocator to address the considerations mentioned above. 
#### Buddy Allocator

The buddy allocator starts with some consecutive, free memory region that is to a power of two, $2^x$. Whenever a request is received, the allocator repeatedly halves the area into until a small enough chunk is found that can satisfy the request. Of course, all halving steps result in chunks that are also powers of two. 

Looking at an example to trace:

![[Pasted image 20231018095209.png]]

1. First, a request for 8 units arrives. The allocator halves the 64 unit chunk repeatedly, picking an arbitrary half to recurse on. It finally encounters a chunk of size 8 and allocates that appropriately.
2. A second request for 8 units arrives. There is already such a chunk, and memory is thusly allocated.
3. A request for 4 units arrives. An 8-unit chunk is halved into two fours. The memory is allocated.
4. When an 8-unit chunk is released, some fragmentation is present. 
5. However, once both 8-unit chunks are released, the free space can be coalesced to form a 16-unit chunk. 

Clearly, fragmentation still exists within the buddy allocator. However, upon each `free` call, a chunk can check with its "buddy" to see if they are both free. If they are, they can aggregate into a larger chunk. In fact, this aggregation can proceed up the tree, until all freed buddy pairs have been aggregated.

Chunks having sizes that are powers of two speeds up computation of the buddy's address. Two buddies will have addresses that differ only by one bit.
> For example, if my buddy and I are each 4 units long, and my starting address is 0x000, my buddy's starting address must be 0x100. 

The fact that allocations with the buddy allocator must be made in a granularity of $2^x$ means that some internal fragmentation exists. This is an issue, since many data structures in the Linux kernel have sizes that are not close to a power of two. For example, the `task` data structure is 1.7 kB.
#### Slab Allocator

To fix the aforementioned internal fragmentation issue with the buddy allocator, Linux also leverages the slab allocator.

The slab allocator builds **custom object caches** on top of "slabs", which are just contiguously allocated physical memory. When the kernel starts, it will pre-create caches for certain objects, such as the `task` struct, or directory entry objects.

![[Pasted image 20231018101623.png]]

When an allocation request occurs, in element in the cache will be immediately used. If none of the cache entries are available, the kernel will allocate another slab, pre-allocating another portion of continuous memory to be managed by the slab allocator.

Internal fragmentation is avoided since the entities within the slab are the exact size of the objects being stored in them. External fragmentation is avoided as well, since only objects of a given size will be stored in a particular slab -- there will never be any gaps within a slab.
## Demand Paging

### Broad Overview

Since physical memory is much smaller than the addressable virtual memory, allocated pages do not always have to be present within physical memory. Instead, the backing physical page frame can be repeatedly saved and restored to/from some secondary storage such as disk.

This process is known as **paging** or **demand paging**. Traditionally, with demand paging, pages are moved in/out of memory and a **swap partition** (e.g., on disk). A swap partition can also sit on some flash device, or it may even sit within the memory of another node.

![[Pasted image 20231018110241.png]]
1. Suppose a program requests for a specific memory address. If the associated page is not present within memory, its **present bit** will be set to 0. 
2. The MMU will raise an exception (page fault) and control will be trapped to the operating system. 
3. At that point, the kernel can determine that a page fault has occurred. It also knows that it has previously swapped out the page to a secondary storage device and thus must issue an I/O operation to retrieve this page.
4. Once the page is brought into memory, the OS will locate some free (typo in figure) frame where this page can be placed.
5. Note that this new location is likely different from the original physical frame number associated within the page table. The page table must therefore be updated with the new mapping.
6. Control is finally handed back to the process that issued the initial reference. Its program counter will be restarted with the same instruction and the exact same reference is made again. This time, however, the reference will succeed.

Occasionally, we may require a page to be **constantly present within memory**, or **maintain its original physical address** throughout its lifetime. In this case, we are allowed to **pin** the page -- disabling swapping. This is particularly useful when a CPU interacts with devices that support Direct Memory Access (DMA).
### Considerations

_When should pages be swapped out of main memory an on to disk?_

This question is rather straightforward to answer. Periodically, when the amount of occupied memory crosses a certain threshold, the OS will run some **page(out) daemon** that will look for pages that can be freed. Therefore, pages should be swapped out:
- When memory usage is above a certain threshold (high watermark)
- When CPU usage is below a certain threshold (low watermark), so as to not disrupt the execution of other intensive CPU-bound tasks.

_Which pages should be swapped out?_

The answer to this question is deceptively simple -- simply swap out pages that will not be used again in the near future. 

A common policy to determine if a page should be swapped out is the **Least Recently Used** (LRU) policy. This policy uses the **access bit** available on modern hardware to keep track of whether or not the page has been referenced.
- If a page has been more recently referenced, it is more likely to be referenced again in the immediate future.

Other viable candidates include pages that do not need to be written out to disk. Writing pages to secondary storage takes time, thus this overhead should be avoided wherever possible. In order to assist in making this decision, the OS keeps track of the **dirty bit** maintained by the MMU hardware to determine if a page has been modified.
> Not too sure why this is phrased in this manner.
> All this is saying is that writing to secondary storage (to reflect any modifications) takes time. The dirty bit informs is if writing to disk is necessary.
> I believe then, if writing to disk is necessary, the page should _not_ be swapped out unless absolutely necessary. If has not been modified at all, go ahead and remove it from physical memory.

On the flipside, there are pages that may be non-swappable. This might include pages used for kernel state or I/O operations. It is important to ensure that these pages are not considered by the swapping algorithm used.
### Case Study

In Linux and most OSes, a number of parameters are available to configure the swapping nature of the system. This includes the **threshold page count** that determines _when_ pages are to be swapped out. It is also possible to specify how many pages are to be replaced in a given period of time.

Linux categorises pages into different types (e.g., claimable or swappable), which helps inform the swapping algorithm as to which pages can be replaced.

Finally, the default replacement algorithm used in Linux is a "second chance" variation of the LRU policy. Two scans a performed before determining the pages that should be swapped out.

_Quiz_
_Suppose you have an array with 11 page-sized entries that are accessed one-by-one in a loop (see below). You have a system with 10 pages of physical memory. What is the percentage of pages that will need to be demand paged using the LRU policy?_

_Assume that the 11 page-sized entries are accessed then manipulated as follows:_
```c
int i = 0;
int j = 0;

while(1) {
    for(i = 0; i < 11; ++i) {
        // access page[i]
    }

    for(j = 0; j < 11; ++j) {
        // manipulate page[i]
    }
    break;
}
```

The answer is 100%. At the start, the first 10 entries are loaded into memory. As the 11th entry is accessed, there is a need for demand paging. By the LRU policy, entry 1 is to be removed.

Unfortunately, entry 1 is manipulated next. Again, by LRU policy, entry 2 is to be removed. This continues on as next-required entries are removed immediately prior to their actual manipulation, resulting in the need for demand paging all 100% of pages.

This is a rather contrived scenario, but this demonstrates how an intuitive policy such as LRU can perform very poorly in certain situations. As such, it is important for the OS to allow for flexible-enough mechanisms to support a variety of policies.
## Addendum

#### Copy On Write

In this lecture, we have encountered how the MMU hardware is leveraged to perform address translation, track memory accesses, and enforce the necessary memory protection. The same hardware can be used to build other useful services as well.

On such mechanism is termed Copy-On-Write (COW). When a process is created, there is a need to re-create the entire parent process by copying over its _entire_ address space. However, many such pages are static -- they will not change. 

In order to avoid unnecessary copying, the new process' virtual address space (or portions of it, at least) is **mapped to the original page**. Of course, the original page must be write protected. 

![[Pasted image 20231018121051.png]]

- If the new process only *reads* the page, then memory as well as the CPU cycles wasted on copying will be saved.
- Otherwise, if a _write_ request is issued, the MMU will raise a page fault. The OS will at this point create a copy of the memory page and. update the page table of the faulting process accordingly. Notice then that only pages that need to be updated (and only by processes that require writing) will be copied.

This mechanism is called Copy-On-Write simply because the cost for copying is paid only when a write request occurs.
### Failure Management Checkpointing

Another useful operating system service that can benefit from hardware supported memory management is **checkpointing**.
###### #DEFINE Checkpointing
> A failure and recovery management technique.

Checkpointing basically involves periodically saving the process state. A process may fail, though we can always restart the process from a recent state instead of having to reinitialise it.

A simple approach to checkpointing would be to simply pause and copy the entire state of an executing process. Needless to say, a better approach would be to try and reduce the disruption of the process that is inherent with checkpointing. 
- As the process continues executing, it will continue to dirty pages. These dirtied pages can be tracked (with hardware MMU support) via the **dirty bit**.
- We will then only have to copy the **diffs** on pages that have been modified. 
- This form of incremental checkpointing will make the recovery processes slightly more complicated, but it is much faster than copying the entire process state.

This basic checkpointing mechanism can be used in other services. 
1. Debugging often relies on a technique known as **rewind-replay**. 
	- "Rewind" involves restarting the execution of a process from some earlier checkpoint.
	- "Replay" involves literally replaying execution from that particular checkpoint to see if the error can be reproduced.
	- Of course, we can step back through various checkpoints until the error is encountered.

2. Migration also benefits from checkpointing (or at least, the mechanisms used).
	- Migration involves moving the checkpointed process to another machine, then restarting execution from there.
	- This is especially important during **disaster recovery**, or during **consolidation** efforts when we try to condense computing resources into as few machines as possible (e.g. for data centers)
	- Migration can be implemented by repeating checkpointing in a fast loop until the minimal dirtied pages results in acceptable pause-and-copy, or until pause-and-copy is simply unavoidable.

_Quiz_
_Which one of these endings correctly completes the following statement?_
_"The more frequently you checkpoint..."_
- the more state you will checkpoint (TRUE)
- the higher the overheads of the checkpointing process (TRUE)
- the faster you will be able to recover from a fault (TRUE)


