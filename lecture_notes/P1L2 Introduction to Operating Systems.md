## Definition

###### #DEFINE Operating System
> An operating system (OS) is a layer of systems software that:
> - Has direct, privileged access to the underlying hardware
> - Hides hardware complexities
> - Manages hardware on behalf of one or more applications according to some predefined policies
> - Ensures that applications are isolated and protected from each other


Examining each term one-by-one:

1. Direct, privileged access to hardware
   - The OS is a **layer** that sits between the hardware components and higher level applications.
   - Naturally, the OS would have to access hardware such as the Central Processing Unit (CPU), physical memory, network interfaces (Ethernet/WiFi Card), GPU, or storage devices (Disk) 

2. Hides hardware complexities
   - Hardware details should be abstracted away from the higher level applications. If necessary, the applications should interface with the hardware via the OS (through system calls -- we'll see that later).
   - For example, the _file_ abstraction comes with operations such as _read_ or _write_, which applications can interact with. Applications need not handle memory management and storage by themselves. 

3. Manages hardware based on predefined policies
   - The OS should control usage of CPU, memory, or peripheral devices by the other software applications.
   - This might include ensuring fair resource access, scheduling of resources, or even limits to resource usage by any given application.
   - This also entails clean-ups, after an application has finished with a specific resource.

4. Ensures isolation and protection between applications
   - For example, applications should not be able to access or overwrite each other's memory. 

*Quiz: Likely Components of an Operating System*

| component     | answer | description                                                                |
| ------------- | ------ | -------------------------------------------------------------------------- |
| File system   | True   | Abstracts "disk storage" as "file" objects                                 |
| Device driver | True   | Makes decisions regarding hardware usage                                   |
| Scheduler     | True   | Distributes access to CPU                                                  |
| Cache memory  | False  | Tricky, but this isn't handled by the OS. Handled by the hardware instead. |
| File editor   | False  | User directly interacts with this and does not handle hardware             |
| Web browser   | False  | User directly interacts with this and does not handle hardware             |

## Operating System Elements

An OS has three main elements:

1. Abstractions _(e.g., process, thread, file, socket, memory page)_
2. Mechanisms _(e.g., create, schedule, open, write, allocate)_
3. Policies _(e.g., least-recently-used, least-frequently-used, etc.)_

> Note the difference between **"abstractions"**, which would be the **simplification** of the underlying hardware, and **"arbitration"**, which would be the **management** of the underlying hardware.

An example:
A higher level software application deals with a memory page, which is an **abstraction** of physical memory. The **mechanisms** put in place by the OS for applications to interact with memory pages would be the ability to _allocate_ or _map to process_, to name a few. However, certain **policies** also dictate if least recently used (LRU) memory would be moved to disk.
#### OS Design Principles

1. Separation of mechanism and policy. 
   - Flexible mechanisms should be able to support multiple different policies (e.g. LRU, LFU, random)
   - For example, an engine should be able to power a car up to 260 km/h (160 mp/h), though it is up to the driver to drive according to speed limits.
2. Optimise for the common case.
   - It is important to understand the common case _then_ pick a specific policy that makes sense.
   - Certain questions like, "where will the OS be used?", "what will the user execute on that machine?", or "what are the workload requirements?" should be thought about.

## OS Protection Boundary

#### User/Kernel Boundary

Computer systems distinguish at least two modes of execution:
- user-level (unprivileged)
- kernel-level (privileged)

The boundary that exists between the user and kernel level is termed the **user/kernel boundary**.

> The word "level" here is interchangable with "layer", "mode", or "land" (sometimes). You might encounter kernel mode, kernel layer, etc. Doesn't really matter.

Bottom line, there is a separation between a privileged (kernel) space and a non-privileged (user) space. The OS must operate within the privileged kernel mode, since it interacts with hardware. **Hardware access can only be performed at the kernel level**.

#### User/Kernel Switching

1. Traps
   - Applications generally operate in user mode. If a user-level operations tries to perform any privileged activities, activity will be interrupted. This is known as a **trap**.
   - The  hardware switches control over to the OS (kernel mode). Note that the hardware provides such a functionality, as well as the ability to flip a bit, checking if the current mode is privileged or otherwise. This is known as a [mode bit](https://stackoverflow.com/questions/13185300/where-is-the-mode-bit).
   - The OS then has a chance to check if whatever caused the trap be allowed to continue, or be killed.
     > This will become more apparent when processes are covered, but the CPU itself interrupts activity and transfers control over to the OS. A **trap handler** is then executed.

2. System Calls
   * The OS also exposes a **system call interface**.
   * This is a set of operations that the applications can explicitly invoke if they would want the OS to perform certain privileged actions on their behalf (e.g., _open file_, _send socket_, _mmap memory_).
     > System calls are "conscious" decisions by the application itself to request privileged operations. Traps on the other hand, are generated automatically by the CPU.

3. Signals
   - To be covered in a later lecture, but basically a means for the OS to send notifications to the application.

### Deeper Dive into System Calls

##### System Call Flow

1. Suppose an application (user) process is currently executing and requires hardware access. The application invokes a system call (potentially passing some arguments).
2. Control is passed to the OS.
3. Some memory within the kernel is accessed to perform instructions corresponding to the system call.
4. Once the system call completes, control and resutls are returned back to the application process. This changes the exection context from kernel to user mode.
5. The user process starts from the exact place where it left off.  
##### System Call Costs

In order to make a system call, an application must (i) write arguments, (ii) save the relevant data at a well-defined location, then finally (iii) make the system call.

The system call itself requires user/kernel switching. While the hardware provides support for user/kernel transitions, this is still an expensive operation to perform. The process of user/lernel switching also results in **locality switching**.

###### #DEFINE Locality
> Locality referes to the **tendency** of the computer system **to access the same set of memory locations** for a particular time period.

Basically, applications might be actively using hardware **cache**. However, the OS itself might require the hardwre cache to perform whatever system calls were made. This replaces some application cached data and the application will be relegated to memory (slower). 

###### #DEFINE Cache
> A **small amount of memory** that stores **frequently accessed data and instructions to speed up processing time**. The speed of access therefore goes memory cache > memory > disk cache > disk.

###### #DEFINE Cold/Hot Cache
> A "**cold **cache" refers to **cache that is blank or just has stale data**. A "**warm/hot** cache" refers to **cache that contains current and useful data**.

In short, user/kernel transitions are expensive, and thus system calls are also expensive.

## OS Organisation and Basic Services

Recall that an OS provides applications with access to the underlying hardware. It does so by exporing a number of services, which are **mapped to components of the hardware**:
- Scheduler for CPU access
- Memory manager for physical memory
- Block device driver for disk management

An OS also provides **higher level abstractions that map to applications**:
- File management (filesystem as a service)
- Memory management
- Device management
- Process management
- Security

### Monolithic OS

Historically, OSes had a monolithic design. In other words, every possible service that any application requires or any hardware may demand will be included. For example, a monolithic OS will implement file systems for random access and sequential access (as well the multitudes of other access types)

Benefit:
- Everything is included
- Everything being packaged at the same time might result in some possibility for compile-time optimisations. 

Downside:
- Naturally, this OS will be huge.
- Portability, customisation, and overall manageability will become difficult.
- Memory footprint and performance will be detrimentally affected.

### Modular OS

Most current OSes are modular in design. A modular OS implements a number of basic services and APIs, but more modules can be added at any given time. The OS itself specifies certain **interfaces** that a new module must implement.

Benefit:
- Ease of maintenance and upgrading.
- Smaller footprint (smaller codebase), directly resulting in
- Less resource intensive OS. The less resources an OS requires, the more applications benefit from resources.

Downside:
- Modularity necessitates an interface, which may impact performance due to **indirection** (though this may not be very significant).
- Maintenance can *still* be an issue, since modules of varying consistency and quality will be thrown together at runtime.

### Microkernel

This OS form only implements the most basic primitives. For example, the OS may only handle address spaces and threads (these terms will make more sense later).

Other processes that we often think of as an "OS component", (e.g., file systems, disk drivers) are to be handled at the application (user) level. This setup requires a lot of **interprocess communications (IPC)** hence the microkernel usually supports IPCs as one of its core abstractions and mechanisms.

> Again, IPC will be covered later, more specifically during the lecture about **processes**.

Benefit:
- Small in size.
- It is easy to very -- important where some embedded devices or control systems would need an OS that behaves properly.

Downside:
- Though it is small, its portability is questionable as it is often specialised to a specific hardware.
- Software development complexity is increased as software has more responsbilities.
- The increased user/kernel crossings are also expensive.
## Linux and MacOS Architecture

### Linux

- Hardware
- Linux kernel
- Standard libraries (syscall interface)
- Utility program (shells/compiler)
- User applications
>  (organised from lowest to highest level)

The kernel consists of several logical components:
- Virtual file system
- Memory management
- Process management

Since the Linux OS is modular, each subcompent within the three main components listed above can be modified or replaced.
### MacOS X

- I/O kit for device drivers
- Kernel extension kit for dynamic loading of kernel components
- Mach microkernel *memory management* thread scheduling *IPC
- BSD component _Unix interoperability_ POSIX API support *Network I/O interface
- All applications sit above this layer
