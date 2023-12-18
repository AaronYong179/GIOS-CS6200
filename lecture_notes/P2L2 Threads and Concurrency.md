---

---
## Definition

Previously in [[P2L1 Processes and Process Management]], we saw how processes can be represented by their address space and Process Control Block (i.e., the **execution context of a process**). The process that is represented in this manner can only be executed on **one CPU** at any given point in time.
###### #DEFINE Execution Context
> An execution context, otherwise known as the **process state**, is internal data by which the OS is able to supervise and control the process. This includes the **state of the processor, which is the value of its program counter and all of its registers**. It also includes the **process' memory map**.

For a process to utilise the multi-core systems of today (multi-CPU), the process needs to have **multiple execution contexts**. We term such execution contexts **threads**. More formally,
###### #DEFINE Thread
> A thread is a **single unit of execution of a process**.

A supplemental paper by [Birrell, 1989](https://s3.amazonaws.com/content.udacity-data.com/courses/ud923/references/ud923-birrell-paper.pdf) describes threads in a fundamental manner, and will be the theoretical basis of this section of the course.

The following are a few key characteristics of threads:
- A thread is an **active entity**. More specifically, a thread executes a unit of work on behalf of a process.
- A thread is able to work simultaneously with other threads. 
- A thread requires **coordination** regarding access to underlying platform resources (such as I/O devices, CPUs, memory, etc.). These must be carefully controlled and scheduled by the OS.

## Process vs. Thread
### Single-Threaded Process Representation

Recall that a process is represented by its address space. The address space contains the virtual to physical address mappings for the process (code, data, heap, etc.). The process is also represented by its execution context, that contains information about values of registers, program counter, stack pointer, etc. The OS represents all this information in Process Control Block.

![[Pasted image 20230923150122.png]]

Importantly, while we are on the subject of threads, the above is a representation of a **single-threaded process**.
> In [[P2L1 Processes and Process Management]], there was some confusion regarding representing processes with address spaces and/or PCB. This course seems to take the view that the PCB contains information regarding the execution context of a process (program counter, etc.) *as well as* the address space.
### Multithreaded Process Representation

Threads, however, represent **multiple, independent execution contexts**. They are part of the same address space (i.e., they share all the virtual to physical address mappings). Therefore, they share the same code, data, and files.

Due to their independent execution contexts, each thread will need to have a different program counter, stack pointer, stack, thread-specific registers, etc. Hence, for each thread, a separate data structure must be available to represent this **per-thread information**. 

The OS representation of such a **multithreaded process** will be as follows:

![[Pasted image 20230923150239.png]]

## Multithreading

### Parallel vs. Specialised Threads

Threads are useful precisely due to their ability to run at the same time on different processors.

1. Parallelisation
   - If threads execute the same code (think matrix multiplication, where a large problem can be subdivided into multiple, smaller, similar problems), then **speed up** is achieved.
   - Of course, remember that threads all have their own execution context. Although the threads are executing the same code, they may be executing _different parts_ of the code at any given point in time.

2. Specialisation
   - Threads can also operate on different portions of the code that correspond to different functions. For example, one thread can handle I/O tasks like display rendering while another handles computation.
   - By virtue of thread specialisation, it is possible to differentiate how these threads are managed. A certain thread executing an important task could be given a higher priority to CPU resources.
   - More importantly, if a thread **only ever executes a specific task**, then more of its state will be present within the processor cache. This results in a hotter cache, which directly translates to performance boosts.
### Multiprocessing vs. Multithreading

Multithreading is preferred over multiprocessing, as **processes require their own, individual address space and execution context**. If, for example, four processes are running, we would need four address spaces and four execution contexts.

![[Pasted image 20230923150430.png]]

A multithreaded process results in **a shared address space**, hence less memory is required. Of course, each thread still requires its own execution context, but multithreading clearly has **lower memory requirements** than its multiprocessor alternative. 

![[Pasted image 20230923150444.png]]

A multithreaded process is therefore more likely able to fit within memory. **Less swaps from disk are required**. Finally, inter-process (IPC) communication is costly. Multithreading sidesteps this issue altogether, since all threads share the same address space.
### Benefits of Multithreading

In sum:
- Threads speed up the execution of a process, especially if the process can be parallelised.
- Threads can be specialised for a specific task, which allows for a greater management (i.e., more important threads can be given more priority). Due to specialisation, threads also benefit from hotter processor caches.
- Multithreading itself uses less memory compared to multiprocessing, as all threads within a process share the same address space.
- The lower memory requirement results in less swapping to/from disk, which leads to an overall speed up.
- Communication between threads is not an issue, since threads of the same process share an address space.
#### Single CPU Multithreading

A valid question would be: are threads useful on a single CPU? More generally, **are threads useful when the number of threads exceeds the number of CPUs?**.

Consider the following image:

![[Pasted image 20230923150639.png]]

- Suppose that T1 reaching a point in execution where it makes a disk request.
- During the time it takes for the disk request to complete (the disk needs to move its disk spindle to the correct position and return the appropriate response), T1 is simply **waiting**. Denote this time $t_{idle}$.
- If $t_{idle}$ is longer than the time it takes to context switch to and back from another thread (T2), then it makes sense to allow T2 to perform something useful with the CPU.
- More specifically, if $t_{idle} > 2*t_{ctx\_switch}$, then context switching should occur to hide any idling time. In plain English, if a thread is idle for more time that it would take to perform two context switches, then context switch please.

Recall that context switching among processes is expensive, due to the need to load the PCB for the new processes. Importantly, the **need to copy the virtual to physical memory mappings** is expensive -- **threads completely bypass the need for this** in particular.

Given that threads share the same address space, there is no need to copy a new virtual to physical memory mapping. Therefore, the time it takes to context switch between threads is much faster than that of processes. 
$$t_{ctx\_switch} \text{ threads} < t_{ctx\_switch} \text{ processes} $$
Since $t_{ctx\_switch}$ for threads is small, there will be more situations where context switching to another thread makes sense. Multithreading is especially useful due to this ability to **hide more of the latency associated with I/O operations**, even on a single CPU.
#### Multithreading the OS

By multithreading the OS kernel, we allow the OS to support multiple execution contexts. Of course, having multiple CPUs fully realises the potential of multithreading for the OS. 
- The OS threads may run on behalf of certain applications, or
- The OS threads may run OS-level services such as daemons or drivers.

_Quiz: Process vs. Thread_

| statement                                           | P/T/B   |
| --------------------------------------------------- | ------- |
| can share a virtual address space                   | thread  |
| take longer to context switch                       | process |
| have an execution context                           | both    |
| usually result in hotter caches when multiple exist | thread  |
| make use of some communication mechanism            | both    |
> We've not really covered inter-thread communication at this point.

## Thread Mechanisms

### Threads and Thread Creation

#### Overview

Before diving into thread mechanisms, consider briefly a "thread" data structure. The data structure should allow us to **identify threads** and to **keep track of a thread's resource usage**.

Mechanisms regarding threads should be able to use these information to:
1. **Create** and **manage** threads, as well as 
2. **Safely coordinate** threads running concurrently in the same address space (we'll get to this last statement later).
#### Thread Data Structure

Note that this is a purely theoretical introduction into threads. A more concrete example will be given in the next lecture about `pthreads`.

A thread data structure should contain:

![[Pasted image 20230923151126.png]]
- information specific to the thread (e.g., *threadID*), and
- information that *describes* the thread (e.g., *program counter, stack pointer*).
> The additional `attributes`  section could be used by thread management systems to schedule threads or help debug threads, etc.
#### Thread Creation

Thread creation proceeds by `fork(proc, args)`. Note that this is distinct from the process fork encountered in the process module.

When a thread `T0` calls fork, a new thread data structure `T1` is created. `T1` fields are initialised such that its program counter points to first instruction of the procedure `proc` and the relevant arguments `args` will be available on the thread's stack.

After the `fork` operation completes, the process has **two threads**, `T0` and `T1`. `T0` will execute the next operation after the `fork` call, while `T1` concurrently executes the first instruction of `proc`. 

![[Pasted image 20230923151220.png]]

When the forked thread `T1` completes, there needs to be a way to notify the main thread `T0` of the result (or an error, if any occurred). This mechanism is known as `join(thread)`.  

If the main thread `T0` calls `join`, the main thread blocks (waits) until the child thread `T1` completes and returns its result. Once the child thread is `join`-ed, the resources allocated for the child thread (data structure, state, etc.) will be freed and the child is at that point terminated.

Note that while a parent thread may be able to `join` a child thread, threads are exactly the same in all other aspects. They can all access the same resources that is available to the parent process. This includes hardware resources such as CPU, memory, etc., or other resources such as process state. 


The following shows a pseudocode example of thread creation and joining:

```
Thread thread1;
Shared_list list;
thread1 = fork(safe_insert, 4);
safe_insert(6);
join(thread1); // Optional
```
1. Suppose that the operation `safe_insert` safely inserts the number into a shared `list`.
2. When the main thread calls `fork`, a new thread is created with the task of inserting `4` into the shared `list`.
3. The main thread itself proceeds to perform `safe_insert(6)`, as it is the next instruction after `fork`. 
4. Note however, that the order of execution is not guaranteed. The CPU may decide to work on `thread1` before the main thread, in which case `4` is inserted prior to `6`. It is also possible that the main thread executes first, inserting its `6` before `thread1`'s `4`.

We've thus encountered an issue related to multithreading and shared resources, which will be expounded upon in the next section. We will then formalise **thread coordination mechanisms** to hopefully alleviate these issues.

### Thread Synchronisation

#### Concurrency and Related Issues

Consider concurrency from the perspective of processes first. Recall that processes operate within their own address space. The OS and the underlying hardware ensures that **unwarranted access from different address spaces** will be disallowed.

The below image shows this succinctly:

![[Pasted image 20230923152245.png]]

- Suppose that physical address X belongs to the address space of Process 1 (`PhAddr_p1`).
- In that case, the mapping between `VA_p1` (virtual address of P1) and `PAx` (physical address X) is valid.
- On the other hand, there will not be any valid mapping between `VA_p2` (virtual address of P2) and `PAx`. The OS guarantees this.
- Hence, from the P2 address space, it is not possible to access `PAx`. 

Threads, however, share the same address space. This means that two concurrent threads (say, T1 and T2) can both legally access the same physical memory.

![[Pasted image 20230923152257.png]]

This poses a few issues, especially if both threads try to access the same physical memory at the same time.
- For example, if T1 is reading data from `PAx` while T2 is writing data to `PAx`. T1 would simply read garbage data.
- Alternatively, if both T1 and T2 are updating data in `PAx`, we might lose data due to overlaps. 

###### #DEFINE Data Race
> A data race occurs when **two threads access the same mutable object without synchronisation**. This is a common problem encountered when multithreading.

###### #DEFINE Race Condition
> A race condition occurs when the **order of execution affects the correctness of the program**. 
#### Synchronisation Mechanisms

As such, the following **thread synchronisation mechanisms** are to be used:

1. Mutual Exclusion
   - Mutual exclusion involves giving exclusive access (i.e., **one thread at a time**).
   - Other threads trying to access the same item must simply wait for their turn.
   - The actual operation itself that must be performed in mutual exclusion might be some update to a state, or a shared data structure among the threads.
   - A **mutex** is commonly used to implement mutual exclusion.

2. Waiting
   - It is also useful for threads to have a mechanism to **wait** on one another.
   - Threads should also be able to specify a condition before they are allowed to proceed (_what_ exactly are they waiting for?). This type of inter-thread coordination is to be handled by a **condition variable**.
     > For instance, a thread handling shipment of orders must wait till all items are processed before any particular order can be shipped. It doesn't quite make sense for the thread to keep checking if the order has been finalised. Rather, it should wait until explicitly notified that the order is ready to be shipped out.

3. Signaling
   - We will not cover this right now.
   - This mechanism involves waking up other threads from a wait state.
#### Mutexes
##### Overview

We've seen in the above section how issues with concurrency can occur, especially if threads are mutating a shared resource. There is therefore a need for **mutual exclusion** between executing concurrent threads.

Most operating systems and threading libraries support a construct known as a **mutex**.
###### #DEFINE Mutex
> A mutex is a **lock-like construct** that should be used whenever accessing data or state shared among threads.

When a thread "locks" (or "acquires") a mutex, it has exclusive rights to the shared resource protected by the mutex. Other threads attempting to "lock" or "acquire" the same mutex will be **blocked**.
###### #DEFINE Thread Blocking
> A blocking thread simply refers to a thread that is waiting. In this specific case, it needs to wait for its turn to acquire the mutex.

Clearly, a mutex data structure should contain the following:
- state (locked or freed?)
- owner (current thread locking)
- blocked_threads (threads currently waiting to lock the same mutex)

The portion of code protected by the mutex is known as the **critical section**. 
###### #DEFINE Critical Section
>  The critical section corresponds to any particular code that requires **only one thread** be performing the operation at any given time. 

A critical section could thus include updating a shared global counter, or modification of a shared list, for example. Of course, threads are free to execute concurrently in other parts of the code.
##### Mutex Lock/Unlock

Pseudocode for mutex locking is shown below:
```
lock(mutex) {
    /* Critical Section */
} // unlock
```
- When a thread encounters a critical section, it should lock the mutex to ensure mutual exclusion.
- After completing the execution of the critical section, it should then **free** the mutex (unlock), allowing other blocked threads to lock the mutex instead.
  > Note that this lecture assumes that unlocking will occur after the end curly braces of `lock` is encountered. However, in most implementations, an explicit call to `unlock` is required. 
  > For simplicity's sake, we will assume that `unlock` is automatically called once the critical section is complete.
- Note that the order in which threads acquire the mutex is not guaranteed. It does not matter if `thread2` blocks before `thread3`, the mutex may very possibly be acquired by `thread3` before `thread2`.

_Quiz: Mutex_

Any thread that has blocked for access to the mutex *before* the mutex has been freed will possibly be given access to the mutex. There is no ordering guaranteed by the mutex construct. 
#### Condition Variables
##### Motivation

A mutex involves a binary operation. A resource is either free (thus available for access) or busy (thus waiting is required). However, it is also possible to require **mutual exclusion only under certain conditions**.

Consider the following **producer/consumer scenario**:

![[Pasted image 20230923152822.png]]
- Suppose we have multiple producer threads that insert their `thread_id` into a shared list.
- We also have one separate consumer thread that prints the contents of a full list before clearing the entire list.
- In this example, the consumer thread should only access the shared list when the condition "list is full" is true.

Consider first the following pseudocode for the producer/consumer scenario, using only the binary mutexes.

```
// main
for i in range(0, 10):
    producers[i] = fork(safe_insert, NULL) // create producers
consumer = fork(print_and_clear, my_list) // create consumer

// producers: safe_insert
Lock(m) {
    my_list -> insert(my_thread_id)
} // unlock

// consumer: print_and_clear
Lock(m) {
    if (my_list.is_full) -> my_list.print_and_remove_all()
    else -> release lock and try again later
} // unlock
```

Note that in this implementation, the consumer `print_and_clear` thread has to continuously acquire the mutex, immediately exit (if `my_list` is not yet full), and try to reacquire the mutex once more. This is **extremely wasteful**, and it would be much more efficient if there was a way to notify that the list is full -- it should now try to acquire the mutex.
##### Waiting and Signaling

The **condition variable** construct should therefore be used **in conjunction with mutexes** to control the behaviour of concurrent threads. Consider now the following implementations of producer and consumer threads:

```
// consumer: print_and_clear
Lock(m) {
    while (my_list.not_full) {
        Wait(m,  list_full) // wait for condition list_full
    }
    my_list.print_and_remove_all()
} unlock

// producer: safe_insert
Lock(m) {
    my_list.insert(my_thread_id)
    if (my_list.is_full) -> Signal(list_full)
} // unlock
```

1. Consumer `print_and_clear`:
   - If the consumer acquires the mutex and realises that the condition `list_full` is not satisfied, it should suspend itself and wait for the condition to be true.
   - Notice here that the `Wait` operation takes in a mutex as an argument. This is because while the consumer is waiting, it **must release the mutex** in order to allow other producer threads to modify the shared `my_list`.
   - Similarly, once the `Wait` operation exits, it **must reacquire the mutex** in order to perform `print_and_remove_all`, which is obviously considered a critical section. 
   - In short, the `Wait` operation takes in a mutex and the condition variable as arguments. Internally, the `Wait` operation must ensure that the mutex is released and reacquired upon entry and exit respectively.

2. Producer `safe_insert`:
   - The producers then have the responsibility of checking if their insertion caused `my_list` to satisfy the condition `list_full`. 
   - If this is true, they should `Signal` the condition `list_full`. This `Signal` operation is primarily for the thread that is waiting on the condition variable (in this case, the consumer).
##### Condition Variable API

An API that allows for condition variables must have the following:
- Naturally, a condition variable **type**.
- The condition variable data structure itself should be able to contain information regarding (i) a list of waiting threads, as well as the reference to the mutex.
- A condition variable API must implement a **wait procedure**, which takes in a mutex and a condition variable.
- The wait procedure, as discussed above, must be able to release and reacquire the mutex depending on the condition variable.

```
Wait(mutex, condition) {
    // atomically release mutex and place thread on wait queue
    // wait
    // condition satisfied
    // remove thread from wait queue
    // reacquire mutex
    // exit wait
}
```

* A condition variable API must implement a **signal procedure**, which allows a thread to signal another waiting thread that the condition has been met.
* A condition variable API may also implement a **broadcast procedure**, which allows a thread to signal **all other waiting threads** that a condition has been met.
	* Broadcasting may not always be useful. Since the mutex will be immediately locked when any of the waiting threads acquire the mutex, we can still only execute one thread at a time (the rest woke up for nothing!).
#### Readers/Writer Problem

In this section, a case study involving multiple reader/writer threads will be studied in conjunction with the use of mutexes and condition variables.

At any given point in time:
- Zero or more reader threads can access a shared resource, but
- Only zero or one writer threads can access the resource concurrently at the same time.
- Of course, reader and writer threads cannot access the shared resource at the same time.

A naive approach to this problem would be to simply lock the shared resource with a mutex. This solution is too restrictive, since mutexes only allow **one thread to access the critical section at any given time**. This impedes the ability of multiple readers to access the shared resource at the same time.

In order to solve this problem, consider the following cases first:
1. If the number of readers is 0 and the number of writers is also 0, then both read and write operations can take place.
2. Otherwise, if the number of readers is more than 0 (i.e. 1 and above), then only read operations can take place.
3. If there is 1 writer, then neither read nor writer operations can take place.

We can therefore express the **state of the shared resource** using a separate variable as follows:
1. If the shared resource is free, then `resource_counter = 0`.
2. Otherwise, if reader threads are reading the shared resource, the `resource_counter > 0`. The resource counter should be the current number of reader threads currently accessing the shared resource.
3. If a writer thread is writing to the shared resource, then `resource_counter = -1`, reflecting that only one writer thread is allowed access.

Then, the **state of the shared resource** (not the resource itself!) should be locked by a mutex. In other words, we enforce that only one thread at a time can update `resource_counter`. 

As long as any access to the shared resource is first reflected by an update to the proxy expression `resource_counter`, locking the `resource_counter` will enable the coordination of different accesses to the shared resource.

Concretising the above:

```c
Mutex counter_mutex;
Condition read_phase, write_phase;
int resource_counter = 0;


/* READERS */
Lock(counter_mutex) {
    while (resource_counter == -1) {
        Wait(counter_mutex, read_phase);
    }
    resource_counter++;
} // Unlock;
// ... read data ...
Lock(counter_mutex) {
    resource_counter--;
    if (resource_counter == 0) {
        Signal(write_phase);
    }
} // Unlock;


/* WRITERS */
Lock(counter_mutex) {
    while (resource_counter != 0) {
        Wait(counter_mutex, write_phase);
    }
    resource_counter = -1;
} // Unlock;
// ... write data ...
Lock(counter_mutex) {
    resource_counter = 0;
    Broadcast(read_phase);
    Signal(write_phase);
} // Unlock;
```

- First thing to note: the actual access of the shared resource is indicated by the lines labelled `... read data ...` and `... write data ...`. These operations are outside any lock construct, both in the case of readers and writers. 

- However, we ensure that _prior_ to accessing the shared resource, both the readers and writers must perform a controlled operation where the `resource_counter` variable is updated. 

- Once a reader or writer is done with the shared resource, they should update the `resource_counter` accordingly and check if other threads need to be signalled. For example, if the final reader is done with the shared resource, it should `Signal` threads waiting for the `write_phase` condition variable.

- If a writer thread is woken up, it will be removed from the waiting queue and will attempt to acquire the `counter_mutex`. However, it might be the case that another thread managed to update the `resource_counter` first. Hence, the `Wait` operation is nested within a `while` loop, as it has to continuously check if the `resource_counter` has not been updated before moving on to update `resource_counter`.

- Once the writer thread is done, it will `Broadcast` to threads waiting on the `read_phase` condition variable (this makes sense, multiple reader threads can access the shared resource) and will `Signal` threads waiting on the `write_phase`. 

- A disclaimer, though both `Broadcast` and `Signal` are used when the writer thread completes writing, there is no guarantee regarding the order of threads that will be executed. A writer thread may acquire the mutex first, or a reader thread might -- it depends on how the scheduler handles these operations.
##### A General Critical Structure

The typical critical section structure can therefore be generalised as follows:

```
Lock(mutex) {
    while (!predicate_indicating_access_ok)
        Wait(mutex, condition_variable)
    update state => update predicate
    signal and/or broadacst
        (condition_variable_with_correct_waiting_threads)
} // Unlock;
```

For situations such as the readers/writer problem where a **proxy variable** is used, the typical critical section structure will be as follows:

```
// ENTER CRITICAL SECTION
perform critical operation (read/write)
// EXIT CRITICAL SECTION
```

The enter/exit will in turn take on these familiar forms:

```
// ENTER CRITICAL SECTION
Lock(mutex) {
    while(!predicate_for_access)
        wait(mutex, condition_variable)
    update predicate
} // Unlock;

// EXIT CRITICAL SECTION
Lock(mutex) {
    update predicate
    signal/broadcast(condition_variable)
} // Unlock;
```
#### Common Pitfalls

##### Brief Comments

While we will investigate two pitfalls in greater detail later, this section will briefly comment on some good habits to follow when dealing with concurrent threads.

1. Ensure that mutex/condition variables used with a shared resources are tracked properly. Code comments help here.

2. Do not use different mutexes to access a single resource. This defeats the purpose of mutual exclusion.

3. Remember to lock and unlock a mutex. Remember also to protect the shared resources **everywhere that it is being used**. Compilers may generate warnings or errors here to aid us.

4. Make sure that signalling or broadcasting is performed on the correct condition.

5. Make sure not to signal when a broadcast was intended. Note that the opposite is alright, though it may incur a performance penalty.

6. Remember that the order of thread execution is not guaranteed. This also applies to threads woken up by signals or broadcasts.

##### Spurious Wake-Ups

This pitfall may not necessarily impact correctness but will most likely affect performance. Take for example the readers/writer problem encountered in the above section. Relevant snippets are placed here for convenience.

```c
// WRITER EXIT CRITICAL SECTION
Lock(counter_mutex) {
    resource_counter = 0;
    Broadcast(read_phase);
    Signal(write_phase);
}

// READER ENTER CRITICAL SECTION
Lock(counter_mutex) {
    while(resource_counter == -1) {
        Wait(counter_mutex, read_phase);
    }
}
```

- Suppose that a writer thread is exiting its critical section. The writer acquires the `counter_mutex`, updates the `resource_counter` proxy variable, and sends `Broadcast` and `Signal`.

- When `Broadcast` is issued, reader threads can now be removed from the wait queue for the `read_phase` condition variable, potentially even before the writer releases the mutex.

- These reader threads will now attempt to acquire the `counter_mutex`, though they are unable to do so since the writer has not freed the `counter_mutex`. These threads will then be placed on the queue associated with acquiring the counter mutex.

- In summary, the reader threads were woken up from one queue (associated with the condition variable), only to be promptly placed on another queue (associated with acquiring the mutex)!

- Cycles will be wasted context switching to threads that will ultimately be placed back onto another queue.
###### #DEFINE Spurious Wake-Ups
> Also known as **unnecessary wake-up**, this occurs when **threads are woken up when there is no possible way to proceed**.

There is a possibility of unlocking the mutex *before* sending out a `Signal` or `Broadcast`. In the case of the writer thread, this is perfectly fine:

```c
// OLD WRITER EXIT CRITICAL SECTION
Lock(counter_mutex) {
    resource_counter = 0;
    Broadcast(read_phase);
    Signal(write_phase);
}

// NEW WRITER EXIT CRITICAL SECTION
Lock(counter_mutex) {
    resource_counter = 0;
} // Unlock;
Broadcast(read_phase);
Signal(write_phase);
```

However, in the case of reader threads, it is not possible to unlock the mutex first, since the call to `Signal` conditionally depends on the `resource_counter` being equal to `0`.

```c
// READER EXIT CRITICAL SECTION
Lock(counter_mutex) {
    resource_counter --;
    if (counter_resource == 0) { // depends on shared counter_resource
        Signal(write_phase) // enclosed within an if-statement
    }
} // Unlock;
```
##### Deadlocks

###### #DEFINE Deadlock
> A situation where two or more threads are waiting on each other to complete, but none of them ever do.

Consider the following example, where two threads `T1` and `T2` require access to shared resources `A` and `B` in their execution. Basically, for each thread to complete, they must first obtain mutexes for `A` and `B`, then perform their respective operations.

![[Pasted image 20230923153742.png]]

- It is possible that `T1` obtains the mutex associated with `A` then seeks to obtain the mutex for `B`. However, at the same time, `T2` obtains the mutex for `B` and now seeks the mutex for `A`.
- Basically, in order of mutex acquisition:
	- `T1`: `acquire(mut_A)` -> `acquire(mut_B)`
	- `T2`: `acquire(mut_B)` -> `acquire(mut_A)`
* However, `T1` will never be able to lock the mutex for `B`, since `T2` is holding it. Similarly, `T2` will never be able to lock the mutex for `A`, since `T1` is holding it.
* Importantly, none of the threads will ever reach a situation where they release whatever mutex they have acquired, since they are *still waiting* to acquire each other's mutex.

There are a few possible solutions to the abovementioned problem:

1. Lock `A`, unlock `A`, then proceed to lock `B`. 
   * However, this does not work in this scenario since both `A` and `B` are required at the same time.

2. Acquire all locks `A` and `B` (or a mega lock for both `A` and `B`), then release all of them when done.
   * This should work as a solution, but it may be too restrictive for some applications. 
   * This severely limits the amount of parallelism that is able to exist.

3. Maintain a **lock order**. In other words, threads must lock the mutex associated with `A` before locking the mutex associated with `B` (the order doesn't really matter here; what matters is that *an order exists in the first place*).
   * This is by far the most common solution used, since a lock order necessarily guarantees that deadlocks will be avoided.

In summary, 

- A cycle in the wait graph is **necessary and sufficient** for a deadlock to occur (edges from thread waiting on a resources to thread that owns a resource).

- For the purposes of this course, simply understand that maintaining a lock order is enough to prevent deadlocks. In reality, programs and applications are complex enough such that deadlocks will most likely occur.

* It is possible to implement **deadlock prevention**. 
	* Checks can be performed if an operation will cause a deadlock (i.e. check for cycles in the graph). 
	* In other words, every lock acquisition should be first checked if the operation will cause a deadlock or not. Clearly, this is **expensive**.

- It is also possible to **detect then recover** from deadlocks. 
	- This perhaps is not as bad as monitoring _every single lock request_ to check for deadlocks (as above), but there is still a need for the ability to rollback from a deadlock.
	- However, this requires that we maintain a stack trace of current and past states. 
	- It is also not always possible to perform rollbacks, e.g., when external inputs are involved.

- Finally, we also have the option to simply **do nothing** (Ostrich Algorithm).
	- The options for preventing or detecting deadlocks are potentially expensive, and it may not be worth it, unless performance is absolutely critical.
	- Otherwise, it might be more beneficial to just assume that no deadlocks occur. If we were wrong, then a reboot is simply in order.

_Quiz:_

Pick the check statements that correspond with the following criteria:
- At any point in time, a maximum of three new orders can be processed.
- If any old orders are currently being processed, then only one new order can be processed.

```
Lock(orders_mutex) {
    [INSERT CHECK HERE]
        Wait(orders_mutex, new_cond)
    new_order++;
}

A) // CORRECT
while ((new_order == 3) OR (new_order == 1 AND old_order > 0))

B) // INCORRECT, while instead of if
if ((new_order == 3) OR (new_order == 1 AND old_order > 0))

C) // INCORRECT, old_order > 0
while ((new_order >= 3) OR (new_order == 1 AND old_order >= 0))

D) // CORRECT
while ((new_order >= 3) OR (new_order == 1 AND old_order >= 1))
```
## Kernel vs. User-Level Threads

Previously, it was briefly mentioned that threads can exist at both the kernel and the user level.

- Kernel-level threads imply that the OS itself is multithreaded. These threads are visible to the kernel, and are managed by kernel level schedulers. 
	- Therefore, it is the OS' job to schedule these threads on the physical CPUs, and to determine which thread is allowed to execute at any given point in time.
	- Some of these kernel-level threads function to support user level processes (and their threads), while other kernel-level threads run OS-level services such as daemons.

- User-level threads of course, imply that the processes are multithreaded.
	- For a user-level thread to actually execute, it must be first associated with a kernel-level thread.
	- Then, the OS-level scheduler must schedule that kernel level thread onto a CPU.
	- Since the user-level threads must interact with the kernel-level threads, we will cover three multithreading models that describe this relationship between the two thread levels.
### Threading Models

#### One-to-One Model

Each user-level thread has a kernel-level thread associated with it when the user-level thread is first created.

Benefit:
- The OS sees that the process is multithreaded, and it will be able to understand what those threads require in terms of synchronisation, scheduling, blocking, etc.
- Since the multithreaded OS already posseses the ability to perform thread mechanisms (scheduling, etc.), the user-level processes can simply leverage the threading support at the kernel level.
- This means that threading management will be handled by the kernel instead of the user.

Downside:
- Every operation must cross the user/kernel boundary, since we must go to the OS for each and every operation running on a thread. This, as we have noted, may be incredibly expensive.
- Since we are relying on the OS to perform threading management, this means that we are bound to the limits the OS imposes on policies, thread count, and so on. This affects portability.     

#### Many-to-One Model

All the user-level threads are to be mapped onto a single kernel-level thread. In other words, there must be a thread management library **at the user level** that decides the mapping of the user-level threads onto the kernel-level thread at any given point in time. The user-level thread will only run once the associated kernel-level thread is scheduled on the CPU.

Benefit:
- Totally portable. Thread management is completely handled at the user level, hence we do not need to rely on any form of kernel support.
- We are also not affected by any specific OS policies or limitations.
- No system calls (user/kernel boundary crossings) are required, since we are not relying on the kernel for scheduling, synchronisation, blocking, etc. 

Downside:
- The OS has no insights into what the application needs. For example, the OS may block the entire process if one user-level thread is blocking on I/O.

#### Many-to-Many Model

The many-to-many model quite literally allows for either many-to-one or one-to-one mappings from user-level to kernel-level threads. 

Benefit:
- This is clearly the best of both worlds. It is possible to have **bound threads**(permanent one-to-one mapping) or **unbound threads** (many-to-one mapping).
- The kernel will know that the process is multithreaded, since it assigns multiple kernel-level threads to the same process. 
- Furthermore, if one user-level thread blocks, other threads of the same process can still be executed if they have mappings to another kernel-level thread.
- Bound threads enable the kernel to know if a thread requires a higher priority. 

Downside:
- Coordination is required between user and kernel-level thread managers. In the other two models, thread management was handled purely at the user level or at the kernel level.
### Scope of Multithreading

At the kernel level, there is system-wide thread management, supported by the OS thread managers. These managers will look at the entire platform before making decisions on how best to run their threads (allocate resources, scheduling, etc). This is the **system scope**.

Conversely, at the user level, a user-level thread management library simply handles the threads for any given process it is linked to. This user-level thread manager has no way of seeing other threads outside their process, hence these managers are said to have **process scope**.

Consider the scenario where there are two processes, A and B, where A has twice as many user-level threads compared to B.
- If the threads have a **process scope**, the kernel is unable to see that A has twice as many threads. The kernel might allocate equal resources to both A and B. Inadvertently, this would mean that any given thread in A will be allocated half the CPU cycles compared to B.
- If the threads have a **system scope**, then the kernel is now able to see that A requires more resources, The kernel will then allocate CPU resources **relative to the total amount of user threads, as opposed to the total amount of processes**.
## Multithreading Patterns

### Boss/Workers Pattern

The boss/workers pattern is characterised by **one boss thread** and a number of **worker threads**. The "boss" is in charge of work assignment, while the "workers" are responsible for completing tasks assigned to them.

The throughput of this pattern is limited by the boss thread. Therefore, it is imperative to keep the boss thread as efficient as possible. Specifically, the throughput of the system is **inversely proportional to the amount of time the boss spends processing each task**. Nevertheless, this pattern is often the most simple -- one thread assigns the work while the others work to complete it.

The following points illustrate some key considerations when designing threads to follow a boss/worker pattern.

1. Work assignment
   - A possible method of assigning tasks to worker threads would be to keep track of idle/working threads, and **directly assign tasks** to currently idling threads (micromanagement). This would mean that the boss must select a worker thread, then wait for that worker to accept the assigned task.
	   - The benefit of this approach would be that workers need not worry about synchronisation amongst themselves.
	   - However, since the boss must keep track of each worker and directly issue tasks, this extra work will decrease throughput.
   - Another possible method would be to establish a task queue shared between the workers and the boss (similar to a producer/consumer queue). In this case, the boss is the producer that adds tasks to the queue, while the workers consume and work on the queued tasks.
	   - This approach allows the boss to not worry about the status of the workers. The boss simply has to place accepted tasks onto the queue and move on to process the next incoming task. Any free worker should look at the queue and attempt to tackle the next task (FIFO queue).
	   - This shared queue would mean that further synchronisation is required, since the queue should be properly modified by both the boss and the workers. Despite this downside, this approach is rather common, since the overall throughput of the system is improved due to the boss thread having to do less work.

2. Number of workers
   - Naturally, if the work queue fills up, the boss can no longer add any items to the queue. The likelihood of the work queue filling up is dependent on the number of workers.
   - It is possible to simply add more threads to help increase/stabilise throughput, though arbitrarily adding any number of threads can introduce unnecessary overheads and complexities.
   - One possible method would be to create a worker (or workers) on demand, i.e., in response to an incoming task. This may be inefficient if the cost of creating a new worker thread is signficant.
   - A more common method would be to create a **pool of worker threads** that can be increased in response to high demand. Contrast this to the creation of a singular thread on demand, where the pool may be instead *doubled or increased by some other multiple*.
	   - A downside would be the overhead involved in managing the worker thread pool. 

3. Worker variants and locality
   - In the boss/worker model where a share queue and the thread pool is used, the boss does not keep track of what the worker threads were doing last. It is possible that a thread was just finishing a task that is very similar to the next incoming task (and can therefore benefit from cached data), though the boss has no way of knowing this.
   - Therefore, it is worthwhile to have specialised worker threads, though the boss will now be slightly more involved in determining which worker threads get which task. This extra work is usually ofset by the fact that the worker threads are now much more efficient.
   - This approach exploits **locality**. By performing only a specialised subset of the task, it is likely that only a subset of the state will need to be accessed. It is thus more likely that this subset will already be present in the CPU cache.
   - In addition, specialised workers allow for better quality of service; more threads can be created for urgent tasks or tasks with higher priorities.
   - However, this approach would require more complex load balancing mechanisms and requirements may be a bit more tricky to figure out.
### Pipeline Pattern

In this pattern, the overall task is divided into subtasks. Each subtask will be assigned a different thread. For example, if we have a six-step process, six separate threads will be assigned, one for each step in the process.

At any given point in time, there may be multiple tasks executing concurently within the overall pipeline. Analagous to a conveyor belt, one item may be at the first stage of completion, while another may be nearing the last, for example.

The throughput of the pipeline will be limited by the **weakest link**; that is, the task that takes the longest amount of time to complete. Not to worry, however, since we can always allocate more threads to that particular step.

The best method to pass tasks between the various stages/steps would be to have a shared buffer between two stages/steps. More concretely, the thread for the first stage will place its completed product on a queue that the thread for the second stage will have to pick up when it is ready.

In sum, a pipeline is a series of subtasks, where a thread performs an individual subtask before handing the its result over to the next thread in the sequence. To keep the pipeline balanced, a particularly slow stage can be executed by more than one thread. A shared buffer-based communication system can be used between stages.

- This approach benefits from specialisation and locality, which can lead to more efficiency. As discussed above, state required for similar jobs will likely already be present in CPU caches.

- However, it is often rather difficult to keep the overall pipeline balanced, especially when the input rate changes, or if resource availability is altered. Rebalancing is often required with this approach.
### Layered Pattern

This pattern involves similar subtasks grouped together into a "layer". Threads being assigned to a particular layer can then perform any of the subtasks classified under that layer. The tasks within a layer need not be sequential -- the point here is to **group similar subtasks together**.
- This approach benefits form specialisation, once again, though it is allowed to be less fine-grained compared to the pipeline pattern.
- This may not be suitable to all applications and synchronisation will be much more complex. Each layer must be able to communicate with and know about the layers above and below it, since the must both receive inputs and pass results around. 
