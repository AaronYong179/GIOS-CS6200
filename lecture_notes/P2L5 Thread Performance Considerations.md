## Introduction and Overview

### Performance Evaluation

#### Threading Models

In the previous section on [[P2L2 Threads and Concurrency]], the execution time of a boss/worker model was compared to that of a pipeline model.

![[Pasted image 20230922210514.png]]

The calculations were already performed in the previous section and will not be rehashed here. The boss/worker model took 360 ms to complete 11 toy orders with 6 total threads while the pipeline model took 320 ms. 

It would be tempting to conclude that the boss/worker model underperforms in comparison to the pipeline model for threading. Let's attempt to calculate a different performance metric, the **average time to complete an order**.

For the boss/worker model, each batch of five orders will take 120 ms to complete. Each subsequent batch must also wait 120 ms multiplied by the number of batches in front of it. Remember that only five worker threads are active at the same time.

```python
total_orders = 11
total_time = (120 * 5) + ((2 * 120) * 5) + (3 * 120)
average_time = total_time / total_orders # ~186 ms
```

For the pipeline model (with 6 steps), the first order will take 120 ms to complete, while each subsequent order will need an additional 20 ms.
```python
total_orders = 11
total_time = (120 + 140 + 160 + ... + 320)
average_time = total_time / total_orders # ~220 ms
```

If were to look at the average time to complete an order, we would conclude that the boss/worker thread model is better instead! The term "better" should therefore take into consideration the desired criteria. If execution time is more important, then the pipeline model is better, and vice versa. 

The performance of a model is therefore always relative to the metric being measured, and different metrics may be important in different contexts.

#### Threads

We have made the claim that threads are useful for a multitude of reasons:
- Being able to parallelise processes results in speed up, 
- A hot cache can benefit performance as well due to thread specialisation,
- Multi-threading in general being cheaper than multi-processing, 
- Latency associated with I/O operations can be hidden even on a single CPU.

However, the notion of "usefulness" is more complex than cursory glance would imply. As examples:
1. For a matrix multiplication application, the _raw execution time_ might be the most important metric of usefulness.
2. For a web server application, the _throughput_ (number of requests per time) or simply the _response time_ might be more important.
3. For a hardware chip, the overall _utilisation_ might be more important.

For any of the given examples, we might care about the average value or the maximum/minimum, or even more complicated calculations can be involved. We might also impose a certain criteria, say, an application is sufficiently "good" if it completes a task in 3 ms 95% of the time.

All in all, in order to evaluate a given solution, it is important to **first determine the properties that we care about**. These properties are termed **metrics**, and will be covered in greater detail below.

## Performance Metrics

### Examples of Performance Metrics

###### #DEFINE Performance Metric
> A metric of a system is a **measurable or quantifiable property of interest**. The metric can be used to evaluate the system's behaviour. 

A metric such as _execution time_ (quantifiable) can be used to evaluate different solutions (system behaviour).

Thus far, we have encountered a few useful metrics, such as _execution time_, _throughput_, _request rate_, or _hardware utilisation_. There are many more, but we will look at a couple of examples to further expand our list:
1. In some systems, it is possible that we care more about the time it takes for a job to actually start and not the time taken to complete. This metric, _wait time_, should be low.
2. Although _throughput_ (the number of tasks per unit time) might be important, _platform efficiency_ is something to consider as well. The end user might deem _throughput_ as an important metric. As a provider (e.g., owner of a data center), the utilisation of resources in order to deliver a certain throughput might be more relevant. 
3. _Performance per dollar_ is a clear measure of monetary gain/loss.
4. Alternatively, _performance per watt_ is a possible metric if energy requirements are more important to consider.
5. The *percentage of SLA violations* can also be a consideration, where SLA stands for "Service Level Agreement". The SLA specifies a definite timeframe for answering a request or picking up a query.
6. Chasing certain metrics might not make too much sense. For example, the rush for higher frames per second (FPS) might not make sense as the human maximally perceives 30 FPS. The goal should then be to roughly stay at or above 30 FPS most of the time, resulting in _client-perceived performance_ as the metric of interest.

Metrics can be considered in combination as well, or perhaps new metrics can be derived from aggregating existing ones. Ideally, measurements of metrics are obtained by running real (production-level) software on real machines with real workloads. This is clearly not feasible, hence we must rely on "toy" experiments to simulate a representation of situations we might actually encounter.
###### #DEFINE Testbed
> These conducted experiments and their relevant parameters are termed "testbeds". The testbed informs the where and how of conducted experiments as well as the relevant metrics that are gathered.
### Rethinking "Usefulness"

There is never a straightforward answer to the question, "Is X useful?". Rather, the answer depends on the metrics we collect and the workload of the system. Different answers may arise depending on the two variables.

For instance, certain graph traversal algorithms work best for sparse graphs, while others perform better with densely-connected graphs. Some filesystems are optimised for read access, while others might be more suited for more write-heavy systems.

The answer is therefore: "it depends". The answer is almost always correct, especially where systems design is concerned. However, this is not too helpful, thus the thesis itself must be modified, extending it to consider the metrics and context of evaluation.
## Case Study: Webserver Implementation

### Overview

Suppose we have a simple webserver, whose flow is presented below. All figures used in this case study are pulled from [Pai, Druschel, and Zwaenepoel, 1999](https://s3.amazonaws.com/content.udacity-data.com/courses/ud923/references/ud923-pai-paper.pdf).

![[Pasted image 20230923155820.png]]

The important steps are summarised as:
1. A client/browser sends a request
2. The webserver accepts the request
3. Multiple server-processing steps take place
4. The server responds by sending the file

Some of these steps are computationally expensive (e.g., request parsing after reading or forming a header to send). These will take place on the CPU. Other steps are potentially blocking, such as accepting a connection, sending data, or finding/reading file from disk.
### Multiprocessing 

An easy way to achieve concurrency would be to simply have multiple instances of the same process. 

![[Pasted image 20230923160152.png]]

This approach is extremely straightforward -- simply spawn multiple processes. As we have already established in [[P2L2 Threads and Concurrency]], running multiple processes comes with several downsides:
- A higher memory footprint due to multiple processes may hurt performance.
- Context switching between different processes is expensive.
- It is possibly difficult/costly to maintain shared state across the processes due to IPC constraints.
- It may also be quite difficult to have multiple processes listening on a singular port.
### Multithreading

We may opt instead for a multithreaded approach, achieving concurrency with a single address space.

![[Pasted image 20230923160426.png]]

In the figure shown above, each thread works through **all steps**. Of course, it is possible to implement the boss/workers pattern (where the boss passes requests to the workers) or even the pipeline pattern (where each thread executes a single step). 

The benefits of multithreading are true here as well:
- All threads share the same address space, resulting in lower memory requirements.
- Context switching at the user-level threading library is much cheaper.

However, as we've seen multiple times now:
- Multithreading is never as simple, requiring care regarding synchronisation-related issues. 
- A multithreaded approach also boldly assumes that the underlying OS supports threading. This, admittedly, is less of an issue today than it was in the past.
### Event-Driven

This approach is **single-threaded** and involves watching out for events and invoking a specific event-handler.

![[Pasted image 20230925141138.png]]


"Events" can correspond to:
- Receipt of a client request
- Completion of a send (header or data)
- Completion of a "find file" (disk reading)

The even dispatcher must therefore have the ability to be notified of these events and invoke the appropriate handler. Since this solution is single-threaded, the execution will simply jump to the specific handler's instruction in the singular process address space.

Notice that as a natural consequence, the handler will **always run to completion**. This is not great if a handler may potentially block. As such, the handler needs to **hand control back to the event dispatcher before proceeding to block**.

#### Concurrency with Single Thread (Event-Driven)

It is rather straightforward to appreciate concurrency in terms of multithreading or multiprocessing -- multiple execution contexts are present, after all. In the event-driven model, it is still correct to claim that concurrency is achieved, since **multiple requests are handled simultaneously by virtue of interleaving**.

Taking an example:
- Suppose one client makes a request. The client connection is accepted and will therefore be parsed by the server.
- At some point in handling the first request, the server might block (e.g. waiting for file). Control is handed back to the event dispatcher, which watches for other events to act upon. 
- Simultaneously, two other requests have been made. One might block on the receipt of the request by the server, while another might still be waiting for a connection acceptance. 
- Since the server has to wait before the first request can be advanced, it can proceed to handle the other requests. Similarly, if the request blocks at any time, the event dispatcher simply searches for another event to handle.
- The server thus interleaves the processing of multiple requests, achieving concurrency.
#### Rationale (Why?)

Why do we opt for a single execution context, rather than assigning each event or request a separate execution context (as we would for multithreading or multiprocessing)?

On a single CPU:
- We have established in [[P2L2 Threads and Concurrency]] that context switching only makes sense if the time spent idling in a particular thread is greater than the time it takes to context switch twice (away and back). 
- Therefore, if idle time is minimal, it **does not make much sense to context switch**; CPU cycles could have instead been used for meaningful processing.
- In the event-driven model, a request will be processed up until it blocks, at which point execution will switch over to another request.
- Contrast this to assigning each request/event a separate thread. There is no telling if a thread will smartly context switch _only_ when it encounters a blocking period.

On multiple CPUs:
- The event-driven model still makes sense, especially when there are more requests than CPUs.
- Each CPU can then be allowed to host a single event-driven process, which can handle multiple requests by itself.
- This achieves less overhead than if each CPU had to context switch between multiple threads/processes handling separate events/requests.
- More care must be taken to direct events to the correct CPU, but there are mechanisms in place for this and this concept will not be covered in further detail in this lecture.

In short, the event-driven model's benefits mainly lie in its design: a single address space with a single flow of control.
- The overheads are lower,
- No context switching is required, and
- Issues with synchronisation are not present (i.e., no mutexes are needed here!)

#### Implementation (How?)

Looking at the lowest level, there needs to be some way to receive events from the network or the disk (where a web server is concerned, at least). 
- The OS uses the abstractions **socket** or **file** to represent network or disk respectively. 
- Although sockets and files are distinct terms, they are ultimately represented by the same data structure, **file descriptors**. 
- Therefore, any event is simply an **input on a file descriptor**.

In order to find a file descriptor with an input:
- The `select()` system call can be used. This system call takes a range of file descriptors and returns the first file descriptor with an input.
- Alternatively, `poll()` can be used; this system call is much faster than `select()`.
- In modern approaches, `epoll()` is commonly used. This newer approach eliminates some issues with older alternatives, namely by avoiding the need to scan large ranges of file descriptors that are mostly not relevant or do not have any input at all.
#### Asynchronous System Calls

Recall in the discussion regarding many-to-one user-level to kernel-level thread mappings, there was a possibility that a single blocking user-level thread causes the entire process to block. Briefly, the kernel only ever sees one blocking thread; it does not know that there are other runnable user-level threads.

A similar issue may arise in the event-driven model if one of the handlers initiates a blocking call. One way to circumvent this issue would be to use **asynchronous system calls**. In other words, the actual results can be checked at a later point in time. 
- The process/thread first makes a system call.
- The OS obtains all relevant information from the stack and either learns where to return results, or tells the caller where to retrieve the results from at a later time. 
- The process/thread can then continue.

Asynchronous system calls clearly requires that the **kernel is multithreaded**. While the caller thread continues execution, there must be another kernel-level thread that waits and handles the incoming results (ensures that the results are made available, for example). 

Asynchronous system calls may also be supported at the **hardware level** itself. A thread or process can simply ask the hardware for information and return later to retrieve the appropriate results. 

Asynchronous system calls will be covered in greater detail later in this course, but the basic takeaway would be that asynchronous system calls allows us to bypass the blocking issue. In this case, a blocking I/O operation can simply be allowed to proceed (with support from the kernel/hardware) and a thread can revisit it later.

#### Alternative Solution

The paper was written in a time where support for asynchronous system calls was not widespread. Even today, asynchronous system calls are not ubiquitous. It might also be the case that asynchronous system calls are not available for a device of interest.

The paper suggested the use of **helpers** as a solution. Whenever a handler encounters an I/O operation that might block, it will enlist a helper and returns to the event dispatcher.

![[Pasted image 20230925202432.png]]

Communication with helpers can be performed with sockets or pipes, which both present a file descriptor-like interface -- `select` and `poll` can still be used in this case as well!

The helper itself blocks, but the main event loop (and thus the process) will not. This provides a neat way to achieve the same result as if asynchronous calls were used. The paper itself describes individual helpers as separate processes (as multithreaded kernels were not common), hence the model shown above was termed the **Asymmetric Multi-Process Event-Driven Model** or AMPED. In principle, a multithreaded approach could also be used for similar effect.

Benefits:
- The usage of helpers resolves portability limitations of the basic event-driven model. Here, no assumptions are required regarding a multithreaded kernel or the presence of asynchronous system calls.
- Having "mini" helpers also results in a smaller memory footprint than a regular worker thread/process that handles all steps of an incoming request.

Downsides:
- This model is not immediately applicable to all types of applications. This only fits nicely for our case study of a webserver. 
- The routing of events might be complicated in multi-CPU systems.

_Quiz_
_Of the three models (Boss/Worker, Pipeline, Event-Driven), which model likely requires the least amount of memory?_
The event-driven model is likely less memory intensive. 
- In the other two models, a separate thread is required each request or each stage in request handling.
- In the event-driven model, the handlers are simply procedures in the same address space, while threads are only required for blocking I/O operations.
- In short, extra memory is only required for threads in charge of concurrent blocking I/O, but not for all concurrent requests.
### Flash Webserver

The discussion thus far basically follows the Flash implementation principles. Flash is an event-driven web server that uses the AMPED model. The blocking I/O operations described by Flash are simply disk reads (this was during the time of Web 1.0).

A extra details:
- Communication from the helpers to the event dispatcher is performed via pipes.
- The helper reads the file into memory via `mmap` 
- The dispatcher checks via `mincore` to determine if the pages are in memory, thus decides if a local handler should be called or a separate helper process.
- As long as the file is in memory, reading will not result in a blocking I/O operation, which means that the local handler is preferred. 

Flash also performs **application level caching**. This is performed on both data and computation:

![[Pasted image 20230925204356.png]]

- It is rather common to cache files (data caching) and should warrant no further explanation.
- Computational results may also be cached. For example, every request for a path must be processed and the result returned (e.g., to return some file). This result can be cached to avoid having to recompute the request with the same path.
- Similarly, HTTP header fields are often file-dependent. If the file itself does not change, the HTTP response header can be cached and reused.

Flash also performs certain optimisations that takes advantage of networking hardware. Note that these are now fairly common optimisations but were novel when the paper was first published.
- All data structures are aligned such that DMA (Direct Memory Access) operations can take place without the need for copying data.
- Flash uses DMA operations with scatter-gather support. In other words, the header and the actual data does not have to be contiguous in memory.
### Apache Webserver

The exact details of Apache will not be covered, but some concepts will be briefly discussed in order to make a meaningful comparison with Flash.

![[Pasted image 20230925204913.png]]
- The core component of Apache provides core server functionality: accepting requests, issuing responses, and managing concurrency.
- The various modules (that can be removed or added as one wishes) simply extend the webserver's functionality. This can be seen as roughly analogous to Flash's event-driven model, where each request will ultimately be parsed by all the handlers -- a request will travel down each module in a pipeline-like fashion.
- Apache combines **multiprocessing and multithreading**. Each instance in Apache is a process, which implements a multithreaded boss/worker pattern -- the thread pool itself is dynamically adjusted. 
- The total number of processes can also be dynamically adjusted based on information such as number of pending requests or outstanding connections.

## Performance Comparisons

This section will use information regarding Flash and Apache. However, the focus will be more on obtaining performance metrics and the associated experimental methodology. This is also what the Flash paper performs -- think of this as a Journal Club.
### Experimental Design

A key note regarding experimental design -- an experiment should always help argue a point you are trying to make. A few leading questions might help:
1. What systems are being compared?
	- Control variables where necessary. If two hardware are compared, then software should be kept constant. 
2. What workloads will be used?
	- What will be the input used?
	- Will the workload be real-world data, or some synthetic, dummy data?
3. How will performance be measured?
	- What metrics are to be considered (execution time, throughput, etc.)?
	- Who is this system designed for? Recall from the above section that this informs the metric to be used.

Let us now investigate the three leading questions in context of the Flash paper:
1. What systems are being compared?
	- Multiprocessed (where each process is single-threaded)
	- Multithreaded (boss/worker threading pattern)
	- Single Process Event-Driven (SPED)
	- Zeus (SPED with 2 process), an existing research implementation
	- Apache (v1.3.1, multiprocessed)
	- Flash (AMPED)
2. What workloads will be used?
	- The authors wanted a realistic request workload, but at the same time, a controlled, reproducible input was desired. 
	- They settled on a **trace-based** approach, where workloads from real webservers were pulled and replayed.
	- The workloads used encompassed the CS Web Server trace from Rice University, Owlnet trace from Rice University, and a synthetic workload. 
		- The first involved a large number of file requests, which would likely not fit in a typical server memory. 
		- The second involved a much more reasonable workload. 
		- The third served to generate specific cases (e.g., best/worst case) for more pointed investigations.
3. How will performance be measured?
	- Bandwidth was considered as a metric. This is a common metric used to evaluate webservers, and can be defined as the **total number of bytes transferred over the total amount of time for transfer**.
	- The authors were primarily interested in Flash's ability to perform concurrently, hence chose to evaluate **connection rate** as a metric. This is defined as the **total number of client connections serviced over the total amount of time passed**.
	- Both of these metrics were evaluated as a function of file size. A larger file size results in amortisation of connection cost, hence a higher bandwidth. However, a larger file size might negatively connection rate, as more work is to be done per request. 
	- File size is therefore a variable that can be used to investigate the two metrics: bandwidth and connection rate.

### Experimental Results

#### Synthetic Best Case

The synthetic workload was first considered, where a best-case scenario was generated (more concretely, all requests were simply for `index.html`). Since only one file will ever be requested, `index.html` can reside in cache, resulting in faster servicing of subsequent requests.

As mentioned briefly above, the file size was varied. In this case, file sizes ranging from 0 to 200 kB were used and the relationship with **bandwidth** was graphed:

![[Pasted image 20230926164732.png]]

A few observations:
- All implementations showed similar _trends_. As file size increases, bandwidth increases as well, before eventually plateauing. 
- SPED demonstrated the best performance, followed closely by Flash. Flash exhibits lower performance here due to the extra checking for memory presence. 
- Zeus contains an anomaly, where bandwidth suddenly drops after a file size of 125 kB -- likely a buggy implementation.
- The MT (multithreaded) and MP (multiprocessed) implementations are understandably slower due to the need for context switching and other synchronisation mechanisms.
- Apache performs the worst due to the lack of any form of optimisations.

#### Realistic Best Case

The Owlnet trace is similar to a best-case scenario. It is a small trace, and thus will **mostly** fit in cache. The results echo this as well, showing a trend that is roughly similar to that observed in the synthetic best-case scenario. Note that Zeus is not looked at here.

![[Pasted image 20230926165935.png]]

One observation:
- In a realistic case, blocking I/O is now potentially required. As such, SPED performs worse than Flash, since Flash outsources blocking to its helpers.

#### Realistic Normal Case

The CS trace is a much larger trace, which would mean that requests are likely not all serviced from cache.

![[Pasted image 20230926170143.png]]

Observations:
- Since the SPED implementation must necessarily block with no way of switching execution, its performance suffers. 
- The MT implementation performs slightly better than the MP implementation as threading involves a smaller memory footprint, thus more memory is available to actually cache files. Naturally, threads are also cheaper in general than processes.
- Flash clearly outperforms other implementations, presumably due to its smallest memory footprint. As a direct result, more files will be available in cache and less blocking I/O operations will be required. 
#### Effect of Optimisations

The authors had also investigated the effect of optimisations on the connection rate.

![[Pasted image 20230926201619.png]]

Observations:
- In all cases, connection rate decreases with increasing file size.
- The connection rate is improved when optimisations are performed (note that path and mmap are optimisations)
#### Results Summary

In short, when data is in cache:
- SPED > AMPED Flash. 
- SPED and AMPED Flash > MT/MP. The former implementations avoid the overhead associated with synchronisation and context switching.

When data is not in cache (i.e., disk-bound workload):
- AMPED Flash > SPED. SPED blocks due to the lack of asynchronous I/O support.
- AMPED Flash > MT/MP. The former is much more memory efficient and context switching is only performed when a blocking I/O is encountered.

_Quiz_
_The image below shows another graph from Pai, Druschel, and Zwaenpoel, 1999. Focus on the performance of Flash and SPED. At about 100 MB, Flash performs better than SPED. Why? Select all that apply._

![[Pasted image 20230926202452.png]]

1. Flash can handle I/O operations without blocking. TRUE.
2. At that particular point in time, SPED starts receiving more requests. FALSE.
3. The workload becomes more I/O bound. TRUE.
	> The issue with SPED once data size increases is that more I/O operations are required. It blocks and incoming requests have to wait, which negatively impacts overall bandwidth. 
4. Flash can cache more files. FALSE
	> SPED and Flash should have similar memory footprints. If anything, Flash should require more memory for the helper processes.

## Summary and Final Notes

### Choosing Experiments

In summary, relevant experiments should enable one to make certain statements about a solution that others will **believe in** and **care for**.

Going back to the webserver example once more, experiments should inform statements about metrics such as _response time_ or _throughput_. Both metrics make sense from the point of view of two key stakeholders:
- Clients care about _response time_, while
- Operators care about _throughput_.

The clients or operators might have certain goals for the webserver. The following lists potential examples, though note that this list is non-exhaustive and goals will mostly change based on context:
1. Increase response time and throughput.
2. Increase response time
3. Increase response time but throughput is negatively affected
4. Maintains response time even under high stress (high request rate)

By **understanding the relevant stakeholders and their goals**, we are able to gain the necessary ideas for choosing metrics or configuring our experiments. 
### Choosing Metrics

A rule of thumb for picking metrics would be to simply look at the "standard" metrics that are used in the target domain. For example, web servers are often concerned with client request rate or server response time. This allows for a broader audience that will be able to understand and interpret the results obtained.

In addition, metrics should answer the who/what/why questions described briefly above.
- Why is this being done?
- What problems are being tackled?
### Choosing the Right Configuration Space

It is important to first determine what may affect the chosen metrics, and from there selecting key parameters to vary.

1. Firstly, **system factors** may affect metrics that are analysed. For example, system resources (such as CPU or memory) can definitely affect the results of a particular metric. 

2. Of course, the input **workload** must be considered. For a web server, it is possible to investigate different request rates, the file size request, access patterns, and much more.

3. Once the configuration space is well understood, some subset of configuration parameters must be chosen. Sometimes, the best approach might involve a range of values for a parameter. These ranges themselves should be meaningful and realistic (a file size of 1kb is not too informative when testing a web server).

4. Note that it is possible to look at worst/best case scenarios, which involves using configurations that are not technically "realistic". However, worst/best case scenarios do have value as they potentially demonstrate certain limitations or opportunities presented by the system in question.

5. Many factors may often describe or argue for a similar point. As such, useful combinations of factors should be chosen, although note that similarity can be potentially useful to solidify an argument or hypothesis -- there is just no need to overdo things.

6. Naturally, certain variables must be kept constant in order to make meaningful conclusions. Compare apples to apples.
### Choosing the Comparison

It is often the best to compare the proposed system to a state-of-the-art implementation, or at least one that is considered "standard". The experiments must show that some improvement has occurred.
### Running the Experiments

Once the experiment is properly designed, conducting the experiment itself is relatively straightforward:
1. Run the test cases `n` times
2. Compute the relevant metrics
3. Represent results (data visualisation)

Of course, all papers must have a Results section followed by a Discussion section. Actually discuss and make conclusions about the results instead of simply stating what is already immediately obvious from a few graphs.

_Quiz_
_A toy shop manager wants to determine how many workers to hire in order to handle a worst-case scenario. Orders range in difficulty from blocks < teddy bears < trains. The shop has three working areas, each with tools for any toy. These work areas can be shared among multiple workers._

_Which of the following experiments represented as (types of orders, number of workers) will allow us to make meaningful conclusions about the manager's question?_

1. { (train, 3), (train, 4), (train, 5) }
2. { (blocks, 3), (bears, 6), (trains, 9) }
3. { (mixed, 3), (mixed, 6), (mixed, 9) }
4. { (train, 3), (train, 6), (train, 9) }

The answer is (4). The worst case scenario is used, where a train order is processed. This is better than (1), as the number of workers per work area is similar (there are three work areas, hence the number of workers should be a multiple of three).




