## Introduction and Overview

Inter-process communication (IPC) refers to OS-supported mechanisms for interaction between processes (coordination and communication). IPC mechanisms are broadly categorised as **message-based** or **memory-based**.
- Message-based IPC abstractions include sockets, pipes, or message queues.
- Memory-based IPC is generally represented as shared memory. This could include completely unstructured pages of physical memory, or may also be in the form of memory-mapped files.

Higher-level semantics for IPC are provided via the filesystem or remote procedure calls (RPC). Both of these will be covered in a later lecture. 
- The term "higher-level semantics" here refers to the fact that these involve much more than a simple channel passing information between two processes. 
- These methods prescribe additional detail on the protocols to be used, how the data will be formatted, how the data will be exchanged, etc.

Finally, coordination and communication amongst processes must imply some form of **synchronisation**. Synchronisation between processes also falls under the IPC umbrella, though this topic will be given special attention in the next lecture.
## Message-Based IPC

### Overview

As the name implies, processes are able to create and send messages, or simply receive messages. The OS is responsible for creating and maintaining a communication channel, such as a buffer or a FIFO queue. The OS also provides the **port** interface to processes. Processes can therefore `read`/`recv` or `write`/`send` from/to a port.

![[Pasted image 20231018145936.png]]

Recall from brief introduction to IPC in [[P2L2 Threads and Concurrency]] that the OS here is responsible in establishing communication between the two processes, **as well as** in performing _each_ IPC operation.
- A `send` must involve a system call and copying of data from the processes to the kernel-level channel. A `recv` must also involve a system and copying of data from the kernel-level channel to the process' address space. 

In message passing IPCs, the overhead involved in crossing the user-kernel boundary is one of the clear downsides. However, the kernel abstracts away the management of the channel as well as any synchronisations involved. Message-based IPCs are much simpler to use in that regard.
### Message Passing Forms

#### Pipes

Pipes are characterised by two endpoints. In other words, only two processes can communicate via a pipe. There is no notion of a "message" with pipes; instead a stream of bytes is pushed from one process to another.

![[Pasted image 20231018150702.png]]

One popular use for pipes is to connect the output of one process to the input of another. The pipe operator is denoted by `|` on the command line.
#### Message Queues

A more complex form (relative to pipes) would be the message queue. Message queues understand the notion of a "message", hence sending processes must properly format a message before submitting it into the channel. 

![[Pasted image 20231018150947.png]]

The OS also allows for message priorities, or even scheduling of message delivery. The usage of message queues is supported through the (archaic) System V API or the POSIX API on Unix systems. 
#### Sockets

The notion of "ports" required in message passing are represented by the "socket" abstraction. The socket API exposes the `send` and `recv` calls to send message buffers in and out of the kernel-level communication buffer. In addition, the user will have to associate any kernel-level processing that needs to be performed alongside message passing. For example, the TCP/IP protocol can be specified, such that the entire TCP/IP protocol stack is associated with data movement in the kernel.

![[Pasted image 20231019093524.png]]

Socket-based communication can occur between processes on different machines. In this case, the communication buffer exists between the process and the network device that will ultimately send data out. Note that if sockets are used for message passing between processes on the same machine, the full protocol stack (e.g., TCP/IP) is often bypassed -- this is hidden from the user, of course.
## Shared Memory IPC

### Overview

In shared memory IPC, processes read and write to a **shared memory region**. The OS first establishes the shared memory region between processes.

![[Pasted image 20231019093853.png]]

- In other words, physical memory pages are mapped to the virtual address space of both processes.
- More concretely, VA(P1) and VA(P2) map to the same physical address, though note that the virtual addresses between processes are not necessarily equal.
- Physical memory itself does not have to be contiguous.
- All of this leverages the memory management support provided by the OS.

The immediately obvious benefit to shared memory IPC is that the OS is removed from the communication path once the initial setup has been performed. **Data copies are reduced** (no need to copy data from user to kernel), but not necessarily eliminated completely.
- For data to be available to both processes, the data actually needs to be explicitly allocated from the virtual addresses that belong to the shared memory region.
- If that is not the case, then data must be copied in and out of the shared memory region.
- Of course, if one process only needs to read certain parts of shared memory, there is no need to copy data in that scenario; data copying can thus be reduced.

However, shared memory IPC is much more complex to use.
- Firstly, explicit synchronisation is required, since multiple processes are accessing the same memory area at the same time.
- The responsibility of managing the communication protocol used, managing the shared buffer, etc. is now placed on the programmer. The OS only aids in the initial setup. 

Unix-based systems support two popular shared memory APIs, System V and POSIX. Shared memory IPC can also be established between processes via a memory-mapped file, which is analogous to using the POSIX shared memory API.

The Android OS using a form of shared memory IPC known as ashmem. This will not be covered, and is more of an FYI.

### Copy vs. Map

The end-result of both shared memory and message-passing IPC is the same: transfer data from one process' address space to another. However, which is "better" depends greatly on context.

In message-based IPC, the CPU *must* be involved in copying data to and from the kernel-level communication channel. There are overheads in terms of burning CPU cycles.

In memory-based IPC, the CPU _must_ be involved in mapping physical memory to the address spaces of processes. The CPU might also be involved in copying data to and from the shared address space, though it is definitely cheaper than user/kernel crossings.

The memory mapping setup is a costly operation, though if shared memory is used multiple times, this results in an overall good payoff.
- Even for one-time usage, the memory-mapped approach may also perform well.
- For example, if the data to be shared is large, the CPU time taken to copy data can **greatly exceed** the CPU time taken to map data.

In fact, Windows systems leverage this difference. If the data that needs to be transferred exceeds a certain threshold, then the memory-mapped approach is used. Otherwise, the data is simply copied in and out of a communication channel via a port-like interface. This mechanism is termed **Local Procedure Calls** (LPC). 
### SysV Shared Memory

#### Details

The OS supports **segments of shared memory**, which do not necessarily correspond to contiguous physical pages. The OS also treats shared memory as a system-wide resource. In other words, there are **system-wide policies** that limit the total number of segments, as well as their total size.
> The system-wide limit is not really an issue nowadays, since Linux currently imposes a generous limit of 4000 segments. In the past, however, this could be as low as 6 segments. Linux previously allowed only for 128 segments.

![[Pasted image 20231019102457.png]]
##### Create
When a process requests for the creation of a shared memory segment, the OS allocates the required amount of physical memory (provided that certain limits are met), then assigns it a **unique key**. Any other process may access this shared memory using this key.
##### Attach
Using the segment identifier (key), the shared memory can be **attached by a process**. In other words, the OS establishes a valid mapping between the virtual address of the calling process and the physical address that backs the segment. 

Multiple processes can attach to the same shared memory segment. Recall that the same physical memory address may be mapped to different virtual addresses in different virtual processes.
##### Detach
Detaching a segment simply involves **invalidating the address mappings** (i.e. addresses that correspond to a shared memory segment) within a process. Quite simply, the page table entries for these virtual addresses will no longer be valid. 

A segment is not truly destroyed upon detach. In fact, a segment may be attached, detached, and reattached any number of times. 
##### Destroy
Shared memory _must be explicitly destroyed_ in order to remove it (unless of course, a system reboot occurs). In comparison, dynamically allocated memory via `malloc` will disappear once a process exits.
#### SysV Shared Memory API

```c
shget(shmid, size, flag);
```
- This is used for shared memory creation. 
- The `flag` can be used to specify certain parameters during creation, such as permissions to read or to write.
- `shmid` is the unique integer key that identifies a shared memory segment, and must be supplied by the calling process. In order to generate this unique integer key, `shmget` is often called together with `ftok`.

```c
ftok(pathname, proj_id);
```
- It is possible to think of `ftok` as a hash function. `ftok` will always return the same key if the same arguments (e.g., same pathname) are provided.

```c
shmat(shmid, addr, flag);
```
- This is used for shared memory mapping or attachment.
- The caller is free to specify the virtual address to which shared memory should be mapped. Otherwise, if `NULL` is passed as the `addr` argument, the OS chooses some arbitrary virtual address.
- `shmat` returns a `void` pointer (i.e., it can be interpreted in arbitrary ways). It is therefore the programmer's responsibility to cast the return value appropriately.

```c
shmdt(shmid);
```
- This is used for shared memory detachment.

```c
shmctl(shmid, cmd, buf);
```
- This is actually an all purpose function that passes certain commands related to shared memory management to the OS.
- For shared memory destruction, the `cmd` is to be specified as `IPC_RMID`.
### POSIX Shared Memory

The POSIX shared memory API is more recent, and has been supported since the Linux 2.4 kernel. The POSIX API is considered the standard, though it might not be as widely supported as the SysV API.

The most notable difference is that the POSIX shared memory API uses files instead of segments. 
- These are not strictly equivalent to other files used by the OS.
- Rather, these exist solely within `tmpfs` (temporary file system). These "files" emulate an actual file -- the OS can then reuse similar mechanisms for files. 
- Quite simply put, the OS uses the same data structure and representation as it would for files, but to represent shared memory instead.
- In that case, there is no longer a need for the awkward use of a key for shared memory. File descriptors can be used.

The functions available simply have different names, but they follow the same workflow: create, attach, detach, destroy.
- `shm_open` returns a file descriptor associated with the created shared memory region.
- `mmap` and `munmap` performs attach and detach operations
- `shm_close` removes the file descriptor from the process' address space.
- `shm_unlink` destroys the shared memory.

## Synchronisation

When data is placed in shared memory, multiple processes can concurrently access and potentially modify the same region. Therefore, as is the case for multithreaded environments, accesses must be synchronised to prevent race conditions. 

There are multiple options for inter-process synchronisation. 
1. It is possible to rely on mechanisms supported by the process' threading library (e.g., `pthread`). 
2. The OS itself supports certain synchronisation mechanisms (though this will be covered in the next lecture).

Regardless of the method chosen, the method must be able to coordinate:
- the number of concurrent accesses to a shared segment (think mutexes), as well as,
- when data is available and ready for consumption (think condition variables/signals).
### Pthreads Sync for IPC

In [[P2L3 PThreads]], it was briefly mentioned that the scope of a mutex or condition variable could be defined during initialisation. More specifically, if a mutex/condition variable is to shared between processes, the `PTHREAD_PROCESS_SHARED` flag is to be used.

Importantly, **synchronisation data structures must be shared**. In other words, they must be placed within shared memory as well. This is analogous to how mutexes can be shared amongst threads by declaring it globally.

Consider the following rough code example:
```c
// make shm data struct
typedef struct {
  pthread_mutex_t mutex;
  char * data;
} shm_data_struct, * shm_data_struct_t;

// create shm segment
seg = shmget(ftok(arg[0], 120), 1024, IPC_CREATE | IPC_EXCL);
shm_address = shmat(seg, (void *) 0, 0);
shm_ptr = (shm_data_struct_t) shm_address;

// create and init mutex
pthread_mutexattr_t(&m_attr);
pthread_mutexattr_set_pshared(&m_attr, PTHREAD_PROCESS_SHARED);
pthread_mutex_init(&shm_ptr.mutex, &m_attr);
```
- Notice that the mutex attributes are first set to allow sharing between different processes.
- Then the mutex is initialised _within shared memory_. 
### Other IPC Sync

Shared memory access can also be synchronised by using OS-provided mechanisms. Notably, the `PTHREAD_PROCESS_SHARED` option is not supported on every platform. Other forms of synchronisation can therefore be leveraged, such as message queues or semaphores. 
#### Message Queues

Message queues can be used for synchronisation, since the process of sending/receiving messages are already synchronised by the OS.
- For example, process A can write data to shared memory and send a "ready" message into the message queue.
- Process B wait until a "ready" message is encountered, reads the data, then sends a "done" message back.
#### Semaphores

This construct is to be covered in greater detail in the next lecture. However, for the purposes of this lecture, note that binary semaphores (i.e. a semaphore that has only two values) is equivalent to a mutex.
- Suppose that a semaphore has a value of `0`. A process that encounters this semaphore will be blocked.
- If the semaphore has a value of `1`, the process is allowed to proceed after it has decremented the value to `0`.

_Quiz_
_For message queues, what are the Linux system calls that are used for the following?_

| Description | Answer |
| --- | --- |
| _send message to a message queue_ | `msgsnd` |
| _receive messages from a message queue_ | `msgrcv` |
| _perform a message control operation_ | `msgctl` |
| _get a message identifier_ | `msgget` |

## Addendum

This section contains information that are likely useful for Project 3. Only the command line tools section is included, since the latter is not really useful content to rewrite. I am copy-pasting the writeup from omscs-notes for the latter section.
### IPC Command Line Tools

`ipcs` lists all IPC facilities. The `-m` flag specifies shared memory IPC only.

`ipcrm` is used to delete an IPC facility. The `-m <shmid>` flag deletes a shared memory segment with the given `shmid`.

### Design Considerations

Let's consider two multithreaded processes in which the threads need to communicate via shared memory.

First, consider how many segments the processes will need to communicate.

Will you use one large segment? If so, you will have to implement some kind of management of the shared memory. You will need some manager that will be responsible for allocating and freeing memory in this region.

Alternatively, you can have multiple segments, one for each pairwise communication. If you choose this strategy, it is probably a smart idea to pre-allocate a pool of segments ahead of time, so you don't need to incur the cost of creating a segment in the middle of execution. With this strategy, you will also need to manage how these segments are picked up for use by the pairs. A queue of segment ids is probably sufficient.

Second, you will need to think about how large your segments should be.

If the size of the data is known up front (and is static), you can just make all your segments that size. Typically an operating system will have a limit on the maximum segment size, so this strategy only works in very restricted cases.

If you want to support arbitrary messages sizes that are potentially much larger than the segment size, one option is to transfer the data in rounds. The sending process sends the message in chunks, and the receiving process reads in those chunks and saves them somewhere until the entire message is received. In this case, the programmer will need to include some protocol to track the progress of the data movement through the shared memory region.
