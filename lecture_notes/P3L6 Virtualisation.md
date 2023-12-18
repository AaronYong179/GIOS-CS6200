## Introduction and Overview

### Motivation

Virtualisation is actually an old idea originating in the 60s at IBM, where it was standard for a small number of large mainframe computers to be shared by many users.

In order to concurrently run diverse workloads on the same physical hardware with out **imposing a singular operating system**, virtualisation was proposed. Briefly put, virtualisation allows for concurrent execution of multiple OSes (and their applications) on the same physical machine.

![[Pasted image 20231028134909.png]]

The individual OSes deployed on the same physical platform will be under the impression that each of them own the underlying hardware resources. More concretely perhaps, an OS may think it owns "all of physical memory", thought it may actually only be accessing a smaller portion of the available physical memory within the overall system.
###### #DEFINE Virtual Machine
> A virtual machine refers to the OS, as well its applications and virtual resources. A virtual machine is distinct from a physical machine, clearly.
> VMs are also referred to as guests (guess VMs) or domains (guest domains).

Supporting the co-existence of multiple VMs on a single physical machine requires some underlying functionality in order to allocate and manage real hardware resources. Of course, it is necessary to provide isolation guarantees across the different VMs.
- This functionality is provided by the **virtualisation layer**, also referred to as a **virtual machine monitor** or **hypervisor**.

This lecture will primarily concern itself with key design choices behind virtualisation. 
### Requirements

A formal outline of virtualisation is provided by Popek and Goldberg in their paper [_Formal Requirements for Virtualizable Third Generation Architectures_](https://s3.amazonaws.com/content.udacity-data.com/courses/ud923/references/ud923-popek-goldberg-paper.pdf) (1974). 

The definition provided in that paper describes a virtual machine as an **efficient, isolated duplicate of the real machine**. As mentioned above, virtualisation is enabled through a virtual machine monitor (VMM). As a piece of software, the VMM has three essential characteristics.

1. The VMM must provide an environment that is essentially identical to the original machine.
	- The capacity itself might differ, but the overall setup (e.g., type of CPU, types of devices) must be identical.
	- Therefore, the VMM must provide some **fidelity** of representation -- hardware visible to the VM must match hardware actually available on the physical platform.

2. The VMM must ensure that programs running in a virtual machine show _at most_ a minor decrease in speed.
	- An important disclaimer: it is obvious that a program within a VM would run slower if the VM itself only sees two out of 4 CPUs (for example).
	- However, the VMM's objective is to ensure that the VM would perform as a native application would if given all the host's resources.
	- In sum, the VMM must provide **performance** to the VMs, as close to the native performance as possible.

3. The VMM is in complete control of the system resources. 
	- The VMM must control accesses to system resources, and must be relied upon to provide **safety and isolation** among the VMs.
	- This does not necessarily imply that all hardware accesses must go through the VMM. For example, a particular VM might be allowed direct hardware access.
	- Once policies regarding system resources are put in place, the VMM must ensure that a VM cannot change these policies, potentially hurting other co-located VMs.

_Quiz_
_Based on the classical definition of virtualisation by Popek and Goldberg, which of the following do you think are virtualisation technologies?_
  
VirtualBox is a virtualisation technology. The Java Virtual Machine and the Virtual GameBoy are not virtualisation technologies.

JVM provides virtualisation for Java applications, but it is very different from the underlying physical machine. There is no _fidelity_ of representation.

Emulators such as Virtual GameBoy emulate the hardware platform, but there is no _fidelity_ of representation once again.

With VirtualBox, the physical hardware visible to the virtual machine is identical (or at least very similar) to the physical platform. 
### Benefits

1. Consolidation
	- Consolidation here refers to the ability to run multiple VMs on a single physical platform, leading to an overall cost efficiency.
	- Fewer machines are required, which directly implies lower space or admin requirements, with potentially smaller electric bills.
	- The same workload can be run, although with a **decrease in cost** and an **increase in manageability**.

2. Migration
	- Since the OS and applications are decoupled from the physical system, it becomes easy to migrate the VM from one machine to another, or even potentially clone a VM on multiple machines.
	- Virtualisation therefore leads to mechanisms that provide greater **availability** of services. For example, if an increase in workload is detected, multiple VMs can be created to address the issue.
	- The ability to migrate also provides greater **reliability**. If a physical machine is about to fail, it is relatively easy to move the VM onto another physical platform.

3. Security
	- VMs are commonly used to contain malicious code to isolated containers without bringing down an entire physical platform.

4. Debugging
	- The encapsulation of an OS and its applications also aided in operating systems research. Researchers can quickly test OSes in development without restarting hardware.

5. Support for legacy operating systems.
	- Virtualisation provides affordable support for legacy OSes.
	- Applications that run on older OSes no longer need a dedicated physical platform just for them. They can run as one VM, sharing the physical resource with other VMs.

_Quiz_
_If virtualisation has been around since the 60s, why has it not been used ubiquitously since then?_
- Virtualisation was not efficient (FALSE)
- Everyone used Microsoft Windows (FALSE)
- Mainframes were not ubiquitous (TRUE)
- Other hardware was cheap (TRUE)

The trend of simply buying hardware to support another OS was not too costly, hence continued for multiple decades.

_Quiz_
_If virtualisation was not widely adopted in the past, what changed?_
- Servers were underutilised (TRUE)
- Datacenters were becoming too large (TRUE)
- Companies had to hire more sysadmins (TRUE)
- Companies were paying high utility bills to run and cool servers (TRUE)

A quick note, companies were using 70% of their budget on operating expenses instead of capital expenses (e.g., buying new hardware or software). As such, virtualisation became important to consolidate workloads on fewer hardware resources.
## Virtualisation Models

### Bare Metal (Type 1)

The bare-metal virtualisation model is also referred to as hypervisor-based or type 1 virtualisation. In this model, the VMM (hypervisor) manages all hardware resources and supports the execution of the VMs. 

![[Pasted image 20231028161820.png]]

Devices immediately pose an issue to this model. According to this model, the hypervisor **must manage all possible devices**. In other words, device manufacturers would have to provide device drivers for the different OSes, as well as the different hypervisors.
- To circumvent this problem, the hypervisor-based model typically integrates a special VM, termed a **service (privileged) VM**. 
- This service VM runs a standard OS, and has full hardware access privileges. It will thus run the hardware drivers and control how devices on the platform are used.
- The service VM will also run some other management/configuration tasks that specifies how the hypervisor would share resources among the other VMs.

This model and its usage of a service VM is adapted by the Xen virtualisation solution and to some loose extent, the ESX hypervisor.
- Both the open source version of Xen as well as the version supported by Citrix (Xen server) refer to the VMs as **domains**.
	- The privileged domain called is `dom0` and the guest VMs are referred to as `domU`.
	- Xen is the actual hypervisor, and all drivers are running in `dom0`.
- The ESX hypervisor is used in VMware. 
	- VMware was among the first to penetrate the virtualisation market, hence it owns the largest percentage of server calls. Due to this fact, VMware is position in such a way that it is able to _mandate_ device vendors to provide drivers for its hypervisor, ESX. 
	- This is not too big of a stretch, since VMware mainly deals with servers, where device diversity is rather limited. Contrast this to client-side platforms, such as home PCs or laptops.
	- In the past, VMware used to have a Linux control core (similar to `dom0`). In recent years, the configurations are performed via remote, open source APIs.
### Hosted (Type 2)

In this model, there will be a **full-fledged** OS that manages all hardware resources. The host OS integrates a VMM module, which is responsible for providing the VMs with their virtual platform interface. The VMM module will then invoke device drivers and other host components if needed.

This model benefits from the fact that it can leverage services and mechanisms that are already developed for the host OS. Much less functionality needs to be redeveloped for the VMM module. In fact, due to the underlying host OS, it is possible to run applications within VMs or directly on the host OS.

![[Pasted image 20231028163515.png]]

The kernel-based VM (KVM) runs a Linux host OS, as shown in the figure above. The host provides all aspects of hardware management and can obviously run regular Linux applications directly.

Support for virtualisation is achieved by the KVM module (VMM) and a hardware emulator called **QEMU**. 
- The QEMU emulator (running in virtualiser mode), matches underlying hardware resources -- the resources available to guest VMs are the exact hardware resources available on the physical platform.
- The QEMU virtualiser intervenes during critical operations (e.g., during I/O management) and passes control over to the KVM module and the host OS. Simply put, since device management is handled by the host OS, the QEMY virtualiser must invoke the host OS via the KVM moduke.

KVM benefits greatly from the Linux open source community, where it can adapt quickly to new features and implement bugfixes. The KVM is now integrated into the standard Linux kernel.

_Quiz_
_Do you think that the following virtualisation products are bare-metal/hypervisor-based (HV) or host-OS-based (OS)?_

| HV | OS |
| --- | --- |
| VMware ESX | KVM |
| Citrix Xenserver | Fusion |
| Microsoft Hyper-V | VirtualBox |
| | VMWare Player |

_Quiz_
_Which of the following do you think are virtualisation requirements?_
- Present virtual platform interface to VMs (TRUE)
	- This is true, since virtualisation should present hardware interfaces to the guest VMs.
- Provide isolation across VMs (TRUE)
	- This is also clearly true. In fact, similar mechanisms used by the OS for process isolation can be used here. 
	- For example, a hypervisor might use techniques such as preemption, or leverage the hardware MMU to validate memory accesses. 
- Protect guest OS from apps (TRUE)
	- The kernel/user boundary must also exist within the VM itself.
	- Guest OSes and apps cannot be run at the same protection level.
- Protect VMM from guest OS (TRUE)
	- Similarly, the VMM and guest OSes cannot be run at the same protection level.
## Hardware Virtualisation

### Hardware Support

From the quiz in the previous section, there appears to be a need to have at least three distinct protection levels. Briefly, the underlying VMM and guest OS and VM software applications will need their own protection level. Fortunately, commodity hardware will often have more than 2 protection levels.

Take for example the x86 architecture (a staple in the server space). There are four protection levels, called **rings**. 
- Ring 0 has the highest privilege -- all resources can be accessed and hardware instructions can be executed at this level. In a native model, the operating system will sit at this level.
- In contrast, ring 3 has the lowest privilege. Applications would sit at this level -- any unprivileged operations will result in a trap to kernel at level 0.
- In a virtualisation setting, it is possible to place the **hypervisor at ring 0**, since it will ultimately need to control hardware access. The OS can then sit at ring 1. 

More recent iterations of the x86 architecture introduce two different protection modes, **root** and **non-root**. Within each mode, four protection levels (rings) exist.
- When running in root mode, all operations are permitted. The hypervisor will therefore run in ring 0 of root mode.
- In non-root mode, certain operations are not permitted. Guest VMs will therefore operate in this mode, and the OS will be placed as it would in a native model -- ring 0 (non-root). The applications will be placed in ring 3 (non-root).
- Attempts by the guest OS to perform privileged operations will result in a trap called **VM-Exits**. When the hypervisor takes over control and completes, control will be passed back to the VM by performing a **VM-entry**.
### Processor Virtualisation

#### Trap-and-Emulate

In order to achieve near native speeds within virtual machines, it is important to note that guest instructions are **executed directly by hardware**. In other words, the VMM does not interfere with _every single_ instruction issued by the guest OS or its applications.

This is akin to how the OS does not interfere with every single instruction memory access. As long as the guest OS (and its applications) operate within the resources allocated by the hypervisor, the instructions will operate at hardware speeds, which underscore the efficiency of virtualisation.

However, when a privileged instruction is accessed, the **processor** traps to the hypervisor. At this point, the hypervisor determines if the operation is legal or otherwise.
- If the operation is illegal, the hypervisor might terminate the VM (some punitive action, for example).
- Otherwise, the hypervisor must **emulate** the behaviour the guest OS expects from hardware. Note that the hypervisor might tweak certain operations (or perform slightly different operations), but this should be invisible to the guest OS. 
- This is known as the trap-and-emulate strategy. 
#### Shortcomings

Pre-2005, x86 platforms did not have the root/non-root modes. Therefore, as discussed above, the hypervisor was allowed to run on ring 0, while the guest OS ran on ring 1. However, this configuration, together with the trap-and-emulate strategy, resulted in 17 privileged instructions that would fail silently.

For example, enabling/disabling interrupts required the manipulation of a bit in a privileged register. When the guest OS issues these instructions (`POPF`/`PUSHF`), control was not passed to the hypervisor. The instructions simply failed silently.
- Since the hypervisor never receives control, it is under the assumption that nothing is to be done.
- At the same time, the OS is never informed of this failure hence simply assumes that the instructions succeeded.
- In this specific example, the OS would continue executing instructions that ultimately resulted in a deadlock. 

As such, the trap-and-emulate strategy was not too effective on these architectures.

_Quiz_
_Given the above scenario, where instructions `POPF` or `PUSHF` fail silently, what do you think might occur as a result?_
- Guest VMs could not requests interrupts to be enabled/disabled (TRUE)
- Guest VMs could not query the status of the interrupts enabled/disabled bit (TRUE)
#### Workarounds

##### Full Virtualisation; Binary Translation

In order to solve the problem of 17 privileged instructions failing silently, the VM binary was rewritten to **never issue those 17 instructions** in the first place. This process is called **binary translation**, and was pioneered by Mendel Rosenblum's group at Stanford -- this group went on to commercialise VMware.

Firstly, note that the goal of **full virtualisation** requires that the guest OS is in no way modified. In other words, there should be no need to install special drivers or add certain policies for the guest OS to run properly in a virtualised environment. 

Instruction sequences that are about to be executed will be **captured** from the VM binary. More specifically, this strategy is known as **dynamic binary translation**. 
- The sequences are typically on the order of a **basic block**, such as a loop or a function. Notice that this has to be done dynamically, since the execution sequence might differ based on parameters that are only available during runtime. 
- Therefore, captured code blocks are inspected prior to execution -- checks are performed to ensure that none of the 17 (infamous) instructions are being executed.
- If there are no bad instructions found, then the block is marked as safe and allowed to run at hardware speeds.
- Otherwise, translation to an alternate instruction sequence is performed. This alternate instruction sequence must emulate the desired behaviour, and might even avoid traps into the hypervisor altogether.

Dynamic binary translation comes with overheads. However, a number of mechanisms are implemented (specifically in VMware) to improve the efficiency of the process.
- For example, translated blocks are cached to avoid further translations in the future.
- It is also possible to distinguish between the components of the VM binary to analyse. For example, only the kernel can be analysed.
##### Paravirtualisation; Hypercalls

Another approach gives up on maintaining completely unmodified guests, instead focuses on performance. In contrast to full virtualisation encountered above, this is known as paravirtualisation.

The guest OS is modified; it is made aware that it is running in a virtualised environment on top of a hypervisor instead of native physical hardware. 
- The guest OS will try to avoid making instructions that it knows might fail. Instead, it makes explicit calls to the hypervisor.
- These calls are termed **hypercalls**.
- Hypercalls behave rather similarly to system calls. Any relevant context would be packaged before the desired hypercall is made. The hypercall will then trap to the VMM.

This approach was adapted and popularised by Xen.

_Quiz_
_Which of the following do you think will cause a trap and exit the hypervisor for both binary translation and paravirtualised VMs?_
- _Access a page that is swapped._ (TRUE)
- _Update to a page table entry_. (FALSE)

The first option is true for both. If a page not present in memory is accessed, the MMU will fault, and will trap to the hypervisor regardless of the virtualisation strategy used.

For the second option, it depends on whether or not the OS has write (update) permissions to the specific page table entry in question.
### Memory Virtualisation

In the previous sections, we discussed how to efficiently virtualise the CPU. In this and subsequent sections, we will look at other physical resources instead.
#### Full Virtualisation

A key requirement for full virtualisation is that the guest OS continues to observe a contiguous, linear address space that starts from address `0`. This is what a native OS would see if it owned the physical memory. 

In order to achieve the above requirement, there is a need to distinguish between **virtual**, **physical**, and **machine** addresses.
- Virtual addresses are used by the applications (processes) in the guest VM.
- Physical addresses are used by the kernel of the guest VM.
- Finally, machine addresses correspond to the true physical addresses on the underlying physical platform.

There is thus a clear need to map virtual addresses (VA) to physical addresses (PA) and finally to actual machine addresses (MA). Recall from [[P3L2 Memory Management]] that hardware (MMU and TLB) aids in address translation -- it would be good to leverage this support.

The naive implementation involves maintaining two separate page tables, a guest page table (VA to PA) and a hypervisor page table (PA to MA).
- The second half of translation can leverage hardware support, however the first half exists solely in software.
- This is too expensive since there will be significant overheads associated with every single memory reference.

Alternatively, the hypervisor can maintain a **shadow page table**. 
- In this implementation, the hypervisor directly creates a VA to MA mapping per guest.
- This page table can be used directly by the hardware MMU to quickly translate VAs into MAs.
- The hypervisor here clearly has additional responsibilities to ensure consistency between the guest's VA to PA page table and its shadow page table:
	- The hypervisor will have to invalidate this shadow page table on a context switch.
	- The hypervisor will also have to write-protect the guest's page table. In this case, if a guest establishes some new mapping, it will trap to the hypervisor. The hypervisor can then update its shadow page table.
#### Paravirtualisation

With paravirtualisation, the OS is aware that it is running in a virtualised environment. Therefore, there is no need to impose the strict requirement that the OS uses physical memory addresses starting from `0`.

In this case, the guest OS can simply register its page tables with the hypervisor -- there is no need for maintaining two page tables at all. 
- However, this page table must be write-protected (now used directly by the hardware). There must be barriers in place preventing a guest VM from overriding another guest's memory.
- Due to this write-protection, every memory update must trap to the hypervisor. Since the guest OS can be modified with paravirtualisation, it is possible to make the guest batch table updates into a single hypercall, amortising the cost of VM-exit across multiple updates.
- There are some other optimisations that can be made. A guest OS might be tweaked such that memory updates are more cooperative to other guest VMs, or memory access can be tweaked to be more optimised for a virtual environment.

Do note that most of the overheads described in full and paravirtualisation have been reduced significantly (or removed completely) on newer x86 hardware platforms.
### Device Virtualisation

CPU or memory virtualisation is relatively less complicated as there is a **significant level of standardisation** at the instruction set architecture (ISA) across different platforms.
- From a virtualisation standpoint, support for a specific ISA is required. Any other lower level details are not important, since it's the responsibility of the hardware manufacturer to standardise at the ISA level.
- For example, x86 platforms will all be standardised at the ISA level regardless of lower level differences.

For devices however, there is much greater diversity involved. There is also a **lack of standard specification** of device interfaces or behaviour.
- For example, the semantics (behaviour) of the device when a particular call is  invoked might not be standardised.

To deal with this diversity, virtualisation solutions adopt one of three key models for device virtualisation. Of course, these were models designed prior to the advent of new, virtualisation-friendly devices.
#### Passthrough Model

In the passthrough model, the VMM-level management driver is responsible for configuring **access permissions** for a device. 
- For example, it will allow a guest VM to access a device's memory registers. 
- Once access is granted, the VM directly accesses said device without interacting with the VMM. 
- Hence, this model is also called the **VMM-bypass model**.

![[Pasted image 20231030154250.png]]

However, by granting _exclusive_ access of a particular device to a single guest VM, **sharing devices** across multiple VMs becomes difficult.
- The hypervisor will need to constantly reassign the device's owner, and a device clearly cannot be concurrently accessed at any given point in time.

The hypervisor being completely out of the way would also mean that the guest VM's device driver will directly operate on a physical device. 
- Therefore, there _must_ be a device of the _exact same type_ that the guest OS expects on the physical platform, since the VMM is unable to intervene and tweak device operations accordingly.

Recall that full virtualisation enables quick and easy migration, due to the decoupling of VMs from the underlying physical hardware. In this model, this is certainly not true.
- The passthrough model breaks the decoupling, binding a device to a VM. 
- This reintroduces migration complexity as there might be some device specific/resident state that would need to be properly copied and moved to a destination node.
#### Hypervisor-Direct Model

In the hypervisor-direct model, the hypervisor intercepts every device access request that is performed by the guest VMs.

![[Pasted image 20231030155109.png]]

Since the hypervisor is allowed to intervene, there is no need for the guest VM's requested device and the underlying physical device to match.
- Specifically, the hypervisor can translate the device access request to some generic representation of an I/O operation for the particular family of devices (e.g., network or disk). 
- The hypervisor-resident I/O stack will then be traversed, where the bottom of the stack lies the actual device driver. The device driver will finally be invoked to perform the requested operation.
- This model was originally adopted by the VMware ESX hypervisor.

This model once again maintains the decoupling of VMs from the underlying physical hardware. 
- Migration becomes straightforward.
- Device sharing also becomes easier, since device access must all go through the hypervisor.

However, device access is slowed. Latency is added due to the intermediate emulation steps. 

Having the hypervisor fully in charge of device access also requires the hypervisor itself to manage the complexities inherent in the device driver ecosystem.
- The hypervisor must now support the multiple possible drivers to perform a device operation successfully.
#### Split-Device Driver Model

Device accesses here involves two components, hence the name "split device driver model".
- Specifically, there is a front-end device driver that sits within the guest VM.
- The actual back-end device driver sits within a service VM (or simply within the host OS for type 2 virtualisation).

![[Pasted image 20231030155842.png]]

Although the back-end driver does not necessarily have to be modified (it is the same driver that a native OS would use, anyway), the front-end driver **does need to be modified**.
- The front-end driver would need to take the device operations requested by the guest VM, and package them in a format to be delivered to the back-end component.
- Due to this modification of the front-end driver, this model can only be used in paravirtualised guests. Recall that full virtualisation should see no modifications to the guests in any form.
- This modification is usually in the form of a device driver wrapper. 

This approach eliminates some emulation overhead associated with the hypervisor-direct model.
- The guest VMs can now explicitly tell the hypervisor what it needs, via the front-end device driver.

Of course, since device requests must go through a centralised back-end component, this allows for better management of shared devices.
### Hardware Improvements for Virtualisation

As the benefits of virtualisation was gradually more well understood and accepted, hardware companies responded appropriately, modifying their architectures to be more virtualisation friendly.
- In the x86 realm, these virtualisation-friendly architectures starting appearing around 2005 with AMD Pacifica and Intel Vanderpool Technology (Intel-VT).

With respect to x86 specifically, 
1. The first thing done was to "close holes" regarding the 17-non virtualisable hardware instructions. Now, these will cause the appropriate trap and pass control over to the hypervisor in privileged mode.

2. As briefly mentioned above, hardware also included two protection modes, "root" and "non-root" (or "host" and "guest"), instead of the single protection mode with four rings.

3. Support was added for the hardware processor to interpret the state of virtual processors (VCPUs).
	- This information is captured in a VM control structure (or VM control block).
	- The fact that the hypervisor understands this data structure (it is able to "walk" the data structure) allows it to specify if a system call should trap.

4. Since hardware was already able to detect the presence of different VMs, the next step was to tag memory structures used by the hypervisor with **VM identifiers**.
	- This resulted in support for extended page tables as well as tagged TLBs, both of which incorporated VM identifiers.
	- When context switching between VMs (also called world switching), there is no longer a need to invalidate or flush entries belonging to the previous VM. 
	- Context switches are now much more efficient.

5. Hardware was also extended to add better support for I/O virtualisation, which included modifications to the processor and chipset, as well as device and system interconnect capabilities.
	- For example, multi-queue capabilities were supported. Think of this as multiple logical interfaces that can be each used by a separate VM. 
	- Better interrupt routing was also implemented, such that only the core on which the VM is running will be interrupted, and not any other.

6. Other features were also added for better security (protection between VMs or VM-hypervisor) and management support. 

7. Of course, will so many new features, new instructions were also added.
	- For example, a new instruction was added to allow switching between the new protection modes.

_Quiz_
_With hardware support for virtualisation, guest VMs can run unmodified and can have access to the underlying devices. Do you think that the split-device driver model is still relevant?_

Yes, it is. With the split-device model, all device accesses are consolidated to the service VM, where device sharing can be better enforced.  

## Addendum

### x86 VT Revolution Summarised

![[Pasted image 20231030164856.png]]