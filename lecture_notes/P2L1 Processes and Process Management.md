## Definition

Recall that the OS manages hardware on behalf of applications.
###### #DEFINE Application
>  An **application is a program on disk, flash memory, or the cloud. Importantly, an application is a _static_ entity.**

An application then _becomes_ a process when it is executing. A process is an active entity.
###### #DEFINE Process
> A **process is, simply put, an executing program.**

A single application may spawn multiple processes. These processes need not be exactly equal (think of two instances of a text editor, both with different contents and states).

## Process Representation

### Address Space

A process, being an actively executing program, needs the following:
- Some way to maintain a state of execution.
- Some way to hold data required during execution
- Occasionally, some access to special hardware (e.g., IO devices)

Every single element of the process has to be uniquely identified by its address in memory. The OS abstraction used to encapsulate all of the process state is an **address space**.

The address space is bounded by a range ($V_0$ to $V_{max}$) which contains elements of the process. As such, a process can be represented as follows:
> The orientation of the figure doesn't really matter. This figure can be flipped vertically with no loss of any meaningful information.

![[Pasted image 20230923142802.png]]
- The stack contains temporary data such as method/function parameters, local variables, etc.
- The heap is just memory that will be dynamically allocated during its run time.
- Note the stack and heap. This will be more relevant when actually executing code, but the stack and heap grow (upwards and downwards respectively) into some shared space. When too much memory is allocated to the stack or the heap, an overflow happens. This could happen, for example, if multiple non-terminating recursive functions are pushed onto the stack.
- Data holds global and static variables.
- Text holds the executable code itself.
- Data and text are static states that will be initialised when the process first loads.

###### #DEFINE Address Space
> An address space is a **representation of a process**. The potential range of addresses represent the maximum size of the process address space. **These addresses are virtual addresses**. 

#### Virtual Addresses

Virtual addresses need not correspond to actual physical addresses. The memory management hardware and certain components of the OS (e.g., page tables) handle the mapping of virtual to physical addresses.
###### #DEFINE Page Table
> An OS abstraction that handles **virtual to physical memory address mapping**.

The decoupling of virtual and physical addresses has a couple of benefits:
- We can maintain simple physical memory management.
- Virtual addresses need not be unique across different processes, since they will all map to distinct locations of physical memory (a guarantee by the page tables)

#### Memory Management for Address Spaces

The potential maximum range of the virtual address of a process may not be fully occupied. Otherwise, this would require large amounts of space (and only for one process!).

In fact, the physical memory itself may not have enough space to allocate all processes. The OS has to dynamically decide *which portion of which address space occupies where in physical memory*.
> That's a mouthful. Basically:
> Which address space (which process)?
> Which portion of that address space? 
> Where to place that portion in physical memory?

Some portions may be swapped temporarily to disk, since physical memory may not have enough space to accommodate all processes.
###### #DEFINE Memory
>  Temporary storage used for fast access. Typically used for computation. 

###### #DEFINE Disk
> Permanent storage with slower access. Some data within memory may be temporarily stored in disk (swapped to disk) to free up space.

Of course, the process never really knows about all of this behind-the-scenes action. It simply deals with the virtual memory addresses it is provided.

#### Bottom Line

For each process, the OS must maintain some information regarding the process address space. The OS uses this information to map virtual addresses to physical addresses, as well as to potentially check the validity of accesses to memory.

Regarding processes and address spaces:
- A process is an abstraction that the OS provides to the user. A process represents what the computer is "doing".
- The address space is an abstraction that the OS provides to the process. A running program should not need to worry about physical addresses. That mapping will be handled by the OS.
> When the lecture introduces a process, they show an address space (which is an abstraction provided to the process). In some confusing way, the address space is a means of abstracting a process to the user as well.
### Process Control Block
#### Motivating the PCB

For an OS to manage processes, it must know certain things. For example, what was the process doing when it was stopped?

Following this simple example, we must first understand how applications actually work. The source code of an application **must** be compiled into binary before being executed. The compiled binary can be thought of simply as a series of instructions:

![[Pasted image 20230923143425.png]]

At any given point in time, the CPU (alongside the OS) must know where exactly in the instruction sequence the process currently is. This "position" is stored by the **program counter**. The program counter is maintained within a **register** in the CPU.
###### #DEFINE Program Counter
> A program counter is a register in the CPU that **contains the address of the instruction being executed at the current time**.

###### #DEFINE Register
> A **small set of memory that the CPU accesses**. A CPU maintains multiple registers and may hold addresses for data, status of execution, etc.

Other than a program counter, the execution status of a process can also be defined by a stack pointer.
> This wasn't explained at all in the lecture

In order to maintain all of this information for *all* processes, the OS maintains what is known as a **Process Control Block (PCB)**.
#### Representing the PCB
###### #DEFINE Process Control Block
> A process control block is a **data structure that the OS maintains for every process that it runs**. 

The following figure shows a non-exhaustive representation of a PCB:

![[Pasted image 20230923143705.png]]
1. A PCB is created when a process is created. Certain fields will be initialised. For example, the program counter will start at the first instruction of a sequence.
2. Certain fields are updated when the process state changes. For instance, if a process requries more memory, then the OS may allocate more memory and record this information.
3. Some other fields may change pretty frequently. The program counter, for one, changes *per instruction*.
#### Maintaining the PCB

We certainly do not wish for the OS to keep track of the ever-changing program counter. This responsibility actually falls to the CPU (briefly alluded to above when introducing the program counter). 

However, it is ultimately the OS' job to collect this information from the CPU and store it within the PCB whenever the process is not being executed on the CPU.

Illustrating the following back-and-forth with an example:
Suppose there are two processes, P1 and P2.
1. When P1 is running, the CPU registers store information regarding P1.
2. Suppose that the OS interrupts P1 to start P2. The OS will save PCB_P1, including the information that was stored within the CPU registers. The OS then **restores** PCB_P2 (i.e. load P2 information to the CPU registers).
3. As P2 runs, the program counter will be updated by the CPU. If it requires additional resources such as memory, the OS will handle this responsibility.
4. When P2 is interrupted, the OS again saves all information within PCB_P2 and restores PCB_P1. Note that the state of PCB_P1 is exactly the same as when the OS first interrupted it, hence it can continue exactly as it left off.
5. Each time a swapping of process occurs, the saving and restoring recurs. This is known as a **context switch**.
#### Deconflicting Address Spaces and PCBs
We saw thus far two "representations" of processes. What the fuck.

![[Pasted image 20230923143919.png]]

Here's what I think:
> A address space refers to the memory regions that a process is using. The address space is something the process can mutate and operate on freely, since it **belongs in the user space**.
> 
> The PCB is an overall "state" of any given process that the OS maintains. The **PCB resides in kernel**. 
> 
> Basically, they are two different things (what I gather from forums anyway). The  PCB stores information regarding the **state of a process**. The address space is an abstraction that **maps virtual address to physical addresses**. **Together**, the PCB and address space describe a process.
> 
> In some cases, the PCB itself contains information regarding the address space (as shown above).
> 
> Linux however, maintains the address space outside of the PCB, `task_struct`. Just to confuse everyone even more.
#### Context Switching
###### #DEFINE Context Switch
> The **switching of the CPU from the context of one process to the context of another.**

As a recap: the PCB of various processes are stored in memory. The CPU registers hold parts of the PCB of a currently executing process.

Context switching is expensive:
- There are direct costs involved, i.e., the number of cycles required for 'load' and 'store' instructions for PCBs.
- There are also indirect costs involved. Whenevr a process is running, some data is stored within the processor cache (owned by the CPU). However, when context switches to another process, the cache is now **cold**, resulting in **cache misses**. Of course, data loaded from memory is much slower. 
###### #DEFINE Cache Miss
> **Data requested from the cache is not present**.
## Process Life Cycle

### Overview

Previously, we've seen how processes can either be running or idle. Recall that as the context switching occurs, a process either starts running or is interrupted (and becomes idle). Here, the process life cycle will be viewed in its entirety.

The figure below shows the five main states of a process' lifecycle:

![[Pasted image 20230923144251.png]]
- When a process is created (**new**), the OS will perform **admission control** and once it is determined that the process can run, it will enter the **ready** state.  The process is allocated some resources, and a Process Control Block is created.
- In the **ready** state, the process has to wait for the CPU scheduler to **dispatch** it onto the CPU. When it is dispatched, then the process is said to be **running**.
- A **running** process can be interrupted when a context switch is performed, moving the process back to a **ready** state.
- A **running** process may also need to wait for I/O operations, or an event like a timer. The process enters the **waiting** state until the I/O or event completes, following which it re-enters the **ready** state.
- Finally, when a **running** process finishes running or encounters an error, it will be **terminated**. The process will return an appropriate exit code (success or error).
###### #DEFINE Admission Control
> Admission control involves **checking that current resources are sufficient for the newly created process.**

<em>Quiz: Process State</em>
The CPU is able to execute a process when the process is in the **running** or **ready** state.

### Process Creation
A process can create child processes. Therefore, all processes ultimately come from a single root.

![[Pasted image 20230923144405.png]]

This is true of most OSes. When the initial boot process is complete and the OS is loaded onto the machine, it will create some number of initial processes. These processes are privileged.

When a user logs into the system, a user shell process is created, *csh(340)* and new processes eventually spawn from the parent shell process. Most OSes support two mechanisms for process creation:

1. Fork
   - The OS creates a new PCB for the child process and copies the values of the parent PCB into the child PCB. Note that the parent and child are *almost* identical (for example, they will differ in terms of process ID).
   - Both the child and parent process will continue their execution at the instruction immediately after the fork, since they have the essentially similar values within the PCB (more importantly, their program counter is the same).
   - In other words, when forking a process, both the parent and child start their execution at the same point.

2. Exec
   - Exec operates on the child PCB created via fork. However, exec will replace the child image by loading a new program, i.e., values associated with the new program will be written to the PCB.
   - Importantly, the program counter of the child will point to the first instruction of the program. The child process starts from the first instruction.
   - In sum, exec operates **after** fork.

<em>Quiz: Parent Process</em>

On UNIX-based OSes, the `init` process is often regarded as the "parent of all processes". On the Android OS, the `zygote` process is considered the "parent of all *app* processes". 
### Process Dispatching

A process can only be dispatched to the CPU when it is **ready**. However, multiple processes may be ready at the same time. This is depicted in the figure below, where ready processes are in a queue:

![[Pasted image 20230923144601.png]]

Given that only one process can be dispatched onto a CPU at any given time, there needs to be a way of determining which process is dispatched (alongside other considerations, as we'll see in a bit).

The responsibility of **scheduling** processes lies with the **CPU Scheduler**.
###### #DEFINE CPU Scheduler
> An **OS component** that determines **which one of the currently ready processes will be dispatched** to the CPU to start running, and **how long it should run** for.

Thus, in order to manage the CPU, the OS must be able to:
- **preempt** (interrupt and save the current context of) the executing process,
- **schedule**: run the scheduler and any associated scheduling algorithm to choose the next process, and
- **dispatch** the process on the CPU, which involves switching into the process' context.

The scheduler itself relies on the CPU to run scheduling algorithms, or other OS-related operations. The CPU resource is precious, hence the OS **must** be efficient in the act of scheduling itself. Algorithms (scheduling) and data structures (queue) must allow for such efficiency in scheduling.

It has already been established that running the scheduler uses precious CPU resources that could otherwise be directed to running processes in the ready queue. Thus, how frequently should the scheduler be run in order to maximise efficiency? A related question would be the following:
#### How long should a process run for?

![[Pasted image 20230923144814.png]]
Basically:
- Useful CPU time can be computed by taking the time-slice ($T_p$) over the total time. The **time-slice** is the time allocated to a process on the CPU.
- In the example shown above, the percentage of CPU time spent on useful work would be ($2 * T_p$) / ($2*T_p + 2*t_{sched}$), where $t_{sched}$ is the time spent running the scheduler.
- Evidently, the longer a process runs (the larger the value of $T_p$), the more time is spent on useful work.

Discussion on scheduling design choices will be conducted at a later time, but there are few considerations to first ponder:
- What are appropriate time-slice values?
- What metrics should be used to choose the next process to run?
### Process Waiting

A process may make an I/O request. The OS then delivers that request, and moves the process to the **I/O queue** associated with that particular I/O device. The process will remain **waiting** in queue until the request is responded to.

Once the I/O response is completed, the process moves back into the ready state and rejoins the ready queue (or might be immediately dispatched on the CPU if the CPU is free).
### Process Ready

We already know that a newly created process will join the ready queue.

A process may **rejoin** the ready queue in the following manners:

![[Pasted image 20230923145041.png]]

- An I/O request completes and the process rejoins the ready queue.
- The timeslice (recall: time allocated for a process to run on the CPU) expires.
- A process is forked. It's child ends up on the ready queue.
- A process waiting for an interrupt will rejoin the ready queue is an interrupt occurs.

_Quiz: CPU scheduler responsibilities_
The following are **not** responsibilities of the CPU scheduler:
- maintaining the I/O queue
- decision on when to generate an event that a process is waiting for (the CPU scheduler has no say on whether or not an I/O event will occur)
  > The phrasing of this is really weird. Basically, this is saying that the CPU scheduler generates an event that a process is currently waiting for. That's a false statement, of course.

## Interaction Between Processes
### Short Answer: Yes

An OS must provide mechanisms for processes to interact. In fact, modern, complex applications are built upon a number of different processes, all running to achieve the same end-goal.
### Case Study: Web Applications

A web application is commonly build with a front-end (web server) and a back-end (database).
- These are two different processes that need to **share information** with each other.
- Recall however, that the **OS is at odds with this behaviour**; the **OS goes through great effort to ensure that processes are protected and isolated from each other**.
- This includes separate address spaces for each of the process, and memory allocated and accessible to each of the process would be different as well.

![[Pasted image 20230923145323.png]]

Therefore, mechanisms that allow processes to cross-talk have to be **built upon these protective restrictions**. These mechanims are known as **Inter-Process Communication (IPC) mechanisms**.

- IPC mechanisms help to transfer data/info between address spaces, all while
- Maintaining the protection and isolation of processes that the OS is trying to enforce.
- Different processes may also require different communication means (e.g., periodic versus continuous stream of data). IPC mechanisms must provide flexibility and performance for these different communications.
#### Message-Passing IPC

This is one of the IPC mechanisms that OSes provides support for. The **OS provides a communication channel**, like a shared buffer. Processes can then write (`send`) or read (`recv`) messages to/from the channel.

![[Pasted image 20230923145427.png]]

Benefit:
- The OS manages the channel and provides APIs (system calls) for writing or reading data from the channel. 

Downside:
- There are overheads involved in 
	- packaging the message from a process (say, P1), 
	- copying the message into the channel, which lies in kernel memory
	- copying the message from the channel to the address space of P2.
#### Shared Memory IPC

In this IPC mechanism, the OS establishes a shared channel (between processes) and maps it into each process' address space. The processes are allowed to directly read and write into this memory (just as they would be able to, to any memory location that is part of their virtual address space).

![[Pasted image 20230923145601.png]]

Benefit:
- The OS is not in "fast path" of the communication between processes. In other words, there are no overheads incurred by passing the message from process to OS to process.

Downside:
* Because the OS is not involved in the communication, it does not provide APIs that fully support this IPC mechanism. This might make this mechanism more error-prone and the usage of shared memory needs to be handled by the software developer.

_Quiz: Shared Memory Communication_

"Shared memory based communication performs better than message passing communication". (True/False/It Depends)
> Answer: it depends. The data exchange itself is cheap, since it avoids the need to copy data in/out of the kernel. However, the actual operation of mapping memory between two processes is expensive.
> 
> Therefore, it only makes sense to implement shared memory if the setup cost can be amortised across a sufficiently large number of messages.
