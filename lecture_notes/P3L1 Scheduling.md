## Introduction and Overview

Recall that the CPU scheduler decides access of processes or threads to the CPU resource itself. In this section, the term "**task**" will be used to refer to both processes and threads, as the details of scheduling are similar for both. 

Recall from [[P2L1 Processes and Process Management]] that processes (or more generally, tasks) must wait in the **ready queue** until it is scheduled to execute on the CPU. As a quick refresher, tasks may become ready in the following ways:
- A blocking I/O request is resolved and the task rejoins the ready queue,
- A time slice expires (more on this later)
- A new task is created
- An interrupt occurred

Whenever a CPU becomes idle, there is a need to run the scheduler to determine which task should be dispatched next. The following are pertinent questions that follow:
1. How should the decision be made? Which task should be schedule?
2. How does the scheduler accomplish its job?

The answer to the first question depends on the scheduling policy/algorithm being used. The second question can be answered by picking apart the actual data structure being used to implement the ready queue, perhaps confusingly named the **runqueue**.

The implementation of the runqueue and the scheduling algorithm used are tightly coupled. Therefore, this lecture will look at various scheduling algorithms in close relation with the actual data structures being used.
## Scheduling Policies

### Broad Overview

This section isn't structured all too well -- this set of notes shifts things around to (hopefully) make things more sensible. A few details must be ironed out before investigating scheduling policies.

We will take a look at three scheduling policies: (i) First-Come-First-Served, (ii) Shortest Job First, (iii) Priority Scheduling, and finally (iv) Round Robin. These policies will be investigated based on some key assumptions.
#### Assumptions

For purposes of simple discussion the below assumptions will be made, though these will gradually be relaxed as we approach "reality". 
1. Assume that there are a group of "ready" tasks all waiting to execute at the same time.
2. Assume that we are able to determine the exact run time of each task.
3. Assume that there is no preemption in the system. A task must run to completion.
4. Assume that there is a single CPU.
#### Performance Metrics

In evaluating different scheduling policies (or schedulers in general), the common metrics are:
- Throughput
- Average job completion time
- Average job wait time
- CPU utilisation.

Some of these metrics will be discussed in relation to the three scheduling policies aforementioned.
### First Come First Served

The simplest algorithm (policy) to implement would be First-Come-First-Served (FCFS). As the name implies, tasks are scheduled in order of arrival. The *FIFO queue* data structure lends itself very nicely to this policy, and the scheduler simply needs to pop the first item in the queue.

Consider the following scenario:
- There are three tasks, `T1`, `T2`, and `T3`.
- Both `T1` and `T3` have tasks that takes 1 second each.
- `T2`'s task takes 10 seconds. 
- Assume the case where the tasks arrive in the order `T1`, `T2`, and `T3`. 

The total amount of time required for all tasks to complete would be 1 + 1 + 10 = 12 seconds (in that particular order). 

The _throughput_ as defined as number of tasks tasks per unit time would be:
$$ 3 / 12 = 0.25 \text{ tasks per second} $$
The _average completion time_ would be:
$$ (1 + 11 + 12) / 3 = 8 \text{ s} $$
The _average wait time_ would be:
$$ (0 + 1 + 10) / 3 = 4 \text{ s} $$
Tabulating the above results:

|  | FCFS |
| ----- | ----- |
| Throughput (tasks/s) | 0.25 |
| Average Completion Time (s) | 8 |
| Average Wait Time (s) | 4 |
### Shortest Job First

#### Run-to-Completion

It is not a major stretch to consider the shortest-job-first algorithm (SJF). Again as the name implies, tasks are scheduled in order of their execution time. Note that the _FIFO queue_ no longer applies in this scenario, rather an **ordered queue** is now necessary. Alternatively, a **tree structure** can be used to maintain the runqueue -- simply re-balance the tree when inserting a new task. 

Consider the same scenario with the three threads mentioned above. The _throughput_ would be the same, where three tasks are completed in 12 seconds. The _average completion time_ now, however, decreases to just 5 seconds.
$$ (1 + 2 + 12)/3 = 5 \text{ s}$$
The _average wait time_ would be:
$$ (0 + 1 + 1) / 3 = 0.667 \text{ s} $$
Now tabulating results from both the FCFS and SJF policies:

| | FCFS | SJF |
| ----- | ----- | ----- |
| Throughput (tasks/s) | 0.25 | 0.25 |
| Average Completion Time (s) | 8 | 5 |
| Average Wait Time (s) | 4 | 0.667 |

Clearly, if the metrics of concern are _average completion time_ or _average wait time_, then the SJF policy should be preferred over FCFS.
#### Pre-emption

Let's remove assumptions (i) and (iii). In other words, a task may be interrupted (preempted) even if it is already executing. Tasks may also arrive at different timings.

The following scenario will still be considered with the above assumptions relaxed:
- There are three tasks, `T1`, `T2`, and `T3`.
- Both `T1` and `T3` have tasks that takes 1 second each.
- `T2`'s task takes 10 seconds. 
- Assume the case where the tasks arrive in the order `T2`, `T1`, and `T3`. 

Since `T2` arrives first, it will be scheduled onto the CPU for execution. However, it must be preempted whenever `T1` or `T3` arrives in line with the Shortest-Job-First policy. Notice then that the scheduler must be invoked **whenever new tasks enter the runqueue**, to make the appropriate scheduling decisions.
#### Unknown Execution Times

Of course, the discussion thus far has been assuming that the exact execution times of tasks are known (ii). In reality, it is not likely that the exact execution times will be known upon scheduling, though certain **heuristics based on historical execution times** can be used for inference.
- Such a heuristic could take into account the time taken for a task to run the last time, or
- The average time taken for a task to run the last `n` times can also be considered. This is known as a **windowed average**.
### Priority Scheduling

It is possible for tasks to have different priority levels, and perhaps we would like to schedule based on that criteria instead.
> As an example of higher priority tasks, think of kernel-level tasks that manage critical system components.

The same requirement from SJF is applicable here -- a lower priority task must be preempted to allow a higher priority task to execute whenever the higher priority task enters the runqueue.

_Quiz_
_An OS scheduler uses a priority-based algorithm with preemption to schedule tasks. Given the values in the table below, compute the finishing times of each task. Assume that the priority P3 < P2 < P1._

| Task | Arrival Time | Exec Time | Priority |
| ---- | ---- | ---- | ----|
| T1 | 5 | 3 | P1 |
| T2 | 3 | 4 | P2 |
| T3 | 0 | 4 | P3 |

T1 finishes at: 8 seconds
T2 finishes at: 10 seconds
T3 finishes at: 11 seconds
#### Implementation Details

The data structure to be used could be as simple as separate FIFO queues for each priority level. The scheduler can then trivially select tasks in order of priorities.
- A caveat: priority scheduling may result in **starvation**. Starvation occurs when a low priority task is never scheduled due to the constant presence of higher priority tasks.
- A workaround would be to implement **priority aging**. In essence, a priority of a task should be a function of the actual priority and the amount of time a task has spent in the runqueue. Older, lower priority tasks will eventually be promoted and scheduled in this manner.
#### Priority Inversion

A scenario known as priority inversion might occur when synchronisation mechanisms are also involved. Consider the following:
- There are three tasks, `T1`, `T2`, and `T3`.
- In order of priority, `T3` < `T2` < `T1`.

Suppose the following occurred:
- `T3` (lowest priority) is currently executing and it acquires a mutex. 
- `T2` arrives after some time, and `T3` is preempted as `T2` is of higher priority.
- `T1` now arrives and preempts `T2`. However, `T1` needs the mutex and blocks to wait. 
- `T2` is the next highest priority thread that is able to run. `T2` gets scheduled and runs to completion.
- `T3` is scheduled next and releases the mutex once it is done.
- Finally, `T1` acquires the mutex and runs to completion. 

![[Pasted image 20231008132838.png]]

Notice here that **priority inversion** occurred. `T1`, the highest priority thread, ran to completion last. 

In order to circumvent this issue, the owner of the mutex should have its priority temporarily boosted. In this case, `T3` should have its priority boosted to the levels of `T1` -- once `T3` actually releases the mutex, its priority should return to normal. Also notice that this is one of the reasons why _knowing about the owner of a mutex is important_; the priority of the mutex owner should be boosted.
### Round Robin Scheduling

A much more popular option (instead of SJF or FCFS) would be round robin scheduling. This method of scheduling is rather similar to FCFS, although tasks are allowed to yield.

This method also allows for dealing with priorities, and the details will not be covered here for brevity (they are mostly repeats of the above anyway). The main idea is to allow tasks to be interleaved with each other. Tasks may do so if they explicitly yield, or they could be preempted after a certain period of time. This mechanism is known as **timeslicing**.

#### Timeslicing 

###### #DEFINE Timeslice
> Also known as a time quantum, a time slice is the **maximum amount of uninterrupted time that a task can run**. 

Notice that a task can also run for a shorter amount of time. For example, if a task encounters an I/O bound operation or synchronise with another task, the task will be preempted and placed on the appropriate queue. Alternatively, a task may also be be preempted if a higher priority task arrives on a runqueue.

The usage of timeslices allows tasks to be interleaved; the CPU is said to be **timeshared**. For I/O bound tasks, timeslices are not too important, as these tasks will likely block and be preempted anyway. For CPU-bound tasks, timeslicing is the only way to achieving timesharing.
#### Metrics

In this section, the performance metrics of round robin scheduling with a timeslice of 1 second will be compared to the FCFS and SJF algorithms.

Suppose we have the following scenario:

| Task | Exec Time | Arrival Time |
| ----- | ----- | ----- |
| T1 | 1 sec | 0 |
| T2 | 10 sec | 0 |
| T3 | 1 sec | 0 |

- `T1` first executes for 1 second and completes.
- Next, `T2` is scheduled and runs for 1 second before its timeslice expires. It is preempted and replaced by `T3`.
- `T3` also runs to completion.
- Finally, since there are no other tasks waiting to run, `T2` runs to completion after 9 more seconds.

The _throughput_ would be the same, where three tasks are completed in 12 seconds. The _average completion time_ is:
$$ (1 + 12 + 3)/3 = 5.33 \text{ s}$$
The _average wait time_ is:
$$ (0 + 1 + 1) / 3 = 0.667 \text{ s} $$

Now comparing all three policies:

| | FCFS | SJF | Round Robin Timeslice |
| ----- | ----- | ----- | ----- |
| Throughput (tasks/s) | 0.25 | 0.25 | 0.25 |
| Average Completion Time (s) | 8 | 5 | 5.33 |
| Average Wait Time (s) | 4 | 0.667 | 0.667 |

All round robin (with timeslicing) metrics are roughly on par with that of SJF metrics. However, a key thing to notice here is that **no prior information regarding a task's execution time was required**. Timeslicing therefore has the following benefits:
- Shorter tasks finish sooner
- The scheduler itself is more responsive
- Lengthy I/O operations can be initiated sooner

However, timeslicing is not without its downsides. Notably, timeslicing must involve context switching between tasks, which comes with overheads. Even if there are no other tasks available to context switch to (as is the case with `T2` above), the scheduler will still be run at every timeslice interval. 

If we consider the overheads involved in context switching and when the scheduler is invoked (every 1 second, here), then the performance metrics will be slightly negatively affected. 
- Slightly more time is required for all tasks to complete, resulting in a lower throughout.
- Average completion time an average wait time will also increase slightly.
#### Timeslice Duration

Recall from previous discussions ([[P2L2 Threads and Concurrency]]) that context switching only makes sense if the timeslice is significantly longer than the time it takes to context switch. 
> More specifically, if $t_{idle} > 2*t_{ctx\_switch}$, then context switching should occur to hide any idling time. In plain English, if a thread is idle for more time that it would take to perform two context switches, then context switch please.

However, this statement *depends* on whether a task is CPU-bound or I/O-bound.
##### CPU-Bound Timeslice Length

Consider two CPU-bound tasks that take 10 seconds to complete. The time for context switching can be assumed to be 0.1 seconds. We will investigate two timeslice durations, 1 second and 5 seconds.

The relevant performance metrics are shown below:

| Alg. | Throughput | Avg. Wait | Avg. Comp. |
| ----- | ----- | ----- | ------ |
| RR (ts=1) | 0.091 tasks/s | 0.55 s | 21.35 s |
| RR (ts=5) | 0.098 tasks/s | 3.05 s | 17.75 s |

A shorter timeslice duration impairs both _throughput_ and _average completion time_, due to the need for context switching. However, it seems like switching to a timeslice duration of 5 seconds significantly impairs _average wait time_ instead.

Notice then that for CPU-bound tasks, the average wait time is hardly a metric for concern -- the user does not perceive when a CPU-bound task starts anyway. Therefore, the metrics _throughput_ and _average completion time_ are arguably more important in this case.

For CPU-bound tasks, **longer timeslices** should be chosen. In fact, the two important metrics are at their best if the timeslice value was set to infinity (i.e., tasks will never be preempted). 

|Alg.|Throughput|Avg. Wait|Avg. Comp.|
|---|---|---|---|
|RR (ts=1)|0.091 tasks/s|0.55 s|21.35 s|
|RR (ts=5)|0.098 tasks/s|3.05 s|17.75 s|
|RR (ts=inf) | 0.1 tasks/s | 5 s | 15 s |
##### I/O-Bound Timeslice Length

Consider now two I/O-bound tasks that take 10 seconds to complete. Context switching takes 0.1 seconds, I/O requests are issued every 1 second, and I/O completes every 0.5 seconds. Again, we shall start by considering timeslice values of 1 second and 5 seconds.

The relevant performance metrics are shown below:

| Alg. | Throughput | Avg. Wait | Avg. Comp. |
| ----- | ----- | ----- | ------ |
| RR (ts=1) | 0.091 tasks/s | 0.55 s | 21.35 s |
| RR (ts=5) | 0.091 tasks/s | 0.55 s | 21.35 s |

In both cases, since tasks yield after 1 second due to them issuing I/O requests, changing the timeslice values do not affect much -- no preemption is taking place. However, consider the case where only one task, say `T2` is I/O bound while `T1` is CPU-bound. These cases are marked with asterisks `*`.

| Alg. | Throughput | Avg. Wait | Avg. Comp. |
| ----- | ----- | ----- | ------ |
| RR (ts=1) | 0.091 tasks/s | 0.55 s | 21.35 s |
| RR (ts=5) | 0.091 tasks/s | 0.55 s | 21.35 s |
| RR (ts=1*) | 0.091 tasks/s | 0.55 s | 21.35 s |
| RR (ts=5*) | 0.082 tasks/s | 2.55 s | 17.55 s |

For the timeslice value of 1, the metrics are unaltered (previously, the I/O task yielded after 1 second. Here, the task is preempted after 1 second. Nothing changed, really).

For the timeslice value of 5, the _average wait time_ increases by 2 seconds. This is not great, especially since the user will perceive wait times associated with I/O tasks. For example, if the CPU-bound task runs first, the I/O task will have to wait a full 5 seconds before it can even begin to make a I/O request (at time=6 seconds). 

From this example, it should be clear that **smaller timeslices** are better for I/O bound tasks.
##### Summary

In short, CPU-bound tasks prefer longer timeslices, while I/O-bound tasks prefer shorter timeslices. The rationale behind both falls under CPU utilisation.
- Longer timeslices for CPU-bound tasks reduces the overheads that context switching and scheduling will produce. The CPU is thus maximally utilised.
- Shorter timeslices for I/O-bound tasks allows I/O requests to be issued as soon as possible, which keeps CPU utilisation high and improves user-perceived performance (low wait time).

_Quiz_
_On a single CPU system, consider the following workload and conditions:_
- _10 I/O bound tasks and 1 CPU-bound task_
- _I/O bound tasks issue an I/O operation every 1 ms_
- _I/O operations always take 10 ms to complete_
- _Context switching overhead is 0.1 ms_
- _All tasks are long running_

_What is the CPU utilisation (%) for a round robin scheduler where the timeslice is 1 ms? What about a 10 ms timeslice? Round the answers to the nearest percent._

**CPU Utilisation Formula:**
$$ \text{CPU Running Time} \div (\text{CPU Running Time} + \text{Context Switch Overhead}) $$
The CPU running time and context switch overhead should be calculated over a consistent, recurring interval. 

1 ms: 0.91% CPU Utilisation
10 ms: 95% CPU Utilisation

> Do note that in this quiz, the metric evaluated in CPU-centric. Of course, the CPU-bound task favours a longer timeslice (reflected in the 95% utilisation). If we were to investigate an I/O-centric metric instead, we may find that a shorter timeslice is preferred. 

## Runqueue Data Structure

### Basic Queue

The runqueue can be implemented as a simple queue, as established. However, it is important to note that the runqueue is only ever _logically_ a queue, and can be implemented in any form. The main consideration would be that the **scheduler must be able to quickly and easily determine the next task to schedule**.

![[Pasted image 20231008152212.png]]
### Multi-Queue

As noted above, I/O-bound and CPU-bound tasks perform better under different timeslice values. It is possible to place both such tasks into the same runqueue and have the scheduler check and determine timeslice durations. Alternatively, these two tasks can be placed in completely separate queues altogether.

A common data structure is therefore a multi-queue structure that maintains distinct timeslice values. Tasks can then be placed on the appropriate queue based on their I/O requirements.

![[Pasted image 20231008152338.png]]

This raises a rather important question that was glossed over in the previous section regarding timeslice durations. How can one determine if a task is I/O-bound or CPU-bound?
- It is possible to once again look to history-based heuristics to determine if a task is likely CPU- or I/O-bound. 
- However, this does not inform new tasks or tasks that may behave dynamically.
### Multi-Level Feedback Queue

The following figure shows a multi-level feedback queue:

![[Pasted image 20231008153305.png]]

- When a new task enters the system, it will be placed on the topmost queue (i.e. shortest timeslice). 
- If the task yields before the timeslices expires, then this was the correct decision; the task is clearly I/O bound. This task will be placed on the same queue when it becomes runnable again.
- If the timeslice expires and the task is preempted, the task will be pushed down a level. This task is likely more CPU-bound, hence a longer timeslice will be provided when the task is encountered again.
- This dynamic shifting also works in the opposite direction (not pictured). If a task in a lower queue yields before the allocated timeslice expires, it is possible to place it in a queue with a smaller timeslice the next time.

This data structure, termed the MLFQ, was coined by Fernando Corbato (and he won the Turing Award for this). The MLFQ allows tasks to find the timesharing schedule that best suits them over time. 

The Linux O(1) scheduler borrows some implementation details from the MLFQ, and the Solaris scheduler is basically using the MLFQ with 60 levels (not covered).

## Linux Schedulers

### O(1) Scheduler

#### Scheduler

The O(1) scheduler, as its name implies, is able to add or select a task in constant time. It is a preemptive, priority-based scheduler with 140 priority levels.

![[Pasted image 20231008154109.png]]

- The priority levels are split into two categories: real-time tasks (0-99), and "other", timesharing tasks (100-139).
- All user-level processes have priority levels that fall under the timesharing task category. The default priority is 120, and can be toggled with a "nice value" (-20 to 19), which essentially spans the entire range of 100 to 139. This toggling is performed via a system call.
- **Different timeslice values** (showed as time quantum) are associated with different priority levels and some **feedback** is accounted for when deciding where to place an incoming task. This can be thought of as roughly similar to the MLFQ.
	- The timeslice values here are associated with priority. A smaller timeslice is allocated for tasks with lower priorities and a larger timeslice is allocated for a higher priority task.
	- The feedback mechanism used depends on sleep time (or rather, time spent waiting or idling). 
		- If a task sleeps for longer, this indicates an interactive task. The task's priority will be boosted, or more specifically, 5 will be subtracted from its numeric priority.
		- If a task sleeps for shorter, this indicates a compute-intensive task. The task's priority will be lowered, or more specifically, 5 will be added to its numeric priority.
	- These seem counter-intuitive, especially since we've covered the fact that I/O bound tasks should have smaller timeslices (and therefore lower priority in this implementation). This will make sense once the actual runqueue is covered.

#### Runqueue

The runqueue associated with the O(1) scheduler is divided into two components, an **active array** and an **expired array**. 

![[Pasted image 20231008155140.png]]

The active array will be used by the scheduler to determine the next task to run.
- Inserting a task takes constant time -- indexing based on priority and enqueuing at the end of the `task list` queue are both constant time.
- Selecting a task also takes constant time. The scheduler itself relies on a sequence of instructions for scheduling. This sequence of instructions maintains information regarding which priority number has tasks waiting to be run (think of this as a sequence of ON/OFF bits). As such, it takes constant time to search for the first set (ON) bit encountered. Indexing into the `task list` and extracting the task also takes constant time.

If a task yields or is preempted, the task will be **returned to the active array** (at the appropriate priority level, of course) if the time spent on the CPU is _less_ than the timeslice allocated. 
- The task is only moved to the expired array when its timeslice is completely exhausted.
- Notice that this means tasks in the expired array will never be scheduled as long as there are tasks in the active array.
- When the active array is empty, the expired and active arrays are swapped.

The rationale behind giving lower priority tasks a smaller timeslice is therefore two-fold:
1. Higher priority tasks (with larger timeslices) are allowed to continually run as long as their timeslice has not expired and will not be blocked by lower priority tasks.
2. Since tasks with expired timeslices will be shifted to an expired array and not be scheduled, this also allows for some form aging mechanism, where lower priority tasks are eventually allowed to run.

#### Final Notes

The O(1) scheduler was introduced in Linux 2.5 by Ingo Molnar. This was eventually replaced with the Completely Fair Scheduler (CFS) in Linux 2.6.23, in order to keep with with changing workloads of newer applications. The CFS scheduler is now the default scheduler used.
### CFS Scheduler

A major issue with the O(1) scheduler is that tasks that enter the expired array _must_ wait for all other active tasks to exhaust their timeslices. This is a problem for the more interactive tasks of today's applications, where **jitter** introduced affects user experience.

The O(1) scheduler also does not make any concrete guarantees regarding **fairness**.
###### #DEFINE Fairness
> In a given time interval, a task should be able to **run for an amount of time proportional to its priority**.

#### Runqueue

A red-black tree is used; self-balancing trees that ensure all paths from the root to leaves are of approximately the same length.

![[Pasted image 20231008160833.png]]

- The tasks are ordered by `vruntime` (virtual runtime), which is defined as the **amount of time spent running on the CPU**. This quantity is tracked to the nanosecond.
- Notice that all nodes to the left of a parent have lower `vruntime` and therefore should be scheduled sooner.
#### Scheduler

The CFS scheduler will always pick the leftmost node. The task that is running on the CPU will have its `vruntime` increased, and the rate of increment depends on the task's priority level.
- Higher priority tasks have a lower rate of `vruntime` increase.
- Lower priority tasks have a higher rate of `vruntime` increase.

Periodically, the `vruntime` of an executing task will be compared with the leftmost node in the runqueue. If the `vruntime` of the executing task exceeds the `vruntime` of the leftmost node, then the executing task is preempted and inserted appropriately into the tree. The leftmost node (task) is then scheduled to run on the CPU.

Task selection from the runqueue tasks constant time -- simply pick the leftmost node. Adding a task takes $log(n)$ time, where $n$ is number of tasks in the system. Clearly, a $log(n)$ time is not the best, although it is considered good enough for current use. 

_Quiz_
_What was the main reason the Linux O(1) scheduler was replaced by the CFS scheduler?_
- Scheduling a task under high loads took an unpredictable amount of time
- Low priority tasks could wait indefinitely and starve
- Interactive tasks could wait an unpredictable amount of time to be scheduled (CORRECT)
## Scheduling on Multiprocessors

### Multiprocessor Architecture

At long last, the assumption made all the way at the start regarding a single CPU (iv) can be relaxed. Before delving into the topic proper, it is important to first understand the architecture of multiprocessor systems.
###### #DEFINE Shared Memory Multiprocessor (SMP)
> Multiple CPUs are involved, where each CPU has its own private L1/L2 cache as well as a **Last Level Cache (LLC)** that may or may not be shared amongst the CPUs. The system memory, **Dynamic Random Access Memory (DRAM)** is shared amongst all CPUs.

###### #DEFINE Multicore
> In a multicore system, each CPU has **multiple internal cores**. Each core has its own private L1/L2 cache and the Last Level Cache is shared amongst all cores in the CPU. DRAM is still shared amongst all CPUs.

![[Pasted image 20231009105052.png]]

Laptops and phones these days usually have two cores, whereas servers might have six to eight cores per CPU. Of course, multiple CPUs might also be present -- multiple multicore CPUs are widely available nowadays. 

As far as the OS is concerned, it views CPUs and the individual cores as entities onto which it can schedule threads (or execution contexts). In other words, the OS does not distinguish cores from actual CPUs. The discussion therefore will mostly focus on SMPs (multiple CPUs) and the concepts are roughly generalisable to multicore CPUs as well.
### Cache Affinity

Recall that performance of threads and processes in general are highly dependent on the state of the cache. More specifically, if the execution state required is present in cache or in memory, the cache is considered to be "hot". Also recall that cache access is faster than memory access (and disk is the slowest, being an I/O operation). 

When a task runs on a CPU, it will (over time) bring most of the required state from memory into the private L1/L2 caches or perhaps the LLC of the current CPU. If the task is then preempted and rescheduled onto a _different CPU_ in the future, the task has to deal with a "cold" cache and performance will be negatively impacted. Therefore, **cache affinity is important** when making scheduling decisions. 
###### #DEFINE Cache Affinity
> The ability to repeatedly schedule a task to a CPU or a range of CPUs (rather than any other CPU), such that the performance benefits from a hotter cache. 

In order to do this, it is possible to have a **hierarchical scheduler architecture**.

![[Pasted image 20231009105804.png]]

- At the top, there should be a load balancing component in place, that divides tasks evenly between the CPUs.
- Then, each CPU would have its own associated scheduler (and of course, a per-CPU runqueue). This would allow the same tasks to be repeatedly scheduled onto the same CPU as much as possible. 
- The load balancing component can achieve its objective by:
	- Balancing based on queue length, or
	- Pulling tasks from other queues if a particular CPU starts to become idle.
#### NUMA

As a quick aside, it is also possible to have multiple memory nodes (depicted as MM in the figures) alongside multiple processors. The CPUs and the memory nodes will be connected via some form of interconnect. For example, on modern Intel platforms, there is an interconnect known as the QuickPath Interconnect (QPI).

It is possible for a particular memory node to be "closer" to a particular set of CPUs. More specifically, the memory node is connected to some **socket** that is in turn attached to multiple processors. In that case, access to that particular memory node (local) will be faster for the attached CPUs versus access to another memory node (remote) associated with a different set of CPUs. 
- Note here that "remote" doesn't imply a disconnect. Everything is still connected, some more closely than others.
- The important take away would be that **access to a local memory node is faster than access to a remote memory node**.
- These platforms are termed Non-Uniform Memory Access (NUMA) platforms.

Schedulers that account for this and keep tasks on a CPU closer to the memory node where their state is are called **NUMA-aware**. 
### Hyperthreading

#### Introduction

Context switching between tasks is necessary as each CPU only has one set of registers used to describe a certain execution context. Over time, hardware architects decided to hide the latency associated with context switching by allowing a single CPU to have multiple sets of registers. Each register can describe a separate execution context.

This is also termed **hyperthreading**. Multiple hardware-supported execution contexts are supported, though there is only ever one CPU. Only one thread can execute at any given point in time but the benefit lies in the fact that context switching is significantly sped up. 

Hyperthreading itself is referred to by various names:
- Hardware multithreading
- Chip multithreading (CMT)
- Simultaneous multithreading (SMT)

Most modern platforms support two hardware threads, though this number can go up to eight. Hyperthreading can also be enabled or disabled upon boot. 

Importantly, this concept should be distinguished from multi-core CPUs. It is possible to think of hyperthreading as **virtual cores**, while multi-core CPUs actually provide **hardware cores**. _Hyperthreading is simply a cheaper alternative to having multiple cores_. However, to the OS scheduler, hyperthreading simply appears as CPUs that it can schedule threads onto. 
#### Scheduling

A consideration that the scheduler has to make would therefore be: which threads should be scheduled onto the virtual cores?
- Again, recall that context switching only makes sense if the time spent idling is more than twice the time taken to context switch (to and from). 
- In SMT (hyperthreaded) systems, the time to context switch is on the order of cycles. DRAM access is on the order of hundreds of cycles. 
- Therefore, it makes sense to context switch here (very fast), and hyperthreading may even help to hide some latency associated with memory access.
- Of course, there are some additional considerations to be made regarding the types of threads that should be scheduled on hardware threads, which will be covered in the section below.

From this point onwards, the discussion regarding scheduling decisions will be made in context of the paper "Chip Multithreading Systems Need a New Operating System Scheduler" by Fedorova, et al. 

A few assumptions must be made first:
1. A thread can issue an instruction every CPU cycle. In other words, CPU-bound threads will be able to maximise the _instructions per cycle_ (IPC) metric. 
2. Memory access takes 4 cycles. 
3. Hardware switching is instantaneous.
4. The thought experiment will be performed with an SMT platform with two hardware threads.

![[Pasted image 20231009131732.png]]

##### Both Compute-Bound

If two threads scheduled are both compute-bound, only one thread will actually be able to run at any given point in time -- there is only ever one CPU, after all. As a result, these two threads are likely to **contend** for CPU resources and **interfere** with each other.

![[Pasted image 20231009131748.png]]

Notice then that both threads' performances degrade by a factor of two. In addition, the memory component is completely idle as nothing is performing a memory-bound task.
##### Both Memory-Bound

If both threads scheduled are memory-bound, then CPU cycles are wasted as the majority of cycles are spent idling.

![[Pasted image 20231009131950.png]]
##### Co-scheduling Mixture

If two threads are scheduled, where one thread is compute-bound and the other is memory-bound, then we achieve a best-case scenario. The CPU-bound thread is scheduled until the memory-bound thread issues a memory request. The memory-bound thread is allowed to issue its request and the execution context is switched back to the CPU-bound thread. 

![[Pasted image 20231009132208.png]]

The benefits are two-fold here:
- Contention on the processor pipeline is avoided or at least limited.
- Both the CPU and memory components are being utilised. 

Although some overhead is involved in context switching, it will be minimal especially when contrasted to scheduling both compute- or memory-bound threads.

#### CPU-Bound or Memory-Bound

##### Hardware Support

To determine if a particular task is CPU-bound or memory-bound, historic information will once again be used. Previously, in order to distinguish I/O-bound or CPU-bound tasks, "sleep time" was used. However, this heuristic will not work in this case.
- The thread is not exactly "sleeping" while waiting on a memory request. It is active within the processor pipeline and not waiting on some software cue (as is the case for I/O).
- Keeping track of the sleep time requires some software methods which takes too long for it to be acceptable -- recall that context switching here is in the order of cycles now.

Thus, there must _some_ hardware support for this decision-making step. Fortunately, modern hardware maintains multiple counters that may aid this decision. Examples include:
- L1, L2, ..., LLC misses (cache usage)
- Number of instructions per cycle (IPC)
- Power or energy data of the CPU or other hardware components

There also exists interfaces and tools to access such hardware components. 
- For example, `oprofile` or the Linux `perf` tool.
- The `oprofile` website actually lists available hardware counters on different architectures (they aren't exactly uniform). 
##### Designing Heuristics

Many practical or research-based scheduling rely on the use of hardware counters to estimate the resources that a thread needs. For example, the scheduler can look at LLC misses. If LLC misses are high, it is likely memory-bound as its execution context does not fit within cache, or it might be that a cold cache is encountered.

There is no surefire way to determine the resources a thread requires. Rather, a scheduler must make informed decisions (guesstimates, arguably) based on:
- Multiple hardware counters
- Models that have been investigated and built _and_ trained for a specific architecture/platform.

This topic is much more involved than an introductory course should handle, so the details will not be covered.
#### Fedorova et al., CPI

##### Methods

Fedorova et al. speculates that the **cycles per instruction** (CPI) is a good concrete metric to use in order to determine if a thread is memory-bound or compute-bound. The rationale is s follows:
1. A memory-bound thread has a high CPI, multiple cycles are required.
2. A compute-bound thread has a low CPI (perhaps even close to 1). An instruction can be completed every cycle (or at least, close to every cycle). 

Fedorova et al. simulates a system with 4 cores. Each core supports a 4-way SMT, resulting in a total of 16 hardware execution contexts. This is the **testbed**. 
- There is no hardware CPI counter available on the platform that Fedorova et al. uses.
- Computing CPI from IPC (1 / IPC) would require software intervention, which is not acceptable.
- A simulation is therefore performed.

A synthetic **workload** was created, where threads have CPI values of 1, 6, 11, and 16. Clearly, threads with CPI values of 1 are more compute-intensive, and those with CPI values of 16 are more memory-intensive. 

The **metric** used was the number of _instructions per cycle_ (IPC). Since there are only 4 actual cores, the maximum IPC value should be 4.

The following table shows the experimental setups:
![[Pasted image 20231009134419.png]]
##### Results

The following graph shows the results obtained:

![[Pasted image 20231009134613.png]]

Clearly, with mixed CPI values, the overall processor pipeline is well utilised and a high IPC is observed (a). With similar CPI values, some cores have increased contention (where compute-bound threads clash) while other cores have cycles wasted (where memory-bound threads idle). 

From this simulation, the conclusion that tasks with mixed CPI values should be scheduled together for higher platform utilisation. However, this simulation is not entirely representative of realistic data.

Fedorova et al. also profiled a number of applications and several respected benchmark suites. CPI values were then computed.

![[Pasted image 20231009134916.png]]

Notice here that most CPI values tend to congregate within the 2 to 5 range. The workload used was therefore not the most realistic. Although CPI values do help to inform scheduling decisions, the reality is that CPI values are not distinct enough to make any proper choices.

### Summary

1. In short, resource contention is something to consider when dealing with SMTs in processor pipelines.
2. Hardware counters can be used to aid workload characterisation.
3. Schedulers must be aware of resource contention and not just be concerned with load balancing. 

*P.S.* LLC usage is a better hardware counter instead of CPI re: Fedorova et al. 



