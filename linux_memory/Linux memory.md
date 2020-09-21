# Linux memory

## Q&A

**Q : **What is the memory? 

**A :** In a computer framework, The memory called RAM(Random Access Memory) is an internal memory that exchanges data with CPU. Memory is used to load various programs and data for CPU to run directly. In nature, it is a part of hardware here.

**Q :** What is virtual memory?

**A :** From above ,we can get that we need to load various programs and data to memory for CPU, but as hardware, it is limited. The memory will be run out, and for solving this issue, the virtual memory arises as a memory management technology. It makes each program feels they have enough memory to use, however they perhaps operate the same memory on physical memory, the virtual memory is just a logic technology. 

## Linux virtual memory

Linux divides the virtual memory into several equal-size storage zones what named page. And in order to keep the physical memory and virtual memory consistent, the physical memory divides into several the same size blocks as what named page frame.

**1.** In Linux, the size of the page and page frame regularly is 4K.

**2.** The memory address divides into two parts, one represents the page or page frame number and another is offset. And the virtual address maps to the physical address, Like this:

![Linux memory](https://user-images.githubusercontent.com/49341598/93020070-c5195180-f60d-11ea-9604-58c8506c0b05.png)

**3.** For searching the correct physical address by virtual memory address, the system will maintain a page table in which stores the page number and corresponding page frame number. Like this:

![Linux memory table](https://user-images.githubusercontent.com/49341598/93020110-ff82ee80-f60d-11ea-92c4-83796d5ab9bc.png)

***Note:*** The offset is unchanged.

###  Page request and exchange

**Request :** While CPU tries to access a virtual address, it will check whether entry of virtual address we need exists in the page table. if exist, it will map to the physical address by entry, if not, the system will trigger a request error interrupt and  check the virtual address's availability. If it's available, it will map and flush to a page table, back to interrupt to continue the process. Otherwise, it will stop this operation.

**Exchange :** If there isn't enough physical memory after page request, it will search a physical memory in which doesn't use now or recently and remove it from the page table.

### TLB(Translation Lookaside Buffer)

We find the program always run on several same pages for a long time. So there is a register to store the page table and achieve access quickly. And MMU always searches the page from that register. it calls TLB.

### Multi level page table

We used the page structure to manage the memory, but these will be many pages in Linux. So we use multi-level page table to manage the page. For example, a page is 4k, it will have 2^20 pages to manage 4G memory space. If it is a primary level page table, it will need 2^20 page tables to manage. Suppose the page table is 4 bytes, a process will need 4M memory to maintain the page table. The cost is huge. 

From the Linux 2.6.11, it adopts a level 4 page table.

- PGD (page global directory) 
- PUD (page upper directory)
- PMD (page middle directory)
- PTE (page table enrty)





[Picture]: https://blog.csdn.net/qq_38410730/article/details/81036768



























