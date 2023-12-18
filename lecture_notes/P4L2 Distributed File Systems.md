## Introduction and Overview

Recall from previous discussions about file systems in [[P3L5 IO Management]] that modern OSes hide different file systems under a VFS interface (Virtual File System). Underneath this VFS, the OS can also hide the fact that files are not even stored on local physical storage. 
- Instead, everything is maintained on a remote machine (or remote file system) that is being accessed over the network. 
- Simply put, **distributed file systems** involve multiple machines delivering a file system service. 
- A simple two-machine setup (one client, one server) can be considered a distributed file system. For example, a client may cache data, while the server actually stores the data.
### DFS Models

More generally perhaps, a distributed file system is a file system that can be organised in any of the following ways:

1. The first model involves client(s) and a singular file server on different machines.

2. The second model involves **distributing** the file server functionality across multiple machines.
	- Each server might carry all files (i.e., **replicated**) to aid in the event of failures, as there will always be a replica of the file on any other server. Additionally, this helps with availability as incoming requests can be split across the servers.
	- Alternatively, each server might carry a percentage/portion of files (i.e., **partitioned**). This benefits scalability, as machines can be added to accommodate more files.
	- More commonly, a combination of replication and partitioning is used. 
		- For example, the files can be partitioned, and each partition replicated.
		- A lot of the "Big Data" companies such as Facebook and Google use this approach.

3. Finally, files may be stored on and served from **all machines**. 
	- This model blurs the distinction between client and server machines -- it is more correct to deem all the nodes in this system **peers**.
	- All peers are responsible for maintaining files and providing the file systems service.

This lesson will primarily focus on the first model of distributed file systems. Caching and consistency management in such environments will also be covered. 

In the next lesson, issues relating to distributed state management among peers will be discussed -- using memory as an example. However, lessons learned from distributing memory can also be applied to file systems.
## Remote File Service

Consider the different ways in which a remote file service can be provided. Only one client and one server will be considered.
### Upload/Download

![[Pasted image 20231102201617.png]]

On one extreme, there is the upload/download model. Whenever a client wishes to make any changes to a file, it will download the entire file, perform local operations, then uploads the file back to server once it completes.

This is less similar to a proper file system, but is more similar to the way FTP servers or version management systems (e.g., SVNs) behave.

A clear benefit is that the client is allowed to perform local reads or writes, speeding up client access once the file is downloaded.

However, the entire file needs to be downloaded and uploaded even for small file accesses, which might be unnecessarily expensive. In addition, this removes control over a file from the server -- the server does not know what a client is going to do to a file, nor does it know when it will receive the modified file in return. This makes file sharing a bit more complicated.
### True Remote File Access

On the other extreme, the client _must_ go to the server for every access to a remote file. The client makes no attempt to store any file locally.

This allows the server to have full control and knowledge over how its clients are modifying potentially shared state and shared files. This makes it a lot easier to reason about consistency, and the server can ensure that multiple clients are not overriding the same portions of a file at the same time.

However, every file operation will have to pay costs associated with remote network latency. Even if a client performs multiple reads, this latency must be paid, since this model bypasses any client means for caching that file locally. Moreover, since all accesses go through the server, this limits the scalability of the server -- it will be overloaded much more quickly.
### A Compromise

In reality, implementation follows something in between the two extreme models discussed. 

Firstly, clients should be allowed to **store parts of files locally**, either in memory or on their disk.
- For example, it makes sense for clients to download some blocks of the file currently accessed.
- Clients may also perform certain prefetching techniques, as one would typically perform when prefetching files from disk to memory.
- This leads to lower latencies on file operations that act on cached blocks.
- This also removes some load on the server, since portions of file accesses are performed on the client -- this leads to server scalability.

Since clients are allowed to store portions of files locally, they must be **forced to interact with the server frequently**. 
- The clients will need to inform the server if any local changes have been made. The clients will also need to find out if a cached has been modified by someone else.
- With some reasonable frequency, this allows the server to gain insights into what the clients are doing. The server is also allowed more control over which accesses can be permitted -- this leads to consistency.

However, this model is more complicated.
- The server would have to perform additional tasks and maintain additional state such that it can provide consistency guarantees.
- The clients would also have to understand somewhat different file sharing semantics -- semantics probably different from the regular file access on a local filesystem.
## Intermission; State

Before going into the design and challenges of the models of remote services described above, it is important to understand the distinction between stateless and stateful file servers.
### Stateless vs. Stateful

#### Stateless

A stateless file server does not maintain any information regarding which clients access which files, how many different clients are serviced, or anything else.

Instead, every request must be self-contained and self-describing, such that it contains all the parameters required. This includes:
- The file names being accessed,
- The file offset, and
- Any data that needs to be written

A stateless file server is able to support the extreme models (upload/download or true remote file access). However, it is not able to support a "practical" model, since it cannot support caching and consistency management. Furthermore, since all requests must be self-contained, more bits are required for a proper request transfer. 

A stateless file server does have its benefits. Firstly, no resources (CPU or memory, for example) are used on the server side for maintaining state. Additionally, the server can be simply restarted on failure, since no state is maintained and all requests are self-contained. A stateless file server is said to be **resilient to failure**.
#### Stateful

As the name suggests, a stateful file server maintains some "state" information regarding file access. For example, a particular file may be tracked for clients that have read or written to it recently, or perhaps information regarding which portions are cached by whom can be maintained. 

With state, the following can be supported:
- Caching,
- Locking, since state allows one to keep track of a client that is currently locking a file, and
- Incremental operations. A client can request to read the "next kilobyte", and this would make sense if the current offset was maintained by the stateful server.

However, being stateful results in less resilience to failure. There needs to be checkpointing and recovery mechanisms put in place to ensure that the file system can be recovered to a consistent state after failure. 

There are clearly overheads involved in maintaining state and consistency as well.
- The exact overheads incurred will depend the caching mechanisms and consistency protocols used.

## DFS Design and Mechanisms

### File Sharing

#### Caching and Coherence

As briefly described above, caching allows clients to locally maintain some portion of state (e.g., file blocks), and clients are allowed to perform operations (read/write/open) on cached state. These local operations can be done **without contacting the remote server**.

Caching requires coherence mechanisms.

![[Pasted image 20231104071801.png]]

Suppose that both Client 1 and 2 have cached a file $F$. Client 2 has then modified $F$ to $F'$, and has updated the server accordingly. Two main questions then, _how_ and _when_ should Client 1 be updated of this change?

In [[P3L4 Synchronisation Constructs]], an analogous scenario can be observed with the maintenance of cache-coherence. The _how_ would be answered by cache-coherence mechanisms such as write-invalidate or write-update, and the _when_ would simply be when a write occurs to a particular cached variable.

However in this case, there are significant communication costs and latencies associated. It might not make sense to to perform coherence mechanisms on every single write. Since files are being shared in portions, it might not even be necessary!

These coherence mechanisms may be triggered in different ways.
- For example, the mechanism may be triggered any time a client tries to access a file (on demand), or it may be triggered periodically, or perhaps only when a client tries to `open` a file.
- The trigger timing also depends on whether or not a coherence mechanism is client- or server-driven. 
- All that is necessary is for these coherence mechanisms to guarantee some level of consistency. The exact details of _how_ and _when_ coherence mechanisms are executed will depend on file sharing semantics.

_Where do you think files or file blocks can be cached in a DFS with a single file server and many clients?_
- In client memory
- On client client storage devices (HDD/SSD)
- In buffer cache in memory on server -- though usefulness will depend on client load or how client requests are interleaved. If many requests are interleaved, locality may not be achieved, rendering this cache not too useful.
#### File Sharing Semantics

##### Motivation

To understand file sharing semantics on a multi-node system, consider first how it applies on a single-node system.

![[Pasted image 20231104133141.png]]

If files are within a single machine, any changes made by one process will be immediately visible to all other processes. In this case, if Process A writes `c` to the original file, the reader Process B will also see `abc`. 
- Do note that even if this file is not written to disk, Process B will still see the edits that Process A performed, since the file is cached in some buffer within memory.

Unfortunately, this immediate visibility is not true for multi-node systems, where file sharing is concerned.

![[Pasted image 20231104133505.png]]

Even if Process A's edit is pushed to the server immediately, the message might take some time to reach the server. It is possible that during this lag time, Process B attempts to read the same file and sees `ab` instead of `abc`.
> In other words, Process B might not read an updated file _even if_ the read operation occurs after Process A's write!

As message latencies are hard to predict, there is no way to set some delay time before a client is allowed to read a file. Therefore, distributed file systems often sacrifice some consistency in favour of maintaining performance. It can be said that **more relaxed file sharing semantics** are acceptable.
##### UNIX Semantics

When the file system is on a single machine, every write will be immediately visible. This is referred to as UNIX semantics.
##### Session Semantics

With session semantics, a client writes-back all changes it has performed on `close()`, and checks with the server for the latest file version (update) on `open()`. Note that on `open()`, a client does not use its cached version of the file. 

A "session" is defined as the duration between a call to `open` and a call to `close`. 
- With session semantics, it is possible for a client to be reading a stale version of a file.
- However, we are at least guaranteed on every `close` or `open` that we are consistent with the rest of the filesystem at that particular moment.

Session semantics are easy to reason about, but they are not great for concurrent access to a file. For example, if two clients want to write to a file and be aware of each other's updates, they will need to constantly `close` and `open` the file.

Also, the longer the session, the more inconsistent a file will become. 
##### Periodic Updates

By introducing timed updates, client will write-back periodically. It is possible to think about this as the client having some "lease" on cached data, though this does not necessarily mean exclusivity such as locking. In a similar manner, the server will also periodically send out invalidations to the clients, notifying them to update their own cached copy. 

By enforcing periodic updates, certain bounds on inconsistency can be provided -- a file can only be inconsistent up to the next update. Since the client might not necessarily keep track of update periods, the filesystem can provide explicit operations to let a client `flush` its updates to the server, or `sync` its state with the server.
> Note that `flush` and `sync` are also used locally to maintain consistency with disk -- these operations are not DFS-specific.
##### Other Strategies

There are multitudes of other strategies that can be put in place for file sharing, but will not be covered in great detail here.

For example, files may be simply **immutable**. A file is never really modified, rather a new file is created instead. 

A useful semantic to provide would include **transactions**. This involves filesystems exporting some API to allow clients to group file updates in an **atomic operation**. 

_Quiz_
_Imagine a DFS where file sharing is implemented via a server-driven mechanism and with session-semantics. Given this design, which of the following items should be part of the per-file data structures maintained by the server? Check all that apply._
- _Readers_ (TRUE)
- _Current writer_
- _Current writers_ (TRUE)
- _Version number_ (TRUE)

A server driven mechanism means that the server will be the one who pushes invalidations to the clients. Session semantics involve making file changes visible when the file (session) is closed and when a file (session) is opened. 
#### File vs. Directory Service

When choosing the best file sharing semantics to support, it is important to first understand file access patterns. For example,
- The sharing frequency of files,
- The write frequency of files, and
- The importance of a consistent view -- whether this can be more or less relaxed.

Again, a common theme would be to optimise for the common case. One issue however, is that filesystems have two types of files: regular files and directories. These two files will often have different file access patterns such as the locality, lifetime, access frequency, and so on. 

As files and directories are accessed differently, it is common for different policies to be chosen for each. 
- For example, session semantics are used for files, and UNIX semantics are used for directories.
- Alternatively, if periodic updates are used for both, then perhaps files require less frequent write-back compared to directories.

Simply put, shared access to a file occur much more frequently than shared access to a file. It therefore makes sense to enforce a greater level of consistency for directories than for files.
### Replication vs. Partitioning

Recall from above that the file server may be distributed across different machines. This can be achieved via **replication** or **partitioning**. 
##### Replication

Replication involves each machine holding all files.

Benefit(s) involve:
- Load balancing, where multiple client requests can be spread out amongst the various server machines,
- Availability, since the system will return results more quickly due to load balancing, and
- Fault tolerance, where a single machine failure is not an issue due to redundancy.

Drawback(s) include:
- Write operations become more complex. In addition to consistency between clients and the server, there is a need to worry about consistency between server replicas.
	- A simple solution involves writing synchronously to all replicas _before_ returning to the client, though this slows down write operations significantly.
	- Alternatively, writes can occur on one server, then propagated to the other replicas. Any differences will need to be resolved with different techniques (e.g., voting and majority-rule), though these are beyond the scope of this course.
##### Partitioning

Partitioning involves each machine holding a subset of files. This may be done with file names (e.g., file names A to M sit on one machine, and the rest on another), or perhaps different directories can sit on different machines. Point being, there are multiple ways to partition all the state in a filesystem.

Benefit(s) include:
- Availability is improved over a single-server DFS design, since request for different files on different machines can be serviced concurrently.
- More importantly, this design improves **scalability**. 
	- With replication, the maximum size of the file system is limited by the machine's capacity
	- With partitioning, the maximum size of the file system can be augmented by simply adding another machine.
- Finally, single-file writes become less complex, since all writes will go to a single machine. 

Drawback(s) include:
- Fault tolerance is decreased. If a machine fails, a portion of data will be lost. 
- In addition, load-balancing the system is more difficult. 
	- If a particular file is accessed more frequently than any others, then **hotspots** will be created on a singular machine.
##### Combination

Replication and partitioning can be combined as well. The overall workload may be balanced by tweaking partitions, repeats, as well as the degree of repetition.
- For example, a partition might only contain read-only files, with a high degree of repetition to allow for more reader access. A partition containing-write only files might be repeated less to lower overheads with synchronising all replicas.
- A file more frequently accessed might also be given a higher degree of repetition versus one that sees little to no access.

_Quiz_
_Consider server machines that hold 100 files each. Using three such machines, a DFS can be configured using replication or partitioning. Answer the following:_

_How many total files can be stored in the replicated vs the partitioned DFS?_
Replicated: 100, Partitioned: 300

_What percentage of the total files will be lost if one machine fails in the replicated vs. partitioned DFS?_
Replicated: 0%, Partitioned: 33%

## Case Studies

### Network File System (NFS)

NFS involves a client accessing files on a remote server across the network. NFS is a popular distributed filesystem originally developed by Sun researchers. In fact, one of the reasons RPC was developed was to help with the use of NFS. 

#### NFS Design

The NFS architecture is shown in this figure:

![[Pasted image 20231106134615.png]]

1. Clients request and access files via the VFS, using the same types of file descriptors and operations they would use to access files in their local storage. 
2. The VFS determines if a file resides locally or if the request needs to be pushed to the NFS client.
3. The NFS client interacts with the NFS server via RPC.
4. The NFS server receives the request, forms it into a proper filesystem operation, and delivers this to the local (server) VFS.
5. The VFS passes this request to the local (server) file system interface. Incoming requests from the NFS server modules are serviced as any other filesystem requests from applications running on the server machine.

For example, when an `open` request arrives at an NFS server, a file handle will be created and returned to the client. 
- This file handle will contain information about the server machine as well as the file itself.
- This file handle will be maintained by the **NFS client**. When the client tries to access files stored on the remote server, the file handle will be passed with every single request --this is handled internally via RPC.
- If the server machine dies or the file is deleted, using this handle will result in error, as stale data is being accessed.

Of course, a client `write` operation will result in the write data being carried as part of the RPC request from client to server. A `read` operation results in data being carried as part of the RPC response from server to client.

_Quiz_
_A file handle can become "stale". What does this mean?_
- _The file is outdated_
- _The remote server is not responding_ (FALSE, this might be due to network issues.)
- _The file on the remote server has been removed_ (TRUE)
- _The file has been open for too long_ (FALSE, some DFS will impose a timed "lease" on files, but this is not true of NFS)

#### NFS Versions

NFS has been around since the 80s, and has since gone through many revisions. The popular versions today that come as part of the standard Linux distributions are `NFSv3` and `NFSv4`.
- The key difference is that `NFSv3` is stateless while `NFSv4` is stateful.
- Since `NFSv4` is stateful, it is able to easily support operations such as file caching and locking. 
- Even though `NFSv3` is stateless, the protocol typically incorporates additional modules to support file caching and locking as well.
##### Caching

For files that are not accessed concurrently, NFS behaves with session semantics. Again, changes are flushed on `close`, and client cached copies are updated on `open`.

As an additional optimisation, NFS also supports periodic updates. This ensures that there are not any changes to a file _during_ a session.
- When multiple clients are concurrently updating a file, periodic updates therefore break the session semantics. 
- The duration in between updates can be configured, though it defaults to 3 seconds for files and 30 seconds for directories.
	- It is assumed that directories are modified less compared to files, and directory modifications are easier to merge than file modifications.

`NFSv4` further incorporates a **delegation mechanism** where the server delegates a single client all rights to manage a file for a given period of time. This avoids the need for periodic update checks.
##### Locking

With server-side state, `NFSv4` supports locking via a lease-based mechanism. When a client acquires a lock, the server assigns a certain time period to the client, during which the lock is valid.
- It is the client's responsibility to either release the lock within the specified time frame or to explicitly extend the lock.
- Using a lease-based mechanism allows the server to guard against an unresponsive client. In the event a lock expires, the server can assign the lock to some other client.
- Suppose the unresponsive client comes back online. It will notice that its lease has expired, hence it must redo any changes it was trying to make.

`NFSv4` also supports a more sophisticated lock, such as a reader/writer lock known as **share reservation**. This also involves certain mechanisms for upgrading readers into writers or vice versa.

_Quiz_
_Which of the following file sharing semantics are supported by NFS and its cache consistency mechanisms?_
- _UNIX_ (FALSE, NFS is a distributed filesystem)
- _session_ (FALSE, recall that periodic updates may break session semantics)
- _periodic_ (FALSE, NFS is not **purely** periodic)
- _immutable_ (FALSE, NFS allows files to be modified)
- _neither_ (TRUE)

NFS is not purely session-based nor is it periodic. 

### Sprite DFS

Discussion regarding the Sprite distributed filesystem will be based around the paper [*Caching in the Sprite Network File System*](https://s3.amazonaws.com/content.udacity-data.com/courses/ud923/references/ud923-nelson-paper.pdf) by Nelson et. al, 1988.

Sprite was not a production-based file system (as was NFS). Rather, it was a research DFS, though it did see some actual use in UC Berkeley. The Sprite DFS paper used trace data on usage/file access patterns to analyse DFS design requirements and justify decisions.
#### Access Pattern (Workload) Analysis

The authors made the following observations while tracing file access patterns:

- 33% of all file accesses are writes.
	- This hints at caching being important to improve performance -- 66% of all operations can be improved.
	- However, a write-through policy is insufficient, since 33% is still a significant portion of the workload.
	- Session semantics were then considered, to write-back when a file is closed.

- 75% of files are open for less than 0.5 seconds, and 90% of files are open for less than 10 seconds.
	- With generally short time periods between a file open and close, session semantics would see high overheads.

- 20-30% of new data are deleted within 30 seconds, and 50% of new data are deleted within 5 minutes.

- File sharing is rare.
	- This observation, as well as the observation that data might be deleted shortly after creation, led to the decision that write-back on close **is not particularly necessary**.
	- It is not necessary therefore to optimise for concurrent access, but since file sharing still occurs, it must be supported somehow.
#### Analysis to Design

The following design choices were made based on the observations made above.

- Sprite will support caching, and it will use a write-back policy.

- Every 30 seconds, write-back blocks that have not been modified for the past 30 seconds.
	- A recently modified block is left alone, as it likely implies that the client is still "working" on that particular portion of the file. Only write-back once no further modifications are expected.
	- The 30 second interval is pulled from the observation that 20-30% of new data are  deleted within 30 seconds of creation.

- When a client comes along and wishes to open a file that is being written to by another client, the server will contact the writer and **collect all dirty blocks**. 
	- In this system, there is no explicit write-back on close, hence it is possible for a client to hold dirtied blocks within its cache.
	- If concurrent access is to take place, the client must be explicitly informed to write-back all changes.

- Additionally, with no explicit write-back on close, it is possible for a client to open-write-and-close multiple times before any data gets written back to the server.
	- This optimises for the 33% of file accesses that are writes.

- Since all open operations must go to the server, directories cannot be cached on the client. 

- Finally, caching is completely disabled on concurrent writes. In this case, all of the writes will be serialised on the server side. Since concurrent writes do not occur frequently, this cost is acceptable.
#### File Access Operations

In this final section, different types of file accesses will be traced, and the events of Sprite's policies will be observed.

Suppose that there are multiple clients that are accessing a file for reading, and there is one writer client.
- All `open` operations will go through the server.
- All clients will be allowed to cache blocks. The writer will have to additionally maintain timestamps on each modified block in order to enforce the 30 second write-back policy.

When the writer `close`s the file, the contents will be stored in the writer's cache. Naturally, the next `open` will go to the server, which means that the server and client must maintain some version number in order to stay in sync.
- On a per file basis, the client keeps track of:
	- file in cache (yes/no)
	- cached blocks
	- timer for each dirty block
	- version
- On a per file basis, the server keeps track of:
	- all readers of the file
	- all writers of the file
	- version
	- cacheable (yes/no)

Suppose then that a new writer arrives _after_ the first writer has closed the file. The new writer is referred to as a **sequential writer**. This scenario is known as **sequential sharing**.
- The server will have to contact the last writer to gather all dirty blocks. As clients keep track of each dirty block, this operation is relatively straightforward.
- The new writer can then proceed to cache its own copy of the file and continue writing.

Suppose that yet another writer arrives while the previous writer is still modifying the file.
- The server first contacts the last writer to gather all dirty blocks. At this point, it will notice that the writer has not closed the file. Caching will be disabled for all writers.
- The cacheable flag helps the server keep track of whether or not a file can be cached by the clients. 
- Both writers will continue to have access to the file, but all operations must go through the server.
- When a writer closes a file, the server will observe this (all operations must go through the server at this point). The server will then make the file cacheable again. 
- Sprite therefore dynamically disables/enables caching depending on whether a sequential or concurrent write is occurring.





