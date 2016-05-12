---
title:  Linux Kernel-Protected Mode
categories: Linux
tags: 
date: 2015-03-22
---

## Paging Mechanism ##


#### Why we need virtual memory? ####

As we know,  the smallest unit of storage is _byte_ which is mapped into a physical address. When using single task OS like *DOS*, the memory is divided into two parts  occupied by the OS and a process. But things change in terms of **multitask** OS, processes are likely to conflict when they're loaded into memory by **physical address** directly because of swapping. If they did so, we need to relocate the later one to resolve it.

Worse,  serious mistakes will be made if process have access to physical address directly.

<!--more-->

So **virtual memory** is developed naturally.

#### How does virtual momory work? ####

Take the 32-bit Operation System as example.

Every process have 4GB virtual address space  and **a page table** to map the virtual memory into the physical ones with the help of physical component [MMU](http://en.wikipedia.org/wiki/Memory_management_unit) to avoid conflict between processes if they access memory that doesn't belong to themself.

Apparently, the one-to-one mapping doesn't seem feasible. In that case, the table will take up 2<sup>30</sup> X 4B = 4GB memory.

We need another mapping method,  **Paging Mechanism**.

To reduce the space the page table takes, the OS divide the memory into **pages** with size of 4KB, 8KB or 16KB. In this case, the table is consist of 2<sup>20</sup> pages with the size of 4KB, taking up 4MB successive memory.

If a process is going to access 0x12345A10, what need to be done?  See below.

The mapping principle Fig1.1
![virtual mapping]({{ site.url }}/images/2015-04-30-Linux Kernel-Protected Mode/Mapping.PNG ) 


1. MMU will look up the start address of mapping table of the process, that is <font color="red">0x10000000</font>
2. Since there are 2<sup>20</sup> pages, we need the highest 20 bit of <font color="red">0x12345A10 </font> to index the page
, the physical address of that page is <font color="red">0x54321000</font>.
3. The low 12 bits is used as offset

The final physical address is 0x54321000+A10 = <font color="red">0x12345678</font>.


For the reason that the size of page is 4KB, the low 12 bits of entry in the page table is always 0 which are used as flags to indicate status of corresponding page. 
For example,

The 0th bit:   1 means the page is loaded in physical memory, 0 means otherwise.When a page with 0th bit 0 is accessed, see [Page fault](http://en.wikipedia.org/wiki/Page_fault).

#### Secondary Page ####

The page table we just talked about required successive memory which is a little demanding for some mechaines with small memory like 64MB.


So the secondary page is invented which means the mapping is not single-step. As a result, the page table is divied into lots of parts in memory.


1. The CR3 register in CPU stores the start address of <font color="red">Page Directory</font> of every process. Taking the highest 10bits of virtual address as index1, we find PDE = CR3 + (index1 X 4).
2. The highest 20bits of PDE indicate the start address of a page, while the lowest 12bits are used as flags. Then we have PTE = PDE + index2. The index2 are represented by the middle 10bits of virtual address.
3. Finally, we get the content = PTE + offset.The offset are represented by the lowest 12bits of virtual address.

![Secondary Page]({{  site.url }}/images/2015-04-30-Linux Kernel-Protected Mode/SecondaryPage.png)


Althought secondary page help us solve this tricky problem, but two step mapping slow down the performance. Inside CPU, TLB buffer are designed to store the recently accessed pages. The process first look up the virtual addresss in TLB, then secondary page.


The Paging Mechanism makes sure the process won't mess up with other processes' memory.
But we still need another mechanism to guarantee that nothing goes wrong between process and kernel.

## Priority Mechanism ##
CPU distinguish process by priority ranging from 0 to 3.  The user process is rated ring0, while kernel process is rated ring3. Code in ring0 can access data or code in ring3 freely.While code in ring 3 need to be check automatically if they attempt to access ring0.Besides, the instruction are rated the same way as process.

Paging Mechanism and Priority Mechanism protect important resource from arbitrary access, we call it **Protected Mode**. The 8086 CPU doesn't support these features and we call it **Real Mode**.


