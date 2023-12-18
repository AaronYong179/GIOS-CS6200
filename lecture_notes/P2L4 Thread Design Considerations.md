## Introduction and Overview

Recall that threads can be supported at the user level, at the kernel level, or both. 
- Naturally, the ability to support threads at the kernel level must imply that the operating system itself is multithreaded. In order to do this, the kernel must maintain some data structure to represent threads as well as scheduling mechanisms for multithreading.
- Similarly, threading at the user level requires a user level library that provides the necessary data structures as well as scheduling mechanisms. Note that different processes **may use completely different user level thread libraries**.

Recall also that user level threads can be mapped onto kernel level threads via a one-to-one, many-to-one, or many-to-many pattern.

This section will take a more detailed look at the "thread" data structure and associated scheduling mechanisms. A simple single-thread and single-CPU example will be used as a starting point, followed by case studies of increasing complexity (multi-thread, multi-process, and multi-CPU).

## Thread Data Structures

### Single PCB

Suppose that we have a single-threaded process, running on a single CPU. In this case, we are already familiar with how this process is represented; the **process control block** should contain information regarding the current stack, registers, and the virtual address mapping.

### User-level Thread Data Structure

Suppose then that the process is now **multithreaded** at the user level. The linked threading library will have to maintain certain information regarding threads, so as to track thread resource use and make scheduling/synchronisation decisions. Therefore, the threading library must maintain a thread data structure containing the following information:
- thread IDs
- thread registers
- thread stacks

### Kernel-level Thread Data Structure

If the kernel is **also multithreaded**, then maintaining a large PCB (as we've encountered in previous lectures) does not make much sense. The kernel-level threads will have different stack and register values, but the overall process shares a few commonalities (e.g., virtual address mapping). Replicating the entire PCB for each kernel-level thread is inefficient in that regard.

Therefore, the solution would be to split the stack and registers information away from the PCB. Each individual kernel-level thread (KLT) would then have their unique stack/registers data structure, but they would all share the virtual address mappings of the PCB.

![[Pasted image 20230914192025.png]]

When there are **multiple processes**, there will need to be multiple copies of user level thread (ULT) structures, PCBs, and KLT structures. There will also need to be relationships between these data structures:
- The user-level thread library must keep track of all threads for a given process, hence must have a relationship with the PCB (which represents a process).
- The process itself must maintain a relationship with kernel-level threads, as the threads that execute on its behalf must be tracked accordingly.

### CPU Data Structure

Suppose there are multiple CPUs on the system. There now needs to be a data structure to represent those CPUs, and there needs to be a relationship between the kernel level threads and the CPUs they execute on. The figure below represents all the relationships between the data structures mentioned thus far:

![[Pasted image 20230914192759.png]]

These relationships aid the handling of multiple processes on a multicore, multithreaded OS. 
- For example, if a kernel-level thread needs to switch between two processes, it can quickly determine that there are two different PCBs involved (KLT-PCB relationship).
- As such, the kernel-level thread can easily decide that context switching is required, and the relevant PCBs can be saved or restored.
### Hard and Light Process State

We have not yet considered the case where two kernel-level threads belong to the same address space (i.e. they service the same process). In this case, there is information within the PCB that will be shared, though certain information (such as signals or sys call arguments) will differ between kernel-level threads.

When context switching between these two threads occur, there is a portion of the PCB that should be preserved. However, there will also be information that is kernel-level thread-specific, and it also depends on the user-level thread that is currently executing (something that the threading library directly impacts).

Therefore, it is possible to split the PCB into a **hard process state**, which contains information that is relevant for all user-level threads and a **light process state**, which is only relevant for a subset of the user-level threads that are currently associated with a kernel-level thread.

### Motivation and Quick Summary

Previously, a large continuous data structure known as the PCB was introduced. For each entity, a separate copy needs to be maintained; threads that share information are not exempt from this. Since each entity has its own PCB, context switching always requires saving and restoring this rather large data structure. Finally, a singular data structure would mean that it is hard to update or customise, since multiple services use the same structure (in other words, no modularity is afforded).

The singular PCB then clearly falls short in the following aspects: 
- scalability (due to its size)
- overheads (each entity needs a private copy)
- performance (especially during context switching)
- flexibility (updates are difficult)

In contrast, by having multiple, smaller data structures, it becomes much easier to share information (where sharing is required). For example, threads that share the same virtual address mapping can simply point to the same location. On a context switch, only elements that need to be changed will be saved/restored. Finally, updates to the smaller data structures is less likely to break everything as a whole, since they are confined to much smaller interfaces.

OSes today generally favour multiple, smaller data structures instead of the singular PCB counterpart.

_Quiz_
_What is the name of the kernel thread structure (name of C struct)?_
```kthread_worker```

_What is the name of the data structure contained in the above structure that describes the process the kernel thread is running (name of C struct)?_
```task_struct```

## Case-Study Solaris 2.0 (Data Structures)

This section investigates the abovementioned data structures, specifically as implemented by the SunOS in Solaris 2.0. This lecture is roughly modelled after the SunOS, though for further details the paper [Beyond Multiprocessing: Multithreading the SunOS kernel](https://www.usenix.org/legacy/publications/library/proceedings/sa92/eykholt.pdf) or [Implementing Lightweight Threads](https://www.usenix.org/legacy/publications/library/proceedings/sa92/stein.pdf)should be consulted.
### Overview

Consider the following figure:

![[Pasted image 20230916192224.png]]
- Note that the OS itself supports multiple cores (CPUs) and the kernel itself is multithreaded.
- At the user level, the processes can be multithreaded or otherwise.
- Both many-to-many as well as one-to-one mappings are supported.
- Each kernel-level thread that is executing a user-level thread has an **associated lightweight process (LWP) data structure**. From the user's perspective, these LWPs are essentially virtual CPUs that it can use to schedule user-level threads on.
- At the kernel level, there will be a scheduler (kernel-level, naturally) that will manage kernel-level threads and controlling their access to the underlying physical CPU hardware. 
### User-level Thread Data Structures

Note that the discussion here will not be specifically regarding `pthread` (POSIX Threads) but is roughly similar. 

Whenever a thread is created, the library returns a **thread ID** (`tid`). This is not actually a direct pointer to the thread data structure itself, rather it is an index to a table of pointers. It is the table of pointers that in turn point to the per-thread data structures.
- If the thread ID was a direct pointer to the thread data structure itself, then the pointer might just point to some corrupt memory if the thread is problematic.
- However, due to this indirection, some additional information regarding the thread can be encoded within the entry of the pointer table.
- This could include some data for meaningful feedback or an error message.

The thread data structure itself contains the following information:
- execution context
- registers
- signal mask
- priority
- thread local storage
- stack (pointer)

The thread local storage includes variables declared within the thread function -- in other words, these are **known at compile time**. The compiler can therefore allocate private storage on a per-thread basis.

In any case, the size required for most of the information within the thread data structure is known at compile time. 
- Therefore, it is possible to create thread data structures and lay them in a **contiguous** manner such locality can be achieved. 
- Additionally, contiguity might make it easier for the scheduler to locate the next thread (simply skip the size of the thread data structure to find the next!).
- The issue with contiguity lies with uncontrolled stack growth. The user-level thread library does not inject itself between any stack updates, nor does the OS know that there are multiple contiguous thread data structures.
- It is therefore possible that a growing stack **overwrites the data structure of another thread**. If this happens, the problem (error) will likely be detected in the thread that was overwritten instead of the thread with the stack overflow.
- The solution introduced by Stein and Shah was the **inclusion of a "red zone"**. This red zone separates thread data structures, and serves as a boundary for stack growth. Basically, if a stack growth tries to write to an address that falls within the "red zone", the OS will cause a fault.
- Now, if a stack overflow occurs, it becomes much easier to find the thread at fault.

![[Pasted image 20230916200003.png]]
### Kernel-level Thread Data Structures

#### Process
Firstly, for each process, the following information is maintained:
- A list of kernel-level threads that execute within the process address space.
- The virtual address mappings of the process.
- User credentials (does the user have rights to a certain operation?)
- Signal handlers (to be covered in a later lecture)

#### Light-Weight Process (LWP)
LWPs contain information that are relevant only to some subset of the process:
- User-level registers
- System call arguments (this and the point above are relevant to one or more user-level threads executing in context of the process)
- Resource usage info
	> Note that the OS tracks resource usage on a per-kernel thread basis. Recall that LWPs correspond to kernel-level threads. Therefore, if we wish to determine the total resource usage of a process, simply sum the information in all LWPs associated with a particular process.
- Signal mask

Note that LWPs contain information similar to that maintained by user-level threads, though these are **visible to the kernel**. These LWPs are not needed when the process is not running.

#### Kernel-Level Threads (KLT)
The kernel-level thread data structure contains the following information:
- Kernel-level registers
- Stack pointer
- Scheduling information (class, etc.)
- Pointers to associated LWP, process, or CPU structures

Note that the kernel-level thread data structure contains information that is **always required**. For example, scheduling information must be maintained and accessible by the OS even if the thread itself is not active. Basically, this information is **not swappable** and has to be always present in memory.

#### CPU
The data structure associated with the CPUs contains the following information:
- The thread currently scheduled (this information can be held on specific registers)
- List of kernel-level threads that executed on the particular CPU
- Dispatching and interrupt handling information

Note that being able to point to a specific thread (even at the CPU) level and the fact that relationships exist among the different data structures means that the entire process can be "reconstructed" simply by following pointers from one data structure to another.
### Data Structures Summary

The paper by Eykholt et al. summarises the relationships above in the diagram below:

![[Pasted image 20230917094533.png]]
- A process data structure contains information regarding the User and points to its address space. The process also points to a list of kernel-level thread structures.
- Each of the kernel-level thread structures points to the LWP that it corresponds to. A pointer exists to its stack (naturally). The LWP and the stack are swappable portions.
- Note that information regarding the CPU is not shown in the above figure. Some other information (e.g., pointers from the thread to the process) are not shown either to reduce clutter.

## Thread Management

### Basic Thread Management Interaction

This section will motivate certain functionalities required for basic thread management, specifically regarding communication between user- and kernel-level threads. 

Consider the simple scenario where we have a single mulithreaded process (suppose the process only has four threads) and one CPU. At any given point in time, the actual level of concurrency required by the process is just **two** (i.e., only two user-level threads are actually executing).

#### User to Kernel Request 
It would be nice if the user-level library could request for a specific number of kernel-level threads (especially if the OS has a limit on the number of kernel-level threads it can support).
- When the process actually starts, the kernel first allocates a default number of KLTs and the accompanying LWP (say, one).
- The process then requests for additional KLTs. The kernel supports a system call `set_concurrency`. In response to this system call, additional KLTs will be created and allocated to the calling process.

#### Kernel to User Signal
Consider the case where the two user-level threads block; suppose they needed to perform some I/O operation hence are moved to the wait queue associated with that particular I/O event. The kernel-level threads will block as well.

![[Pasted image 20230917102047.png]]

Now, the process as a whole is also blocked, since the only two KLTs allocated to it are blocked on some I/O operation. This happens as the user-level library _does not know what is happening in the kernel_.

It would have been useful for the kernel to **notify the user-level library before it blocks the kernel-level threads**. A notification signal should be sent back to the user-level library. The user-level library can then act accordingly, and ask for more KLTs/LWPs.

At a later time, when the blocking I/O operations have been resolved, there will be extra kernel-level threads that are constantly idle (recall that the process itself really only needs two threads of execution). The kernel should then **notify the user-level library that the idling kernel-level thread will be removed; scheduling on that thread is no longer allowed**. 

#### System Calls and Signals
In sum, we saw in the simple example above that the user-level library does not have much information regarding the kernel, nor does the kernel know what is happening in the user-level library. 

System calls and signals allow the user and kernel to interact and coordinate with each other.

_Quiz_
_In the `pthread` library, which function sets the concurrency level? (give the function name)_
`pthread_setconcurrency()`

_For the above function, which concurrency value instructs the implementation to manage the concurrency level as it deems appropriate? (give an integer)_
`0` -- let the underlying implementation decide.
### Visibility Issues

Previously, we've seen how certain aspects of thread management are only visible to the user or kernel. The user-level library only sees ULTs and available KLTs, while the kernel only sees KLTs, CPUs, and the kernel-level scheduler.

In a one-to-one model, all user-level threads are associated with a kernel-level thread. In that case, the user essentially "sees" the kernel-level threads, although the kernel handles the scheduling of kernel-level threads. Even if a one-to-one model is not used, it is possible for the user-level library to **request to bind its user-level threads to a kernel-level thread**. 
###### #DEFINE Bound Thread
> A user-level thread that is permanently associated with a kernel-level thread. In a one-to-one model, all user-level threads are bound threads.

###### #DEFINE Pinned Thread
> A kernel-level thread that is permanently associated with a CPU.

Suppose that one of the user-level thread and its associated kernel-level thread acquires a lock (i.e. a critical section is entered). Suppose that all other user-level threads are also waiting to acquire that same lock.

![[Pasted image 20230917145249.png]]

If the kernel **preempts** said kernel-level thread and schedules another kernel-level thread, nothing of use can be done. Instead, the newly scheduled kernel-level thread will simply cycle through the user-level threads (that are all waiting for the lock anyway).
###### #DEFINE Kernel Preemption
> The ability to interrupt a currently executing CPU for the assignment of other tasks. Specifically, the scheduler is **forced to context switch**.

In essence, time will wasted, as the only way the process can continue would be if the kernel-level thread associated with the lock is rescheduled onto the CPU once more. The problem is a visibility problem -- the kernel itself does not see mutex variables or wait queues of the user-level library. This leads to a disconnect between state and decisions of the kernel and the user-level library.

Clearly, having a one-to-one mapping alleviates some of these issues.
### User-level Library Scheduler

Since the user-level library plays such an important role in managing user-level threads, this section is dedicated to investigating _when_ exactly it gets involved during the execution loop.

The user-level library is part of the user process, hence is part of the process **address space**. The process jumps to the user-level library scheduler when:
- ULTs explicitly yield
- A timer set by the user-level library expired.
- ULTs call library synchronisation functions such as `lock` or `unlock`.
- Blocked threads become runnable.

Other than the abovementioned user-level thread operations, the user-level library scheduler also runs on signals from timer or the kernel.
### Multi-CPU Issues

In previous examples, a single-CPU was considered. In that case, any changes in thread scheduling by the user-level library would be immediately reflected on that singular CPU. However, when multiple CPU cores are involved, a change made by the user-level library in the context of one CPU must be somehow communicated to threads running on other CPUs.

Consider the following situation, where there are three threads, `T1`, `T2`, and `T3` and two CPU cores. The thread order of priority is as follows: `T3` > `T2` > `T1`. Suppose that `T2` is currently running on one of the kernel-level threads and is locking a mutex. `T3`, the highest priority thread, is waiting to lock that mutex hence `T1` is scheduled to run instead on the other kernel-level thread.

![[Pasted image 20230917201207.png]]

Suppose that at some point, `T2` unlocks the mutex -- `T3` then becomes runnable. Now all three threads are runnable, though in accordance of priority, `T3` should take precedence. There is thus a need to preempt `T1` (lowest priority), but it is currently running on another CPU. The realisation that `T3` should context switch with `T1` was made in context of `T2`'s execution. 

It is not possible to modify the registers of one CPU when executing on another. Instead, there needs to be some **signal** to the other kernel-level thread to **execute the user-level library code locally**. Once this happens, the user-level library on the second CPU will determine that it needs to schedule the highest priority `T3`.

In short, once multiple CPUs are involved, alongside multithreaded user-level processes and a multithreaded kernel, interactions between user- and kernel-level thread management becomes much ore complex. 
### Synchronisation-Related Issues

Consider now the following scenario: there are four threads `T1`, `T2`, `T3`, and `T4` and two CPUs. `T1` is currently executing on one kernel-level thread and has locked a mutex. Suppose that `T4` is about to be scheduled on the other CPU and `T4` is also waiting for the same mutex held by `T1`. 

![[Pasted image 20230917204916.png]]

The normal behaviour as discussed in [[P2L2 Threads and Concurrency]] section would be to place `T4` on a wait queue associated with the mutex. However, on a multi-CPU system, it is possible that the `T1` is just about to release the mutex. In that case, **if the critical section is short**, it might make more sense for the thread that needs the mutex to _spin_ on the CPU rather than being context switched and placed on the blocking wait queue.

If it faster to wait for `T1` to release the mutex than to be placed on the wait queue, then `T4` should just burn a few CPU cycles, spin, and wait for `T1` to unlock the mutex.

For short critical sections, don't block -- spin. For long critical sections, carry on with the default behaviour of blocking and waiting. Mutexes that are able to block or spin are called **adaptive mutexes**. Clearly, these only make sense on multi-CPU systems as the decision to spin will only take place **if the owner of the mutex is currently executing on another CPU**. 

Recall that in the discussion of mutexes, it was said that information regarding the owner of the mutex should be maintained. This is useful in this particular situation, as a thread trying to lock a mutex can check to see if the current owner is currently executing or otherwise. 
### Thread Cleanup

Once a thread is no longer needed (i.e. when it exits), it should be destroyed the memory associated with it freed. This might include the thread's data structure, stack, etc. However, since thread creation incurs a time cost (data structures need to be initialised), it might be better to **reuse the existing data structures instead**. 

Therefore, when a thread exits, it is put on **"death row"**. It is not immediately freed or destroyed. Periodically, a special **reaper thread** will perform garbage collection, freeing up threads that have been marked for destruction. 

If a request for a thread comes in before the thread is reaped, its data structures/stack will be reused, resulting in an overall performance gain.

_Quiz_
_In the Linux kernel codebase, a minimum of how many threads are needed to allow a system to boot?_
20 threads.

_What is the name of the variable used to set this limit?_
`max_threads`
## Interrupts and Signals

### Overview

#### Definitions

###### #DEFINE Interrupts
> Events **generated externally** by components other than the current CPU. Examples of such components are I/O devices, timers, or even other CPUs. 
> 
> The interrupts that can occur are **determined based on the physical platform** (i.e. the underlying hardware/architecture).
> 
> Interrupts appear **asynchronously**. Basically, they are not generated in response to a current action taken by the CPU.

###### #DEFINE Signals
> Events **triggered by the CPU and the software running on it**. Signals can therefore be generated by the software, or the CPU hardware itself.
> 
> The signals that can occur are **determined based on the operating system**. Two identical hardware will generate similar interrupts, but signals may differ if two different OSes are compared. 
> 
> Signals can appear **synchronously or asynchronously**. Signals can occur in direct response to a CPU action (e.g. if invalid memory is touched) or they can occur similar to interrupts.

#### Interrupts and Signals Similarities

1. Both have a unique ID that depends on the hardware (interrupts) or on the OS (signals).
2. Both can be masked and disabled/suspended via the corresponding **mask**.
	- An interrupt can be masked on a per-CPU basis (interrupts affect a CPU)
	- A signal can be masked on a per-process basis (signals affect a single process)
	- Basically, masking allows one to disable to delay the notification of an interrupt or signal.
3. If the mask enables the interrupt/signal, the corresponding interrupt/signal handler will be triggered.
	- Interrupt handlers are set **for the entire system by the OS**.
	- Signal handlers are set on a **per-process basis** by the process itself.
#### Summary

In short, interrupts and signals:
- can be handled in specific ways via interrupt/signal handlers
- can be ignored via interrupt/signal masks
- can be expected or unexpected (synchronous or asynchronous)

### Interrupt/Signal Handling

#### Interrupt Handling

A device interrupts a CPU by sending a "message" through the interconnect that connects the device to the CPU complex. In the past, dedicated wires were used for this purpose, though most modern devices use a special message termed MSI (Message-Signaled Interrupt) that can be carried on the same interconnect that connects the device to the CPU. 
###### #DEFINE Message-Signaled Interrupt
> An alternative to using dedicated pins to trigger interrupts. Devices that use MSI trigger an interrupt by writing a value to a particular memory address.

Based on the pins that receive the interrupt or the MSI, the interrupt can be uniquely identified. This information allows to identify the exact device that generated the interrupt.

There exists an **interrupt handler table** which maps the given interrupt number to the starting address of the associated interrupt handler routine. In the event an interrupt occurs, the CPU first looks at the interrupt handler table. The program counter of the interrupted thread will then be set to the interrupt handler starting address and the interrupt handling routine is executed. Note that all of this happens in the context of the interrupted thread.

![[Pasted image 20230918172843.png]]
A final note on the interrupt handler table: the supported interrupt numbers are defined by the hardware, while the OS defines the handler associated with each interrupt.

#### Signal Handling

Signal handling slightly differs from interrupt handling -- recall that signals are not generated by external entity. For example, if a thread is performing an illegal memory access, a signal `SIGSEGV` will be generated **by the OS**. In Linux, `SIGSEGV` corresponds to a signal number of `11`.

Roughly similar to that of interrupts, a **signal handler table** is also present, which maps signal numbers to the starting address of the associated handler routine. However, in this case, the signals that can occur are defined by the OS, while the handlers can be defined by the process itself.

![[Pasted image 20230918200719.png]]

The operating system provides default responses to signals. The following shows some examples of default signal responses:
- Terminate
- Ignore
- Terminate and Core Dump (core dump for inspection of what caused a crash)
- Stop
- Continue (from stopped)

As mentioned however, it is possible for the process to define its own signal handler via system calls such as `signal` or `sigaction`. Note that some signals cannot be caught in this manner and must be handled with the default behaviour. 

Examples of synchronous signals include:
- `SIGSEGV`
- `SIGFPE` (division by zero)
- `SIGKILL(kill, id)` (from one process to another)

Examples of asynchronous signals include:
- `SIGKILL(kill)` (from the perspective of the killed process, this signal is async)
- `SIGALARM` (timeout from timer expiration)
### Interrupt/Signal Disabling

#### Motivation

Interrupts and signals are handled in context of the thread being interrupted/signaled. In other words, the handler executes on the thread's stack, which may cause certain issues.

Whenever interrupt/signal handling occurs, the program counter of the thread will point to the starting address of the handler. The **stack pointer**, however, **remains the same**.

Consider the scenario where a handler needs to access _some shared state_ that is used by other threads in the system. In this case, mutexes must be used. However, if the thread being interrupted/signaled already locked the mutex, a **deadlock will occur**.
- The thread itself cannot unlock the mutex until the handler returns, but
- The handler cannot return until it locks the mutex.

We may choose to alleviate this issue by enforcing simple handler codes -- handlers should not be able to acquire mutexes, for example. This is too restrictive.

#### Masks

A better solution would be to use interrupt/signal **masks**. These masks allow us to dynamically decide if a particular interrupt/signal can stop the execution of a thread.
- The mask itself is a sequence of bits. Each bit corresponds to an interrupt/signal and the value 0 or 1 signifies a disabled or enabled state respectively.
- When an event occurs, the mask is first checked to determine if the given interrupt/signal is enabled.
- If enabled, the handler is allowed to execute.
- Otherwise, the interrupt/signal is made pending and will be handled when the mask changes.

In short, to solve the deadlock situation described above, the thread must first disable the interrupt/signal prior to mutex locking. After the mutex is released, then interrupt/signal should be re-enabled. The handler will then be executed.
> Note that while an interrupt/signal is pending, other incoming interrupts/signals can become pending as well. Typically, the handling routing is only executed once. Therefore, if we would wish for a handling routine to be executed more than once, it is not sufficient to generate the same signal multiple times.
#### Distinguishing Interrupt/Signal Masks

Interrupt masks are maintained on a per-CPU basis.
- If a mask disables an interrupt, the hardware interrupt routing mechanism will simply not deliver the interrupt to the CPU. 

Signal masks are per-execution context (rather, depending on the ULT on top of KLT)
- If a mask disables a signal, the kernel sees the mask and will not interrupt the corresponding thread.
### Addendum
#### Interrupts on Multicore Systems

On a multi-CPU system, the interrupt routing logic usually directs the interrupt to any CPU that has interrupts enabled. To make things slightly cleaner, it is possible to only enable interrupts on **one** CPU. This avoids the overheads and perturbations related to interrupt handling on any of the other cores. Overall, this results in improved performance.

#### Types of Signals

1. One-shot signals
	- If there are multiple instances of the same signal, these signals will be handled **at least once**.
	- Therefore, it is likely that regardless of the number of signals pending, only one handler execution will occur.
	- One-shot signals must also be explicitly re-enabled per signal. If a user wants to install a custom signal handler, the handler will only be called on the first signal encountered. Any other subsequent signals of the same number will be handled as specified by the default OS action (which may include ignoring!)
2. Real-time signals
	- Basically, if a signal is raised `n` times, the handler will also be called `n` times.
	- In essence, real-time signals exhibit more of a "queueing" behaviour as opposed to one-shot signals' "overriding" behaviour.

_Quiz_
_Using the most recent POSIX standard, indicate the correct signal names for the following events:_

| Event | Signal | 
| -------- | -------- | 
| terminal interrupt signal | `SIGINT` |
| high bandwidth data is available on a socket | `SIGURG` |
| background process attempting write | `SIGTTOU` |
| file size limit exceeded | `SIGXFSZ` |

These can be found within the `signals.h` file.
### Threads and Interrupt Handling

Recall the possibility of deadlocks occurring when performing an interrupt. Previously, it was discussed that masks should be used to delay the execution of the interrupt handler. In the paper by Eykholt et al., it was proposed that interrupts should be allowed to **become full-fledged threads**. This should occur whenever a potentially blocking operation is performed.

Taking one more look at the example regarding deadlocks given above, if an interrupt is given its own thread, this would mean that it possesses its own execution context as well. Importantly, the thread scheduler is able to schedule the original thread (which is currently locking the mutex) back onto the CPU while the **interrupt thread simply blocks and waits**.

However, **dynamic thread creation is expensive**. There is a need to dynamically decide if a thread is to be created or otherwise:
- If the handler does not block, then simply execute on the interrupted thread's stack.
	> The paper by Eykholt et al. defines "does not block" as a handler never locking a mutex.
- If the handler potentially blocks, then create a new thread.

The solution to that is then to simply have the kernel pre-create and pre-initialise thread structures for interrupt routines. In that case, the time cost for thread creation is not paid for _during_ interrupt handling itself. 
#### Top/Bottom Half

In Linux, interrupt handling are split into two parts:
1. The **top half** occurs in the context of the interrupted thread. This half must be fast, non-blocking, and require minimum processing. 
	> Resolving deadlocks by enforcing simple interrupt handlers or by masking would fall under this half.
2. The **bottom half** occurs once we move into the context of a separate handler thread. This can contain arbitrary complexity. 

#### Analysing Performance

Interrupt handling with threads (as described by Eykholt) was really motivated by performance. The operations necessary to perform the appropriate checks and, if necessary, create a new thread, resulted in an overhead of 40 SPARC instructions per interrupt. 

As a direct result of this however, 12 instructions are saved per mutex. There is no longer a need to disable/re-enable interrupts for every single mutex encountered.

Since mutex locking and unlocking occur much more frequently compared to interrupts, the overall instruction count is lower when handling interrupts with threads. 

This brings to light an important concept, _always optimise for the common case_. 
### Threads and Signal Handling

Recall that signals are mediated by kernel and are handled by individual processes. The paper by Eykholt et al. describes a signal mask that is associated with each user-level thread (which are a part of the user-level process). However, there is also a signal mask associated with the kernel-level thread, or rather, the LWP that it is attached to. 

![[Pasted image 20230920191713.png]]

Suppose a user-level thread wants to disable a specific signal. It would then clear the appropriate bit in the signal mask. However, note that this is **happening at the user level** -- this mask is not visible to the kernel. When the signal occurs, how should the kernel infer what to do?

#### Case 1: Trivial Case

Let's first consider an easy case, where the mask at the kernel and user level had the signal enabled. When a signal occurs, the kernel sees that the signal is enabled. It will interrupt the currently executing user-level thread, which is also perfectly fine as the user-level thread had the signal enabled as well.

#### Case 2: Signaling Another ULT

Consider now the case where the kernel-level mask is one (i.e. the signal is enabled). However, the user-level thread that is currently running on top of the kernel-level thread has disabled the signal (mask = 0). There is another user-level thread that is waiting to be executed, but this particular thread has enabled the signal (mask = 1).

The user-level threading library that manages both of these threads will know about the two threads and their respective statuses.

![[Pasted image 20230920194021.png]]

If a signal occurs, the kernel simply assumes that the process is able to handle the signal. However, only the user-level library knows that there are two user-level threads with different signal-handling masks. It should therefore figure out a way to allow the thread with (mask = 1) to handle the signal. 
- Firstly, recall that all signal numbers are mapped to their respective handler starting address. It is thus possible to map all signals to a library handling routing (i.e. one that invokes the user-level threading library).
- In this case, when a signal occurs, the library-provided handler will first execute.
- The library will then notice that the currently executing user-level thread is not able to handle the signal, but there is another runnable thread that is able to.
- It should then schedule the second thread onto the kernel-level thread, where the proper handling routine can be executed.
- It is therefore fair to claim that the library handling routine _wraps_ the intended handling routine.
#### Case 3: Signaling Another KLT

![[Pasted image 20230920195704.png]]

Suppose that there are two kernel-level threads and two user-level threads executing on top of them. Only one of the user-level threads has the signal bit enabled (similar to the case above).

Suppose then that a signal occurs in the context of the kernel-level thread with the user-level thread that has disabled the signal (mask = 0). 
- The library has to then **generate a directed signal to the kernel-level thread** (or more correctly, the LWP where the user-level thread is currently executing).
- When the OS delivers this signal to the kernel-level thread, it sees that the signal mask is enabled.
- Technically, the library handling routine is called first.
	> The diagram above is slightly misleading here. The short width of the `thread_lib` box seems to imply that only the first KLT-ULT pair is handled by the threading library. This is not true. All user-level threads are maintained by the user-level threading library.
- The library handling routine will see that the associated user-level thread is able to handle the signal, and will finally call the appropriate signal handler.

#### Case 4: Setting the Kernel Mask

![[Pasted image 20230920202139.png]]

The final case considers both user-level masks to be 0 (signal disabled) while both kernel-level masks are 1 (signal enabled). 

As usual, once a signal is generated at the kernel level, the user-level threading library is first invoked. The user-level threading library however notes that there are no user-level threads that have the signal bit set to 1 (all of them are disabled). 
- The user-level threading library then **requests the kernel-level thread to set its own mask to 0**.
- The signal will then be reissued. The OS will have to find another kernel-level thread and the procedure will repeat. The kernel-level thread (with mask = 1) will invoke the library signal handler. The library signal handler notes that all user-level threads have disabled signals. It will request that particular kernel-level thread to change its mask value to 0.

There will come a point in time where the user-level threads are ready to handle signals once more.
- The threading library knows that it has already disabled all of the kernel-level signal masks, hence it will have to perform a system call and update one of the signal masks appropriately.
#### Final Notes

The solution for signal handling once again demonstrates the concept of optimising for the common case. 
- Signals themselves occur much less frequently compared to the need to update the signal mask (for example, the signal mask will be disabled then re-enabled when entering and exiting a critical section). 
- Therefore, the signal mask updates themselves are relatively cheap -- everything only occurs on the user level and no system calls are required.
- It is only the signal handling that becomes more expensive and more complex. \

## Tasks in Linux

### Task Data Structure

The abstraction that Linux uses to represent an execution context of a kernel-level thread is called a **task**. A single-threaded process will therefore have one associated task, while a multi-threaded process will have many tasks associated with it.

The following snippet shows key elements in the `tast_struct` that Linux provides. Note that the overall data structure is quite large -- not all information can be shown here:
```c
struct task_struct {
	// ...
	pid_t pid;
	pid_t tgid;
	int prio;
	volatile long state;
	struct mm_struct *mm;
	struct files_struct *files;
	struct list_head tasks;
	int on_cpu;
	cpumask_t cpus_allowed;
	// ...
}
```
1. A task is identified by its `pid` (a bit of an unfortunate misnomer due to legacy reasons). A single-threaded process will share its `pid` with the task's `pid`. A multi-threaded process will have tasks with multiple distinct `pid`s and the process itself will be identified by the `pid` of the first task created. This information is also captured in `pid_t tgid`.
2. A list of all tasks associated with a process is stored by each task under `struct list_head tasks`.
3. Pointers to other data structures are maintained, such as `struct mm_struct *mm` or `struct file_struct *files`. Linux never had one large process block, instead the multiple data structures are split up as described at the start of this current lecture.

### Task Creation

Linux supports an operation `clone` in order to create a new task. The function call is given below:
```c
clone(function, stack_ptr, sharing_flags, args);
```
Note that this is roughly similar to `pthread_create` or `fork`. This function however takes in an additional parameter `sharing_flags` which denotes the portion of a task that will be shared between the parent and child tasks.

| `sharing_flags` | meaning when set | meaning when cleared |
| -- | -- | -- |
| `CLONE_VM` | Create a new thread | Create a new process |
| `CLONE_FS` | Share unmask, root, and working dirs | Do not share unmask, root, and working dirs |
| `CLONE_FILES` | Share the file descriptors | Copy the file descriptors |
| `CLONE_SIGHAND` | Share the signal handler table | Copy the signal handler table |
| `CLONE_PID` | New thread gets old PID | New thread gets own PID |
| `CLONE_PARENT` | New thread has same parent as caller | New thread's parent is caller |

If all bits are set, a new thread is created where the state is shared with the parent thread. If all bits are cleared, nothing is being shared -- rather akin to creating a new process. In fact, `fork` in Linux actually calls `clone` with all `sharing_flags` cleared.

### Threading Model

Linux implements the **Native POSIX Threads Library** (NPTL). This is a 1:1 model; that is, one kernel-level thread for each user-level thread. This implementation replaces the earlier LinuxThreads, which implemented a many-to-many model.

NPTL uses the 1:1 model, affording the kernel to see **every user level thread**. This reduces the need for complex thread management. In recent years, kernel trapping has become much cheaper (kernel/user crossings are cheaper) and more memory is available -- removing the need for extreme constraints on the number of kernel-level threads.

However, if exascale computing or high-level processing is required (multiple heterogeneous processes, etc.), it makes more sense to look to custom threading policies to make systems more scalable. For most practical purposes, the 1:1 model is more than good enough.
