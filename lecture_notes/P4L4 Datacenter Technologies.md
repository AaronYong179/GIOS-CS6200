## Introduction and Overview

This final lecture will serve as a brief and high-level overview of challenges and technologies facing datacenters. The main objective of this lecture is not to go into overly technical details, rather serves to contextualise mechanisms learned in previous lectures in real world applications.

_Quiz (FYI)_
_How many datacenters are there in the world in 2015?_
~510,000 data centers

_How much space (in sqft) is required to house all of the world's datacenters in 2015?_
285.5 million square feet.
## Internet Services

### Preamble

In context of datacenters, it is first important to establish the term "internet service".
###### #DEFINE Internet Service
> An internet service is any type of service that is accessible via a web interface.

Most commonly, internet services can be decomposed into three components (or tiers):
- Presentation, or static content
- Business logic, or dynamic content,
- Database, or some data store.

Although these are separate components, they are not necessarily separate processes on separate machines.
- There are many available open source and proprietary technologies that can be used to implement these different tiers.
- For example, the Apache HTTP web server can fulfil the presentation and the business logic tier.
- Many **middleware** components also exist to connect and connect these tiers.

In multi-process configurations of internet services, some form of IPC is often used.
- This might include RPC or RMI (for Java), or perhaps even shared memory if the processes are running on the same machine.
- The IPC concept was covered in [[P3L3 Inter-Process Communication]], and RPCs are covered in [[P4L1 Remote Procedure Calls]].
### Architectures

For services that deal with high or variable request rates, it becomes important to choose a configuration involving **multiple processes configured on multiple nodes**.

The service can therefore be **scaled out** by launching the same service on multiple machines (nodes). Such architecture could be typically laid out as follows:

1. A front-end dispatching or load balancing component would route an incoming request to the appropriate machine.

2. All nodes then execute any possible step in request processing for any incoming request.
	- This is also termed "functionally homogenous".

3. Alternatively, specialised nodes might be implemented, where certain nodes execute some specific steps(s) in request processing. Some nodes might even handle different request types altogether.
	- This is also termed "functionally heterogenous".

These are reminiscent of the boss/worker threading pattern discussed back in [[P2L2 Threads and Concurrency]]. The worker threads could be general workers, or it was also possible to implement specialised worker threads.
#### Functionally Homogenous

Having all workers be able to perform any step of any incoming request allows the front end to be relatively simple. All the front end has to do is simply assign incoming requests to the next available node.
- For some sophistication, perhaps the front end can keep track of CPU utilisation for each node as well to inform its decision on task assignment.

This design does not necessarily require all nodes to have all data required to service an incoming request. 
- Instead, such data might be replicated or distributed across all nodes.
- Bottom line, each node will be able to retrieve any information necessary for the execution of the service.

![[Pasted image 20231123190625.png]]

A downside would be that there is little opportunity to benefit from caching. A simple front end, after all, does not have enough information about the locality of each task on a per-node basis.

_Quiz_
_Consider a toy shop where every worker knows how to build any toy (homogenous). If higher order rates start arriving, you can keep the homogenous architecture balanced by:_
- Adding more workers (processes)
- Adding more workbenches (servers)
- Adding more tools, parts, etc. (storage or other resources).

_Quiz_
_Consider a toy shop where every worker knows how to build any toy (homogenous). Higher order rates start arriving, so the manager keeps "scaling out" by adding workers, workbenches, parts, etc. This works until..._
- The manager can no longer manage all of the resources.
- It is no longer possible to fit more stuff and staff in the toy shop.
- There are no more shops to outsource to.
#### Functionally Heterogenous

In this design, data does not have to be uniformly accessible everywhere. Data can still be distributed, though this would only be for some subset of the set of nodes.

![[Pasted image 20231123200905.png]]

This approach benefits from locality and caching, since each node is specialised for a specific task.

However, this results in a more complex front end, since it would need to parse an incoming request and route the request appropriately.

The overall management of the system also becomes more complex.
- If the workload increases, it is no longer as simple as adding another process or machine that can handle the incoming requests. There must be sufficient understanding of the type of workload as well as the necessary steps to take.
- This system also suffers when the workload changes. If a specific type of request floods the system, some nodes will be rendered essentially useless.
- In other words, this system sees a higher occurrence of **hotspots**.

_Quiz_
_Consider a toy shop where every worker knows how to build some specific toy (heterogenous). If higher order rates start arriving, you can keep the heterogenous architecture balanced by:_
- Profiling the toys in demand
- Profiling the resources and expertise required
- Add more of the appropriate type of workers, workbenches, and parts.

## Cloud Computing
### Preamble
#### Animoto; Poster Child of Cloud Computing

In the mid 2000s, Amazon was already servicing large volumes of online sales transactions. 
- The vast majority of such transactions occur during peak holiday periods such as Thanksgiving or Christmas. 
- As such, it was necessary to provision the required hardware resources in order to cope with the holiday workloads.
- However, the resources were left to idle for the rest of the year. 
- In 2006, Amazon decided to allow access to its resources via web-based APIs -- for a fee, of course. In other words, third party workloads were allowed to execute on Amazon hardware.
- This was the birth of Amazon Web Services (AWS) and Amazon's Elastic Compute Cloud (EC2).

The company Animoto appeared roughly around the same time.
- Animoto provided a rather intensive service: converting images into videos.
- They started out by renting "compute instances" on EC2, and would only really require around 50 machines for proper functioning.
- In April of 2008, Animoto became available to Facebook users, and the required EC2 instances shot up from 50 to 3400 in the span of one week.

Notice that it would be near impossible to cater to such a large jump in resource requirements if traditional in-house machines were used. The only reason this was possible was due to the usage of cloud computing.
#### Motivation

In the traditional approach, where hardware machines/resources are bought and configured, it is important to determine the required capacity based on the expected demand.

However, it *is* quite difficult to properly estimate the required capacity. In the figure below, it is possible for the demand to exceed the allocated capacity.
- This leads to dropped requests, and lost opportunities in general.

![[Pasted image 20231124152809.png]]

In the ideal scenario, it would be preferable to scale capacity elastically with demand. 
- This scaling should be instantaneous, and should be able to occur in both directions.
- In other words, the overall cost (for maintaining capacity) should be proportional to demand, and to revenue opportunity.

![[Pasted image 20231124152742.png]]

This form of scaling should occur automatically, and can be accessed from anywhere, anytime.

A potential downside would be that these resources are "owned" by someone else. However, this would potentially be a reasonable compromise, provided that the above promises are upheld. 

The goal of cloud computing is to therefore provide the above "ideal scenario", or as least as closely as possible. The **requirements** of cloud computing can therefore be distilled as follows:
- On-demand, elastic resources and service,
- Fine-grained pricing based on usage, 
- Professionally managed and hosted,
- Available via API-based access
#### Overview

Cloud computing provides a pool of **shared resources**.
- These resources can be **infrastructure** resources; that is, compute, storage, or networking devices.
	- Of course, these are not physical hardware resources, rather some virtual machines along with the ability to store some amount of state.
- Alternatively, these resources may also be higher-level **software/services**.
	- These can correspond to email or database services.

These resources are made available via **APIs for access and configuration**.
- These resources must be accessible and configurable over the internet.
- These APIs might be web-based, library-based (for certain programming languages), or command-line based. 

Cloud computing providers offer different types of **billing and accounting services**.
- Some marketplaces incorporate spot pricing, reservation pricing, or potentially other pricing models.
- Note that billing is often not done by raw usage, as overheads associated with such fine-grained monitoring are rather high. 
	- Instead, billing is usually done based on some discrete step function.
	- For example, compute resources are often billed according to "size": small, medium, and large, for example.

All of the above are **managed by the cloud provider** via some sophisticated software stack.
- Common software stacks include the open source OpenStack and VMWare's vSphere.
#### Fundamental Theories of Approach

Two basic principles provide the fundamental theory behind the cloud computing approach.

1. The Law of Large Numbers.
	- The average resource required will remain fairly constant across a large number of clients.
	- This is the reason a cloud provider can operate with fixed amounts of resources.
	- Clients will have their individual (variable) peaks, which would average out to be roughly constant.

2. Economies of Scale.
	- A cloud provider is able to leverage a large number of customers on a single piece of hardware, which amortises the cost of hardware across those customers.
	- This is of course assuming a threshold number of customers is met.
#### Vision

Interestingly, the vision for cloud computing predates the technology by almost half a century. 

_John McCarthy, MIT Centennial, 1961_
> If computers of the kind I have advocated become the computers of the future, then computing may some day be organized as a public utility, just as the telephone system is a public utility. This computer utility could become the basis of a new and important industry.

Based on this vision, cloud computing should be a fungible utility. 
- In other words, the underlying hardware resources used should not be a concern.
- Quite simply put, the resource should be readily used without much understanding of the underlying implementation (like water or electricity as a utility)
- Virtualisation technology is definitely an enabler in this process.

However, there are challenges to this idealistic vision.
- Some hardware dependencies cannot be masked even with virtualisation.
- Different providers also provide different APIs, thus it becomes tedious to switch between providers -- rewriting code is likely necessary.
- There are also privacy and security concerns associated with placing core business code in the hands of an external provider.

_Quiz_

>Cloud computing is a model for enabling **ubiquitous**, convenient, **on-demand network access** to a **shared pool** of configurable computing resources (e.g., network, servers) that can be **rapidly provisioned** and released with **minimal management** effort or service provider interactions.

_Categorise the bolded phrases based on the cloud computing requirement they best describe. Multiple phrases might describe the same requirement. Leave blank if none._

| Requirement | Phrase(s) |
| --- | --- |
| _Elastic resources_ | "on-demand...", "shared pool...", "rapidly provisioned..." |
| _Fine-grained pricing_ | |
| _Professionally managed_ | "minimal management" |
| _API based_ | "ubiquitous" |

### Cloud Models

The National Institute for Standards and Technology (NIST) adopted the term "cloud computing" in 2011 and defined different types of clouds.
#### Deployment Models

Clouds can be classified by their deployment types.

1. Public clouds
	- Amazon's EC2 cloud is an example of a public cloud.
	- The infrastructure belongs to the provider, but any third party customer/tenant can rent the hardware to run their own tasks.

2. Private clouds
	- Both the infrastructure and the applications that run on top of the infrastructure are owned by the same entity.
	- Amazon also maintains private clouds to run its own workloads.

3. Hybrid clouds
	- A private cloud is used primarily in this case, but it interfaces with a public cloud.
	- The private cloud house the main compute resources for the applications, with failover/spikes being handled on the public cloud resources.
	- In addition, public clouds may be used for auxiliary tasks, such as load testing.

4. Community clouds
	- These are essentially public clouds, with a stricter subset of users. 
	- In this case, the clouds are not used by any arbitrary customer, but by a **certain type of users**.
#### Service Models

Clouds can also be distinguished by the type of service that they provide.

On the extreme end _without_ cloud computing, one would run their applications **on premises** -- everything has to be taken care of themselves.

At the other extreme, cloud computing services can be used to provide a complete application. 
- This is known as **Software as a Service** (Saas), and Gmail is such an example.

Cloud computing might also provide certain APIs for the development of certain types of applications.
- The execution environment, which includes the OS and other necessary tools, are provided. 
- This is known as **Platform as a Service** (Paas), and the Google App Engine is such an example, used to develop Android applications.

At the lowest possible level, clouds can simply provide infrastructure instances.
- These instances consist of CPUs with the accompanying memory, storage, and network infrastructure.
- This is known as **Infrastructure as a Service** (Iaas), and the Amazon EC2 is such an example.
- Note that these clouds do not provide access to physical resources directly, rather only the virtualised resources.
- It is often the case that the physical resources are shared with other tenants, though Amazon does provide high-performance instances that are single tenant.

The figure below illustrates the distinction between the cloud service models succinctly:

![[Pasted image 20231125062710.png]]
### Cloud Service Requirements

The following section lists some requirements that cloud services must fulfil, although there were four such requirements already covered above. I'm not too sure what the distinction is for.

1. Cloud resources must be **fungible**.
	- In other words, these resources must be easily repurposed to support different customers who potentially have different requirements.
	- If cloud service providers have to match the diverse customer demands, then the cloud model will cease to be economic.
	- This also allows potential heterogeneity of underlying resources to be hidden from the customer (e.g., different generations or types of server machines).

2. Cloud resources must allow **elastic and dynamic resource allocation methods**.

3. Cloud resources must be **scalable**. 
	- This includes scalability in terms of resource management.
	- This also includes scalability from the customer's perspective, in terms of on-demand resource allocations.

4. Cloud resources must be able to **deal with failure**.
	- Eventual failures are expected with increasing scale. As such, mechanisms must be put in place to deal with the possibility of failure.

5. Cloud resources must be able to **handle multi-tenancy**, more specifically with regard to **performance and isolation**.
	- Misbehaving tenants should not be able to wreak havoc on other tenants and/or the entire system.

6. Cloud resources must also make **security guarantees**.
	- Tenants should not be able to breach data privacy amongst each other.
	- The providers themselves should also not be able to take advantage of the state running on their resources.

_Quiz_
_A hypothetical cloud has $N=10$ components (CPUs). Each has a failure probability of $p=0.03$. What is the probability that there will be a failure somewhere in the system?_
_What if the system has $N = 100$ components?_

N = 10
> 26%

N = 100
> 95%

### Cloud-Enabling Technologies

1. **Virtualisation** is needed to provide fungible resources that can be dynamically repurposed for different customer needs.

2. Dynamic resource provisioning (**scheduling**) is required as well.
	- Certain platforms such as Mesos, or Hadoop's Yarn serve this role.

3. In order to address the customer's needs for scale, there is a need for big data **processing and storage**.
	- For processing, Hadoop's MapReduce or Spark are popular platforms.
	- For storage, distributed file systems are often used in "append-only" mode, such that deletions or modifications are disallowed.
	- Distributed in-memory caches are used to reduce overheads associated with repeated disk accesses. NoSQL databases can be used to help with access and manipulation of data at scale as well.

4. Cloud computing would need software that can enable and configure defined, isolated slices of resources.

5. Finally, efficient monitoring technologies must be incorporated to regulate cloud resources.
	- This is immediately useful for the provider, but may also be provided to the customer.
	- Flume (open source), CloudWatch (Amazon), and Log Insight (VMWare) are all monitoring technologies.
### Big Data

#### Requirements

Cloud computing essentially empowers anyone to have potentially infinite compute resources, as long as money is not an issue. This is especially helpful in the era of big data.

Cloud platforms that offer big data processing must -- at minimum -- have a **data storage layer** and a **data processing layer**.

Often a **caching layer** is also incorporated, to avoid paying the cost of going to disk. This may involve simple caching in memory, but most likely involves distributed shared memory across multiple nodes.

Big data stacks commonly incorporate some language front-end that allows developers to query their data in a language that they are familiar with (e.g., SQL or Python).

Apart from supporting common language front-ends, big data platforms also incorporate common data analytics libraries, such as machine learning algorithms, or even data visualisation.

Finally, the data being analysed might be continuously generated (in fact, they most commonly are). The overall observation might have to be updated over time as more data is retrieved. As such, these platforms have to support ingesting and staging data that is continuously being **streamed** to the platform.
#### Examples

##### Hadoop

![[Pasted image 20231125103908.png]]
##### Berkeley Data Analytics Stack (BDAS)

![[Pasted image 20231125104018.png]]

