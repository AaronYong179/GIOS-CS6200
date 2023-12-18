## Introduction and Overview

### DFS Review

In the previous lecture [[P4L2 Distributed File Systems]], the "state" (or more specifically, the files) were **stored** on some set of server nodes and **accessed** by some set of client nodes.
- The servers own and manage the state (files), and provided a service (file access operations) for clients to operate on said state.

The primary focus of the lecture on DFS was the **caching mechanisms**.
- Briefly, clients were able to operate on a locally cached file, which resulted in improved performance from the client's perspective.
- Since the server does not have to worry about _every_ file operation, this resulted in improved scalability for the servers as well.

We did not cover:
- How multiple servers are to share and coordinate state amongst themselves.
- The scenario in which all nodes in the distributed system are both servers and clients (peers).

In this lecture on distributed shared memory, we will consider the scenario involving peers. This lesson is also generalisable to other forms of applications, but the discussion will be centered about distributed shared memory.
### Peer Distributed Applications

In most modern distributed applications, the total state is shared across all nodes, with different nodes both requesting and serving state at different points in time.

In particular, each node "owns" some portion of the state -- some state is locally stored on a particular node, for example. Therefore, it can be said that the overall applications state is the **union of all the pieces of state** on all nodes.

These nodes are thus said to be **peers**. They all require accesses to some state located elsewhere in the system and at the same time provide accesses to their local state. Examples of peer distributed applications include:
- big data analytics,
- web searches,
- content sharing, or
- distributed shared memory (DSN)

Note that it is likely for there to be some nodes that provide overall control or management services for the entire service. In a true "peer-to-peer" system, these tasks would be performed cooperatively by all peers.
### DSM Motivation

Distributed shared memory is a service that manages memory across multiple nodes, such that applications running on top will be under the impression that they are running on a single shared-memory machine.
- Each node in the system owns some portion of the total physical memory.
- Each node must of course provide read and write operations on its portion of memory. These read/write operations may originate from any of the other nodes in the system. 
- Each node must also be involved in some **consistency protocol**, to ensure that shared accesses have meaningful semantics.
	- More concretely, read and write operations must be observed by the other nodes in consistent ways -- ideally emulating a singular shared memory machine.

DSMs are important as they permit scaling **beyond single machine memory limits**. The monetary cost starts to increase rapidly with increasing memory resources within a single machine. For instance, a machine with large amounts of memory might break the half million dollar range. Therefore with DSMs, we are able to add "shared" memory at lower costs. 

Naturally, accessing remote memory will be slower than accessing local memory. However, it is possible to hide these delays by optimising and making smart design choices.

DSMs are becoming more relevant today, as commodity interconnect technologies offer increasingly low latencies between nodes in a system via **Remote Direct Memory Access** (RDMA) interfaces.

## Hardware vs. Software DSM

Hardware supported DSM relies on some (high-end) physical interconnect. The OS running on an individual node will be under the impression that it has access to a much larger physical memory, and will be allowed to establish virtual to physical mappings that point to physical addresses on other nodes.

![[Pasted image 20231121133509.png]]

Memory accesses that do not correspond to local, physical memory addresses, will be passed to the Network Interconnect Card (NIC). 
- The requester's NIC will be able to translate the memory access into interconnect messages that are then passed to the NIC of the correct node.
- The NICs are involved in all aspects of memory management, ensuring consistency, and even support atomics -- as we have discussed in section on shared memory.

While it is certainly very convenient to rely on hardware, this type of hardware in particular is typically very expensive and thus usually reserved for high-end machines. Instead, DSM is often realised in software. The software will therefore have to:
- Detect and distinguish local versus remote memory accesses.
- Create and send messages to the appropriate node.
- Accept messages from other nodes and perform the encoded memory operations.
- Be involved in all aspects of memory sharing and consistency support.

Software handling of DSM can be performed at the **OS level** or with a **programming language runtime**.

_Quiz_
_According to the paper "Distributed Shared Memory: Concepts and Systems", what is a common task that is implemented in software in hybrid (hardware + software) DSM implementations?_
- _prefetch pages_ (TRUE)
- _address translation_
- _triggering invalidations_

Address translations and triggering invalidations are well defined and easier to implement in hardware. The need to prefetch pages might differ (based on the type of application, for example), and is therefore best implemented in software instead.
## DSM Design

Several design points are necessary to consider when discussing DSM design.
### Sharing Granularity

In shared memory multiprocessor (or symmetric multiprocessor, SMP) systems, the **granularity of sharing is the cache line**.
- The hardware tracks concurrent memory accesses at the granularity of a single cache line.
- The necessary coherence mechanisms are triggered if a shared cache line has been modified.

For DSM, generating coherence traffic at this granularity will be too expensive, given that this traffic has to travel over a network. Instead, we might consider larger granularities for sharing:
- Variable granularity
- Page granularity
- Object granularity
##### Variable Granularity

Variables are a meaningful unit from a programmer's perspective, although variables are likely too fine-grained. For example, integer variables are merely a few bytes long, and the overheads are likely not justifiable.
##### Page and Object Granularity

Pages and Objects are preferred as they offer larger granularity. 

If the DSM system is to be integrated at the OS level, page-based granularity should be used, as OSes have no concept of "Objects". 
- The OS then can track pages that are modified, and trigger the necessary exchange of messages between remote nodes. 
- Pages are larger (recall, 4 kB in common environments), hence it is possible to amortise the cost of remote access across these larger granularities.

For granularity at the level of Objects, there are two options.

Firstly, it is possible lay out application-level Objects on separate pages (with some help from a compiler). The page-based OS-level mechanism can then be easily leveraged.

Alternatively, it is possible to have a DSM that is supported by the programming language, where the runtime understands which Objects are local or remote.
- For remote Objects, the runtime will then handle all necessary communications with remote nodes and the necessary operations for providing distributed shared memory.
- The OS does not need to be modified, since it is not involved in DSM.
- However, this method assumes the need for a programming language to support DSM.
##### False Sharing

With larger pages or objects being used for sharing, **false sharing** might occur.

Consider a page that has two variables, `x` and `y`. 
- A process on one node is exclusively accessing and modifying `x` (it does not care about `y`).
- Another process on another node is exclusively accessing and modifying `y` (it does not care about `x`).

As `x` and `y` are on the same page, any modifications will trigger coherence mechanisms (and all the associated overheads) although these actions are functionally superfluous.

The programmer must then be careful when allocating such variables, and place them on different pages, or group them into different objects if necessary. Alternatively, one could rely on some smart compiler to determine if there is any shared state at all.
### Access Algorithm

It is important to understand the types of memory accesses that higher-level applications are expected to perform. 

The simplest application involves a **single reader and a single writer**. 
- The main role of the DSM layer here is to provide the application with the ability to access additional, remote memory.
- There is no need to worry about consistency of sharing at the DSM layer.

However, when the application involves **multiple readers and multiple writers**, the DSM has more complex roles.
- It is now important that the reads return the most recent value at a memory location.
- It is also important that all writes are correctly ordered and performed appropriately.

Briefly put, with multiple readers/writers, care must be taken to present a consistent view of the distributed state to all nodes in the system.
### Migration vs. Replication

For a DSM solution to be useful, it must provide good performance to the applications above it. As the core service provided by DSM is memory access, the key performance metric to optimise would be **access latency**.

Clearly, accessing local memory is much faster than accessing remote memory. Therefore, it is important to maximise the proportion of local memory accesses. There are two ways of doing this, **migration** or **replication**.

**Migration** involves copying required state over to a requesting node.
- Migration in particular works in the single writer/reader scenario, since only one node will be accessing state at any given point in time.
- However, moving data does incur overheads.
- As such, if the other node only sees a single access, then migrating will not be amortised sufficiently.

**Replication** involves copying state to multiple nodes.
- This is necessary for multiple readers and writers, since the state will need to be accessed at multiple nodes at the same time.
- This solution is much more general, but it requires consistency management (e.g., caching techniques). 
- In particular, all writes must be ordered correctly, and the state copies must reflect the must recent updates for incoming reads.
- Overheads involved in maintaining consistency can be controlled by limiting the number of replicas that can exist in the system at any given time.
- After all, the consistency overhead is proportional to the number of replicas.

_Quiz_
_If access latency (performance) is a primary concern, which of the following techniques would be best to use in a DSM design? Check all that apply._
- _Migration_ (FALSE; only for single writer single reader)
- _Caching_ (TRUE)
- _Replication_ (TRUE)

For many concurrent writes, the overheads associated with caching or replication may be high. Recall from [[P4L2 Distributed File Systems]] that a solution to this was simply disabling caching when multiple concurrent writes are encountered. This is to avoid the overheads associated with repeated cache invalidations and consistency maintenance.

Caching versus invalidation?
> Caching involves moving data "closer", such that data access time is reduced. Replication involves copying data to multiple locations, perhaps to distribute the load of accessing data across multiple nodes.
### Consistency Management

Of course, once state is allowed to exist concurrently on multiple locations, it becomes important to manage consistency. DSM systems are expected to behave similarly to SMP systems, so discussion should first start by considering consistency strategies for SMP systems.

Recall from [[P3L4 Synchronisation Constructs]] that there are two such strategies, namely:
- Write-invalidate, where all cached values are invalidated except for that of the writer.
- Write-update, where all cached values are updated to reflect the new value.

Recall also that these coherence operations are triggered by the shared memory support in hardware on _every single write operation_. Simply using this for DSM is not valid, since the associated overheads (with network costs) will be too high.

1. One option is to **push** invalidation messages when data is being written to. This is similar to the server-driven approach mentioned in [[P4L2 Distributed File Systems]], though in the context of DSM, all nodes are peers and there is no distinction between "servers" and "clients".
	- This approach is also referred to as "proactive", "eager", or "pessimistic".
	- This method is "eager" as it forces propagation of information immediately.
	- Pushing is "pessimistic" as it assumes that information will be required by nodes immediately after modification.

2. Alternatively, one node could **pull** modifications from other nodes in the system either periodically or on-demand.
	- This approach is also referred to as "reactive", "lazy", or "optimistic".
	- This method is "lazy" as it only retrieves information when convenient or when absolutely necessary.
	- Pulling is "optimistic" as it assumes that information is not needed urgently by other nodes, and it would be better to accumulate more modifications before paying the cost of moving data across nodes in the system.

Regardless the strategy chosen, _when_ these methods get triggered will depend on the consistency model for shared state. Options for consistency models will be discussed below.
## DSM Architecture

#### Basics

A DSM system will consist of multiple nodes, each with its own physical memory.
- Each node may contribute either a **portion** or **all** of its physical memory towards distributed shared memory.
- Assume here that only a portion is contributed. In other words, only a portion of this each node's physical address can be explicitly addressed.
- The rest of physical memory might be used for caching, replication, or to simply store some metadata required for the DSM layer.

![[Pasted image 20231123095720.png]]

The pool of memory that each node contributes forms the global shared memory available for applications running on this system.
- Therefore, every address in this system must be uniquely identified by a combination of the **node identifier** and the **page frame number**.
- A node where a page is located is typically referred to as the **home node** for that page.
#### Home and Owner Nodes

Consider the multiple readers/writers scenario. In order for the system to deliver acceptable performance and achieve low latency (for memory accesses), the DSM layer should incorporate **local caching**.
- In particular, pages will be cached on the nodes where they are accessed.
- The home node for that page should need to drive all coherence related operations.
- The home node will therefore have to maintain some state:
	- Pages accessed,
	- Modifications,
	- Caching enabled/disabled (in the event of increased write events)
	- Lock status, 
	- and much more

When a page is repeatedly (or perhaps, exclusively) accessed on a node that is **not its home node**, it becomes quite expensive to repeatedly contact the home node for the necessary state.
- As such, it is important to consider separating the notion of a **home node** from the **owner node**.
- It is owner that controls state modifications and drive coherence mechanisms relating to its page.
- The owner node might change over the lifetime of an application's execution, as a page might be accessed by different nodes at different times.
- The role of the home node is then to keep track of the current **owner** of a particular page.

> This section is _very poorly explained and structured_.
>
> Quite simply, one would think that the home node is responsible for maintaining coherence. However, this is not very feasible due to the possibility of access repeatedly coming from a different node altogether -- this becomes expensive.
>
> As such, the owner node would be the one responsible for maintaining coherence. In this case, notice that all nodes will be responsible for the management of the overall distributed memory -- there is no "server" and "client" in that sense.

#### Additional State

In addition to creating page copies via caching, page replicas can be explicitly created for load balancing, performance, or reliability reasons.
- For example, data centers often triplicate shared state (one on the main machine, one on a nearby machine, and one on a remote machine).
- The consistency of these nodes is either managed by the home node or by some manager node.
#### Summary

Here are the key architectural features of DSM systems (specifically page-based DSM).
- Each node contributes a part of its memory pages to DSM.
- Local caches are needed for performance by minimising latency of memory accesses.
- All nodes are responsible for some portion of distributed memory.
- The home node manages accesses and tracks page ownership.
- Explicit replication is possible for load balancing, performance, or reliability. 
#### Indexing Distributed State

As mentioned briefly above, each page must be identified by its home node ID as well as the page frame number -- this constitutes the address of the page/object. The term "page" and "object" will be used interchangeably here.

It follows that all nodes must be able to identify a given home node (or manager node) ID, hence manager information is available on each node.
- In fact, a map from the page/object ID or address to the corresponding manager node ID must be made available to all nodes.
- This can be achieved by maintaining a global map, which must be available across all nodes.
- This global map is therefore said to be **replicated** across all nodes.

Each manager will in turn maintain the per-page metadata data necessary for specific access to that page or to force its consistency.
- In other words, metadata for local pages is **partitioned** across the system.
- Just a quick recap, contrast this to the global map of manager information being **replicated** across the system.

For additional flexibility, it is possible to _not_ encode manager ID information within a particular address.
- In this case, there might be some additional mapping table that converts an input object ID into the corresponding manager node.
- This perhaps could entail a hashing function, where `hash(object)` would return the manager ID.
- With this approach, the manager node can always be swapped by updating the mapping table, without manipulating the object's address.
## DSM Implementation

An important consideration for DSM: the DSM layer _must_ intercept every single access to the shared state. This is required for two reasons:
1. The DSM layer would need to distinguish between remote or local accesses. In particular, it should handle remote accesses by sending a message to the corresponding node.
2. The DSM layer should also be able to detect if a node is updating a locally-controlled portion of distributed shared memory, such that it can trigger the necessary coherence messages.

These overheads should be avoided when accessing local, non-shared state. Therefore, it would be best to **dynamically engage and disengage the DSM layer** whenever necessary based on the type of access being performed.

To achieve these goals, it is possible to leverage hardware support at the MMU level. 
- Recall from [[P3L2 Memory Management]] that the MMU will trap into the OS if no valid mapping for a virtual address is available, or if it detects a write attempt to protected memory.

Whenever access to a remote address is required, there will not be a valid local mapping on a particular node.
- The hardware thus generates a trap, and the OS can then invoke the DSM to send out the appropriate remote message.

Whenever content is cached locally on a node, the DSM layer will ensure that the memory is write-protected.
- As such, any modifications will result in a trap to the OS, which can then invoke the DSM to generate coherence messages.

Additionally, it can be useful to leverage other information made available by the MMU. For example, it is possible to track whether a page is dirty or has been accessed -- this is particularly useful to implement different consistency policies.

For an object-based DSM system that is implemented at the level of the programming language runtime, the implementation can consider similar mechanisms that leverage underlying OS services. Alternatively, everything can be realised in software. 
## Consistency Models

### Preamble

The design of a DSM system or how coherence mechanisms will be triggered depends on the consistency model being used. Consistency models exist in the context of the implementations of applications/services that manage distributed state.
###### #DEFINE Consistency Model
> The consistency model is a **guarantee** that state changes will **behave a certain way** as long as the upper software layers **follow a certain set of rules**.

For memory (state) to "behave correctly", the guarantee is that:
- Memory accesses are ordered similar to how the operations were issued in the first place.
- Memory updates will be propagated/visible such that a reader will see a value representative of what was recently written.

The software must therefore use a specific set of exported APIs, in order to be given the above guarantees. Of course, if a stronger set of guarantees are required, it is up to the software to enforce these itself.
> This is similar to synchronisation constructs being used to enforce guarantees such as mutual exclusion. The CPU does not guarantee any particular ordering, but simply promises that all threads will eventually be executed. Requiring an additional trait would see the software enforcing it itself.

In the subsequent sections, consistency models will be examined with the use of timeline diagrams. Notations used will be as follows:
- $R_{m1}(X)$ refers to a **read** of value $X$ from memory location $m1$.
- $W_{m1}(Y)$ refers to a **write** of value $Y$ from memory location $m1$.

It will be assumed that all memory locations are initialised to $0$ at the start.
### Strict Consistency

For a perfect consistency model, updates would be visible everywhere, immediately. These updates would also occur in the exact same manner as they were performed.

![[Pasted image 20231123135134.png]]

In practice, even on a single SMP system, there are no guarantees on the ordering of accesses from different cores unless synchronisation constructs are used. In distributed systems, the additional latency and possibility of message loss or reordering make strict consistency **impossible to guarantee**.

Therefore, strict consistency is purely a theoretical model, and is not sustainable in practice. Other consistency models are used instead.
### Sequential Consistency

_The lecture illustrates sequential consistency very poorly. This is my rephrasing_.

Sequential consistency ensures that the order in which operations are executed by different processes in the system appears to be consistent with a **global order of execution**.

Quite simply put, all processes **must see the same order of operations**, though the operations themselves can be interleaved arbitrarily. Consider the following example:

![[Pasted image 20231123143614.png]]

In this case, a read at location $m1$ does not reflect Process 1's write with a value of $X$. However, recall that operations between processes can be interleaved arbitrarily. It might be possible that Process 2's write at $m2$ with value $Y$ occurred first.

However, the following is not allowed:

![[Pasted image 20231123144029.png]]

In the above (incorrect) case, Process 3 sees (via its reads) that $W_{m1}(X)$ occurred before $W_{m2}(Y)$. Process 4, on the other hand, sees a different sequence, where $W_{m1}(X)$ did not occur before $W_{m2}(Y)$.

In summary, sequential consistency provides the following guarantees:
- Memory updates from **different processors** may be arbitrarily interleaved.
- However, **all processes** will see the same interleaving.

It follows that operations from the **same process** will always appear in the order they were issued. In other words, updates made by the same process will not be arbitrarily interleaved.
> This makes sense, and one would not expect operations in a singular process to jump all over the place -- that would make programming very difficult indeed.

### Causal Consistency

Perhaps it is not too important to force all processes to "see" the exact same order of operations.

![[Pasted image 20231123144635.png]]

For example, P3 and P4 might perform very different operations with values at $m1$ and $m2$. In fact, the update to $m2$ by P2 was completely independent of the update to $m1$ by P1.

![[Pasted image 20231123144725.png]]

In the figure above, the value at $m1$ and $m2$ are no longer completely independent. Clearly, P2 reads $X$ from $m1$ before writing $Y$ to $m2$. As such, the observation made by P4 as depicted above is no longer valid.

**Causal consistency** here guarantees that these causal relationships between operations will be detected. Causally related operations are guaranteed to be ordered correctly.

In the situation above, a causal consistency model will enforce the reading of $X$ from $m1$ before reading $Y$ from $m2$. 

For writes that are not related -- or as referred to in a causal consistency model, **concurrent** writes -- there are no such guarantees. 

Similar to sequential consistency, writes performed by the same process will be visible in the exact same order on other processes.
### Weak Consistency

In weak consistency models, an additional operation is exported alongside the standard `read` and `write` operations. This additional operation is known as `sync`.

A call to `sync` introduces a **synchronisation point**, which has two main functions:
- A synchronisation point ensures that all updates on the calling process will become visible to all other processes.
- A synchronisation point ensures that all updates from other processes will be visible to the calling process.

In particular, consider the figure below. 

![[Pasted image 20231123150907.png]]

Even if P1 calls `sync` after writing to $m1$, there is no guarantee that P2 will see that particular update at this moment. P2 must explicitly synchronise with the rest of the system as well. Only once P2 calls `sync`, it is guaranteed that P2 will read the value $X$ written to $m1$. There are no guarantees to be made between calls to `sync`.

Note that in the example shown above, a single synchronisation operation was used for all variables in the system. However, it is also possible to have synchronisation operations at different levels of granularity (e.g., per-page).

Perhaps this should have been mentioned earlier, but a system generally provides two distinct calls for synchronisation.
- An entry/acquire point can be invoked when a calling process wishes to see the updates by other processes. This is similar to the `sync` call by P2. 
- An exit/release point can be invoked when a calling process wishes to make its updates known to other processes. This is similar to the `sync` call by P1.

These finer-grained operations allow the system to control overheads imposed by the inclusion of the DSM layer. In other words, data movement and coherence operations between nodes in the system are limited unless explicitly invoked.

However, the DSM layer will have to maintain some additional state to allow it to support the additional `sync` operation.

_Quiz_
_Consider the following sequence of operations. Is this execution sequentially consistent?_

![[Pasted image 20231123152222.png]]
Yes.

_Quiz_
_Consider the following sequence of operations. Is this execution sequentially consistent? Is it causally consistent?_

![[Pasted image 20231123152308.png]]
It is both sequentially and causally consistent.

_Quiz_
_Consider the following sequence of operations. Is this execution sequentially consistent? Is it causally consistent?_

![[Pasted image 20231123152445.png]]
It is neither.

_Quiz_
_Consider the following sequence of operations. Is this execution weakly consistent?_

![[Pasted image 20231123152549.png]]
Yes.

_Quiz_
_If the above timeline was used and the sync operations were ignored, is this execution causally consistent?_

![[Pasted image 20231123152649.png]]
No.
