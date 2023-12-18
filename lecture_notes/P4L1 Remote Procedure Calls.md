## Introduction and Overview

### Motivation

Consider the following applications:
1. The first application (GetFile app) involves a client retrieving a file from a server.
2. The second application (ModImage app) involves a client sending an image to the server for modification, before receiving the modified image.

In both applications, the following steps are required:
- Set up client-server interaction by creating and initialising sockets.
- Allocate and populate buffers with file/image data.
- Include some information regarding the protocol ("GETFILE" or image modification parameters)
- Copy protocol data into buffers.

In fact, almost all programs that implement some form of client/server functionality would require very similar steps. It is therefore productive to simply this process -- the need for simplification gave rise to Remote Procedure Calls (RPCs).
### RPC Benefits

RPC is intended to **simplify the development of cross-address space and cross-machine** interactions.

1. RPC offers a higher-level interface for all aspects of data movement and communication.
	- This includes communication establishment, requests, responses, acknowledgements, etc.

2. RPCs therefore captures and automates a lot of the errors that may arise from implementing low-level communication.
	- The programmer does not need to worry about these errors as much, or at least, the programmer no longer has to re-implement error handling for each program.

3. RPCs hide the complexities of cross-machine interactions.
	- The developer will be hidden from the complexities inherent in communicating between different machines. 
	- Machines or the network connecting them might fail, but this will be handled by RPCs.
## RPC Requirements

Firstly, it is important to establish that RPC is intended for client/server IPC interactions **only**. A server might perform some complex task, but all the client needs to do is issue requests to the server.

Since most of the languages at the time of RPC inception were procedural languages, RPCs were expected to follow synchronous procedure call semantics.
- Examples of such procedural languages include C, Basic, Fortran, or Pascal.
- Consider first what happens when a _local_ procedure call is made (i.e. a procedure call in a single address space). The calling thread will block until the procedure completes and returns its result.
- Similarly, for _remote_ procedure calls, the calling process will block until a response is received.

RPCs share the type checking feature with regular procedure calls as well.
- Importantly, RPCs will throw an error if an argument of the wrong type is being used.
- Type checking allows to optimise for the RPC runtime. 
	- When packets are being transmitted between two machines, they are technically just a stream of bytes.
	- Having information regarding the _type_ can be useful when the RPC runtime is trying to interpret these bytes.

Since the client and the server may run on different machines, RPCs should also hide any potential differences from the programmer.
- For example, certain data types might be represented differently. Machines might use different endianness to represent integers, or differ in their representations of floating point numbers or negative numbers.
- RPCs should therefore handle any of the necessary conversions in cross-machine communications.
- One way to deal with this is to have both machines agree upon a single data representation. For instance, both can agree that data transmitted should be in the network format. 
- Once an agreement is set, both machines do not have to worry about their own representation of data -- simply ensure that the necessary conversions are performed when sending or receiving data.

Finally, RPC is intended to be a **higher-level protocol**, rather than a transport-level protocol such as TCP/UDP.
- In other words, RPC should support these lower level transport protocols. It should not matter if the machines use TCP or UDP, for instance. 
- In addition, higher level functionalities such as access control, authentication, and fault tolerance (e.g., retry connection) should be included.
## RPC Structure and Steps

#### Case Study

The typical structure of an RPC system will be illustrated with an example. Suppose a client machine needs to perform addition operations but must rely on a server machine "calculator". 
- The client must therefore send the operation it wishes to perform (addition) as well as the necessary arguments to the server.
- The server will perform this addition on behalf of the client and return the results.

To simply all communication aspects, including socket creation, buffer allocation, and so on, RPCs will be used instead.

![[Pasted image 20231031145237.png]]
##### Client

Consider the event where the client wishes to `add` two numbers, `i` and `j`. 
- It does not have the implementation of `add` (that belongs to the server), but it is still allowed to call something that looks like a regular procedure `add`.
- In a regular program, calling a procedure would result in a jump to the address space that holds the first instruction of the procedure.
- In this case, the procedure call will also result in a jump to an address space -- this time containing a **stub implementation of `add`**.
	- This client `add` stub should create and populate some buffer with all the required information for the server. In this case, the desired operation `add` and the arguments `i` and `j`.
	- This stub code itself is automatically generated as part of the RPC package. The programmer need not concern themselves with this.
- After the buffer is created, the RPC runtime will send a message to the server process. This may be via TCP/IP sockets, or some other transport-level protocol.
- Not shown are some additional information (IP address of server, etc.) that will be used by the client and the RPC runtime to establish a connection.
##### Server

Packets are first received by the server.
- Upon reception, they will be handed over to the server stub. This stub knows how to parse the incoming bytes, and will determine that this is an RPC request for `add` with arguments `i` and `j`.
- The server stub will handle the extraction of data, which includes allocating data structures for containing arguments `i` and `j`, as well as their creation within the address space of the server process.
- Once extraction completes, the stub is ready to make a call in the server process to the local `add` procedure. The result will be stored within the process' address space.
- Finally, the server stub allocates a buffer and stores the addition result and sends it back to the client. The client will receive said packet and store the result in its address space.
- The procedure (client-side) will return and client execution will proceed.
#### Generalised

This section lists the generalised steps involved in an RPC.

0. Register
	- The server first needs to execute a registration step.
	- This lets the "world" know what procedures it supports, what arguments it requires, what location it can be accessed (IP address, port number, etc.)
1. Binding
	- The client finds and binds to the desired server. 
	- For connection-based protocols such as TCP/IP, the connection will be established during this step.
2. Call
	- The client then makes the actual RPC.
	- This involves a call to the stub; control is passed to the stub and the client code blocks.
3. Marshall
	- The client stub creates a data buffer and populates it with the necessary information. This process is known as **marshalling**.
	- The arguments may be located in arbitrary, non-contiguous locations within the client's address space, but a contiguous buffer is required for transmission. 
	- In other words, the arguments are **serialised** into the buffer.
4. Send
	- Once the buffer is available, the RPC runtime will send the message via the transmission protocol that was agreed upon during binding (TCP, UDP, or even shared memory IPC). 
5. Receive
	- Data is received by the RPC runtime on the server, which performs the necessary checks to determine the destination stub for delivery.
	- Additional checks for access control can be performed here as well.
6. Unmarshall
	- The server stub unmarshalls the data, extracting arguments from the received byte stream.
	- Data structures to hold these arguments are also created and populated.
7. Actual call
	- Once the arguments have been allocated and set to appropriate values, the actual procedure call is made.
	- This calls the implementation that is part of the server process.
8. Result
	- The server will compute the result (or throw an error) and return its response to the client, using similar steps in reverse.
## RPC Implementation

### Interface Definition Language

#### Broad Overview

Using RPC allows for the client and server to be developed separately. They might be implemented by completely different developers, or use a completely different language altogether. 

However, since the client and server must communicate eventually, there _must_ be some type of agreement.
- Particularly, the server needs to give explicit details about the **types of procedures supported** as well as the **arguments required** for a particular operation.
	- This information allows a client to decide the server it wishes to bind to.
- Most obviously, standardisation allows for the RPC runtime to incorporate tools for automating the generation of the stub functionality.

To address these needs, RPC systems rely on the use of **interface definition languages** (IDLs). The IDL serves as a protocol for how the client-server agreement will be expressed.
#### IDL Specification

###### #DEFINE Interface Definition Language
> An IDL is used to **describe the interface that the particular server exports**.

At the bare minimum, an IDL will export the name of the procedure, the types of different arguments, as well as the return type. Notice that this is very similar to a function signature. 

An IDL should also include version number as a piece of information.
- If there are multiple servers providing the same procedure, it is beneficial for the client to locate the most up-to-date server.
- Otherwise, in the event of incremental upgrades, a client may find server versions that are compatible with its own version until an upgrade is strictly necessary.

An IDL can either be language-agnostic or language-specific.
- SunRPC (to be covered later) uses an IDL that is called XDR (External Data Representation). This is an example of a language-agnostic IDL, since XDR's specification is distinct from any other programming languages.
- The Java RMI (Java equivalent of RPC) uses Java to specify the interfaces that the RMI server exports.
- Language-specific IDLs benefit programmers that are familiar with the particular language, while language-agnostic IDLs are generally easier to pick up compared to learning an entirely new programming language.
- Do note that the choice of IDL language is not reflective of the actual implementation -- it is simply used for interface specification.
	- The IDL is used to help the RPC system generate stubs or information used in the service discovery process.
### Marshalling

Consider the `add` example from above once more. Suppose that arguments `i` and `j` are passed into `add` from the client side.

Initially, `i` and `j` will reside in some arbitrary location within the client's address space. At the lowest level, these arguments must eventually be passed as part of a contiguous buffer of information to a socket (that connects to the server).
- The serialising of these arguments depends on the format specified, but for this example, suppose that these arguments will be packed into a buffer after the procedure `add`.
- The procedure identifier `add` is of course necessary, so that the server can make sense of what exactly needs to be done.

This buffer is generated by the **marshalling code**.

Generally, the marshalling process encodes the data into an agreed upon format such that is can be correctly interpreted by the server. The encoding specifies the layout of the data upon serialisation into a byte stream.
- For example, strings are terminated by a `'\0'` character. A server would recognise this when it is parsing an incoming byte stream.
### Un-marshalling

Using the procedure descriptor and the data types required as arguments for the procedure, the byte stream is parsed and relevant information extracted. Data structures that correspond to the argument types are created upon un-marshalling as well. 

As an example, the `i` and `j` arguments will be allocated somewhere in the server address space, and they will be initialised with values concordant with the message received by the server.

Again, it bears repeating that the programmer **does not have to worry about implementing marshalling or un-marshalling**. 
- Instead, RPC systems typicaly include a special compiler that takes an IDL specification. This specification describes the procedure prototype and the data types for the arguments required.
- From the IDL specification, this compiler will generate marshalling and un-marshalling routines. 
	- These routines would naturally handle the encoding (or "formatting") of arguments and procedures into a byte stream, as well as how to parse an incoming byte stream.
	- Encoding will also encompass conversion from little-endian to big-endian for integers, for example.
- Once the IDL specified routines are compiled, all the programmer has to do is to simply link it with the source code for the server/client when generating executables.
### Binding and Registry

Binding is used by the **client**, where it determines _which_ server to connect to and _how_ it will connect to said server.
- For example, _which_ server could encompass a decision regarding the server name or the server version number.
- A client decides on _how_ to connect to a server based on options regarding network protocols, or it can be as simple as a connection to an IP address.

In order for a client to make these decisions, there needs to be a form of **registry**, which provides a database of available services to the client.
> This might be analogous to the Yellow Pages, where a client can search for a service name to find the appropriate service (which) as well as the necessary contact details (how).

On one extreme, this registry can be **distributed** online. 
- Servers can then register themselves here, while clients have a well-known contact point; they can simply look to this online registry to find a server that meets their needs.

On the other extreme, the registry can be machine-specific. 
- In this case, the registry can be a dedicated process that runs on a server machine, and knows only about those services that run on that particular machine.
- This would require the clients to know the machine address (for registry lookup in the first place). 
- The registry can then inform clients regarding the specifics (port number, for example) if they wish to connect to a particular server on that machine.

In either cases of registry implementation, some naming protocol must be established.
- It is possible to only search for exact matches -- if a client wishes for the `add` procedure, they must specify `add` exactly.
- Otherwise, a more sophisticated naming scheme can be used, where matches can be inferred -- `summation` or `addition` might link to `add`, for example. This is clearly more involved and will not be covered any further here.
### Pointers in RPC

In regular local procedures, it makes sense to allow pointers to be passed as arguments. The pointer will reference the same address, hence the pointer can be dereferenced with no issue. In RPC however, a pointer that points to some location within the _caller's_ address space would make no sense to the _callee_. 

It is possible to completely bypass this issue by disallowing pointers to be used as arguments in RPC. An alternative solution involves marshalling the **referenced data** (not the pointer) into the transmitted buffer.
- More concretely, the RPC runtime should be able to generate marshalling code that understands pointer arguments.
- This marshalling code should then serialise the pointer; i.e. copy the data that the pointer refers to into the transmitted buffer.
- On the server side, the un-marshalling code should be able to unpack this data into its own data structure. The **pointer** to this data structure should then be passed into the local implementation of the procedure.
### Handling Partial Failures

When a client hangs while waiting on a remote procedure call, it is often difficult to pinpoint the exact issue. The service might be down or the server itself might be down. Perhaps the network or router went down.

Even if the RPC runtime incorporates some timeout and retry mechanisms, there is still no guarantee that the problem will be resolved, or that the runtime will be able to provide some better insight into the problem encountered.

RPC systems typically incorporate a special error notification that _tries_ to capture what went wrong with an RPC request without claiming to provide exact details. This serves as a catch-all for multiple errors, and can even catch partial failures.

_Quiz_
_Assume an RPC call fails and returns a timeout message. Given this timeout message, what is the reason for RPC failure?_
- _Client packet lost_
- _Server packet lost_
- _Network link down_
- _Server machine down_
- _Server process failed_
- _Server process overloaded_

Any of the above might occur. In fact, all of them might occur at the same time as well. 
### Summary

In this section, we discussed the various mechanisms involved in RPC implementation, as well as the different design choices behind them.

Client binding to a server might involve a distributed registry or a machine-specific registry, for example. Pointers as arguments can be completely disallowed, or the underlying data might be serialised by the marshalling routine. Point being, there are multiple ways to think about designing RPCs. 

The next section will look at the choices made in the SunRPC implementation, as well a quick look at the Java RMI for comparison.  
## SunRPC

### Introduction and Design Choices

SunRPC is an RPC package originally developed by Sun in the 80s for their Network File System (NFS) for UNIX systems. It became very popular, and is now widely available on other platforms.

SunRPC makes the following design choices:
- It is assumed that the server machine is known up-front, there there is one registry per machine (instead of a distributed registry). When a client wishes to talk to particular service, it must first talk to the registry on that particular machine.
- There are no assumptions made regarding the programming language used by the client or the server. To maintain neutrality, a language-agnostic IDL (XDR) is used for specifying the interface as well as the encoding performed on data types.
- Pointer usage is allowed and pointed-to-data will be serialised.
- Mechanisms for handling errors are supported.
	- It includes a retry mechanism for re-contacting a server when a connection times out.
	- Meaningful errors are returned as much as possible, to allow the client to distinguish between different failure points.
### Overview

SunRPC allows the client to interact with a server via a procedure call interface, similar to the generic RPCs discussed above.

The server specifies the interface it supports in a `.x` file written in XDR. SunRPC includes a special compiler `rpcgen` that will compile the interface specified in the `.x` file into a language-specific stub for both the client and the server.
> Of course, this interface compilation must occur separately for the client and the server.

Upon starting, the server process registers itself with the **registry daemon** available on its local machine. This per-machine registry will keep track of information that includes the name of the service, version, protocol names supported by the service, and the port number that needs to be contacted when a client requests for the RPC server.
- A client must **explicitly** contact the registry on the server machine to obtain information regarding the desired service.

When binding occurs, a client RPC handle is created. 
- This handle is used whenever the client performs any RPC calls.
- This handle also allows the RPC runtime to track all of the RPC-related state on a per-client basis.

For more information, the following can be consulted:
1. Documentation on TI-RPC (Transport Independent-RPC) is maintained by Oracle. TI-RPC basically means that the communication protocol used by the client and server does not have to specified at compile time.
2. Oracle also provides SunRPC/XDR documentation and code examples.
3. Older online references are of course still relevant.
4. Finally, the Linux man pages for `rpc` can be used.
### XDR Interface

#### XDR Specification Example

The code block below shows the XDR specification for a simple program in which a client sends an integer `x` to the server who then squares and returns the result.

```c
struct square_in {
  int arg1;
}

struct square_out {
  int res1;
}

program SQUARE_PROG { /* rpc service name */
  version SQUARE_VERS {
    square_out SQUARE_PROC(square_in) = 1; /* proc ID 1 */
  } = 1 /* version 1 */
} = 0x31230000 /* service id */
```
##### Datatypes

In the `.x` file, the server specifies all **datatypes** need for the arguments and results of the procedures it supports. In this case, the server supports one procedure `SQUARE_PROC` that has one argument of type `square_in` and returns a result of type `square_out`.

The datatypes `square_in` and `square_out` are therefore defined in the `.x` file. They are both structs with one `int` member -- this `int` is similar to that of C, a 32-bit integer.
##### Procedures

The `.x` file also describes the actual RPC service and all of the procedures it supports.

Firstly, there is the name of the RPC service, `SQUARE_PROG`. This is the name to be used by clients when finding an appropriate service to bind with. 

A single RPC server can support one or more procedures. In this case however, the `SQUARE_PROG` service supports one procedure, `SQUARE_PROC`.
- Each procedure is identified by some number (in this case, `1`). This ID will not be explicitly used by the programmer, rather the RPC runtime will use it to identify the procedure being called.
- Each procedure is also identified by some version. The version number might also apply to an entire collection of procedures. Also, a single server is able to support multiple versions of the same procedure.
##### Service ID

Finally, the `.x` file will specify a service ID. This ID is to be used by the RPC runtime to distinguish the different services available.

Briefly put, the client will use identifiers such as the server name, procedure name, and the version number. However, the RPC runtime will internally refer to the service ID, procedure ID, and the version ID.

Only a specific range of service IDs are allowed for usage. The remaining values either have some predefined services (e.g., for the network file system) or are reserved for future use.
- `0x00000000`-`0x1fffffff` : defined by Sun
- `0x20000000`-`0x3fffffff` : range to use
- `0x40000000`-`0x5fffffff` : transient
- `0x60000000`-`0x7fffffff` : reserved
#### XDR Datatypes

All types defined within the XDR file must be XDR-supported datatypes. Some of the default types supported by XDR are commonly found in other programming languages as well, such as `char`, `byte`, `int`, or `float`.

XDR also supports a `const` datatype which will be compiled into a `#define` statement in C.

The `opaque` datatype corresponds to uninterpreted binary data, similar to the C `byte` type. For instance, if an image is to be transferred, the image could be represented as an array of `opaque` elements.

Regarding arrays, XDR supports two types of arrays, **fixed-length** and **variable-length**. 
- Fixed-length arrays are defined with square braces, such as `int data[80]`.
	- Here, the RPC runtime will allocate the specified amount of memory when arguments of this data type are sent or received.
	- The RPC runtime will also know how many bytes to read out in order to populate a variable of this datatype.
- Variable-length arrays are defined with angular braces, such as `int data<80>`.
	- The number here denotes the **maximum expected length** instead of the number of elements within this array.
	- When compiled, the variable-length array will be translated into a data structure with two fields: (i) and integer `len` that refers to the actual size of the array, and (ii) `val` which is the address of the data being stored. 
	- The sender will have to specify `len` and set `val` to point to the memory location where data is stored.
	- The receiver will know that it must expect four bytes to be read as `len`, following which it will read `len` number of bytes into an appropriately allocated buffer.
	- The only (weird) exception to this involves strings. Strings are stored as normal null-terminated sequences of characters in memory. It is only when a string is encoded for transmission will it be stored as a pair of `len` and the remaining data.

_Quiz_
_An RPC routine uses the following XDR data type: `int data<5>`_.
_Suppose the array is full. How many bytes are needed to represent this 5-element array in a C client on a 32-bit machine?_

$$ N(\text{bytes}) = \text{data structure bytes} + \text{data bytes} $$ 
The data structure will consist of two items, and `int len`, as well as a pointer to the actual data, `int * val`. These two result in 8 bytes (4 B + 4 B).

Of course, the data itself requires 4 bytes per element, resulting in a total of $8 + (4 \times 5) = 28$ bytes.

#### XDR Compilation

As mentioned above, XDR compilation uses a compiler known as `rpcgen`. The `-C` flag specifies generation of C code. Therefore, the following command is to be used:
```
$ rpcgen <interface>.x -C 
```

A number of files will be generated as a result:
1. `<interface>.h` contains all (language-specific) definitions of the datatypes and function prototypes.
2. `<interface>_svc.c` and `<interface>_clnt.c` contain code for the server- and client-side stubs respectively.
3. `<interface>_xdr.c` contains code for marshalling/unmarshalling routines to be used by the client and the server.

We will quickly take a closer look at the source code for client- and server-side stubs.
1. `<interface>_svc.c` (server)
	- The first part of this code will contain the `main` function for the server. This includes code for the registration step and some additional housekeeping operations.
	- The second part (e.g., `square_prog_1`) contains code related to a particular RPC service, which includes request parsing and argument marshalling. 
	- Finally, the autogenerated code will include the prototype for the actual procedure to be invoked (e.g., `square_proc_1_svc`). **This has to be implemented by the developer**.
2. `<interface>_clnt.c` (client)
	- The client stub includes a procedure that is automatically generated. This would actually be a wrapper for the RPC call that the client makes to the server-side procedure.

Other than the implementation of the server-side procedure, the rest of the functions are readily available for usage. This is what makes RPC so appealing -- there is no need to repeatedly handle code regarding communication.
#### XDR Routines

As mentioned above, compilation of the XDR file will automatically generate routines within the `<interface>_xdr.c` file. This includes marshalling and unmarshalling routines, as well as some **clean-up operations**.

These clean-up operations (e.g., `xdr_free`) are typically called within a user-defined procedure, in order to specify state that should be cleaned after the RPC runtime is done servicing a request. 

This user-defined procedure should be named `<name>_freeresult` (e.g., `square_prog_1_freeresult`), and will be called automatically by the RPC runtime after it is done computing the results. 

#### XDR Usage Summary

![[Pasted image 20231101204925.png]]

The figure above summarises the steps involved in using the XDR interface and its eventual compilation.
1. The XDR `.x` file must be first defined and compiled with `rpcgen`.
2. Compilation results in the header files, stubs, and even a skeleton implementation of the server. An `<interface>_xdr` file will also be generated, which contains helpful marshalling/unmarshalling routines.
3. The developer has to provide the actual implementation of the service procedure on the server side. 
4. On the client side, the developer has to simply call the wrapper procedure which would communicate with the server under the hood.
5. Other than those two items, the developer has to include the relevant header files, and link the client and server code with the stubs.
6. The RPC runtime will handle everything else (OS interactions, communication management, etc.).
#### Multithreading

As a brief aside, `rpcgen` generates code that is **not** thread safe by default. To generate thread safe code, the flag `-M` should be used:
```
$ rpcgen <interface>.x -C -M 
```

The `-M` flag does not generate a multithreaded server, it simply ensures that the code produced is thread safe. On Linux, a multithreaded implementation must be put in place manually by the developer. On Solaris, the `-A` flag can be passed to automatically generate a multithreaded server.

_Quiz_
_For the `square.x`  code given below, what is the return type of `square_proc_1` if this `square.x` file is compiled with:_

```c
struct square_in {
  int arg1;
}

struct square_out {
  int res1;
}

program SQUARE_PROG { /* rpc service name */
  version SQUARE_VERS {
    square_out SQUARE_PROC(square_in) = 1; /* proc ID 1 */
  } = 1 /* version 1 */
} = 0x31230000 /* service id */
```

```
$ rpcgen square.x -C 
```
The return type will be `square_out*`

```
$ rpcgen square.x -C -M 
```
The return type will be `enum clnt_stat`
### Server Registry

The code that the server needs in order to register itself with the machine's registry is autogenerated in the call to `rpcgen`. Recall that with SunRPC, the registry process must run on every single machine. This process is called `portmapper`, and must be started on Linux using the following command:

```
$ sudo /sbin/portmap
```

The `portmapper` process must be contacted by the **server to register its service**. The **client** on the other hand, must contact `portmapper` to **find contact information** for a particular service on a machine.

Once the RPC daemon is running, it is possible to explicitly check the current running services with the following command:

```
$ /usr/sbin/rpcinfo -p
```

Note that the full path to `rpcinfo` is often required. This command will return the program ID, version, protocol, socket port number, and the service name for every service running on that particular machine.

The `portmapper` service itself runs with both TCP and UDP on the same port number 111 (two sockets with same port number).
### Client Binding

Binding is initiated by the client using the following operation:
```c
CLIENT * clnt_create(char * host, unsigned long prog, 
					 unsigned long vers, char * proto);
```

For the specific `SQUARE_PROG` service created, the operation will look as follows:
```c
CLIENT * clnt_handle;
clnt_handle = clnt_create(rpc_host_name, SQUARE_PROG, 
						  SQUARE_VERS, "tcp"); 
```

The arguments `SQUARE_PROG` and `SQUARE_VERS` are auto-generated during compilation of the XDR file and will included in the header file as `#define`-d values.
- If a client wishes to use a different version number (for example), re-compilation will have to take place, since these arguments are static variables. However, the client code does not need to be modified at all -- only re-compiling is necessary.

The `clnt_create` function returns a client handle, which will be used for all subsequent RPC calls. 
- This handle can also be used to track the status of the current request, or handle any error messages or authentication-related information.
### Data Encoding

Since a server might support multiple programs, each with different versions and procedures, it is not enough to simply pass procedure arguments to the server.

Rather, RPCs must also contain information about:
- Server procedure ID,
- Version number,
- Request ID (so that repeated retries can be detected).

These are included in the **RPC header**, which must be sent to both the client and the server. 

Of course, **actual data** must also be encoded -- data is encoded into a byte stream dependent on its datatype.
- There might be a 1-to-1 mapping between how data is represented in memory versus how it is represented on the wire, but this may not always be the case.
- More importantly, data has to be encoded in a standard format that can be deserialised by both the client and the server.

Finally, the data packet must be preceded by the **transport header** (e.g., TCP/UDP) to ensure that the destination address is captured, and all the protocol-specific operations will take place.
#### XDR Encoding

XDR specifies both an IDL as well as an encoding. For example, the XDR specifies how strings will be represented (what is the syntax, essentially), as well how a string will be encoded into some binary representation when it is being transmitted.

XDR encoding rules are as follows:
- All data types are encoded in multiples of four bytes. Encoding a single byte argument would result in 3 bytes of padding.
	- This helps with alignment and in moving data to/from memory and the network card.
- Big-endian is used as the transmission standard. 
	- Regardles of the endian-ness of the client or server, data must be converted to big-endian prior to transmission.
- Two's complement representation of integers is used.
- The IEEE format for floating point numbers is followed.

Taking a look at an example, suppose we have defined `string data<10>` in the `.x` file and some string data `"Hello"` needs to be transmitted.
- In C (regardless of client or server), the string `"Hello"` will be represented as 6 bytes, `'H'`, `'e'`, `'l'`, `'l'`, `'o'`, `'\0'`.
- In the transmission buffer, this string will be encoded to take 12 bytes.
	- 4 bytes will be used for the integer length (`len = 5`)
	- 5 bytes will be used for the characters (no null terminator is included, recall)
	- 3 bytes will be used for padding to fit a multiple of four.

_Quiz_
_An RPC routine uses the following XDR data type, `int data<5>`_.
_Assume the array is full. How many bytes are needed to encode this 5-element array to be sent from the client to the server? Do not include the bytes used for headers/protocol._

24 bytes. 4 bytes to be used for `len` and 5 $\times$ 4 bytes for each element of the array. 
## Java RMI

The Java Remote Method Invocations (RMI) is another popular form of RPC. This was also popularised by Sun as a form of client-server communication among address spaces in the Java virtual machine (JVM). The IDL is thus language-specific to Java.

Since Java is an object-oriented language, entities interact via **method invocations** instead of procedure calls (hence the name Remote _Method Invocations_).

The RMI runtime is separated into two components.

![[Pasted image 20231102135313.png]]

1. The **Remote Reference Layer** 
	- This contains all of the common code needed to provide reference semantics. 
	- For example, it can support **unicast**, in which a client interacts with a single server, or **broadcast**, in which a client interacts with multiple servers.
	- In addition, this layer specifies different return semantics, such as _return-first-response_ or _return-if-all-match_. 

2. The **Transport Layer** implements all of the transport protocol-related functionality (e.g., TCP, UDP, shared memory, etc.)