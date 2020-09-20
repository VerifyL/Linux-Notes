# Linux memory

## Q&A

**Q : **What is the memory? 

**A :** In computer framework, The memory called RAM(Random Access Memory), is an internal memory that exchange data with CPU. Memory is used to load various programs and data for CPU to run directly. In nature, it is a part of hardware here.

**Q :** What is the virtual memory?

**A :** From above ,we can get that we need load various programs and data to memory for CPU, but as a hardware, it is limited. The memory will be run out, and for solving this issue, the virtual memory arises as memory management technology. It makes each program feels they have enough memory to use, however they perhaps operate the same memory on physical memory, the virtual memory is just a logic technology. 

## Linux virtual memory

Linux divides the virtual memory into several equal size storage zones what named page. And in order to keep the physical memory and virtual memory consistent, the physical memory divides into several the same size blocks what named page frame.

**1.** In Linux, the size of page and page frame regularly is 4K.

**2.** The memory address divides into two parts, one represents the page or page frame number and another is offset. And the virtual address maps to physical address, Like this:

![Linux memory](https://user-images.githubusercontent.com/49341598/93020070-c5195180-f60d-11ea-9604-58c8506c0b05.png)

**3.** For searching the correct physical address by virtual memory address, the system will maintain a page table in which stores the page number and corresponding page frame number. Like this:

![Linux memory table](https://user-images.githubusercontent.com/49341598/93020110-ff82ee80-f60d-11ea-92c4-83796d5ab9bc.png)

***Note:*** The offset unchanged.

###  Page request and exchange

**Request :** While CPU try to access a virtual address, it will check whether entry of virtual address we need exist in page table. if exist, it will map to physical address by entry, if not, the system will trigger a request error interrupt and  check the virtual address's availability. If it's available, it will map and flush to page table, back to interrupt to continue process. Otherwise, it will stop this operation.

**Exchange :** If there isn't enough physical memory after page requesting, it will search a physical memory in which don't use now or recently, and remove it from page table.

### TLB(Translation Lookaside Buffer)

We find the program always run on several same page for a long time. So there is a register to store the page table, and achieve access quickly. And MMU always search the page from that register. it calls TLB.

### Multi level page table

We used the page structure to mange the memory, but these will be many page in Linux. So we use multi level page table to manage the page. For example, a page is 4k, it will have 2^20 pages to manage 4G memory space. If it is primary level page table, it will need 2^20 page tables to mange. Suppose the page table is 4 bytes, a process will need 4M memory to maintain the page table. The cost is huge. 

From the Linux 2.6.11, it adopts level 4 page table.

- PGD (page global directory) 
- PUD (page upper directory)
- PMD (page middle directory)
- PTE (page table enrty)





[Picture]: https://blog.csdn.net/qq_38410730/article/details/81036768



























