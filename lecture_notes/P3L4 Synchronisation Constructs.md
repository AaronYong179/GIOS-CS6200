## Introduction and Overview

Although we have discussed synchronisation multiple times before, there are still some concepts that have yet to be covered in details. 

For example, consider mutexes and condition variables.
- We have seen how the usage of these constructs are error-prone (forgetting to unlock a mutex, or signalling the incorrect condition variable).
- When discussing the readers/writers problem in [[P2L2 Threads and Concurrency]], a helper variable was required -- mutexes by themselves can only represent binary states. They are said to **lack expressive power**. In fact, with the synchronisation constructs encountered thus far, arbitrary synchronisation conditions _cannot be expressed_.

Finally, it is important to understand how synchronisation constructs require hardware support. This is provided in the form of **atomic instructions**, which will be examined in greater detail below.

This lecture will first introduce alternative synchronisation constructs, followed by a deeper dive into how we can achieve efficient implementations of synchronisation by leveraging hardware support.
## Examples of Synchronisation Constructions

### Spinlocks

The most basic synchronisation construct would be a **spinlock**. They are a synchronisation primitive that can be used to build more elaborate implementation, but discussion on this will continue at a later section in this lecture.

Spinlocks are similar to mutexes, but not exactly the same.
- Spinlocks provide mutual exclusion -- locking and unlocking a spinlock guarantees this.
- However, a spinlock does not block threads (as per mutexes) if it is currently locked. 
- A thread is allowed to 'spin' (i.e. run) on the CPU, repeatedly checking if the spinlock has become free.
- Recall that with mutexes, a thread encountering a locked mutex will relinquish the CPU and allow another thread to run.
- For spinlocks, threads will only relinquish the CPU if it is preempted (e.g., its time slice expired, or a higher priority thread entered the runqueue).
### Semaphores

Semaphores are a common synchronisation construct in OS kernels. 
> Semaphores were originally designed by E. W. Dijkstra, who was Dutch. As such, the initial operations for semaphores were termed `proberen`, which meant to "try" and `verhogen`, to "increase". 
> Semaphore operations may use the term "P" and "V" due to this historical reason.

Semaphores allow for the expression of **count-related** synchronisation requirements. 
- Semaphores are initialised (on `init`) with some integer maximum value. 
- Threads arriving at a semaphore (on `try`) with a non-zero value will decrement that value and are allowed to proceed. Threads arriving at a semaphore with a zero value will wait.
- Quite simple, the initialised maximum value would be the maximum number of threads allowed to run concurrently. 
- When a thread exits the critical section (on `post`), it will signal the semaphore which results in incrementing the semaphore's counter.

If a semaphore is initialised with a value of `1`, the semaphore behaves like a mutex. This type of semaphore is known as a binary semaphore.
#### POSIX API

The following code snippet shows a quick glance at the POSIX Semaphore API.
```c
#include <semaphore.h>

sem_t sem;
sem_init(sem_t * sem, int pshared, int count);
sem_wait(sem_t * sem); // try and/or wait
sem_post(sem_t * sem); // exit and post
```
- Note that if inter-process communication is desired, `pshared` must be set accordingly so that the semaphore is shared amongst different processes.

_Quiz_
_Complete the code snippet so that the semaphore behaves identically to a mutex used by threads within a process._

```c
#include <semaphore.h>

sem_t sem;
sem_init(sem_t * sem, 0, 1); // set pshared to 0, count to 1

sem_wait(sem_t * sem);
// critical section
sem_post(sem_t * sem); 
```
### Reader/Writer Locks

When specifying synchronisation requirements, it is sometimes useful to distinguish the different **access types** that a shared resource would have to deal with.

In this particular case, there can be **read accesses**, where the shared resource is never modified, as well as **write accesses**, where the shared resource might be modified.
- Therefore, multiple read accesses are allowed. This is also termed **shared access**.
- Only a single writer is allowed. This is termed **exclusive access**.

Operating systems (Windows) and language runtimes (Java, POSIX) support the notion of reader/writer locks. These are defined and used as per mutexes, although the type of access can be further specified -- the lock will then behave accordingly.

Do note that the implementation of reader/writer locks might differ across different platforms.
- For example, during recursive calls to `read_lock`, some implementations unlock all recursive locks, while some others only unlock the most recently acquired lock.
- Some implementations also allow for lock upgrading or downgrading. Briefly, a reader may "upgrade" its lock to a writer lock, though some implementations do not allow this behaviour -- a reader must unlock its reader lock then acquire a writer lock.
- The scheduling policies might also differ. For example, some implementations will block a reader if there is already a writer waiting, while others will allow the reader to acquire its lock.
#### API

The following shows a code snippet of reader/writer locks in Linux:
```c
#include <linux/spinlock.h>

rwlock_t m;
read_lock(m);
read_unlock(m);

write_lock(m);
write_unlock(m);
```
### Monitors

All the construct encountered thus far require developers to explicitly call the paired lock/unlock or signal/wait operations. Clearly, this introduces a point of fault, where most errors likely occur.

Monitors are a **high-level synchronisation construct** that hides the manual invocation of the synchronisation operations. A monitor typically specifies the shared resource, the entry procedures for accessing that resource, and any condition variables required.
- When invoking the entry procedure, all of the necessary locking and checking will occur behind the scenes.
- Similarly, when exiting a critical section, the unlocking and signalling (if any) is performed, hidden from the user.

Historically, monitors were included in the MESA language runtime (by XEROX PARC). 

Today, Java supports monitors as well; every single object in Java has an internal lock, and objects declared to be synchronised are entry points into the monitor. When compiled, all the necessary locking and unlocking will be added -- the only exception being `notify()` (signal), which has to be invoked explicitly.

As a side note, the term "monitors" may also refer to the programming style that uses mutexes and condition variables to describe critical section entry and exit.
- Therefore, the code encountered in [[P2L2 Threads and Concurrency]] can be considered to follow the monitors programming style.

### ...and More

This section quickly lists other synchronisation constructs without going into too much detail. Do note that all synchronisation constructs require some support from the underlying hardware to _atomically update a particular memory location_. This is in fact, the only way a lock acquisition (for example) is guaranteed to be safe and race-free. 
#### Serialisers
Serialisers make it easier to define priorities, and hide the need for explicit use of condition variables and signalling. 
#### Path Expressions
These require a specified regular expression that captures the correct synchronisation behaviour. As opposed to using locks, the programmer would specify something akin to 'Many Reads, Single Write' and the runtime will ensure that operations satisfy the regular expression. 
#### Barriers/Rendezvous Points
Basically reverse semaphores, where the barrier blocks _until_ a specified number of threads arrive. Rendezvous points also wait for a certain number of threads before proceeding.
#### Wait-Free Synchronisation
In order to boost scalability and efficiency metrics, there are efforts to achieve concurrency without the need to explicitly lock and wait. These wait-free synchronisation constructions are therefore termed **optimistic** -- they bet on the fact that there will not be any concurrent writes and it will be safe to allow multiple reads to proceed. An example of this is the **read-copy-update** (RCU) lock that is part of the Linux kernel.
## Spinlock Primitive

### The Need for Hardware Support

Recall that spinlocks are one of the basic synchronisation constructs that often serve as a building block for other, more complex constructs. As such, this section will predominantly focus on spinlock implementation, closely mirroring the paper [The Peformance of Spin Lock Alternatives for Shared-Memory Multiprocessors](http://content.udacity-data.com.s3.amazonaws.com/courses/ud923/references/ud923-anderson-paper.pdf) by Anderson, 1990.
- The paper discusses alternative implementations of spinlocks, and these concepts are indeed generalisable to other synchronisation constructs.
- The discussion on atomic instructions can also be used to inform techniques relating to other constructs.

_Quiz_
_Consider the following possible implementation of a spinlock:_
```
spinlock_init(lock):
  lock = free; // 0 = free, 1 = busy

spinlock_lock(lock):
  spin:
    if(lock == free) { lock = busy; }
    else { goto spin; }

spinlock_unlock(lock):
  lock = free;
```
_Does the implementation guarantee mutual exclusion? Is it efficent?_

The implementation is both inefficient and incorrect. 
- If the `lock` is set to `busy`, CPU cycles will be spent spinning until the `lock` becomes `free`. This is clearly inefficient.
- Multiple threads may also encounter the `if` statement at the same time, and will all proceed to set `busy` in their own execution contexts. 

_Consider this alternative implementation of a spinlock:_
```
spinlock_init(lock):
  lock = free; // 0 = free, 1 = busy

spinlock_lock(lock):
  while (lock == busy);
  lock = busy;

spinlock_unlock(lock):
  lock = free;
```
_Again, is this implementation efficient, and does it guarantee mutual exclusion?_

This implementation is also both inefficient and incorrect.
- The inefficiency comes from the fact that there is continuous spinning within the `while` loop as long as the `lock` is `busy`.
- Again, multiple threads may be stuck in the `while` loop until the `lock` becomes `free`, although at that point the multiple threads will exit the `while` loop and set `lock` to be `busy` concurrently.

The overall takeaway is that **software implementation is simply not enough** to handle mutual exclusion. There is always no guarantee that the checking and setting operations occur atomically (i.e. both must happen together, indivisibly). This issue is made worse if there are multiple CPUs, where multiple checks/updates can overlap concurrently. Therefore we _must rely on support from hardware_ for atomic instructions.
### Atomic Instructions

##### Hardware Support

Hardware will generally support a number of atomic instructions. These include:
- `test_and_set`
- `read_and_increment`
- `compare_and_swap`

Notice that the above perform some multi-step, multi-cycle operation. However, the hardware makes certain guarantees regarding the execution of these instructions:
- They will occur atomically. For example, both `test` and `set` will occur one after the other or not at all.
- They are mutually exclusive. Threads on multiple cores cannot perform the atomic instructions in parallel. All concurrent attempts will be queued and performed serially.

Consider the following implementation of a spinlock, this time using an atomic instruction:
```c
spinlock_lock(lock) {
  while(test_and_set(lock) == busy);
}
```
- Since `test_and_set` is atomic, it is guaranteed that the first thread encountering a `free` lock will manage to lock it successfully and correctly.
- All other threads will have to spin in the `while` loop.
##### Software Considerations

Note that different hardware will support different atomic instructions.
- `test_and_set` is widely supported on most hardware.
- `read_and_increment` might be implemented differently on specific hardware. For example, some may only allow for the `increment` operation, but does not return any value `read`. Some others might also implement an atomic `read_and_decrement`.

There will also be differences in terms of efficiency depending on the hardware used. `test_and_set` might be more efficient than `read_and_increment` on a specific hardware, for instance.

For the reasons outlined above, software implementations using these atomic instructions must be **ported**. In other words, the implementation must use instructions that are available on a target platform, and the most efficient atomics are to be used as well.
### Intermission: Shared Memory Multiprocessors

Before continuing the discussion on spinlock alternatives presented in Anderson's paper, a brief recap of shared memory multiprocessors are required. This concept has been covered briefly under [[P3L1 Scheduling]].
#### Shared Memory Units

A multiprocessor system consists of multiple CPUs, as well as memory unit(s) that is/are available to all CPUs. These shared memory multiprocessors are also referred to as symmetric multiprocessors, or simply as SMPs. There are two possible configurations:

![[Pasted image 20231020153955.png]]

- An interconnect-based configuration allows multiple memory references to be "in-flight" at any given moment. 
- A bus-based configuration only allows **one** memory reference to be "in-flight" at any given point in time. 
#### Cache and Memory

Each of the CPUs will also have their own cache. In general, cache access is much faster compared to memory access; caches are useful to hide memory latency.

![[Pasted image 20231020155834.png]]

- Memory latency is aggravated in shared memory systems, as there is contention on the shared memory module. Due to this contention, certain memory references are delayed, ultimately resulting in an increased latency.

When a CPU performs a write operation, multiple scenarios may happen:
1. A CPU may be disallowed from writing to cache in the first place. A write will therefore go directly to memory, and any cache references will be invalidated. This is known as a **no-write**.
2. A CPU may also be allowed to write to both cache and memory. This is known as a **write-through**.
3. A CPU may also write immediately to cache, and only perform memory writing at some later point in time (perhaps, when the cache entry is evicted). This is known as a **write-back**.
#### Cache Coherence

When multiple CPUs reference the same data, there may be some point in time where the data within their respective caches do not match. 
- For example, suppose a CPU changes the value of $X$ to $X'$. The value $x$ is still on the cache of the other CPUs.

On some architectures, this issue needs to be resolved in software. The hardware makes no attempt to update any changed value across multiple caches. These architectures are known as **non-cache-coherent** (NCC).

On **cache-coherent** (CC) architectures, the hardware will ensure that caches are coherent with one of the following strategies:
1. **Write-invalidate**. The write-invalidate strategy **invalidates** all cache entries of $X$ once a CPU updates its cached copy of $X$. Future references to invalidated entries will have to go to memory.
2. **Write-update**. The write-update strategy ensures that all cache entries of $X$ will be **updated** once a CPU updates its cached copy of $X$. 

There are potential upsides to both cache-coherent strategies.
1. Write-invalidate poses a lower bandwidth requirement on the shared interconnect within the system. 
	- The value of $X$ itself is not needed, rather only the address of $X$ is required to invalidate all other cache entries of $X$. 
	- Subsequent changes to $X$ on the same CPU will also require further invalidations; the cost is therefore amortised if $X$ is not needed on any other CPU anytime soon.
2. Write-update allows for the value of $X$ to be available on other CPUs immediately. 
	- Programs that would require access to $X$ would not have to pay the cost of going to memory.
	- Therefore, if $X$ is required on multiple CPUs back-to-back, write-update is likely a better strategy.

With all that said, the developer has no control over the policy used. This is determined by hardware.

#### Cache Coherence and Atomics

Suppose that there are two CPUs, each intending to perform some atomic operation regarding $X$. Both CPUs have a reference to $X$ present in their caches.

![[Pasted image 20231020162244.png]]

It is rather challenging to ensure cache coherence with atomicity. There may or may not be multiple CPUs with $X$ cached, there is no control over the cache coherence policy employed by the hardware, and there is even latency on the chip. Given these issues, atomic operations **always bypass the cache and are issued directly to memory**. 

By forcing atomic operations to go directly to the memory controller, this creates a central entry point where all references can be ordered and synchronised. In other words, race conditions that might occur if atomics are allowed to access cache would simply not happen.

This solves the correctness issue, though this also means that atomics is a slow operation. 
- Atomics will always have to access memory.
- Atomics will always have to contend for memory.
- Coherence traffic must be generated -- the updating or invalidating of all cache references must take place, regardless of whether or not $X$ changed (to err on the side of caution).

A few key takeaways from this intermission:
- Atomic instructions are more expensive on SMP systems than single CPU systems, due to bus or I/C contention.
- Atomic instructions in general are also more expensive as they bypass the cache for memory, and will always generate coherence traffic.
###### #DEFINE Coherence Traffic
> Coherence traffic is simply the data traffic on cache-coherent systems.

### Spinlock Performance Metrics

Having covered atomic instructions, cache coherence, and SMPs, we are now finally ready to discuss the design and performance of different spinlock implementations. These implementations will be evaluated based on the following metrics:

1. Low latency.
	- The time taken to acquire a **free** lock (i.e., latency) should be minimal and ideally zero.
	- In essence, it would be ideal for a thread to immediately acquire a free lock with a single atomic instruction.
2. Low delay/wait time.
	- The time taken for a thread to **stop spinning** and acquire a lock that has been freed should be minimal and ideally zero.
	- Again, a free lock should ideally be acquired immediately. The thread must stop spinning quickly.
3. Low contention.
	- Contention is defined here as being caused by atomic memory references and the subsequent coherence traffic.
	- Ideally, contention should be low on the shared bus or interconnect.
	- Contention will clearly negatively impact performance, especially if the release of a spinlock is delayed due to contention from a waiting thread.

_Quiz_
_Given the above performance metrics, are there any metrics which conflict with each other?_

(1) conflicts with (3). If low latency is required, then the atomic operation must occur immediately, regardless of the load currently on the system.
Similarly, (2) conflicts with (3). Since a low delay time is required, the thread must continuously spin and check if the lock is free. This is clearly bad for (3) since each check will add to contention. 
### Spinlock Implementations

#### Test and Set

```c
spinlock_init(lock):
  lock = free; // 0 = free, 1 = busy

spinlock_lock(lock):
  while (test_and_set(lock) == busy);

spinlock_unlock(lock):
  lock = free;
```

The `test_and_set` instruction is a very common atomic that most hardware platforms support. This particular implementation is therefore very portable.

1. Only a single atomic operation is performed -- latency is as good as it gets.
2. There is very minimal delay between a lock free and lock acquisition as well, since a waiting thread will continuously spin until it manages to acquire the lock.
3. However, from the contention perspective, this lock performs poorly. Threads will spin on the `test_and_set` instruction, which means that the processor will have to go memory on each instruction (remember that cache is bypassed). 

In essence, the issue with this implementation is the continuous spinning on the atomic instruction. Regardless of cache coherence, time is wasted going to memory upon each invocation of `test_and_set`.
#### Test and Test and Set

Since the issue with the previous implementation was that CPUs were left to spin on an atomic operation, it is feasible to suggest separating the checking of the lock value.

```c
spinlock_lock(lock):
  while( (lock == busy) OR (test_and_set(lock) == busy) )
```

The idea behind this is that the CPU can test the cached copy of the lock and only execute the atomic if its cached copy is `free`.
> An important concept here is that OR statements will short circuit if any of the predicates evaluate to True. Therefore, if the `lock == busy` predicate is True, the second half of the OR statement will not be executed.

This implementation is known as the **test-and-test-and-set** spinlock, or otherwise called spin-on-read or spin-on-cached.

1. From both a latency and delay standpoint, this lock is not too bad. It might be worse than test-and-set, due to the extra checking prior to the atomic operation.
2. From the contention perspective, this may vary.
	- If the architecture is non-cache-coherent (NCC), there is no difference since every memory reference will go to memory.
	- If there is cache coherence with the write-update strategy, then contention improves. However, all processors will encounter a free lock and **all of them will try locking at once**. This is evidently bad.
	- If there is cache coherence with the write-invalidate strategy, then contention is terrible. Every single attempt to acquire the lock will generate contention as well as create invalidation traffic. 

Recall that cache coherence is triggered regardless if any change had occurred.
- In the write-update situation, this isn't too big of an issue. If the lock remains busy after the write-update event, the CPU simply keeps spinning on the cached copy.
- However, in the write-invalidate situation, invalidation will occur even if the value itself has not changed. 
- As an example, suppose one CPU executes an atomic but encounters a busy lock. Nothing has changed, but since the cache coherence strategy is triggered, all other CPUs will have their cached lock invalidated and _all of them_ will have to go to memory.

_Quiz_
_In an SMP system with N processors, what is the complexity of memory contention (access), relative to N, that will result from releasing a test-and-test-and-set spinlock?_

CC (cache coherence) with write-update: O($N$)
- All executing processors will see that the lock is free and will all issue a `test_and_set` atomic. 
CC with write-invalidate: O($N^2$)
- Upon first free, all caches will be invalidated and all of them will have to go into memory.
- In the worst case scenario, all processors will encounter a `free` lock in the first test predicate and will _all_ call `test_and_set`. There will only be one successful `test_and_set`, but every processor attempts to execute it anyway. 
- This means that each processor will invalidate all other processor's caches.
- However, the other processors are spinning on the test predicate. Having their cached invalidated, they must all go to memory as well. This occurs for all N processors, each time (N times in total) a processor executes `test_and_set`.
#### Delay

```c
spinlock_lock(lock):
  while( (lock == busy) OR (test_and_set(lock) == busy) ) {
    while(lock == busy); // keep spinning
    delay(); // delay on exit
  }
```

The implementation above delays once a thread notices that a lock is `free`. This prevents every thread from executing the atomic instruction at the same time, thereby reducing contention.
- Delayed threads will try to re-check the value of the lock (outer `while` loop), and it is now fully possible that another thread had already executed the atomic.
- In other words, the event in which threads see a `busy` lock as `free` and attempt to execute an atomic will be reduced.

1. Latency-wise, this lock is alright. A memory reference is required to bring the lock back into cache (outer `while` loop) and then to perform the atomic. This is essentially similar to the test-and-test-and-set spinlock.
2. Clearly the delay metric has been negatively affected. Once a thread sees that a lock is free (upon exiting the inner `while` loop), it still has to waste time delaying before attempting to acquire the lock.

```c
spinlock_lock(lock):
  while( (lock == busy) OR (test_and_set(lock) == busy) ) {
    delay();
  }
```

This alternative implementation delays after each reference.
- This implementation works well on NCC architectures. Since a thread has to go to main memory on every reference, introducing some delay decreases the number of memory references required while the thread spins.
- Unfortunately, this worsens the delay metric.
##### Picking a Delay

There are two main strategies to picking a delay: (i) static delay or (ii) dynamic delay.

With a static delay, some fixed information can be used together with the length of the critical section, such as the CPU ID where the process is running. In other words, a CPU ID of 32 would result in a delay of 32 multiplied by the length of the critical section.
- This is an extremely straightforward approach that works well under high loads, where the atomic references will likely be spread out such that there is little to no contention.
- However, under low load (low contention), unnecessarily long delays can be introduced. For example, suppose two CPUs are running, one with an ID of 1 and another with an ID of 128. Although there is little contention, the CPU with ID 128 will have to wait a ridiculously long time.

Dynamic delays are the more commonly favoured strategy. 
- Each thread will take a random delay value from a range of possible delays, depending on the _perceived_ contention in the system.
- A higher load will see a range of higher delay values, and vice versa.
- Under high load, both dynamic and static delays will perform well, reducing contention within the system.
- To estimate the contention within a system, it is possible to track the number of failed `test_and_set` operations. The more these operations fail, the more likely it is that there is a higher degree of contention.
- A caveat: if the delay is called after each lock reference, the delay will grow both as a function of contention as well as **the length of the critical section**. If a large critical section is currently being executed, all other `test_and_set` operations will fail (hence delays will keep increasing) even though the contention in the system has not changed. There must be safeguards against this case.
#### Queueing Lock

The delay was introduced to guard against the case where every thread tries to acquire a freed lock. Alternatively, if we prevent all threads from _seeing_ that the lock is free at the same time, this directly avoids the aforementioned situation.

Enter the **queuing lock** (as proposed by Anderson), which controls the threads that "see" a free lock.

![[Pasted image 20231020194442.png]]
> `hl` represents `has_lock`, while `mw` represents `must_wait`.

This lock works as follows:
- An array of flags up to `N` elements are used, where `N` is the number of processors in the system. 
- Each element will have one of two values, `hl`, or `mw`. 
- There will be two pointers, one to indicate the current lock owner (which must have the value `hl`), and one to indicate the last element of the queue.
- A newly arriving thread will be placed after the last element of the queue (i.e., assigned a **unique ticket**). Since multiple such threads may arrive at the same time, it is important to increment `queuelast` atomically. This uses the `read_and_increment` atomic, though note that this is not as widely supported as `test_and_set`.
- A thread will only be allowed to attempt locking once their flag is set to `hl`. As such, a thread exiting a critical section must set `queue[ticket + 1] = hl`.
- Notice that this lock requires O($N$) space, which is much larger than the single memory location required for the previous locks encountered.
##### Implementation

```
init:
  flags[0] = has-lock;
  flags[1 ... p-1] = must-wait;
  queuelast = 0; // global variable

lock:
  myplace = r&inc(queuelast); // get ticket, read and increment
  // spin
  while (flags[myplace mod p] == must-wait);
  // now in critical section
  flags[myplace mod p] = must-wait;

unlock:
  flags[myplace+1 mod p] = has-lock;
```
- Initially, the first element is set to `has-lock`, and all other elements are set to `must-wait`. 
- A thread arriving at the lock must first get its ticket with the `read_and_increment` atomic. It must spin if its current ticket is flagged as `must-wait`, otherwise it may proceed into the critical section. Once it is done with the critical section, it must set its current ticket to `must-wait` in order to prep for future tickets.
- Unlocking simply involves changing the flag of the next element to `has-lock`, signalling that the lock is now free and ready for acquisition.

1. Latency-wise, this lock performs quite poorly. A more complex operation, `read_and_increment` is performed, which takes more cycles than `test_and_set`. A modular shift is also required in order to index into the `flags` array, and this occurs even _before_ the decision to spin or acquire the lock. 
2. Delays are good. Each lock holder will directly signal the next element in the queue when the lock is freed. All waiting threads are also constantly spinning, waiting for their turn.
3. Contention-wise, this lock is much better than any of the other alternatives discussed.
	- The atomic is only executed once and is not part of the spinning code.
	- Further, the atomic operation and the spinning code involve separate variables (hence different memory addresses), thus cache invalidation on the atomic operation does not impact spinning on local caches
	- However, this necessitates a cache coherent architecture.
	- Each element of the queue must also be on a separate **cache line**. If elements are on the same cache line, changing the value of one would invalidate (potentially) the caches of other processes spinning on other elements.

In brief, this lock enforces the fact that only one CPU/thread sees that the lock is free and then tries to acquire the lock. By having what is essentially a "private lock", only one thread at a time can see that the lock has become free.

_Quiz_
_Assume we are using Anderson's queueing spinlock implementation where each array element of the queue can have one of two values, has-lock (0) and must-wait (1). If the system has 32 CPUs, how large is the array data structure?32 bits, 32 bytes, or neither?_

Answer: Neither. This is a bit of a trick question. Recall that in order for the queuing implementation to work correctly, the elements must be in a different cache line.
- The size of the data structure will therefore depend on the size of the cache line.
- For example, with a cache line of 64 bytes, the size of the data structure would be 32 multiplied by 64 bytes.
### Spinlock Performance Comparisons

![[Pasted image 20231020201054.png]]

The figure above is pulled from Figure 3 of Anderson, 1990. 
- A program with multiple processes was executed. 
- Each process executed a critical section in a loop, one million times.
- The number of processes in the system was varied such that there was only one process per processor. (i.e. number of processors = number of _processes_).
- The platform used was [Sequent's Symmetry](https://en.wikipedia.org/wiki/Sequent_Computer_Systems) with 20 processors, hence the maximum number of processors was capped at 20. This platform is **cache coherent with write-invalidate**. 
- The metric computed was _overhead_. A theoretical limit (assuming no delay or contention) of the execution time was used. The _overhead_ was then defined as the **difference between the observed execution time and this theoretical limit**.
- The different lock types previously discussed were tested, except for the primitive `test_and_set` lock, which would easily perform the worst.

Under heavy load:
- The queueing lock performs the best. Overhead does not increase even with more processors (i.e. more contention).
- The `test_and_test_and_set` (spin on read) lock performs the worst. This platform uses write-invalidate to maintain cache coherence, and this forces O($N^2$) memory references as discussed above.
- Comparing the delay-based alternatives:
	- The static locks perform better than the dynamic locks, since the dynamic locks have some measure of randomness and will have more collisions as a result.
	- Delaying after each reference ("ref") is better than delaying after each release; delay after reference avoids certain invalidations (since the platform is write-invalidate).

Under light load:
- The `test_and_test_and_set` (spin on read) lock performs rather well, since it has low latency. 
- The queuing lock performs the worst here, since there is higher latency involved in the form of the more complex atomic `read_and_increment`, as well as the additional computation required (modular indexing, etc.).
- Comparing the delay-based alternatives:
	- The dynamic locks now perform better than the static ones. Static locks may enforce unnecessarily large delays under small loads, while dynamic delays can self-adjust.

Again, there is no clear answer as to "which is the best" strategy or design. It depends on the workload, the platform used (write-invalidate or write-update?), and so much more.

