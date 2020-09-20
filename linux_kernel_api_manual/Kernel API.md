# Kernel API

> 基本只适用64位系统且至少4.x的内核

## 寻址

### 页表与页

1. 线性地址

> 1. `#include <asm/pgtable.h>`

每个页表目录或页表项都用一页表示

| macro                         | comments                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| `PAGE_SHIFT`                  | 值为12，即页表大小的位数，对应虚拟地址的低12位，刚好表示4K大小 |
| `PAGE_SIZE`                   | 页大小，`1 << PAGE_SHIFT`                                    |
| `PAGE_MASK`                   | 值为 `~(PAGE_SIZE - 1)`，用于屏蔽低12位，得到页大小对齐的值  |
| `PMD_SHIFT`                   | 值为21，单个中间目录页表项对应的虚地址位数，刚好表示2MB大小  |
| `PMD_SIZE`                    | PMD管理的页总字节数，也是虚地址大小，`1 << PMD_SHIFT`        |
| `PMD_MASK`                    | 值为 `~(PMD_SIZE - 1)`，用于屏蔽低21位，得到PMD管理空间对齐的值 |
| `PUD_SHIFT`                   | 值为30，单个上级目录页表项对应的虚地址位数，刚好表示1GB大小  |
| `PUD_SIZE`                    | PUD管理的页总字节数，也是虚地址大小，`1 << PUD_SHIFT`        |
| `PUD_MASK`                    | 值为 `~(PUD_SIZE - 1)`，用于屏蔽低30位，得到PUD管理空间对齐的值 |
| `PGD_SHIFT`                   | 值为39，单个全局目录页表项对应的虚地址位数，刚好表示512GB大小 |
| `PGD_SIZE`                    | PGD管理的页总字节数，也是虚地址大小，`1 << PGD_SHIFT`        |
| `PGD_MASK`                    | 值为 `~(PGD_SIZE - 1)`，用于屏蔽低39位，得到PGD管理空间对齐的值 |
| `PTRS_PER_PTE`                | 值为512，表示页表项中有多少个表项 ———— 页                    |
| `PTRS_PER_PMD`                | 值为512，表示中间页表目录中有多少个表项 ———— 页表项          |
| `PTRS_PER_PUD`                | 值为512，表示上级页表目录中有多少个表项 ———— 中间页表目录    |
| `PTRS_PER_PGD`                | 值为512，表示全局页表目录中有多少个表项 ———— 上级页表目录    |
| `PAGE_OFFSET`                 | 内核虚地址空间的起始                                         |
| `PAGE_ALIGN(unsigned long)`   | 将值对齐页大小                                               |
| `PAGE_ALIGNED(unsigned long)` | 值是否与页大小对齐                                           |

1. 页表处理

> 1. `#include <asm/pgtable.h>`
> 2. `#include <linux/mm.h>`
> 3. 不论是应用层还是内核层，页表页不参与页框回收

内核使用以下类型和宏进行各种表项值的表示和转换

| type       | type to uint       | uint to type     | comments                            |
| ---------- | ------------------ | ---------------- | ----------------------------------- |
| `pte_t`    | `pte_val(type)`    | `__pte(uint)`    | 页表项 与 无符号 之间的转换         |
| `pmd_t`    | `pmd_val(type)`    | `__pmd(uint)`    | 中间页目录表项 与 无符号 之间的转换 |
| `pud_t`    | `pud_val(type)`    | `__pud(uint)`    | 上级页目录表项 与 无符号 之间的转换 |
| `pgd_t`    | `pgd_val(type)`    | `__pgd(uint)`    | 全局页目录表项 与 无符号 之间的转换 |
| `pgprot_t` | `pgprot_val(type)` | `__pgprot(uint)` | 页表项标志位 与 无符号 之间的转换   |

内核使用以下内联函数进行页表标志位的查看和设置。

由于pte和其他表项类型有很多相似的操作，暂时仅列出pte的操作。而且如果在pgd、pud、pmd中出现巨型页的情况，那么请将对应的表项强转为pte_t进行操作。

| func                                        | comments                                                     |
| ------------------------------------------- | ------------------------------------------------------------ |
| `pte_none(pte_t*)`                          | 如果pte的值没有设置则返回真                                  |
| `pte_clear(pte_t*)`                         | 清零pte的值                                                  |
| `ptep_get_and_clear(pte_t*)`                | 返回pte值，并清零pte                                         |
| `pte_write(pte_t)`                          | 读取 `_PAGE_RW` 可写标志，比如代码页是不可写的               |
| `pte_exec(pte_t)`                           | 读取 `_PAGE_NX` 可执行标志，比如数据页是不可执行的，仅pte有此标志 |
| `pte_dirty(pte_t)`                          | 读取 `_PAGE_DIRTY` 脏标志，表示最近对关联的页进行了写操作    |
| `pte_young(pte_t)`                          | 读取 `_PAGE_ACCESSED` 访问标志，表示最近对关联的页进行了读操作 |
| `pte_present(pte_t)`                        | 如果关联的页号有效，即虚地址可以读写，如果`pte_present()`为真，则`pmd_present()`等一定为真 |
| `pte_wrprotect(pte_t)`                      | 清除可写标志，用于标记需要写时复制操作                       |
| `pte_mkwrite(pte_t)`                        | 设置可写标志                                                 |
| `pte_mkclean(pte_t)`                        | 清除脏标志                                                   |
| `pte_mkdirty(pte_t)`                        | 设置脏标志                                                   |
| `pte_mkold(pte_t)`                          | 清除访问标志                                                 |
| `pte_mkyoung(pte_t)`                        | 设置访问标志                                                 |
| `pte_special(pte_t)`                        | 读取 `_PAGE_SPECIAL` 编程者标志                              |
| `pte_modify(pte_t o, pte_t n)`              | 使用`n`的值替换`o`的值                                       |
| `pte_update(pte_t)`                         | 更新pte，在改变写权限和访问标志后调用                        |
| `pte_same(pte_t*, pte_t)`                   | 如果pte指针的值与比较值相等，则返回真                        |
| `set_pte(pte_t*, pte_t)`                    | 使用pte的值设置pte指针，在构造一个局部pte值时使用            |
| `mk_pte(struct page*, pgprot_t)`            | 将页描述符对应的页号和页表权限标志作为参数创建一个pte值      |
| `vm_get_page_prot(unsigned long)`           | 将对虚地址的权限（`VM_XXX`）转换为pgprot_f值                 |
| `void ptep_set_wrprotect(*mm, addr, *ptep)` | 类似 `pte_wrprotect()` ，但进行了关联的 `mm_struct` 和 虚地地址 的更新操作。 |
| `pfn_pte(phy, prot)`                        | 使用 `prot` (`_PAGE_RW` 等) 页权限将 `phy` 构造为一个 `pte` 项。 |

页目录项

| func        | comments                                                     |
| ----------- | ------------------------------------------------------------ |
| `pmd_bad()` | 页表页不可访问，未设置读写、不在内存、被标记为脏页、被用于数据页寻址等。 |

巨型页表

| macro or func      | comments                                                     |
| ------------------ | ------------------------------------------------------------ |
| `pgd_large(pgd_t)` | pgd级别的巨型页，如果 `pte_present((pte_t)pgd)` 有效， 则`pte_pfn((pte_t)pgd)` 页号是 512GB 的巨型页 |
| `pud_large(pud_t)` | pgd级别的巨型页，如果 `pte_present((pte_t)pud)` 有效， 则`pte_pfn((pte_t)pud)` 页号是 1GB 的巨型页 |
| `pmd_large(pmd_t)` | pgd级别的巨型页，如果 `pte_present((pte_t)pmd)` 有效， 则`pte_pfn((pte_t)pmd)` 页号是 2MB 的巨型页 |

内核使用以下宏或函数对页表目录进行遍历。

pte没有这样的一套函数，而pgd、pmd、pud都有，仅列出pgd

| macro or func                  | comments                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| `pgd_index(va)`                | 获取线性地址在pgd中的索引                                    |
| `pgd_offset(mm, va)`           | 获取线性地址在指定内存描述符的页表中的pgd_t项                |
| `pgd_offset_k(va)`             | 获取线性地址在内核页表中的pgd_t项，仅pgd有此宏               |
| `pgd_addr_end(va, va_end)`     | 获取虚地址va所在pgd索引对应的结束虚拟地址，如果va_end小于该虚拟地址，则返回va_end |
| `pgd_bad(pgd_t)`               | 如果pgd的标志或权限不是内核页表特有的，则返回真，需要清除    |
| `pgd_none_or_clear_bad(pgd_t)` | 如果pgd_t的值为0返回真，或无效清零后返回真                   |
| `pgd_addr_end(addr, end)`      | 返回地址区间 `[addr, end)` 紧接着的PGD中所在的下一个边界地址。 |

内核使用以下宏或函数对页表项进行遍历

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `pte_offset_kernel(pmd_t*, unsigned long)`                   | 获取线性地址在pmd中的pte_t项的指针，pmd对应的页必须是直接映射 |
| `lookup_address(unsigned long, unsigned int *)`              | 在内核页表中查找虚地址对应页表项，如果指针值返回: 1. `PG_LEVEL_1G` 返回值是 `pud_t*` 类型，表示地址映射在1G巨型页内 2. `PG_LEVEL_2M` 返回值是 `pmd_t*` 类型，表示地址映射在2M的巨型页内 3. `PG_LEVEL_4K` 返回值是 `pte_t*` 类型，表示地址映射在常规页内 |
| `lookup_address_in_pgd()`                                    | 与 `lookup_address()` 类似，但是可以使用指定的页表目录       |
| `pte_offset_map(pmd_t*, unsigned long)`                      | 将虚地址转换为pmd中的页表项指针，之后要使用 `pte_unmap(pte_t*)`对得到的pte_t*基础映射，X86不做任何事 |
| `pte_offset_map_lock(struct mm_struct*, pmd_t*, unsigned long, spinlock_t*)` | 与 `pte_offset_map()` 类似，但是会锁住所在的pmd页表          |
| `pte_unmap_unlock(pte_t*, spinlock_t*)`                      | 与 `pte_unmap()` 类似，是 `pte_offset_map_lock()` 的反操作   |

内核使用以下函数对页表（目录）项和页描述符进行转换

| macro                   | comments                                              |
| ----------------------- | ----------------------------------------------------- |
| `pgd_page(pgd_t)`       | 获取pgd_t项关联的页描述符 ———— pud使用的页            |
| `pud_page(pud_t)`       | 获取pud_t项关联的页描述符 ———— pmd使用的页            |
| `pmd_page(pmd_t)`       | 获取pmd_t项关联的页描述符 ———— pte使用的页            |
| `pte_page(pte_t)`       | 获取pte_t项关联的页描述符 ———— 寻址虚地址得到的物理页 |
| `pgd_page_vaddr(pgd_t)` | 返回关联页的内核虚地址，pud、pmd都有此函数            |
| `pte_pfn(pte_t)`        | 获取pte_t项关联的页号，pmd、pud都有此函数             |

内核使用以下函数进行页表的分配

| macro                                                        | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `pgtable_t pte_alloc_one(struct mm_struct *, unsigned long)` | 分配关联虚地址的pte的页表页 ———— 存放所有的页表项，它对页进行了pte使用的字段初始化 |
| `void pte_free(struct mm_struct*, pgtable_t)`                | `pte_alloc_one()`的反操作，释放pte页表页                     |
| `pte_t *pte_alloc(struct mm_struct*, pmd_t*, unsigned long)` | 与 `pte_alloc_one()` 类似，如果pte页表页未分配则分配，并将该pte设置到了关联pmd内，`pmd_t*`参数必须是对应的存放pte目录项页的虚拟地址 |
| `pte_alloc_map()`                                            | 是 `pte_alloc()` 和 `pte_offset_map()` 的结合体 ，使用`pte_unmap()`进行解除映射操作 |
| `pte_alloc_map_lock()`                                       | 是 `pte_alloc()` 和 `pte_offset_map_lock()`的结合体，使用`pte_unmap_unlock()`解除映射操作 |
| `pte_alloc_kernel(pmd, address)`                             | 与 `pte_alloc()` 类似，但是操作的内核页表                    |
| `pmd_t *pmd_alloc_one(struct mm_struct*, unsigned long)`     | 分配关联虚地址的pmd的页表页 ———— 存放所有的pte页表页，它对页进行了pmd使用的字段初始化 |
| `void pmd_free(struct mm_struct*, pmd_t*)`                   | `pmd_alloc_one()`的反操作，释放pmd页表页                     |
| `pmd_t *pte_alloc(struct mm_struct*, pud_t*, unsigned long)` | 与 `pmd_alloc_one()` 类似，如果pmd页表页未分配则分配，并将该pmd设置到了关联pud内，`pud_t*`参数必须是对应的存放pmd目录页的虚拟地址 |
| `pud_t *pud_alloc_one(struct mm_struct*, unsigned long)`     | 分配关联虚地址的pud的页表页 ———— 存放所有的pmd页表页         |
| `void pud_free(struct mm_struct*, pud_t*)`                   | `pud_alloc_one()` 的反操作，释放pud页表页                    |
| `pud_t *pud_alloc(struct mm_struct*, pgd_t*, unsigned long)` | 与 `pud_alloc_one()` 类似，如果pud页表页未分配则分配，并将该pud设置到了关联pgd内，`pgd_t*`参数必须是对应的存放pud目录页的虚拟地址 |
| `pgd_t *pgd_alloc(struct mm_struct*)`                        | 分配pgd的页表页 ———— 存放所有的pud页表页，该函数对页描述进行了一些构造 |
| `void pgd_free(struct mm_struct*, pgd_t*)`                   | 释放gpd页表页                                                |
| `spinlock_t *pte_lockptr(struct mm_struct*, pmd_t*)`         | 获取`pmd_t*`指向的pte目录页的页锁                            |
| `spinlock_t *pmd_lock(struct mm_struct *mm, pmd_t *pmd)`     | 锁定pmd页表页，并返回锁的指针（为了`spin_unlock()`）         |
| `pmd_lockptr()`                                              | 与 `pte_lockpte()` 类似，返回pmd页表页中的锁指针             |

1. 页与页描述符

> 1. `#include <asm/page.h>`
> 2. `#include <linux/pfn_t.h>`

内核使用以下函数对内核直接映射虚地址、页描述符与页号进行转换

| marco                             | comments                                                     |
| --------------------------------- | ------------------------------------------------------------ |
| `pfn_to_page(unsigned long)`      | 通过页号得到页描述符                                         |
| `page_to_pfn(struct page*)`       | 通过页描述符得到页号                                         |
| `pfn_t_to_page(pfn_t)`            | 通过页号类型的页号得到页描述符                               |
| `page_to_pfn_t(struct page*)`     | 通过页描述符得到页号类型的页号                               |
| `__va(phys_addr)`                 | 将有效的物理地址转换为内核虚拟地址                           |
| `__pa(virt_addr)`                 | 将有效内核虚拟地址转换为物理地址                             |
| `virt_to_page(kaddr)`             | 将虚拟地址转换为对应的页描述符 ———— 直接映射                 |
| `virt_addr_valid(kaddr)`          | 内核虚拟地址是否是直接映射页对应的地址                       |
| `clear_page(void *va)`            | 清零虚地址对应的物理页中的数据                               |
| `copy_page(void *to, void *from)` | 将`from`对应的物理页的数据拷贝到`to`对应的物理页中           |
| `__pa_symbol(ksymbol)`            | 内核镜像中符号（变量等）的物理地址                           |
| `slow_virt_to_phys(void*)`        | 将内核虚地址指针转换为物理地址，可以处理巨型页映射中的虚地址 |
| `pfn_valid(unsigned long)`        | 判断无符号长整型页号的有效性，返回真假值                     |
| `pfn_t_valid(pfn_t)`              | 判断`pfn_t`类型的页号的有效性，返回真假值                    |

内核使用下列变量表示运用各个层次的页框

| var                  | comments                                   |
| -------------------- | ------------------------------------------ |
| `highstart_pfn`      | 非直接映射的起始页号，x64不用关心          |
| `highend_pfn`        | 非直接映射的结束页号，x64不用关心          |
| `max_pfn`            | 操作系统管理的最大页号                     |
| `max_low_pfn`        | 直接映射的最大页号                         |
| `min_low_pfn`        | 直接映射的且可用的（内核镜像后的）最小页号 |
| `totalram_pages`     | 可用的页框数                               |
| `totalreserve_pages` | 保留的页框数                               |

## TLB

> 1. `#include <asm/tlbflush.h>`

| func                               | comments                                                     |
| ---------------------------------- | ------------------------------------------------------------ |
| `__native_flush_tlb()`             | 刷新当前用户空间的TLB。                                      |
| `__native_flush_tlb_global()`      | 刷新全局TLB。                                                |
| `__native_flush_tlb_single(vaddr)` | 刷新用户空间的某个虚地址的TLB。                              |
| `__flush_tlb_all()`                | 刷新本地或全局TLB。如果CPU支持PGE，等效于 `__native_flush_tlb_global()` 。 |
| `__flush_tlb_one(vaddr)`           | 刷新内核空间的某个虚地址的TLB。                              |
| `local_flush_tlb()`                | 刷新本地TLB，等效于 `__flush_tlb_all()` 。                   |
| `flush_tlb_mm(mm)`                 | 刷新 `mm_struct` 管理的虚地址的TLB。                         |
| `flush_tlb_range(vma, start, end)` | 刷新 `vma` 管理的某段虚地址空间的TLB。                       |

## NODE、CPU与寄存器

> 1. `#include <asm/processor.h>`
> 2. `#include <asm/special_insns.h>`

### 寄存器操作

1. 控制寄存器
   寄存器读写操作大同小异，仅列出cr3的整体读写操作函数

| func                                      | comments                                                     |
| ----------------------------------------- | ------------------------------------------------------------ |
| `unsigned long long rdtsc()`              | 读取当前指令已运行周期数，将前后读取的指令周期数相减得到中间运行代码的指令周期花费 |
| `void load_cr3(pgd_t *)`                  | 将页表写入cr3寄存器，会刷新整个TLB                           |
| `void cpu_relax()`                        | 忙等待循环里的短暂放弃CPU，常利用此函数实现自旋操作          |
| `safe_halt()`                             | 空转CPU，可以被中断打断。                                    |
| `void write_cr3(unsigned long)`           | 通过PVOP回调函数将物理地址（页表的物理地址）写入cr3          |
| `void native_write_cr3(unsigned long)`    | 通过汇编指令直接将物理地址（页表的物理地址）写入cr3          |
| `unsigned long read_cr3()`                | 通过PVOP回调将cr3的中的页表物理地址返回                      |
| `unsigned long native_read_cr3()`         | 通过汇编指令直接将cr3的中的页表物理地址返回                  |
| `void cr4_set_bits(unsigned long mask)`   | 设置cr4的某个控制位                                          |
| `void cr4_clear_bits(unsigned long mask)` | 清除cr4的某个控制位                                          |
| `unsigned long cr4_read_shadow(void)`     | 读取cr4的影子值 ———— 位于内存中的值                          |

1. cr4控制位

| ctl mask        | comments                                                     |
| --------------- | ------------------------------------------------------------ |
| `X86_CR4_PCIDE` | 开起PCID。进程PCID的全称是 `Process-Context Identifiers`，如果没有PCID，那么运行在处理器上的软件每次切换CR3， 都会造成整个处理器的地址翻译缓存信息（包括TLB和paging-structure cache）被刷掉，在开启这个标志后，如果CR3的低12位不为0（标识ID）， 则CR3关联的页表在地址翻译缓存信息不会被刷掉。当创建地址翻译缓存信息时，它会将该条目和当前的PCID挂钩， 当翻译地址需要使用这些条目的时候，它也只会查找那些和当前PCID匹配的条目要求CPU必须支持 `X86_FEATURE_PCID` |
| `X86_CR4_PGE`   | 开启PGE，通过global page的机制标记相应的地址翻译缓存信息。 如果PTE（或者大页的最有一级页表项）中的G bit（即第8个bit）是1的话， 则该TLB条目就被标记成global，当切换CR3时关联的TLB信息不会被刷掉 |
| `X86_CR4_PSE`   | 巨型页支持。                                                 |

### CPU特性

| func or feature                            | comments                                                     |
| ------------------------------------------ | ------------------------------------------------------------ |
| `struct cpuinfo_x86 & cpu_data(int cpuid)` | 通过CPUID返回指定CPU信息的**引用**，对返回值进行取址操作获取指针 |
| `cpu_has(cpuid, feature_bit)`              | 查看指定CPU是否支持指定特性，支持多物理CPU的机器中可以有不同厂商的CPU，自然特性就不同 |
| `this_cpu_has(feature_bit)`                | 与 `cpu_has()` 类似，但是参看当前执行代码路径的CPU的特性     |
| `X86_FEATURE_PCID`                         | 是否支持`X86_CR4_PCIDE`                                      |
| `X86_FEATURE_PGE`                          | 是否支持 `X86_CR4_PGE`                                       |
| `struct cpuinfo_x86.phys_proc_id`          | CPU物理ID，按板卡上的CPU数量编号                             |
| `struct cpuinfo_x86.core_id`               | CPU核心ID，按CPU内的核编号，所以全局来看可以重复，每个核心可以包含多个超线程核 |
| `struct cpuinfo_x86.cpu_index`             | CPU超线程核索引，全局唯一的。能运行代码基本CPU核单位，也就是 `smp_processor_id()` 返回值 |

1. NODE与CPU布局
   1. 下面的布局是非常接近大部分物理机的多物理CPU的ID布局的。
   2. 一般超线程核ID的奇偶编号都是按物理CPU分割开的。
   3. 下面的示例表示有2个物理CPU，每个物理CPU有2个物理核，每个物理核又有2个超线程核。
   4. 根据ID排序的特性，在多个线程同时需要屏蔽中断要特别小心，必须要在每个物理CPU上空出一个超线程核来做中断处理，否则系统将发生很多不可预期的事故。
   5. 假设每个物理CPU一个NODE。

```
 +---------------------------------+ | +---------------------------------+
 |    Physical processor 00        | | |      Physical processor 01      |
 | +-------------+ +-------------+ | | | +-------------+ +-------------+ |
 | |   Core 00   | |   Core 01   | | | | |   Core 00   | |   Core 01   | |
 | | +---+ +---+ | | +---+ +---+ | | | | | +---+ +---+ | | +---+ +---+ | |
 | | |S01| |S05| | | |S03| |S07| | | | | | |S02| |S06| | | |S00| |S04| | |
 | | +---+ +---+ | | +---+ +---+ | | | | | +---+ +---+ | | +---+ +---+ | |
 | +-------------+ +-------------+ | | | +-------------+ +-------------+ |
 +---------------------------------+ | +---------------------------------+

12345678910
```

### PerCPU变量操作

> 1. `#include <asm/percpu.h>`
> 2. 如果对PerCPU变量的操作是非原子的，则需要禁用调度或中断后才能操作

| func                                          | comments                                                     |
| --------------------------------------------- | ------------------------------------------------------------ |
| `DEFINE_PER_CPU(type, name)`                  | 定义静态PerCPU变量（非动态分配空间的），type是类型可以是基础类型、结构体、枚举或**某个类型的数组**， name是变量的名字。比如定义一个文件内的局部int类型的数组perCPU变量： `static DEFINE_PER_CPU(int[3], intarray)` |
| `DECLARE_PER_CPU(type, name)`                 | 如果想在头文件导出定义的PerCPU变量，则使用这个宏导出定义，当然定义时不能是`static`起头 |
| `DEFINE_PER_CPU_ALIGNED(type, name)`          | 定义cacheline对齐的PerCPU变量                                |
| `DECLARE_PER_CPU_ALIGNED(type, name)`         | 导出cacheline对齐的PerCPU变量                                |
| `DEFINE_PER_CPU_PAGE_ALIGNED(type, name)`     | 定义pagsize对齐的PerCPU变量                                  |
| `DECLARE_PER_CPU_PAGE_ALIGNED(type, name)`    | 导出pagsize对齐的PerCPU变量                                  |
| `EXPORT_PER_CPU_SYMBOL(var)`                  | 如果想在其他中模块中使用PerCPU变量，则使用这个宏导出符号     |
| `EXPORT_PER_CPU_SYMBOL_GPL(var)`              | 如果想在其他中模块中使用PerCPU变量，则使用这个宏导出符号，属于GPL的许可 |
| `alloc_percpu(type)`                          | 动态分配PerCPU变量，返回指针                                 |
| `alloc_percpu_gfp(type, gfp)`                 | 以指定的获取页的标志动态分配PerCPU变量                       |
| `free_percpu(var_ptr)`                        | 释放动态分配的PerCPU变量                                     |
| `phys_addr_t per_cpu_ptr_to_phys(void *addr)` | 获取某个CPU上对应的PerCPU变量指针的物理地址，如 `per_cpu_ptr_to_phys(this_cpu_ptr(var))` |
| `per_cpu(var, cpu)`                           | 获取指定CPU上的PerCPU变量的引用，PerCPU变量引用作为参数，使用取址符`&`可以得到指针，一般用于静态定义的PerCPU变量。 |
| `per_cpu_ptr(ptr, cpu)`                       | 获取指定CPU上PerCPU变量的指针，PerCPU变量指针作为参数，一般用于动态分配的PerCPU变量。 |
| `this_cpu_ptr(ptr)`                           | 获取当前CPU上的PerCPU变量的指针，PerCPU变量指针作为参数      |
| `raw_cpu_ptr(ptr)`                            | 与`this_cpu_ptr()`类似，但是不检查变量                       |
| `get_cpu_var(var)`                            | 与`per_cpu()`类似，在获取当前CPU的PerCPU变量引用之前，禁用抢占 |
| `put_cpu_var(var)`                            | `get_cpu_var()` 的反操作，开启抢占                           |
| `get_cpu_ptr(ptr)`                            | 与`per_cpu_ptr()`类似，获取当前CPU的PerCPU变量指针之前，禁用抢占 |
| `put_cpu_ptr(ptr)`                            | `get_cpu_ptr()` 的反操作，开启抢占                           |
| `percpu_modcopy(var, src, size)`              | 将src地址的size大小的数据拷贝到PerCPU变量中（每个CPU对应的数据都被相投的数据覆盖），涉及的一致性可能无法保证 |
| `raw_cpu_read(var)`                           | PerCPU的单指令读取操作，同时还有其他的写(`write`)、加减法(`add`/`sub`)、自增自减(`inc`/`dec`)、与或操作(`and`/`or`)、交换(`xchg`)、比较交换(`cmpxchg`) 。 这个操作是非原子，所以如果是多指令实现的话，还是需要禁用调度或中断。 比如 `raw_cpu_read_4(__preempt_count)` 和 `this_cpu_read(irq_stat.__softirq_pending)`的运用。 |
| `__this_cpu_read(var)`                        | 与`raw_cpu_read()` 类似，但是会检查是否禁用调度和中断。      |
| `this_cpu_read_stable(var)`                   | 返回 `var` 的本地引用，比 `this_cpu_read()` 效率更高。       |

### CPU位掩码

> 1. `#include <linux/cpumask.h>`

cpu的标识通过 `cpumask_t` 位图结构进行标识。

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `unsigned int smp_processor_id(void)`                        | 获取当前CPU的ID（超线程）。                                  |
| `num_online_cpus()`                                          | 在线的 CPU 数量，这是使用全局的CPU位掩码进行计算的。         |
| `num_possible_cpus()`                                        | 可用的 CPU 数量，因为热插拔的特性，这个宏得到的结果可能和 `num_online_cpus()` 不一致。 |
| `num_present_cpus()`                                         |                                                              |
| `num_active_cpus()`                                          |                                                              |
| `cpu_online(cpu)`                                            | 判断 CPU ID是否在线。                                        |
| `cpu_possible(cpu)`                                          | 判断 CPU ID是否有效。                                        |
| `cpu_present(cpu)`                                           |                                                              |
| `cpu_active(cpu)`                                            |                                                              |
| `for_each_cpu(cpu, mask) { ... }`                            | 使用局部变量 `int cpu` 遍历 `cpumask_t mask` 中的设置的位，每一位代表CPU ID。 遍历结束后 `cpu >= nr_cpu_ids`，`nr_cpu_ids` 是系统中最大可能的 CPU ID。 |
| `for_each_cpu_not(cpu, mask) { ... }`                        | 类似 `for_each_cpu()`，但是遍历没有设置的位。                |
| `for_each_cpu_and(cpu, mask, and) { ... }`                   | 类似 `for_each_cpu()`，但是遍历 `mask` 和 `and` 掩码中都设置的位。 |
| `for_each_possible_cpu()`                                    | 类似 `for_each_cpu()` ，但是使用系统变量，变量可能存在的CPU。 |
| `for_each_online_cpu()`                                      | 类似 `for_each_possible_cpu()`，遍历在线的CPU。              |
| `for_each_present_cpu()`                                     |                                                              |
| `void cpumask_set_cpu(unsigned int cpu, struct cpumask *dstp)` | 将 `dstp` 中的 `cpu` 位置位，表示一个CPUID被设置。           |
| `void cpumask_clear_cpu(int cpu, struct cpumask *dstp)`      | 类似 `cpumask_set_cpu()` ，但是是位置零，表示一个CPUID被清除。 |
| `int cpumask_test_cpu(int cpu, const struct cpumask *cpumask)` | 测试 `cpu` 是否在 `cpumask` 中被设置，如果被设置返回真。     |
| `cpumask_test_and_set_cpu()`                                 | 如果位没有设置则设置，返回操作位的旧值。                     |
| `cpumask_test_and_clear_cpu()`                               | 如果位已设置则清除，返回操作位的旧值。                       |
| `void cpumask_setall(struct cpumask *dstp)`                  | 全部置位。                                                   |
| `cpumask_clear()`                                            | 全部清零。                                                   |
| `int cpumask_and(struct cpumask *dstp, struct cpumask *src1p, struct cpumask *src2p)` | 进行 `*dstp = *src1p & *src2p` ，如果 `*dstp == 0` 则返回 0，否则返回 1。 |
| `cpumask_or()`                                               | 进行 `*dstp = *src1p | *src2p`，没有返回值。                 |
| `cpumask_xor()`                                              | 进行 `*dstp = *src1p ^ *src2p`，没有返回值。                 |
| `cpumask_andnot()`                                           | 类似 `cpumask_and()` ，进行 `*dstp = *src1p & ~*src2p`。     |
| `cpumask_complement()`                                       | 进行 `*dstp = ~*srcp`，没有返回值。                          |
| `cpumask_equal()`                                            | 判断是否相等。                                               |
| `cpumask_intersects()`                                       | 进行 `*src1p & *src2p != 0` ，如果有交集则，则返回真。       |
| `cpumask_subset()`                                           | 类似 `cpumask_intersects()`，进行 `*src1p & ~*src2p == 0`，如果属于子集，则返回真。 |
| `cpumask_empty()`                                            | 如果没有置任何位，则返回真。                                 |
| `cpumask_full()`                                             | 全部被置位，则返回真。                                       |
| `cpumask_weight()`                                           | 返回掩码中有多少个位被设置为 1 。                            |
| `void cpumask_shift_right(struct cpumask *dstp, struct cpumask *srcp, int n)` | 进行 `*dstp = *srcp >> n` 操作。                             |
| `cpumask_shift_left()`                                       | 类似 `cpumask_shift_right()` ，进行 `*dstp = *srcp << n` 。  |
| `cpumask_copy()`                                             | 进行 `*dstp = *srcp`。                                       |
| `cpumask_any()`                                              | 返回掩码中的任何一位被设置的位的位数。                       |
| `cpumask_of(cpu)`                                            | 使用 `cpu` 作为位下标，返回一个 `cpumask_t` 常量。           |
| `cpumask_of_node(node)`                                      | 使用 `node` 上的CPU作为位下标，返回一个 `cpumask_t` 常量。   |
| `get_cpu_mask()`                                             | 等效 `cpumask_of()` 。                                       |

### NODE操作

> ```
> #include <linux/topology.h>
> ```

| macro or func                   | comments                                                    |
| ------------------------------- | ----------------------------------------------------------- |
| `nr_cpus_node(node)`            | 获取指定nodeID上的CPU核心个数                               |
| `node_online(node)`             | nodeID 是否在线                                             |
| `cpu_to_node(cpu)`              | 获取CPU所在的NODE号                                         |
| `numa_node_id()`                | 获取当前的NODE号                                            |
| `numa_mem_id()`                 | 获取当前的NODE号，`numa_node_id()` 包裹函数                 |
| `for_each_online_node(node)`    | 遍历在线的NODE，node是一个用于标识NODEID的局部变量。        |
| `for_each_node_with_cpus(node)` | 类似 `for_each_online_node()` 但只检索有对应在线CPU的NODE。 |

## 内核同步

### 内核抢占

| macro or func                  | comments                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| `preempt_count()`              | 返回当前抢占计数                                             |
| `preempt_disable()`            | 禁用抢占，在路径上可嵌套                                     |
| `preempt_enable()`             | 启用抢占，`preempt_disable()`的反操作，必须成对出现。如果在调用之前发生抢占，则主动调度放弃CPU。 |
| `preempt_enable_no_resched()`  | 与 `preempt_enable()` 类似，但是如果已被抢占，不会发生主动调度。 |
| `get_cpu()`                    | 禁用抢占后，返回当前CPUID，与PerCPU配合使用的辅助函数        |
| `put_cpu()`                    | 启用抢占                                                     |
| `preempt_schedule()`           | 如果当前抢占计数为0，并已发生抢占且中断已启用，则主动调度放弃CPU。 |
| `set_preempt_need_resched()`   | 设置抢占计数为强制调度状态，即使抢占被禁用。参见 `resched_curr()` |
| `clear_preempt_need_resched()` | 清除 `set_preempt_need_resched()`设置的状态，实际调度时使用  |
| `test_preempt_need_resched()`  | 测试是否被强制调度                                           |

### 原子操作

> 1. `#include <asm/atomic.h>`

1. 对封装为`atomic_t`32位或`atomic64_t`(`atomic_long_t`)64位类型的原子操作

| func `atomic_t`                                      | func `atomic64_t`                                          | comments                                                     |
| ---------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------ |
| `ATOMIC_INIT(i)`                                     | `ATOMIC64_INIT(i)`                                         | 对原子进行初始化                                             |
| `int atomic_read(const atomic_t*)`                   | `long atomic64_read(const atomic64_t*)`                    | 返回原子类型变量对应的整型值                                 |
| `void atomic_set(atomic_t*, int)`                    | `void atomic64_set(atomic64_t*, long)`                     | 将原子类型的值设置为对应的整型值                             |
| `void atomic_add(int, atomic_t*)`                    | `void atomic64_add(long, atomic64_t*)`                     | 将原子类型加上对应的整型值                                   |
| `void atomic_sub(int, atomic_t*)`                    | `void atomic64_sub(long, atomic64_t*)`                     | 将原子类型减去对应的整型值                                   |
| `void atomic_inc(atomic_t*)`                         | `void atomic64_inc(atomic64_t*v)`                          | 原子自增                                                     |
| `void atomic_dec(atomic_t*)`                         | `void atomic64_dec(atomic64_t*v)`                          | 原子自减                                                     |
| `bool atomic_sub_and_test(int, atomic_t*)`           | `bool atomic64_sub_and_test(long, atomic64_t*)`            | 类似 `atomic_sub()` ，但是如果执行后原子类型的值为0，则返回真 |
| `bool atomic_dec_and_test(atomic_t*)`                | `bool atomic64_dec_and_test(atomic64_t*)`                  | 类似 `atomic_dec()` ，但是如果执行后原子类型的值为0，则返回真 |
| `bool atomic_inc_and_test(atomic_t*)`                | `bool atomic64_inc_and_test(atomic64_t*)`                  | 类似 `atomic_inc()` ，但是如果执行后原子类型的值为0，则返回真 |
| `bool atomic_add_negative(int, atomic_t*)`           | `bool atomic64_add_negative(int, atomic64_t*)`             | 类似 `atomic_add()` ，但是如果执行后原子类型的值为负数，则返回真 |
| `int atomic_add_return(int, atomic_t*)`              | `long atomic64_add_return(long, atomic64_t*)`              | 类似 `atomic_add()` ，但是返回执行后原子类型的值             |
| `atomic_sub_return`                                  | `atomic64_sub_return`                                      | 类似`atomic_add_return()`，但是执行减法操作                  |
| `atomic_inc_return()`                                | `atomic64_inc_return()`                                    | 类似 `atomic_add_return()`，但是执行自增                     |
| `atomic_dec_return()`                                | `atomic64_dec_return()`                                    | 类似 `atomic_sub_return()`，但是执行自减                     |
| `atomic_fetch_add()`                                 | `atomict64_fetch_add()`                                    | 类似 `atomic_add_return()`，但是返回执行前原子类型的值       |
| `atomic_fetch_sub()`                                 | `atomict64_fetch_sub()`                                    | 类似 `atomic_sub_return()`，但是返回执行前原子类型的值       |
| `int atomic_cmpxchg(atomic_t *v, int old, int new)`  | `long atomic64_cmpxchg(atomic64_t *v, long old, long new)` | 比较交换。如果 `*v` 指向的值与 `old` 相同，则将 `new` 值赋值给 `*v`。始终返回 `*v` 原来的值，如果返回 `old` 说明成功发生了期望的比较交换操作。 |
| `int atomic_xchg(atomic_t *v, int new)`              | `long atomic64_xchg(atomic64_t *v, long new)`              | 原子交换。将 `new` 赋值给 `*v` ，并返回 `*v` 原来的值        |
| `int __atomic_add_unless(atomic_t *v, int a, int u)` | /                                                          | 类似 `atomic_fetch_add()`，但是只有 `*v` 不等于 `u` 时才相加。始终返回 `*v` 的原来的值 |
| `atomic_add_unless()`                                | `bool atomic64_add_unless(atomic64_t *v, long a, long u)`  | 类似 `__atomic_add_unless()` ，但是返回真假值来说明是否真实的发生了加操作。真值表示发生了加操作 |
| `atomic_inc_not_zero()`                              | `atomic64_inc_not_zero()`                                  | 对 `atomic64_add_unless(v, 1, 0)`的封装，如果不为0则加1      |
| `atomic_dec_if_positive()`                           | `long atomic64_dec_if_positive(atomic64_t *v)`             | 如果 `*v` 自减后不为负数，则自减。不论是否自减都返回自减后的值 |
| `atomic_inc_unless_negative()`                       | /                                                          | 非负数则自减1                                                |
| `atomic_dec_unless_positive()`                       | /                                                          | 非正数则自减1                                                |
| `void atomic_or(int, atomic_t*)`                     | `void atomic64_or(long, atomic64_t*)`                      | 原子或                                                       |
| `void atomic_and(int, atomic_t*)`                    | `void atomic64_and(long, atomic64_t*)`                     | 原子与                                                       |
| `void atomic_andnot(int, atomic_t*)`                 | `void atomic64_andnot(long, atomic64_t*)`                  | 原子与非 `*v &= ~i`                                          |
| `void atomic_xor(int, atomic_t*)`                    | `void atomic64_xor(long, atomic64_t*)`                     | 原子异或                                                     |
| `int atomic_fetch_or(int, atomic_t*)`                | `long atomic64_fetch_or(long, atomic64_t*)`                | 原子或，返回执行前的值                                       |
| `int atomic_fetch_and(int, atomic_t*)`               | `long atomic64_fetch_and(long, atomic64_t*)`               | 原子与，返回执行前的值                                       |
| `int atomic_fetch_xor(int, atomic_t*)`               | `long atomic64_fetch_xor(long, atomic64_t*)`               | 原子异或，返回执行前的值                                     |

1. 对整型进行原子操作

> 1. `#include <asm/cmpxchg.h>`

| macro                                          | comments                                                     |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `xchg(ptr, v)`                                 | 原子交换，将 `*ptr` 与 `v` 进行交换，始终返回 `*ptr` 交换前的值 |
| `cmpxchg(ptr, old, new)`                       | 比较交换，如果 `*ptr` 的值与 `old` 相等，则将 `*ptr` 设置为 `new`。始终返回 `*ptr` 原来的值，参见 `atomic_cmpxchg()` |
| `xadd(ptr, inc)`                               | 将 `*ptr` 加上 `inc`，并返回 `*ptr` 原来的值                 |
| `cmpxchg_double_local(p1, p2, o1, o2, n1, n2)` | 双重比较交换，如果 `*p1` 的值与 `o1` 相等，且 `*p2` 的值与 `o2` 相等，则将 `*p1` 设置为 `n1`，将 `*p2` 设置为 `n2`。如果条件成立发生了交换，则返回真，否则返回假。**p1、p2必须是相同大小的基础类型，且内存分布上必须连续。** |

1. 位操作

> 1. `#include <asm/bitops.h>`

一些宏

| macro           | commenst         |
| --------------- | ---------------- |
| `BITS_PER_LONG` | 机器字节的位数目 |
| `BIT_64(n)`     | 即 `1ULL << (n)` |

字整数设置或测试位是否设置

> 1. 位操作分为原子和非原子两种

| atomic func                                             | non-atomic func                   | comments                                                     |
| ------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------ |
| `void set_bit(long nr, unsigned long *addr)`            | `__set_bit(nr, addr)`             | 将 `*addr` 中的第 `nr` 位置 1                                |
| `void clear_bit(long nr, unsigned long *addr)`          | `__clear_bit(nr, addr)`           | 将 `*addr` 中的第 `nr` 位置 0                                |
| `bool test_and_set_bit(long nr, unsigned long *addr)`   | `__test_and_set_bit(nr, addr)`    | 将 `*addr` 中的第 `nr` 位置 1， 返回旧值 ———— 如果返回假，说本次操作成功 |
| `bool test_and_clear_bit(long nr, unsigned long *addr)` | `__test_and_clear_bit(nr, addr)`  | 将 `*addr` 中的第 `nr` 位置 0， 返回旧值 ———— 如果返回真，说本次操作成功 |
| `bool test_and_set_bit_lock(nr, *addr)`                 | /                                 | 与 `test_and_set_bit()` 一样，用于实现位锁                   |
| `bool clear_bit_unlock(nr, *addr)`                      | `__clear_bit_unlock(nr, addr)`    | 与 `clear_bit()` 一样，但是执行前加入了内存屏障，用于实现位解锁。是 `test_and_set_bit_lock()` 的反操作 |
| `void change_bit(long nr, unsigned long *addr)`         | `__change_bit(nr, addr)`          | 如果 `*addr` 中的第 `nr` 位是 1，则置位0，否则置 1 ———— 进行异或操作 |
| `bool test_and_change_bit(long r, unsigned long *addr)` | `__test_and_change_bit(nr, addr)` | 如果 `*addr` 中的第 `nr` 位是 1，则置位0，否则置 1。然后返回旧值 |
| `bool test_bit(long nr, unsigned long *addr)`           | /                                 | 判断 `*addr` 中的第 `nr` 是否为 1，是 1 则返回真             |

字整数的位查找

> 1. 都是非原子的，且只能对值进行查找，而非地址引用（没有原子的概念）
> 2. 位下标从 0 开始

| func                                      | comments                                                     |
| ----------------------------------------- | ------------------------------------------------------------ |
| `unsigned long __ffs(unsigned long word)` | 返回 `word` 中第一位被设置成 1 的位下标，**必须保证`word`不为 `0` ，否则结果未定义** |
| `unsigned long ffz(unsigned long word)`   | 返回 `word` 中第一位被设置成 0 的位下标，**必须保证`word`不为 `~0UL` (字的最大值)，否则结果未定义** |
| `unsigned long __fls(unsigned long word)` | 返回 `word` 中最后一位被设置成 1 的位下标，**必须保证`word`不为 `0` ，否则结果未定义** |
| `unsigned long __ffs64(u64 word)`         | 类似 `__ffs()`，用于64位整数                                 |
| `__sw_hweight8(value)`                    | 返回8位整数中被置位的位数                                    |
| `__sw_hweight16(value)`                   | 返回16位整数中被置位的位数                                   |
| `__sw_hweight32(value)`                   | 返回32位整数中被置位的位数                                   |
| `__sw_hweight64(value)`                   | 返回64位整数中被置位的位数                                   |
| `hweight_long(value)`                     | 返回机器字节整数中被置位的位数                               |
| `get_bitmask_order(value)`                | 获取 `value` 值掩码位的 `order` 值                           |
| `get_count_order(value)`                  | 获取 `value` 值的最大 `order` 值，如果 `value` 不是 `2^order` 先向上收缩到此值。 |
| `get_count_order_long(value)`             | 类似 `get_count_order()`，但用于机器字节整数                 |
| `rol64(word, shift)`                      | 将64位整数左边的 `shift` 位与剩余的位交换，然后返回这个值。  |
| `ror64(word, shift)`                      | 将64位整数右边的 `shift` 位与剩余的位交换，然后返回这个值。  |
| `rol32(word, shift)`                      | 参见 `rol64()` 。                                            |
| `ror32(word, shift)`                      | 参见 `ror64()` 。                                            |
| `rol16(word, shift)`                      | 参见 `rol64()` 。                                            |
| `ror16(word, shift)`                      | 参见 `ror64()` 。                                            |
| `rol8(word, shift)`                       | 参见 `rol64()` 。                                            |
| `ror8(word, shift)`                       | 参见 `ror64()` 。                                            |

模拟库函数的整数位查找

> 1. 位下标从 1 开始
> 2. 最后一位为 整数的位大小

| func                         | comments                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| `int ffs(int x)`             | 返回 `x` 中第一位被设置成 1 的位下标，**如果 `x` 的值为 0，则返回 `0`** |
| `int fls(int x)`             | 返回 `x` 中最后一位被设置成 1 的位下标，**如果 `x` 的值为 0，则返回 `0`** |
| `int fls64(__u64 x)`         | 类似 `fls()`，但是对64位整数进行操作                         |
| `int fls_long(unsiged long)` | 类似 `fls()`，用于机器字节                                   |

### 位锁

> 1. `#include <linux/bit_spinlock.h>`
> 2. 在执行周期上位锁没有自旋锁快，但可以使锁的粒度更细
> 3. 位锁也会加锁先禁用、解锁再开启调度

| func                                                  | comments                                                     |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| `void bit_spin_lock(int nr, unsigned long *addr)`     | 自旋锁定 `*addr` 中的第 `nr` 位 （成功置1）                  |
| `int bit_spin_trylock(int nr, unsigned long *addr)`   | 类似 `bit_spin_lock()` ，但仅尝试一次，成功锁定返回真，否则返回假 |
| `void bit_spin_unlock(int nr, unsigned long *addr)`   | 解锁 `*addr` 中的第 `nr` 位 （直接置0）                      |
| `int bit_spin_is_locked(int nr, unsigned long *addr)` | 测试 `*addr` 中的第 `nr` 位是否被锁定（是否置1），锁定返回真，否则返回假 |
| `void __bit_spin_unlock(int nr, unsigned long *addr)` | 解锁 `*addr` 中的第 `nr` 位 （直接置0），**非原子的**，用于锁嵌套（被另一个锁保护） |

### 内存屏障

> 1. `#include <asm/barrier.h>`

| macro       | comments                                                     |
| ----------- | ------------------------------------------------------------ |
| `nop()`     | 空指令                                                       |
| `mb()`      | 通用内存屏障                                                 |
| `rmb()`     | 读内存屏障，保证执行前，寄存器已从内存加载了数据             |
| `wmb()`     | 写内存屏障，保证执行前，寄存器已将数据写入了内存             |
| `smp_mb()`  | 多CPU上通用内存屏障，保证在所有CPU上，屏障上下的操作都是顺序的 |
| `smp_rmb()` | 多CPU上读内存屏障                                            |
| `smp_wmb()` | 多CPU上写内存屏障                                            |

### 自旋锁

> 1. `#include <linux/spinlock.h>`
> 2. `#include <linux/rwlock.h>`
> 3. 加锁与解锁必须正确的配对使用
> 4. 读写自旋锁不能相互转换
> 5. 2.6的内核在忙等待的过程中可以被抢占，4.x的则一直禁用抢占。
> 6. 读写自旋锁的中断和抢占事项请参考自旋锁。
> 7. 自旋锁以及衍生都不能在临界区休眠

1. 初始化

| macro                      | comments                                 |
| -------------------------- | ---------------------------------------- |
| `DEFINE_SPINLOCK(varname)` | 定义静态的自旋锁，初始化为未加锁状态     |
| `spin_lock_init(lockptr)`  | 初始化自旋锁指针                         |
| `DEFINE_RWLOCK(varname)`   | 定义静态的读写自旋锁，初始化为未加锁状态 |
| `rwlock_init(lockptr)`     | 初始化读写自旋锁指针                     |

1. 加锁解锁

| func lock                                       | func unlock                                          | comments                                                     |
| ----------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| `void spin_lock(spinlock_t *)`                  | `void spin_unlock(spinlock_t*)`                      | 禁用抢占后，忙等待加锁； 解锁后，启用抢占。 对同一个锁而言，此种加锁不能同时出现在（软）中断上下文中和普通上下文中， 否则出现死锁 —————— 资源仅能被一种上下文同时使用。 比如tasklet或timer同时访问资源，他们都是软中断上下文。 |
| `spin_lock_bh(spinlock_t *)`                    | `spin_unlock_bh(spinlock_t *)`                       | 禁用软中断后，忙等待加锁； 解锁后，启用软中断。普通路径和软中断上下文中。 |
| `spin_lock_irq(spinlock_t *)`                   | `spin_unlock_irq(spinlock_t *)`                      | 禁用本地中断后，忙等待加锁；解锁后，启用本地中断 （一定会开启中断，如果当前已是中断上下文，就有可能发生中断嵌套）。普通路径和非嵌套中断上下文中。 |
| `spin_lock_irqsave(spinlock_t*, unsigned long)` | `spin_unlock_irqrestore(spinlock_t*, unsigned long)` | 禁用并保存中断标志到局部变量中，忙等待加锁；解锁后，将中断标志恢复到加锁前（不是开启中断）。可以使用在任何上下文中。 |
| `void write_lock(rwlock_t*)`                    | `void write_unlock(rwlock_t*)`                       | 为写获取或释放读写锁，多个写操作之间或与读操作时间都是互斥的，同一时刻仅有一个路径能请求成功。 对抢占和中断的影响，请参考 `spin_lock()`。 |
| `void read_lock(rwlock_t*)`                     | `void read_unlock(rwlock_t*)`                        | 为读获取或释放读写锁，多个读操作之间共享的，同一时刻可以有多个路径能请求成功，但与写操作是互斥的，必须等待写锁释放。 |
| `write_lock_irq(rwlock_t*)`                     | `write_unlock_irq(rwlock_t*)`                        | 参考 `spin_lock_irq()`                                       |
| `read_lock_irq(rwlock_t*)`                      | `read_unlock_irq(rwlock_t*)`                         | 参考 `spin_lock_irq()`                                       |
| `write_lock_hb(rwlock_t*)`                      | `write_unlock_hb(rwlock_t*)`                         | 参考 `spin_lock_irq()`                                       |
| `write_lock_irqsave(rwlock_t*, unsigned long)`  | `write_lock_irqrestore(rwlock_t*, unsigned long)`    | 参考 `spin_lock_irqsave()`                                   |
| `read_lock_irqsave(rwlock_t*, unsigned long)`   | `read_lock_irqrestore(rwlock_t*, unsigned long)`     | 参考 `spin_lock_irqsave()`                                   |

1. 锁的补充操作

| func                                                    | comments                                                     |
| ------------------------------------------------------- | ------------------------------------------------------------ |
| `int spin_trylock(spinlock_t *)`                        | `spin_lock()`的补充，试图加锁，加锁失败时不会禁用抢占。加锁成功返回真 |
| `int spin_trylock_bh(spinlock_t *)`                     | `spin_lock_bh()`的补充，试图加锁，加锁失败不会禁用软中断。加锁成功返回真 |
| `int spin_trylock_irq(spinlock_t *)`                    | `spin_lock_irq()`的补充，试图加锁，加锁失败不会禁用中断。加锁成功返回真 |
| `int spin_trylock_irqsave(spinlock_t *, unsigned long)` | `spin_lock_irqsave()`的补充，试图加锁，加锁失败不会禁用中断。加锁成功返回真 |
| `void spin_unlock_wait(spinlock_t*)`                    | 自旋等待，直到自旋锁处于未锁状态。                           |
| `int spin_is_locked(spinlock_t *)`                      | 如果自旋锁处于已锁状态，则返回真。                           |
| `int spin_is_contended(spinlock_t *)`                   | 如果有其他进程正在争抢加锁，则返回真。                       |
| `int spin_can_lock(spinlock_t *)`                       | 如果可以加锁，则返回真。                                     |
| `read_trylock(rwlock_t*)`                               | `read_lock()`的补充，试图为读获取锁。参考 `spin_trylock()`。 |
| `write_trylock(rwlock_t*)`                              | `write_lock()`的补充，试图为读获取锁。参考 `spin_trylock()`。 |
| `read_can_lock(rwlock_t*)`                              | `read_lock()`的补充，是否能为读获取锁，仅测试不操作抢占和中断使能，可以加锁则返回真。 |
| `write_can_lock(rwlock_t*)`                             | `write_lock()`的补充，是否能为写获取锁，仅测试不操作抢占和中断使能，可以加锁则返回真。 |
| `spin_lock_irqsave_nested()`                            | 与 `spin_lock_irqsave()` 一样，但是添加了一些调试信息输出    |
| `int cond_resched_lock(spinlock_t * lock)`              | 如果当前进程需要被调度，或其他进程想持有该锁，则解锁后 `schedule()` / `cpu_relax()`，再次运行时再加锁。防止饿死其他在该CPU上运行的进程。 |

### 位图操作

> 1. `#include <linux/bitmap.h>`
> 2. 位图就是大于或等于机器字大小的数组构成，对每一个元素都进行位操作

1. 定义位图

| macro                             | comments                                          |
| --------------------------------- | ------------------------------------------------- |
| `DECLARE_BITMAP(varname, bitnum)` | 定义变量 `varname` 的位图，一共包含 `bitnum` 个位 |
| `BITS_TO_LONGS(bitnum)`           | 将 `bitnum` 位数量转换为 `long` 整型的数量        |

1. 操作位图

| func or macro                                              | comments                                                     |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| `void bitmap_zero(unsigned long *dst, unsigned int nbits)` | 将 `nbits` 个位的位图 `*dst` 置零                            |
| `bitmap_fill(*dst, nbits)`                                 | 将 `nbits` 个位的位图 `*dst` 置一                            |
| `bitmap_copy(*dst, *src, nbits)`                           | 将 `*src` 位图中的低 `nbits` 位拷贝到 `*dst` 位图中          |
| `bitmap_and(*dst, *src1, *src2, nbits)`                    | 将 `*src1` 位图和 `*src2` 位图的低 `nbits` 位做与计算， 将各个位与的结果存入 `*dst` 的低 `nbits` 位。简单表示为 `*dst = *src1 & *src2`。 **如果结果位不都为0，则返回真** |
| `bitmap_or(*dst, *src1, *src2, nbits)`                     | 类似 `bitmap_and()`，`*dst = *src1 | *src2`，但没有返回值    |
| `bitmap_xor(*dst, *src1, *src2, nbits)`                    | 类似 `bitmap_and()`，`*dst = *src1 ^ *src2`，但没有返回值    |
| `bitmap_andnot(*dst, *src1, *src2, nbits)`                 | 类似 `bitmap_and()`，`*dst = *src1 & ~(*src2)`，**如果结果位不都为0，则返回真** |
| `bitmap_complement(*dst, *src, nbits)`                     | 将 `*src` 位图的低 `nbits` 取反存入 `*dst`，`*dst = ~(*src)`，但没有返回值 |
| `bitmap_equal(*src1, *src2, nbits)`                        | 将 `*src1` 位图的低 `nbits` 与 `*src2` 对应位做比较，`*src1 = *src2`，相等则返回真 |
| `bitmap_intersects(*src1, *src2, nbits)`                   | `*src1` 位图的低 `nbits` 与 `*src2` 对应位有交集，则返回真。在 `nbits` 中8位的整数倍部分，按8位比较，小于8位的部分从起始位算起。 |
| `bitmap_subset(*src1, *src2, nbits)`                       | `*src1` 位图的低 `nbits` 与 `*src2` 对应位有一方是另一方的子集 |
| `bitmap_empty(*src, nbits)`                                | 如果 `*src` 位图的低 `nbits` 位没有任何一位置1，则返回真     |
| `bitmap_full(*src, nbits)`                                 | 如果 `*src` 位图的低 `nbits` 位没有任何一位置0，则返回真     |
| `bitmap_weight(*src, nbits)`                               | 返回 `*src` 位图的低 `nbits`位中被置1的位个数                |
| `bitmap_shift_right(*dst, *src, shift, nbits)`             | 将 `*src` 位图的低 `nbits` 位右移 `shift` 位存入 `*dst`      |
| `bitmap_shift_left(*dst, *src, shift, nbits)`              | 将 `*src` 位图的低 `nbits` 位左移 `shift` 位存入 `*dst`      |
| `bitmap_parse(*buff, len, *src, nbits)`                    | 将长度为 `len` 的 `*buff` 中的16进制字符编码为对应的位图， 存入 `*src` 中的低 `nbits` 位，忽略头部的空白字符，且与`,` 和 `\0`就停止解析 |
| `bitmap_release_region(*src, pos, order)`                  | 从 `*src` 的 `pos` 位下标开始，将 `2^order` 个位置0。        |
| `bitmap_allocate_region(*src, pos, order)`                 | 从 `*src` 的 `pos` 位下标开始，将 `2^order` 个位置1，如果这个范围内有任何为1，则返回 `-EBUSY`，否则成功返回0 |

1. 遍历和查找

| func or macro                                | comments                                                     |
| -------------------------------------------- | ------------------------------------------------------------ |
| `find_first_bit(*addr, nbits)`               | 返回 `*addr` 中低 `nbits` 中第一个被置1的位的下标，如果返回 `nbits` 则表示没有找到 |
| `find_first_zero_bit(*addr, nbits)`          | 与 `find_first_bit()` 类似，返回第一个被置0的位的下标        |
| `find_last_bit(*addr, nbits)`                | 与 `find_first_bit()` 类似，返回最后一个被置0的位的下标      |
| `find_next_bit(*addr, nbits, offset)`        | 在 `*addr` 中低 `nbits` 位的 `offset` 起查找， 返回下一个被置1的位的下标，如果返回 `nbits` 则表示没有找到 |
| `find_next_zero_bit(*addr, nbits, offset)`   | 与 `find_next_bit()` 类似，返回下一个被置0的位的下标         |
| `for_each_set_bit(bit, *addr, nbits)`        | 使用 `bit` 作为迭代位下标，遍历 `*addr` 中低 `nbits` 的所有置1的位 |
| `for_each_set_bit_from(bit, *addr, nbits)`   | 使用 `bit` 作为起始迭代位下标，遍历 `*addr` 中低 `nbits` 的所有置1的位 |
| `for_each_clear_bit(bit, *addr, nbits)`      | 使用 `bit` 作为迭代位下标，遍历 `*addr` 中低 `nbits` 的所有置0的位 |
| `for_each_clear_bit_from(bit, *addr, nbits)` | 使用 `bit` 作为起始迭代位下标，遍历 `*addr` 中低 `nbits` 的所有置0的位 |

### 顺序锁

> 1. `#include <linux/seqlock.h>`
> 2. 写锁直接互斥，但是比读锁有更高的优先权，不必等待读锁
> 3. 写者不能修改读者间接引用的数据。
> 4. 读者操作的临界区应该具有一致性，并且应该相当简短。

1. 初始化

| macro                      | comments           |
| -------------------------- | ------------------ |
| `DEFINE_SEQLOCK(varname)`  | 定义静态的顺序锁   |
| `seqlock_init(seqlock_t*)` | 初始化顺序锁的指针 |

1. 加锁与解锁

| func lock                                          | func unlock                                            | comments                                                     |
| -------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| `unsigned read_seqbegin(const seqlock_t*)`         | `bool read_seqretry(const seqlock_t*, unsigned start)` | **不需要禁用抢占**，读锁操作是通过判断在加锁和解锁时，通过判断顺序锁的状态（计数器） 是否被写锁操作改变来判定保护的临界区是否存在竞争。 如果前后状态不一致，则再次尝试“加锁“获取状态、”解锁”判断状态的方式，直到前后状态一直为止。 1. `read_seqbegin()`获取一个状态后开始临界区的操作。 2. `read_seqretry()` 离开临界区时，使用`read_seqbegin()`返回的状态和当前锁内部状态比较，如果一直，则返回真，说明没有写锁操作过临界区保护的数据；相反说明有写锁改变了临界区保护的数据，可以再次尝试。比如： `unsigned stat; do { stat = read_seqbegin(lptr); ... } while (read_seqretry(lptr, stat));` |
| `void write_seqlock(seqlock_t*)`                   | `void write_sequnlock(seqlock_t*)`                     | 写锁操作仅在加锁后操纵内部计数器，对内部的自旋锁的操作就是普通的自旋锁操作， 因此对抢占和中断使能的注意事项可以参考 `spin_lock()`。 |
| `write_seqlock_bh(seqlock_t*)`                     | `write_sequnlock_bh(seqlock_t*)`                       | 参考 `spin_lock_bh()`                                        |
| `write_seqlock_irq(seqlock_t*)`                    | `write_sequnlock_irq(seqlock_t*)`                      | 参考 `spin_lock_irq()`                                       |
| `write_seqlock_irqsave(seqlock_t*, unsigned long)` | `write_sequnlock_irqrestore(seqlock_t*)`               | 参考 `spin_lock_irqsave()`                                   |

1. 高级读锁操作

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void read_seqbegin_or_lock(seqlock_t *lock, int *seq)`      | 开始读操作，`*seq` 值如果是奇数则说明与写操作发生竞争，这时会强制加锁内部自旋锁来开始读操作。 一般将 `*seq` 初始化为0（偶数都可以），那么初次是普通读操作，除非发生了竞争。参考内核 `__dentry_path()` 中的使用。 抢占与中断的使能参考 `spin_lock()`。 |
| `bool need_seqretry(seqlock_t *lock, int seq)`               | 如果与写者发生了竞争，则返回真。这时需要再次调用 `read_seqbegin_or_lock()`，这次此函数就强制加锁来开始读操作 |
| `void done_seqretry(seqlock_t *lock, int seq)`               | 完成读操作，根据 `seq` 判断是否需要解锁内部自旋锁（奇数需要解锁） |
| `unsigned long read_seqbegin_or_lock_irqsave(seqlock_t *lock, int *seq)` | `read_seqbegin_or_lock()`的补充，返回保存的中断标志位。参考 `spin_lock_irqsave()`。 |
| `void done_seqretry_irqrestore(seqlock_t *lock, int seq, unsigned long flags)` | `read_seqbegin_or_lock_irqsave()` 对应的 `done_seqretry()` 补充。参考 `spin_unlock_irqrestore()` |

### 信号量

> 1. `#include <linxu/semaphore.h>`
> 2. `#include <linux/rwsem.h>`
> 3. 信号量保护的临界区可以休眠
> 4. 信号量可以是多值的 ————— 多个路径可以同时一个资源，但一般都使用二值信号量，即互斥量。
> 5. 读写信号量是二值的。
> 6. 读写信号量不能相互转换。
> 7. 不能用于（软）中断上下文中，但可以用于异常、系统调用中。

1. 初始化

| macro or func                                    | comments                                         |
| ------------------------------------------------ | ------------------------------------------------ |
| `DEFINE_SEMAPHORE(varname)`                      | 定义静态的信号量变量，二值信号量（0和1）。       |
| `void sema_init(struct semaphore *sem, int val)` | 初始化定义值为 `val` 的多值信号量                |
| `DECLARE_RWSEM(varname)`                         | 定义静态的读写信号量变量，读写信号量仅是二值的。 |
| `init_rwsem(semptr*)`                            | 初始化读写信号量                                 |
| `DEFINE_MUTEX(mutexname)`                        | 定义静态的互斥量                                 |
| `mutex_init(mutexptr*)`                          | 初始化互斥量                                     |

1. 获取与释放

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void down(struct semaphore *sem)`                           | 获取信号量，如果资源可用，则成功返回。否则将当前进程置位 `TASK_UNINTERRUPTIBLE`， 然后挂起，期间不会被中断，直到其他路径释放资源后被唤醒，并在此尝试获取资源。我们将信号量的计数器的值比作资源数。 |
| `int down_interruptible(struct semaphore *sem)`              | 类似 `down()` ，但是挂起进程期间可以被信号中断。如果被中断返回 `-EINTR` ，否则返回0，表示成功获取资源。 |
| `int down_killable(struct semaphore *sem)`                   | 类似 `down_interruptible()`，但是只能被 `KILL` 信号中断。    |
| `bool down_trylock(struct semaphore *sem)`                   | 尝试获取资源，**如果失败返回真**，成功返回假。**不能用于中断上下文** |
| `int down_timeout(struct semaphore *sem, long jiffies)`      | 在一定时间节拍内尝试获取资源，`jiffies` 是一段的节拍时间数。 如果 `jiffies` 为0，则与 `down_trylock()` 功能类似。如果超时返回 `-ETIME`，如果被中断返回 `-EINTR`，成功返回0。 |
| `void up(struct semaphore *sem)`                             | 释放信号量，增加资源数。                                     |
| `void down_read(struct rw_semaphore *sem)`                   | 以读方式获取读写信号量，同一时刻可以有多个路径同时持有读信号量。进程挂起行为参考 `down()`。 |
| `int down_read_trylock(struct rw_semaphore *sem)`            | 以读方式尝试获取读写信号量。**如果成功返回真**，失败返回假。进程挂起行为参考 `down()`。 |
| `void down_write(struct rw_semaphore *sem)`                  | 以写方式获取读写信号量，同一时刻只能有一个路径持有写信号量。进程挂起行为参考 `down()`。 |
| `int down_write_killable(struct rw_semaphore *sem)`          | 类似 `down_write()`。进程挂起行为参考 `dow_killable()`。     |
| `int down_write_trylock(struct rw_semaphore *sem)`           | 尝试写方式尝试获取读写信号量。**如果成功返回真**，失败返回假。 |
| `void up_read(struct rw_semaphore *sem)`                     | 释放以读方式获取的信号量。                                   |
| `void up_write(struct rw_semaphore *sem)`                    | 释放以写方式获取的信号量。                                   |
| `void downgrade_write(struct rw_semaphore *sem)`             | 降低当前进程的优先级，释放信号量，以让读方式获取信号量的进程有机会被调度，从而获取信号量。 |
| `void mutex_lock(struct mutex *)`                            | 获取互斥量，如果有其他进程持有，则挂起进程等待。进程挂起行为参考 `down()`。 |
| `int mutex_lock_interruptible(struct mutex *lock)`           | 类似 `down_interuptible()`。                                 |
| `int mutex_lock_killable(struct mutex *lock)`                | 类似 `down_killable()` 。                                    |
| `int mutex_trylock(struct mutex *lock)`                      | 类似 `down_trylock()`。但是 **如果成功返回真**，失败返回假。 |
| `void mutex_unlock(struct mutex *lock)`                      | 释放互斥量，**加锁和解锁互斥量必须是同一个进程**。           |
| `bool mutex_is_locked(struct mutex *lock)`                   | 判断互斥量是否已被持有。                                     |
| `enum mutex_trylock_recursive_enum mutex_trylock_recursive(struct mutex *lock)` | 以可递归的方式尝试加锁，如果自己已持有互斥量，则返回 `MUTEX_TRYLOCK_RECURSIVE` 。 |
| `int atomic_dec_and_mutex_lock(atomic_t *cnt, struct mutex *lock)` | 如果 `*cnt` 大于1，原子递减 `*cnt` 的值，然后返回假。 否则获取 `*lock` 互斥锁，递减 `*cnt`。如果递减后的 `*cnt` 为0，则持有互斥量（说明有竞争），返回真；否则释放互斥量，返回假。 配合计数器使用，特别在销毁由单个计数器管理的多个资源时。 |

### PerCPU 读写信号量

> ```
> #include <linux/percpu-rwsem.h>
> ```

由于 cacheline 的失效对普通读写信号量有效率上的影响，特别是对读获取。所以开发了 per-cpu 读写信号量，其功能与普通信号量一直。实现等效于，读获取只需要获取本地CPU上的信号量，而写获取需要获取所有CPU上的信号量，这可以减少读获取带来的 cacheline 失效，提高读获取效率；但是写获取由于需要操作多个信号量，所有写获取的效率变低了。尽管通过上述的手段减少了 cacheline 失效的问题，但是由于获取信号量后，可能发生调度，cacheline 的失效还是在所难免的。

1. 定义与初始化

| macro or func                      | comments                                                   |
| ---------------------------------- | ---------------------------------------------------------- |
| `DEFINE_STATIC_PERCPU_RWSEM(name)` | 定义一个静态的perCPU读写信号量                             |
| `percpu_init_rwsem(semptr)`        | 动态初始化。                                               |
| `percpu_free_rwsem()`              | `percpu_init_rwsem()` 的反操作，释放变量内的动态分配内存。 |

1. 操作

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void percpu_down_write(struct percpu_rw_semaphore *sem)`    | 写获取。                                                     |
| `void percpu_up_write(struct percpu_rw_semaphore *sem)`      | 释放写。                                                     |
| `void percpu_down_read(struct percpu_rw_semaphore *sem)`     | 读获取。                                                     |
| `void percpu_down_read_preempt_disable(struct percpu_rw_semaphore *sem)` | 禁用调度后，读获取。                                         |
| `int percpu_down_read_trylock(struct percpu_rw_semaphore *sem)` | 尝试读获取，成功返回真。                                     |
| `void percpu_up_read_preempt_enable(struct percpu_rw_semaphore *sem)` | 释放读，开启中断。 与 `percpu_down_read_preempt_disable()` 配对。 |
| `void percpu_up_read(struct percpu_rw_semaphore *sem)`       | 释放读。                                                     |

### 完成变量

> 1. `#include <linux/completion.h>`
> 2. 完成变量的语义是对信号量的补充 ———— 定义一个完成变量，
>    将完成变量传递给给其他进程，然后等待完成变量资源计数可用；
>    其他获得完成变量的进程增加资源计数，已通知等待资源的进程；
>    等待资源的进程被唤醒。
>    可以参考用户态的 `pthread_cond_t`的使用。
> 3. 可以用在中断上下文，但是不能用于中断上下文嵌套中。

1. 初始化

| macro or func                          | comments           |
| -------------------------------------- | ------------------ |
| `DECLARE_COMPLETION(varname)`          | 定义一个完成变量   |
| `init_completion(struct completion *)` | 初始化一个完成变量 |

1. 唤醒与等待

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void reinit_completion(struct completion *x)`               | 重新初始化完成变量的计数，相当于丢弃以前的 `complete()` 操作 |
| `void wait_for_completion(struct completion *x)`             | 等待完成变量，如果当前资源不可用，则以 `TASK_UNINTERRUPTIBLE` 进程状态挂起进程等待，此时进程不会被信号中断。 |
| `void wait_for_completion_io(struct completion *x)`          | 类似 `wait_for_completion()` ，但用于等待实际的IO资源就绪的上下文中。 |
| `int wait_for_completion_interruptible(struct completion *x)` | 类似 `wait_for_completion()`，但等待可以被中断。如果被中断则返回 `-ERESTARTSYS`；返回0则资源可用。 |
| `int wait_for_completion_killable(struct completion *x)`     | 类似 `wait_for_completion_interruptible()`，但仅响应 `KILL` 信号。 |
| `unsigned long wait_for_completion_timeout(struct completion *x, unsigned long timeout)` | 类似 `wait_for_completion()`，在指定节拍数时间内等待资源可用。**如果返回非0，则表示资源可用，返回0表示超时**。 |
| `wait_for_completion_io_timeout()`                           | 等价 `wait_for_completion_timeout()` 和 `wait_for_completion_io()` 的结合体。 |
| `wait_for_completion_interruptible_timeout()`                | 等价 `wait_for_completion_timeout()` 和 `wati_for_completion_interruptible()` 的结合体。 |
| `wait_for_completion_killable_timeout()`                     | 等价 `wait_for_completion_timeout()` 和 `wati_for_completion_killable()` 的结合体。 |
| `bool try_wait_for_completion(struct completion*)`           | 尝试获取资源，如果成功则返回真。                             |
| `bool completion_done(struct completion *x)`                 | 如果没有等待完成变量的进程，则返回真。                       |
| `void complete(struct completion *)`                         | 增加一个资源，仅能唤醒一个等待资源的进程。                   |
| `void complete_all(struct completion *)`                     | 增加足够多(2^30 - 1)的资源，唤醒全部等待资源的进程。 如果有这样的唤醒需求，在消费资源的进程，应该调用 `reinit_completion()` 进行资源重置。 |

### 中断操作

> 1. `#include <linux/irqflags.h>`
> 2. `#include <linxu/bottom_half.h>`
> 3. 中断发生时不可能发生异常（系统调用和缺页等）
> 4. 同一个软中断（tasklet）只会在一个CPU上串行执行（非并行、非嵌套）
> 5. 软中断可以被硬中断中断。
> 6. 硬中断处理不能被抢占。
> 7. 执行一个短的不阻塞的任务。

1. 中断使能和中断上下文

| macro or func                | comments                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| `irqs_disabled()`            | 如果本地中断（线）被禁用则返回真。                           |
| `irqs_disabled_flags(flags)` | 如果给出的本地中断标志中本地中断被禁用，则反真。这个标志由 `local_irq_save(flags)` 获得。 |
| `local_irq_save(flags)`      | 保存本地中断标志，**并禁用本地中断**。                       |
| `local_irq_restore(flags)`   | 以指定值恢复本地中断标志，**并没有开启本地中断，这时阻止中断嵌套的最好的方法**。 |
| `local_save_flags(flags)`    | 仅保存本地中断标志，一定要和 `local_irq_save()` 做区分。     |
| `local_irq_enable()`         | 开启本地中断，**在中断上下文使用时，一定要当心，开启中断就可能发生中断嵌套**。 |
| `local_irq_disable()`        | 禁用本地中断。                                               |
| `local_bh_enable()`          | 开启软中断，如果有软中断正等待处理，就立即处理。**不要在禁用中断或硬中断处理中调用，否则发出警告**。 |
| `local_bh_disable()`         | 禁用软中断。                                                 |
| `in_irq()`                   | 如果正在硬中断上下文，则返回真。                             |
| `in_softirq()`               | 如果正在软中断上下文（包括下半部处理），则返回真。           |
| `in_interrupt()`             | 如果正在中断上下文，则返回真。                               |
| `in_serving_softirq()`       | 如果正在处理软中断，则返回真。                               |
| `in_atomic()`                | 如果在一个临界区上下文（包括中断和禁止抢占），则返回真。**在非抢占内核中不能正确的检查，但只要不在驱动代码中使用，应该没有什么问题**。 |
| `local_softirq_pending()`    | 如果有待处理的软中断，则返回值大于0，各个位就是待处理软中断的下标。 |

1. `tasklet` 使用

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `DECLARE_TASKLET(name, func, data)`                          | 定义一个`name`的tasklet变量，`func`和`data` 分别是回调函数和其数据 |
| `DECLARE_TASKLET_DISABLED(name, func, data)`                 | 类似 `DECLARE_TASKLET()` 但是初始化是禁用状态（即使被软中断调度，也不会执行） |
| `void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data)` | 动态初始化 tasklet                                           |
| `void tasklet_schedule(struct tasklet_struct *t)`            | 以 `TASKLET_SOFTIRQ` 模式调度 tasklet ———— 即与 `TASKLET_SOFTIRQ` 等级的软中断关联，排队到等处理 tasklet 队列的尾部。 同一个 tasklet 只能与某一个CPU上的某一个等级软中断关联。 一旦软中断被执行，tasklet 关联的函数就可能被执行，如果要再次运行需要再次调度。 |
| `tasklet_hi_schedule()`                                      | 类似 `tasklet_schedule()` ，但以 `HI_SOFTIRQ` 模式关联。     |
| `tasklet_hi_schedule_first()`                                | 类似 `tasklet_hi_schedule()` ，但是排队到队列的首部 ———— 一旦软中断执行，可以立马执行关联的函数。**使用时要非常小心**。 |
| `tasklet_kill()`                                             | 杀死 tasklet，是所有类 `tasklet_schedule()` 函数的反操作。如果还没有执行关联函数，则以后都不会被执行，否则等待执行完毕。 **尽量不要在中断上下文中使用，因为此函数会调度；禁止在tasklet的回调函数中执行，会发生死锁**。 |
| `void tasklet_disable_nosync(struct tasklet_struct *t)`      | 禁用 tasklet ———— 可以被调度，但不会执行关联的函数。         |
| `tasklet_disable()`                                          | 类似 `tasklet_disable_nosync()` ，但是如果现在关联的函数正在执行，则等待完成。 |
| `tasklet_enable()`                                           | 启用 tasklet ———— 若被调度，则可以执行关联的函数。           |
| `void tasklet_hrtimer_init(struct tasklet_hrtimer *ttimer, enum hrtimer_restart (*function)(struct hrtimer *), clockid_t which_clock, enum hrtimer_mode mode)` | 初始化一个高精度定时执行的 tasklet。 1. 参数 `function` 是tasklet的回调，它的返回值，如果不是 `HRTIMER_NORESTART` 则这个 tasklet 会在未来某个时间再次触发； 它的参数 `struct hrtimer*`就是 `struct tasklet_hrtimer` 中的成员，对其使用 `container_of()` 可以得到这个 tasklet 本身。 2. 参数 `which_clock` 是指定使用的时钟源类型，比如 `CLOCK_REALTIME` 或 `CLOCK_MONOTONIC` 等。 3. `mode` 是定时器模式，比如 `HRTIMER_MODE_ABS` 、 `HRTIMER_MODE_REL`等。 |
| `void tasklet_hrtimer_start(struct tasklet_hrtimer *ttimer, ktime_t time, const enum hrtimer_mode mode)` | 启动 tasklet，`time` 是定时器触发时间，`mode` 是计时模式，以绝对时间还是相对时间计时。触发定时器后，才是真正的调度这个 tasklet 。 |
| `void tasklet_hrtimer_cancel(struct tasklet_hrtimer *ttimer)` | 取消定时器和杀死 tasklet ， `tasklet_hrtimer_start()` 的反操作。 |

### 工作队列

> 1. 工作队列与 tasklet 在功能上类似，但是运行在进程上下文
> 2. 工作队列可以运行阻塞函数，切换换进程等。
> 3. 排队这个动作使用的自旋锁，所以可以在中断上下文中使用，其他的大多数都会休眠，使用时要时刻注意自己所处的上下文。

1. 工作队列操作

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `alloc_workqueue(fmt, flags, max_active, args...)`           | 创建一个通用工作队列。 1. `fmt` 和 `args...` 是这个队列的名字格式化数据。 2. `flags` 决定了工作队列的属性，比如：  1. `WQ_UNBOUND` 不绑定执行任务线程的CPU；  2. `WQ_CPU_INTENSIVE` 可以执行CPU密集型的任务；  3. `WQ_MEM_RECLAIM` 可以进行内存回收等工作，而且会创建新的线程来工作。 3. `max_active` 可以同时执行排队任务的数量，如果为0，则是默认数量。 |
| `alloc_ordered_workqueue(fmt, flags, args...)`               | 创建一个排队的工作队列，所有任务都是按加入顺序执行的。       |
| `create_workqueue(name)`                                     | 为了匹配老旧的接口的创建工作队列。                           |
| `create_freezable_workqueue(name)`                           | 创建一个在待机时可以挂起所有工作的队列。                     |
| `create_singlethread_workqueue(name)`                        | 单线程模式的工作队列，上面的工作队列都是线程池模式。         |
| `void destroy_workqueue(struct workqueue_struct *wq)`        | 销毁工作队列。**如果带非绑定属性的队列有未完成的任务，则中断销毁，否则都保证挂起的任务都会执行完成**。 |
| `int workqueue_set_unbound_cpumask(cpumask_var_t cpumask)`   | 设置所有工作队列中的没有绑定CPU的工作线程的CPU亲和性（仅在这些CPU上运行）。 1. 返回0，成功。 2. 返回 -EINVAL，CPU掩码参数无效。 3. 返回 -ENOMEM。在分配属性时没有内存了。 |
| `struct workqueue_attrs *alloc_workqueue_attrs(gfp_t gfp_mask)` | 分配一个内存相关的工作队列属性。                             |
| `void free_workqueue_attrs(struct workqueue_attrs *attrs)`   | 释放属性。                                                   |
| `int apply_workqueue_attrs(struct workqueue_struct *wq, const struct workqueue_attrs *attrs)` | 将属性指针中的数据拷贝到工作队列中，从而改变原来的属性。     |
| `void flush_workqueue(struct workqueue_struct *wq)`          | 等待当前所有排队的任务完成。如果在等待开始后，不会理会之后添加的任务。 |
| `void drain_workqueue(struct workqueue_struct *wq)`          | 等待所有排队的任务完成。在等待过程中，除已排队或正在执行的任务可以再次排队，其他的任务不能排队到工作队列，但应该尽量避免这种情况。 |

1. 内核预创建的工作队列

| var                                                          | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `extern struct workqueue_struct *system_wq;`                 | 普通的工作队列，不要排队长时间运行的任务                     |
| `extern struct workqueue_struct *system_highpri_wq;`         | 高优先级的工作队列                                           |
| `extern struct workqueue_struct *system_long_wq;`            | 可以排队长时间运行的工作队列                                 |
| `extern struct workqueue_struct *system_unbound_wq;`         | 不会将工作线程绑定在固定CPU上的工作队列                      |
| `extern struct workqueue_struct *system_freezable_wq;`       | 可以在系统休眠时冻结工作线程的工作队列                       |
| `extern struct workqueue_struct *system_power_efficient_wq;` | 如果内核有能效特别设置，工作队列里的工作线程会自动解除CPU绑定 |
| `extern struct workqueue_struct *system_freezable_power_efficient_wq;` | 以上两者的结合物                                             |

1. 工作任务初始化

| macro or func                              | comments                                                     |
| ------------------------------------------ | ------------------------------------------------------------ |
| `INIT_WORK(workptr, func)`                 | 初始化`workptr`普通工作任务，`func`是这个任务的回调函数。    |
| `INIT_WORK_ONSTACK(workptr, func)`         | 初始化在栈上定义的`workptr`普通工作任务，`func`是这个任务的回调函数。 **你必须保证在退出这个变量的作用域前，工作任务已经不再在工作队列中排队或运行**。 |
| `INIT_DELAYED_WORK(workptr, func)`         | 初始化定时的工作任务，类似 `INIT_WORK()` 但是任务一定会被推迟到某个时间点运行。延迟工作任务是使用的延迟定时器，优先级非常低。 |
| `INIT_DELAYED_WORK_ONSTACK(workptr, func)` | 类似 `INIT_DELAYED_WORK()` 和 `INIT_WORK_ONSTACK()` 的结合物 |
| `DECLARE_WORK(name, func)`                 | 定义`name`的普通工作任务                                     |
| `DECLARE_DELAYED_WORK(name, func)`         | 定义`name`的定时工作任务                                     |

1. 工作任务排队

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `bool queue_work_on(int cpu, struct workqueue_struct *wq, struct work_struct *work)` | 在工作队列的指定CPU上排队一个工作任务，**在内部工作线程在这个CPU队列上拿到这个任务后，不保证一直在这个CPU上运行任务，因为工作线程是可以在所有CPU上漂移的，要根据工作队列的属性而定**，成功返回真，否则表示已经排队。 |
| `queue_work()`                                               | 类似 `queue_work_on()`，但由内核自动选择一个最合适的CPU执行。 |
| `bool queue_delayed_work_on(int cpu, struct workqueue_struct *wq, struct delayed_work *work, unsigned long delay)` | 类似 `queue_work_on()`，期望在`delay`节拍时间过后排队任务，如果为0，则立刻排队。 |
| `bool mod_delayed_work_on(int cpu, struct workqueue_struct *wq, struct delayed_work *dwork, unsigned long delay)` | 如果已排队（正在执行的话，直接失败返回），可以修改延迟任务的超时时间。 |
| `int schedule_on_each_cpu(work_func_t func)`                 | 在每个CPU排队一个函数式任务。 成功返回0，出错返回错误码的负数表示。这个任务运行在 `system_wq` 预创建工作队列上。 |
| `bool schedule_work(struct work_struct *work)`               | 在 `system_wq` 上排队一个任务，如果已经排队则返回假。        |
| `int execute_in_process_context(work_func_t fn, struct execute_work *ctx)` | 使用 `ctx` 这个工作任务作为 `fn` 这个函数任务的参数，然后排队。 如果当前不是中断上下文，立马以 `ctx` 执行这个函数任务； 否则丢弃 `ctx` 以前的任务，重新排队，如果 `ctx` 已排队则什么事都不会发生。 |
| `bool flush_work(struct work_struct *work)`                  | 等待工作任务完成，如果工作任务没有排队，则返回假。           |
| `bool cancel_work(struct work_struct *work)`                 | 异步取消排队的任务。                                         |
| `bool cancel_work_sync(struct work_struct *work)`            | 同步取消排队的任务。                                         |
| `bool flush_delayed_work(struct delayed_work *dwork)`        | 等待延迟工作任务完成，如果工作任务没有排队，则返回假。       |
| `bool cancel_delayed_work(struct delayed_work *dwork)`       | 异步取消排队的延迟任务。                                     |
| `bool cancel_delayed_work_sync(struct delayed_work *dwork)`  | 同步取消排队的延迟任务。                                     |
| `unsigned int work_busy(struct work_struct *work)`           | 检查工作任务的状态，返回一组掩码位，`WORK_BUSY_PENDING` 或 `WORK_BUSY_RUNNING`。这个结果仅供参考，因为任务状态随时可能变化。 |
| `work_pending()`                                             | 查看普通工作任务是否已排队，排队则返回真。                   |
| `delayed_work_pending()`                                     | 查看延迟工作任务是否已排队，排队则返回真。                   |

### 等待队列

> 1. `#include <linux/wait.h>`
> 2. 用于挂起进程，等待资源就绪后由其他进程唤醒。
> 3. 等待队列由两部分组成 ———— 等待队列头和等待队列实体。当我们想挂起进程等待某个资源就绪时，我们定义一个局部的或全局的等待队列实体，然后插入已定义的、其他使用资源的进程也可见的等待队列头中，然后设置进程的状态（可能从调度队列中移除进程），然后切换进程去等待他人唤醒。其他进程使用完资源后（或制造资源），通过等待队列头唤醒一个或多个挂起的进程。

1. 等待队列头和实体的通用初始化操作

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `DECLARE_WAIT_QUEUE_HEAD(name)`                              | 定义一个等待队列头                                           |
| `init_waitqueue_head(headptr)`                               | 初始化一个等待队列头                                         |
| `int waitqueue_active(wait_queue_head_t *q)`                 | 判断等待队列里是否有等待者。                                 |
| `bool wq_has_sleeper(wait_queue_head_t *wq)`                 | 类似 `waitqueue_active()`，建议使用此函数。                  |
| `void init_wait_entry(wait_queue_t *wait, int flags)`        | 类似 `init_waitqueue_entry()`，初始化等待队列实体，将当前进程作为需要被唤醒的进程标识符， 并将 `autoremove_wake_function()` 作为唤醒函数，它与 `default_wake_function()` 不同之处就是，等待者被唤醒后，自动从等待队列中删除。 如果 falgs 为 `WQ_FLAG_EXCLUSIVE` ，则初始化为互斥的。 |
| `init_wait(waitptr)`                                         | 类似 `init_wait_entry()`，但是用默认标志（不互斥）。         |
| `DEFINE_WAIT(name)`                                          | 类似 `init_wait()`，定义一个默认参数的等待实体，使用 `autoremove_wake_function()` 作为唤醒函数。 |
| `DEFINE_WAIT_FUNC(name, function)`                           | 类似 `DEFINE_WAIT()`，以指定函数定义等待实体。               |
| `DECLARE_WAIT_QUEUE_HEAD_ONSTACK(name)`                      | 在栈上定义一个等待队列头                                     |
| `DECLARE_WAITQUEUE(name, tsk)`                               | 定义一个等待队列实体，使用 `default_wake_function()` 函数作为默认唤醒函数， 该函数唤醒等待者之后，不会将等待实体从等待队列中删除（需要手动删除）， `tsk` 是需要被唤醒的进程标识符，用作唤醒函数的进程参数。 |
| `void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)` | 类似 `DELARE_WAITQUEUE()` 初始化一个等待队列实体。           |
| `void init_waitqueue_func_entry(wait_queue_t *q, wait_queue_func_t func)` | 使用自定义唤醒函数去初始化一个等待队列实体。                 |
| `default_wake_function()`                                    | 默认唤醒函数，仅使用 `try_to_wake_up()` 唤醒等待的进程，返回真表示唤醒成功。 |
| `autoremove_wake_function()`                                 | 类似 `default_wake_function()` ，但唤醒成功之后之后还从等待队列中删除实体。 |

1. 普通等待队列操作

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `wake_up(wait_queue_head_t*)`                                | 以 `TASK_NORMAL` 模式唤醒一个等待者。`TASK_NORMAL` 被传递给每一个队列中等待队列实体定义的`wait_queue_func_t()`的mode参数。所有唤醒函数都是先唤醒队列头部的等待者，如果唤醒函数 `wait_queue_func_t()` 返回真，且当前被唤醒的等待者是互斥的，并唤醒数量等于0（负数不算），那么停止唤醒。 |
| `wake_up_nr(x, nr)`                                          | 类似 `wake_up()` ，但指定唤醒的数量。                        |
| `__wake_up_sync()`                                           | 同步唤醒，可能主动抢占被唤醒进程所在CPU当前运行的进程。      |
| `wake_up_all(x)`                                             | 唤醒所有等待者，包括互斥标志的等待者。                       |
| `wake_up_locked(x)`                                          | 在持有等待队列锁的情况下，唤醒一个等待者。                   |
| `wake_up_all_locked(x)`                                      | 类似 `wake_up_locked()`，唤醒所有。                          |
| `wake_up_interruptible(x)`                                   | 类似 `wake_up()`，但是以 `TASK_INTERRUPTIBLE` 传递给唤醒函数。 |
| `wake_up_interruptible_nr(x, nr)`                            | 类似 `wake_up_interruptible()`，唤醒指定数目的等待者。       |
| `wake_up_interruptible_all(x)`                               | 类似 `wake_up_interruptible()`，唤醒所有的等待者。           |
| `void prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)` | 准备挂起进程等待资源，以非互斥的方式（不管原来的标志）将等待队列实体插入等待队列头部（如果没有插入），并将进程状态设置为 `state` （比如 `TASK_INTERRUPTIBLE`）。 |
| `prepare_to_wait_exclusive()`                                | 类似 `prepare_to_wait()`，但是以互斥的方式等待，并插入到等待队列的尾部。 |
| `prepare_to_wait_event()`                                    | 类似 `prepare_to_wait()`，但是根据原来的标志判断是否以互斥方式等待，并且如果当前进程有信号待处理，则失败返回 `-ERESTARTSYS` ，否则成功返回 0 。 |
| `void finish_wait(wait_queue_head_t *q, wait_queue_t *wait)` | 结束等待资源，从等待队列里删除实体。                         |

1. 普通等待队列的条件等待操作

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `wait_event(wq, condition)`                                  | 挂起当前进程直到条件变为真。 `wq` 等待队列头，`condition` 条件表达式，不可被信号中断。 |
| `io_wait_event(wq, condition)`                               | 类似 `wait_event()`，但为了等待某个相关的IO资源，不可被信号中断。 |
| `wait_event_freezable()`                                     | 挂起并试图冻结进程直到条件变为真，响应信号可中断。           |
| `wait_event_timeout(wq, condition, timeout)`                 | 在指定的 `timeout` 节拍数内挂起进程，等待条件变为真，不可被信号中断。 当返回 `0` 时表示超时， 大于或等于 `1` 表示剩余节拍数，此时条件变为真。 |
| `wait_event_freezable_timeout()`                             | 挂起并试图冻结进程直到条件变为真，类似 `wait_event_timeout()`，但是响应信号可中断。 |
| `wait_event_cmd(wq, condition, cmd1, cmd2)`                  | 类似 `wait_event()`，但会在休眠前执行 `cmd1` 语句，在唤醒后执行 `cmd2` 语句。 |
| `wait_event_exclusive_cmd(wq, condition, cmd1, cmd2)`        | 类似 `wait_event_cmd()`，但以互斥标识等待唤醒。              |
| `wait_event_interruptible(wq, condition)`                    | 类似 `wait_event()`，但可被信号中断。                        |
| `wait_event_interruptible_timeout(wq, condition, timeout)`   | 类似 `wait_event_timeout()`，但可被信号中断。                |
| `wait_event_hrtimeout(wq, condition, timeout)`               | 类似 `wait_event_timeout()`，但是用高精度的 `ktimer` 作为计时器，不可被信号中断。 |
| `wait_event_interruptible_hrtimeout(wq, condition, timeout)` | 类似 `wait_event_hrtimeout()`，但可被信号中断。              |
| `wait_event_killable(wq, condition)`                         | 类似 `wait_event()`，但可以被 `KILL` 信号唤醒。              |
| `wait_event_killable_exclusive(wq, condition)`               | 类似 `wait_event()`，但以互斥方式等待唤醒，且可以被 `KILL` 信号唤醒。 |
| `wait_event_freezable_exclusive(wq, condition)`              | 类似 `wait_event_freezable()`，但以互斥方式等待唤醒。        |
| `wait_event_interruptible_locked(wq, condition)`             | 类似 `wait_event_interruptible()`，但是调用者已获取了等待队列的锁。 |
| `wait_event_interruptible_locked_irq(wq, condition)`         | 类似 `wait_event_interruptible_locked()`，但是调用者已经禁用了本地中断。 |
| `wait_event_interruptible_exclusive_locked(wq, condition)`   | 类似 `wait_event_interruptible_locked()`，但以互斥方式等待唤醒。 |
| `wait_event_interruptible_exclusive_locked_irq(wq, condition)` | 类似 `wait_event_interruptible_locked_irq()`，但还以互斥方式等待唤醒。 |
| `wait_event_lock_irq_cmd(wq, condition, lock, cmd)`          | 类似 `wait_event_interruptible_locked_irq()`，但不可中断，且在休眠前调用 `cmd` 语句。 |

1. 高级等待队列操作

这种定义会在唤醒和等待时判断 `WQ_FLAG_WOKEN` 标志（这时内部使用的标志，调用者不应该使用它），保证是被等待队列唤醒的，减少其他进程或调度程序的干扰。

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)` | 以非互斥的方式将等待者排队到队列头。非互斥的等待者可以被同时唤醒。 |
| `void add_wait_queue_exclusive(wait_queue_head_t *q, wait_queue_t *wait)` | 以互斥的方式将等待者排队到队列尾。互斥的等待者一般情况下只同时唤醒一个。 |
| `void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)` | 从等待队列头中移除指定的实体。                               |
| `long wait_woken(wait_queue_t *wait, unsigned mode, long timeout)` | 期望在 `timeout` 节拍时间内等待资源就绪。 直接挂起进程等待资源，在切换进程之前，设置进程为 `mode`状态，检查 `WQ_FLAG_WOKEN` 标志， 如果已设置，则直接返回。在返回之前设置进程为 `TASK_RUNNING` 状态，并清除 `WQ_FLAG_WOKEN` 状态。 返回值是剩余时间，大于0，则表示资源可能已就绪；等于0，则表示超时，资源未就绪。 |
| `woken_wake_function()`                                      | 高级等待队列的通用唤醒函数。                                 |

1. 位等待队列操作

使用位等待锁时，可以使用系统中的专用位等待队列，一般用于页标志位等待操作。

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void __wake_up_bit(wait_queue_head_t *wq, void *word, int bit)` | 唤醒在 `wq` 上等待 `word` 这个字的 `bit` 位改变的等待者。    |
| `int __wait_on_bit(wait_queue_head_t *wq, struct wait_bit_queue *q, wait_bit_action_f *action, unsigned mode);` | 等待前，先观察 `q` 关联的位地址，如果观测的位为 `1`（资源不可用） 则调用 `action` 函数进行互斥等待（一般是放弃CPU）。如果 `action` 返回 0，则一直继续循环前面的动作进行等待。**返回 0 表示位变为了0， 否则位仍然是 1。** |
| `int __wait_on_atomic_t(wait_queue_head_t *wq, struct wait_bit_queue *q, int (*action)(atomic_t *), unsigned mode);` | 类似 `__wait_on_bit()`，但原子的测试（测试否是为0值）作用在整个原子值上，而非某个位上。 |
| `__wait_on_bit_lock()`                                       | 类似 `__wait_on_bit()`，但是如果 `action` 返回 0（不希望放弃等待），则试图主动的原子抢占等待位，如果成功将其置位 1 （自己占有了资源），如果然后返回 0。 |
| `void wake_up_bit(void *wq, int bit)`                        | 类似 `__wake_up_bit()` ，但直接使用 系统预定义 的等待队列。  |
| `void wake_up_atomic_t(atomic_t *word)`                      | 唤醒等待 `word` 原子改变的等待者，也直接使用系统预定义的等待队列。 |
| `wait_queue_head_t *bit_waitqueue(void *word, int bit)`      | 根据 `word` 和 `bit` 算出一个系统预定义的等待队列。          |
| `DEFINE_WAIT_BIT(name, word, bit)`                           | 定义一个位等待者，使用默认唤醒唤醒函数 `wake_bit_function()` |
| `wake_bit_function()`                                        | 默认位等待队列唤醒函数，如果等待的位仍为 `1` 则**放弃**唤醒。 |
| `int (*wait_bit_action_f)(void *key, int mode)`              | 位等待者的动作函数，`key` 是观测的位所在内存地址， `mode` 是调用者希望放弃CPU时，如何设置进程状态，如果返回0，则希望继续等待位改变。 |
| `int wait_on_bit(unsigned long *word, int bit, unsigned mode)` | 快捷等待函数（比作可休眠的位锁），等待 `bit` 位变为0（没有主动抢占，你需要使用原子操作再次确认），`mode` 为放弃CPU时设置进程的状态。 返回 0 成功，否则失败。 |
| `int wait_on_bit_io()`                                       | 用于IO上下文。                                               |
| `int wait_on_bit_timeout()`                                  | 类似 `wait_on_bit()` ，但是 仅等待 `timeout` 的节拍数。返回 0 成功，否则 `-EAGAIN` 表示超时，`-EINTR` 表示被信号中断（`mode` 参数被忽略）。 |
| `int wait_on_bit_action(unsigned long *word, int bit, wait_bit_action_f *action, unsigned mode)` | 类似 `__wait_on_bit()`，使用系统预定义的队列。               |
| `int wait_on_bit_lock(unsigned long *word, int bit, unsigned mode)` | 类似 `__wait_on_bit_lock()`，但在进入休眠循环前也尝试获取抢占位，成功直接返回 0。 |
| `wait_on_bit_lock_io()`                                      | 类似 `wait_on_bit_lock()` 但用于IO上下文。                   |
| `wait_on_bit_lock_action()`                                  | 类似 `wait_on_bit_lock()` 和 `wait_on_bit_action()` 的结合物。 |
| `int wait_on_atomic_t(atomic_t *val, int (*action)(atomic_t *), unsigned mode)` | 类似 `__wait_on_atomic_t()` 但是用内核预定义队列，并在休眠前测试原子是否为0值，是则直接返回0，表示成功。 |

## 内存管理

> 1. `#include <linux/gpf.h>`

### 物理页管理

1. 分配标志

分配表示用于说明如何得到页，页会做什么样的处理等。很多标志可以彼此组合。

下列标志是获取内存区域（按高端向低端排列），他们是互斥的只能有一个。

| flags           | comments                                                     |
| --------------- | ------------------------------------------------------------ |
| `__GFP_MOVABLE` | 可移动区域的内存，如果不足，则使用其他低区域的内存。此区域是软件层面的区域，通过页框压缩和回收算法，可以将这个区域的页移动到同一个node中的其他zone中。 |
| `__GFP_HIGHMEM` | 高端内存区域，64位没有高端内存概念。如果不足使用低端内存。   |
| `__GFP_DMA32`   | 32位直接存取内存区域，如果不足则使用更低断内存。             |
| `__GFP_DMA`     | 直接存取内存区域。在物理内存急缺时，内核分配内存最后的手段。 |

下列标志控制分配时内存回收等动作，可以组合使用

| flags                  | comments                                                     |
| ---------------------- | ------------------------------------------------------------ |
| `__GFP_RECLAIMABLE`    | 主要用于 类`slab` 分配，在页框压缩中，通过注册`shrinkers` 进行回收没有使用的页。 |
| `__GFP_WRITE`          |                                                              |
| `__GFP_HARDWALL`       | 使用 `cpuset` 分配策略。                                     |
| `__GFP_THISNODE`       | 指示不适用备用node的页，仅分配期望node上的页。               |
| `__GFP_ACCOUNT`        | 增加统计信息用于 `kmemcg` 。                                 |
| `__GFP_ATOMIC`         | 用于原子上下文（分配不能休眠）。如果常规内存不足，直接使用紧急内存。 |
| `__GFP_HIGH`           | 高权限分配，低于 `__GFP_ATOMIC`，比如用于 IO 上下文。        |
| `__GFP_MEMALLOC`       | 如果内存不够，先使用紧急内存，再就压缩回收内存，甚至杀死别的进程。这个过程会相当耗时，但是能尽可能的保证分配页成功，一般用于用户层的请求。 |
| `__GFP_NOMEMALLOC`     | 与 `__GFP_MEMALLOC` 是互斥的，与其同时指定，会覆盖 `__GFP_MEMALLOC` 标志。这个标志是在常规内存不足时，禁止使用紧急内存。 |
| `__GFP_IO`             | 如果发生内存回收，可以进行物理的磁盘回写。                   |
| `__GFP_FS`             | 回收文件系统使用的缓存（inode缓存等），不使用此标志可以避免递归的进入文件系统。 |
| `__GFP_DIRECT_RECLAIM` | 如果本地内存水平位低时就开启回收。当有其他节点的后备页可用于分配时，不设置此标志可以减少分配延迟。 |
| `__GFP_KSWAPD_RECLAIM` | 如果本地内存水平位过低时开启回收，且直接唤醒交换守护进程，进行实际的页框回写回收。 |
| `__GFP_RECLAIM`        | 即开启回收，并唤醒交换守护进程。                             |
| `__GFP_REPEAT`         | 如果整个系统的内存都不足时，尝试重复的回收、等待，直到有内存可用。但根据不同VM的实现，还是可能失败。 |
| `__GFP_NOFAIL`         | 分配一定不会失败，直到有内存可用为止。非常耗时的分配方案。   |
| `__GFP_NORETRY`        | 如果内存不足，执行了回收之后，不再尝试分配，返回错误。       |

下列标志控制分配页的本身，可以组合使用

| flags                          | comments                                                    |
| ------------------------------ | ----------------------------------------------------------- |
| `__GFP_COLD`                   | 分配冷页，一般都是分配热页，这样可以使用硬件高速缓存和TLB。 |
| `__GFP_NOWARN`                 | 分配失败时，不打印警告信息。                                |
| `__GFP_COMP`                   | 分配组合页，仅分配多页时有效。                              |
| `__GFP_ZERO`                   | 将分配的页清零。                                            |
| `__GFP_NOTRACK`                | 避免被跟踪（内存泄露检查机制）。                            |
| `__GFP_NOTRACK_FALSE_POSITIVE` | 就是 `__GFP_NOTRACK`                                        |
| `__GFP_OTHER_NODE`             | 要求分配**非本地node**的内存。                              |

使用中的快捷标志组合

| flags                  | comments                                               |
| ---------------------- | ------------------------------------------------------ |
| `GFP_ATOMIC`           | 中断或原子上下文使用                                   |
| `GFP_KERNEL`           | 内核层使用的，开启所有可能的回收机制。比较耗时的分配。 |
| `GFP_KERNEL_ACCOUNT`   | 内核层使用的，还记录统计信息。                         |
| `GFP_NOWAIT`           | 不会阻塞在回收机制上。                                 |
| `GFP_NOIO`             | 回收机制不执行磁盘IO。                                 |
| `GFP_NOFS`             | 回收机制不回收文件系统的缓存。                         |
| `GFP_USER`             | 用户层使用。                                           |
| `GFP_DMA`              | 分配DMA的内存，主要是低级驱动程序使用。                |
| `GFP_DMA32`            | 高级驱动程序使用。                                     |
| `GFP_HIGHUSER`         | 用户层使用，且优先分配高端内存。                       |
| `GFP_HIGHUSER_MOVABLE` | 用户层使用，分配的页可以迁移。典型的是作用在LRU的页。  |

1. 页的分配与释放
   1. 除非特别说明，所有释放函数都要查看页的引用计数。只有为0的页才能释放。
   2. 只能分配最多分配 `2 ^ (MAX_ORDER - 1)` 个连续页，即 4MB 大小。

基础分配释放函数

| alloc func                                                   | free func                                                  | comments                                                     |
| ------------------------------------------------------------ | ---------------------------------------------------------- | ------------------------------------------------------------ |
| `struct page *alloc_pages_node(int nid, gfp_t gfp_mask, unsigned int order)` | `void __free_pages(struct page *page, unsigned int order)` | 在指定的node上分配 2^order 页，返回页描述符。释放时，使用页描述符来释放这 2^order 个页。node 为 -1 则分配本地节点上的页。 |
| `void *alloc_pages_exact(size_t size, gfp_t gfp_mask);`      | `void free_pages_exact(void *virt, size_t size);`          | 分配虚地址大小为 `size` 的页，返回这些页的直接映射的虚地址。释放时，也使用这个虚地址和大小释放这些页。 |
| `unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);` | `void free_pages(unsigned long addr, unsigned int order);` | 使用 2^order 张页，返回连续页的虚地址。释放时使用虚地址和 order 数进行释放。 |

辅助分配释放函数

| func or macro                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `struct page * alloc_pages(gfp_t gfp_mask, unsigned int order)` | 使用 `mempolicy` 策略进行内存分配，使用 `__free_pages()` 释放。 |
| `struct page * alloc_pages_vma(gfp_t gfp_mask, struct vm_area_struct *vma, unsigned long addr, int order, int node, bool hugepage)` | 为挂载在 vma 管理虚地址区域分配页。使用 `__free_pages()` 释放。 |
| `alloc_hugepage_vma()`                                       | 类似 `alloc_pages_vma()`，但是在本地节点分配巨型页。         |
| `alloc_page_vma()`                                           | 类似 `alloc_pages_vma()` ，在本地节点分配一页普通页。        |
| `alloc_page_vma_node()`                                      | 类似 `alloc_pages_vma()`，在指定node分配一页普通页。         |
| `alloc_page(gfp_mask)`                                       | 分配单页。 使用 `__free_page()` 释放。                       |
| `unsigned long get_zeroed_page(gfp_t gfp_mask)`              | 分配一张页，清零数据后，返回页的虚地址。 使用 `free_page()` 释放。 |
| `void free_hot_cold_page(struct page *page, bool cold)`      | 释放单页，`cold` 为真表示将页释放到热页页缓存池中。**不管页的引用计数直接释放**。 |
| `void free_hot_cold_page_list(struct list_head *list, bool cold)` | 类似 `free_hot_cold_page()`，将 `list` 中的页全部按单页释放（`page->lru`链接到该链表）。 |
| `__get_free_page()`                                          | 分配单页，返回单页的虚地址。                                 |
| `__get_dma_pages(gfp_mask, order)`                           | 返回 2^order 个连续的DMA内存区域的页。                       |
| `__free_page(page)`                                          | 按页描述符释放单页。                                         |
| `free_page(addr)`                                            | 按单页的虚地址释放单页。                                     |
| `void release_pages(struct page **pages, int nr, bool cold)` | 释放 `pages[]` 数组中 `nr` 个页。 `cold` 指示以什么方式将这些单页释放到冷热页缓存中。 |

1. 页的引用计数

| func or macro                                               | comments                                                     |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| `get_page()`                                                | 增加页的引用计数，如果是作用于 组合页 则只增加头页的引用计数。 |
| `put_page()`                                                | 减少页的引用计数，如果是作用于 组合页 则只减少头页的引用计数，但引用计数变为0时，自动释放页到伙伴系统。 |
| `page_count()`                                              | 获取页的引用计数，如果是作用于 组合页 则获取的是头页的引用计数。 |
| `page_ref_count()`                                          | 获取页的引用计数，但忽略 组合页。                            |
| `set_page_count(struct page *page, int v)`                  | 强制设置也的引用计数为 `v`，但忽略 组合页。                  |
| `void init_page_count(struct page *page)`                   | 初始化页的引用计数为 1，但忽略 组合页。                      |
| `void page_ref_add(struct page *page, int nr)`              | 强制增加 `nr` 页的引用计数，但忽略 组合页。此外有减、自增、自加等函数，就不一一列举了。 |
| `int page_ref_sub_and_test(struct page *page, int nr)`      | 强制增加 `nr` 页的引用计数，如果为引用计数为 0 ，则返回 真，忽略组合页。 |
| `page_ref_inc_return()`                                     | 强制自增，然后返回增加后的页引用计数，忽略组合页。           |
| `page_ref_dec_and_test()`                                   | 强制自减，如果为引用计数为 0 ，则返回 真，忽略组合页。       |
| `page_ref_dec_return()`                                     | 强制自减，然后返回减少后的页引用计数，忽略组合页。           |
| `int page_ref_add_unless(struct page *page, int nr, int u)` | 如果页的引用计数不是 `u` 则增加 `nr` 个计数。如果有增加，则返回真。 |
| `int page_ref_freeze(struct page *page, int count)`         | 如果页的引用计数为 `count` 则将计数设置为0，成功则返回真。   |
| `void page_ref_unfreeze(struct page *page, int count)`      | `page_ref_freeze()` 的反操作。                               |

1. 其他页辅助函数

| func                                                     | comments                                                     |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| `void split_page(struct page *page, unsigned int order)` | 将 2^order 连续的页全部拆开，按单页使用。执行了此函数后，你只能使用 `put_page()` 或 `__free_page()` 按单页释放。 |
| `enum zone_type page_zonenum(const struct page *page)`   | 返回页所在的zone区域的枚举值。                               |
| `int page_zone_id(struct page *page)`                    | 返回页所在的zone区域的整数值。                               |
| `int page_to_nid(const struct page *page)`               | 返回页所在的node节点值。                                     |
| `struct zone *page_zone(const struct page *page)`        | 返回页所在的zone区域对象指针。                               |
| `pg_data_t *page_pgdat(const struct page *page)`         | 返回页所在的node节点对象指针。                               |
| `void *page_address(const struct page *page)`            | 返回页对应的虚地址，如果是高端页返回的是固定映射的区域（32位才有），否则返回直接映射的虚地址。 |
| `struct page *compound_head(struct page *page)`          | 获取组合页的头页。                                           |
| `int PageMappingFlags(struct page *page)`                | 是否有映射额外信息。                                         |
| `int PageAnon(struct page *page)`                        | 是否是匿名页。                                               |

1. 页的标志

页表示说明

| flags             | comments                                                     |
| ----------------- | ------------------------------------------------------------ |
| `PG_locked`       | 页被锁住，页在多个数据结构中传递时使用。比如 回写与回收等。  |
| `PG_error`        | 页在I/O传输中发生了错误。                                    |
| `PG_referenced`   | 页最近被引用过。 比如换入、缺页新分配、引用文件高速缓存的页等。 |
| `PG_uptodate`     | 用于描述有后备存储的页（交换页、文件页）中数据更新了（有效页）。比如异步的从磁盘加载完数据后，设置此标志。 |
| `PG_dirty`        | 页中的数据被改写了。文件页会根据此标志回写磁盘。             |
| `PG_lru`          | 页在lru链表中。                                              |
| `PG_active`       | 活动页，最近操作过。与 `PG_referenced` 配合使用，描述页是否不应该被回收。转移到活动lru中时设置。 |
| `PG_slab`         | 页用于 `slab` 分配。此时 `page->lru` 存储了 `kmem_cache` 和 `slab` 描述符。 |
| `PG_reserved`     | 页是保留的，不要写此类页，可能用于存储指令数据。比如存储内核镜像使用的页。 |
| `PG_private`      | 文件系统使用，表示 `page->private` 字段有有效数据。          |
| `PG_private_2`    | 类似 `PG_private` ，但用于描述不同子系统使用的数据。         |
| `PG_writeback`    | 页正在被回写，开始回写交换页、文件脏页时标记。               |
| `PG_head`         | 表示此页是组合页的头页。                                     |
| `PG_swapcache`    | 表示此页正处于交换缓存中。如果此时页的引用计数小于等于2，页可能可以被回收掉。 |
| `PG_mappedtodisk` | 页的数据是磁盘的映射。                                       |
| `PG_reclaim`      | 页正在被回收，配合 `PG_writeback` 使用。                     |
| `PG_swapbacked`   | 页有在交换文件或其他页的备份。                               |
| `PG_unevictable`  | 页不参数回收机制。不会在活动或非活动lru中出现。              |
| `PG_mlocked`      | 页用于`mmap()`的 `MAP_LOCKED`，不参与缺页机制和回收机制。    |
| `PG_uncached`     | 页已经被作为非缓存映射。？                                   |
| `PG_hwpoison`     | 页用于内存校验，不要动此类的页，否则内核会崩溃。             |
| `PG_fscache`      | 网络文件系统使用。                                           |
| `PG_pinned`       | Xen 架构使用，这些页是只读的。                               |
| `PG_slob_free`    | 该页是 SLOB 分配器中空闲的页。                               |
| `PG_double_map`   | 组合页被双重映射时，将标记为设置在尾页的标志中。             |

页标志的操作。

很多标志都有 `SetPageXXX()`设置、`PageXXX()`测试、`ClearPageXXX()`清除、`TestSetPageXXX()`（设置返回旧值）、`TestClearPageXXX()`（清除返回旧值）等，`XXX`和上述标志后缀一直，这些都是原子操作，另外可能有一套 `__PageXXX()` 前面以 `__` 开始的非原子操作函数系列，只有在 `PageLocked()`为真 或 持有其他的相关锁 的情况下才可以使用。下标只列出一部分进行说明。

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `PageCompound()`                                             | 页是否是组合页的一部分。                                     |
| `PageTail()`                                                 | 页是否是组合页的尾页（尾页可以是多个）。                     |
| `SetPageLRU()`                                               | 设置 `PG_lru` 标志。                                         |
| `PageLRU()`                                                  | 测试 `PG_lru` 标志，设置了就返回真。                         |
| `ClearPageLRU()`                                             | 清除 `PG_lru` 标志。                                         |
| `__SetPageLocked()`                                          | 设置 `PG_locked` 标志。                                      |
| `TestClearPageReferenced()`                                  | 尝试清除 `PG_referenced` 标志，返回旧标志。返回真表示清除成功，否则失败。 |
| `TestSetPageWriteback()`                                     | 尝试设置 `PG_writeback` 标志，返回旧值标志。返回真表示失败，否则成功。 |
| `PageUptodate()`                                             | 页是否更新，可以处理组合页。组合页的此标志设置在头页里。     |
| `PageBuddy()`                                                | 页在伙伴系统中，返回真，否则返回假。                         |
| `SetPageUptodate()`                                          | 设置更新，不可能处理组合页，你必须传递组合页的头页。         |
| `void page_endio(struct page *page, bool is_write, int err)` | 在IO结束时根据IO结果更新页标志。`is_write` 是否是写页操作， `err` 是否发生传输错误。 |

页标志的同步操作

使用位等待队列进行标志位的同步

> ```
> #include <linxu/pagemap.h>
> ```

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void __lock_page(struct page *page)`                        | 锁住页，如果需要休眠，进程不会被信号中断。可以处理组合页。**在对页的操作不是原子的就需要锁页，特别是操作缓存页相关的时候**。 |
| `int __lock_page_killable(struct page *page)`                | 类似 `__lock_page()`，但可以被 `KILL` 信号中断，此时返回 `-EINTR`。返回 0 表示成功锁住。 |
| `int __lock_page_or_retry(struct page *page, struct mm_struct *mm, unsigned int flags)` | 返回 1，页被锁定，仍然持有信号量，返回 0，页未被锁定，除非 `FAULT_FLAG_ALLOW_RETRY` 和 `FAULT_FLAG_RETRY_NOWAIT` 都被设置，这样仍然持有信号量，否则信号量被释放 |
| `void unlock_page(struct page *page)`                        | 解锁页，唤醒等待此页的进程。                                 |
| `int trylock_page(struct page *page)`                        | 尝试锁页，成功返回真。                                       |
| `void lock_page(struct page *page)`                          | 类似 `__lock_page()`。                                       |
| `int lock_page_killable(struct page *page)`                  | 类似 `__lock_page_killable()`。                              |
| `lock_page_or_retry()`                                       | 类似 `__lock_page_or_retry()`。                              |
| `void wait_on_page_bit(struct page *page, int bit_nr)`       | 等待 页的 `bit_nr` 标志清零。 不会被信号中断等待。           |
| `wait_on_page_bit_killable()`                                | 类似 `wait_on_page_bit()`，但是可以被 `KIL` 信号中断，如果被中断则返回 `-EINTR` ，成功返回 0。 |
| `wait_on_page_bit_killable_timeout()`                        | 类似 `wait_on_page_bit_killable()` 但是只等待 `timeout` 节拍数的时间，如果超时 `-EAGAIN`。 |
| `int wait_on_page_locked_killable(struct page *page)`        | 类似 `wait_on_page_bit_killable()`，等待 `PG_locked` 清零，在等待前测试是否已经加锁。 |
| `void wake_up_page(struct page *page, int bit)`              | 唤醒页上等待 `bit` 标志位的进程，顶多唤醒一个。              |
| `void wait_on_page_locked(struct page *page)`                | 自己持有锁，等待其他路径解锁，页异步操作时使用。             |
| `void wait_on_page_writeback(struct page *page)`             | 自己设置回写标志，等待异步回写完成。                         |
| `void end_page_writeback(struct page *page)`                 | 回写路径唤醒等待回写完成的进程。                             |
| `void wait_for_stable_page(struct page *page)`               | 检查页是否后备（可以回写的页），然后等待回写完成。           |
| `void add_page_wait_queue(struct page *page, wait_queue_t *waiter)` | 使用自己定义的等待实体，等待页的标志改变。                   |

### 非整页内存管理

> ```
> #include <linux/slab/h>
> ```

内核使用高速缓存将小块内存作为“对象”的概念进行管理，速度非常快。对象的最小大小为 `KMALLOC_MIN_SIZE`(一般为 16 bytes），最大为 `KMALLOC_MAX_SIZE`（一般为 2^22 bytes = 4MB）。

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `struct kmem_cache * kmem_cache_create(const char *name, size_t size, size_t align, unsigned long flags, void (*ctor)(void *))` | 创建新的高速缓存并返回描述符指针。可能休眠，所以注意调用的上下文。 1. `name` 对其命名； 2. `size` 每个对象的大小； 3. `align` 是否按某种长度对齐（比如cacheline大小），如果不需要则传0值； 4. `flags` 期望的行为，`SLAB_XXX`的或集合； 5. `ctor` 初始化构造函数，可以为空。 |
| `void kmem_cache_destroy(struct kmem_cache *)`               | 释放所有涉及的内存，并销毁描述符。如果其中有任何一个对象在使用，都会失败。 |
| `int kmem_cache_shrink(struct kmem_cache *cachep)`           | 压缩缓存，释放其中未使用的内存页。                           |
| `void *kmem_cache_alloc(struct kmem_cache *, gfp_t flags)`   | 从缓存中分配一个未使用的对象，返回内核空间的地址指针。`flags`用于在缓存的对象不足时，分配新内存页使用。 |
| `void kmem_cache_free(struct kmem_cache *, void *)`          | 释放对象，先会判定对象是否属于传入的缓存。                   |
| `int kmem_cache_alloc_bulk(struct kmem_cache *, gfp_t, size_t count, void **ptr)` | 批量分配`count`个对象存入 `ptr` 指针数组中，返回分配的个数。调用此函数时，必须打开中断。 |
| `void kmem_cache_free_bulk(struct kmem_cache *, size_t, void **)` | `kmem_cache_alloc_bulk()` 的反操作，调用时也必须打开中断。当不提供缓存描述符时，使用对象本身获取目的缓存，也就是说，**批量释放的对象不必来自同一个缓存**。 |
| `void *kmem_cache_alloc_node(struct kmem_cache *, gfp_t flags, int node)` | 类似 `kmem_cache_alloc()`，但可以分配指定node上的内存。      |
| `void *kmem_cache_zalloc(struct kmem_cache *k, gfp_t flags)` | 类似 `kmem_cache_alloc()`，但分配后清零对象。                |
| `unsigned int kmem_cache_size(struct kmem_cache *s)`         | 返回缓存管理对象的大小。                                     |
| `KMEM_CACHE(__struct, __flags)`                              | 创建一个专用的用于分配 `__struct` 类型的高速缓存。           |

对 SLAB_XXX 描述

| flags                 | comments                                       |
| --------------------- | ---------------------------------------------- |
| `SLAB_HWCACHE_ALIGN`  | 忽略对齐参数，每个对象都是使用缓存线大小对齐。 |
| `SLAB_CACHE_DMA`      | 对象使用的内存都是DMA区域的页。                |
| `SLAB_PANIC`          | 如果创建失败，直接停止内核。                   |
| `SLAB_DESTROY_BY_RCU` | 销毁SLAB及其中的对象时，使用RCU模式。          |
| `SLAB_NOLEAKTRACE`    | 避免内存泄露跟踪。                             |
| `SLAB_NOTRACE`        | 避免内存页跟踪。                               |

为了方便使用，并减少缓存本身的数量，内核预创建了一些通用缓存，通过要分配的大小使用下列函数自动的适配并分配

| func or macro                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void *kmalloc(size_t size, gfp_t flags)`                    | 分配大小为 `size` 的对象并返回，`flags` 用于分配新内存页使用。**该函数在分配大小超过`KMALLOC_MAX_SIZE`时，直接分配整页**。 |
| `void *kzalloc(size_t size, gfp_t flags)`                    | 类似 `kmalloc()`，清理新分配的内存。                         |
| `__kmalloc()`                                                | 类似 `kmalloc()`，但不处理分配大于`KMALLOC_MAX_SIZE`大小的对象。 |
| `void *kmalloc_node(size_t size, gfp_t flags, int node)`     | 类似 `__kmalloc()`，但可以分配指定node的内存。               |
| `__kmalloc_node()`                                           | 类似 `kmalloc_node()`，但当 `size` 参数是字面量时不会提供加速。 |
| `void *kmalloc_order(size_t size, gfp_t flags, unsigned int order)` | 直接分配 `2^order` 个的组合页，返回首页的虚地址。            |
| `void *kmalloc_large(size_t size, gfp_t flags)`              | 对 `kmalloc_order()` 的封装。使用分配大小的去计算（`get_order()`）一个 `order` 值，然后分配组合页。 |
| `void kfree_bulk(size_t size, void **p)`                     | 批量释放 `size` 个 `**p` 指向的指针数组中的对象地址，根据对象地址判断所属缓存。 |
| `void *kmalloc_array(size_t n, size_t size, gfp_t flags)`    | 分配 `n` 个连续的 `size` 大小的对象，返回地址。              |
| `void *kcalloc(size_t n, size_t size, gfp_t flags)`          | 对 `kmalloc_array()` 的封装，分配后清零新分配的内存。        |
| `void kfree(const void *)`                                   | 释放分配的内存。                                             |
| `void kzfree(const void *)`                                  | 在释放内存前清零内存，再释放。                               |
| `size_t ksize(const void *)`                                 | 获取分配对象的大小。                                         |
| `void *krealloc(const void *p, size_t new_size, gfp_t flags)` | 为 `*p` 重新分配一块新大小的内存，如果 `new_size` 为0，则仅释放 `*p`。 |
| `__krealloc()`                                               | 类似 `krealloc()`，但当 `new_size` 为0时，不会释放 `*p` 指向的原对象。 |

### 非连续内存管理

> `#include <linux/vmalloc.h>`
> 不能在NMI中断中使用

在请求内存大于 `2^MAX_ORDER` 页时，内核使用非连续映射单页的形式提供超大内存请求的分配，但是这些分配函数的效率并不高，且不能用于不能休眠的上下文中。

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void *vmalloc(unsigned long size)`                          | 分配 `size` 大小的非连续内存页的空间，并返回内核空间的虚地址。 |
| `zmalloc()`                                                  | 类似 `vmalloc()` ，返回之前清零分配的内存页数据。            |
| `vmalloc_user()`                                             | 类似 `zmalloc()`，但返回虚地址用户空间可以访问？。           |
| `void *vmalloc_node(unsigned long size, int node)`           | 类似 `vmalloc()`，但指定分配node上的页。                     |
| `zmalloc_node()`                                             | 类似 `vmalloc_node()` ，但清零分配的内存页。                 |
| `void *vmalloc_exec(unsigned long size)`                     | 为代码页分配一段内核虚地址空间。                             |
| `void *vmalloc_32(unsigned long size)`                       | 分配DMA32的内存页，然后进行非连续映射。                      |
| `vmalloc_32_user()`                                          | 类似 `vmalloc_32()`，但为用户空间分配。                      |
| `void *__vmalloc(unsigned long size, gfp_t gfp_mask, pgprot_t prot)` | 类似 `vmalloc()`，但在映射页表时，可以使用指定页表项的标志。 |
| `void vfree(const void *addr)`                               | 释放非连续映射的内存。                                       |
| `void *vmap(struct page **pages, unsigned int count, unsigned long flags, pgprot_t prot)` | 使用非连续映射的虚地址空间，将 `pages` 指针数组中的 `count` 个页映射到一块虚地址上。`flags` 是 `VM_XXX` 的或集合，`prot` 用于映射页表时，指定页表项的标志。 |
| `void vunmap(const void *addr)`                              | `vmap()` 的反操作，解除映射。解除映射时对关联的页进行 `__free_pages()` 操作。 |
| `int remap_vmalloc_range_partial(struct vm_area_struct *vma, unsigned long uaddr, void *kaddr, unsigned long size)` | 将 长度为 `size` 的 `kaddr` 内核非连续映射虚地址空间关联的页，**增加引用后**共享的映射到 `vma` 关联的 用户虚地址空间 `uaddr` 起始的位置处。如果原 长度为 `size` 中的 `uaddr` 地址不属于 `vma` 范畴 或 中已经映射了页，则出错返回负数错误码，否则成功返回0。 |
| `int remap_vmalloc_range(struct vm_area_struct *vma, void *addr, unsigned long pgoff)` | 对 `remap_vmalloc_range_partial()` 的封装，要重映射的用户空间和长度由 `vma` 的起止地址觉得。 |

对 VM_XXX 的描述

| flags         | comments                                     |
| ------------- | -------------------------------------------- |
| `VM_ALLOC`    | 新分配页用于映射。                           |
| `VM_MAP`      | 已分配页用于映射。                           |
| `VM_IOREMAP`  | 使用硬件的内存用于映射。                     |
| `VM_NO_GUARD` | 每个非连续映射之间不插入 4K 保护虚地址空隙。 |

## 时间管理

### 节拍软定时器

> 1. `#include <linux/timer.h>`
> 2. 定时器软中断执行的，所以在回调执行期间不会发生抢占，当然也不能主动调度自己。
> 3. 该类定时器使用节拍作为定时单位

初始化

| func or macro                                  | comments                                                     |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `__TIMER_INITIALIZER(function, expires, data)` | 静态初始化定时器。 1. `function` 是定时器的回调， 2. `data` 回调的参数， 3. `expires` 触发时间，未来的某个绝对时间节拍数（使用 `jiffies + e` 得到），普通定时器是 |
| `DEFINE_TIMER(name, function, expires, data)`  | 定义名为 `name` 的定时器。参数见 `__TIMER_INITIALIZER()` 的说明。 |
| `timer_setup(timer, fn, flags)`                | 动态初始化或重新设置定时器的回调和回调参数。 `flags` 见下表标志，可以是或运算的组合值。 |
| `timer_setup_on_stack()`                       | 类似 `timer_setup()`，但用于设置栈上定义的定时器，另外这个系列也有带其他的标志操作定时器的宏，就不一一列举了。 |
| `from_timer()`                                 | 获取`timer_list`变量的包裹结构体的指针。                     |

定时器标志

| flags              | comments                                                     |
| ------------------ | ------------------------------------------------------------ |
| `TIMER_DEFERRABLE` | 定时器是可延迟的，也就是说这种定时器在CPU处于idle状态时，即使到期也是不调度的，除非CPU状态改变或有一个非可延迟定时器已经到期。 |
| `TIMER_PINNED`     | 如果没有设置此标志，内核将定时器分发给最近在忙的CPU定时器队列进行计时，否则使用当前或指定的CPU进行处理。 |
| `TIMER_IRQSAFE`    | 定时器的回调是中断安全的，这样定时器子系统在调用回调时，不会禁用中断。 |

定时器操作

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `int timer_pending(const struct timer_list * timer)`         | 如果定时器还没有启动则返回真。                               |
| `int del_timer(struct timer_list *timer)`                    | 取消定时器激活。如果此定时器已经激活，则返回真，否则返回假。不处理定时器正在被执行的情况。 |
| `int mod_timer(struct timer_list *timer, unsigned long expires)` | 多个非同步的使用者可以同时安全修改定时器的超时时间。 如果定时器尚未激活，则激活并返回真，否则返回假。 |
| `int mod_timer_pending(struct timer_list *timer, unsigned long expires)` | 类似 `mod_timer()`，但如果定时器当前没有激活，则不激活，返回假。 |
| `int timer_reduce(struct timer_list*, unsigned long newexpires)` | 可以减少以`pending`状态的过期时间？                          |
| `void add_timer(struct timer_list *timer)`                   | 在当前CPU上激活定时器，**必须保证定时器尚未激活**。          |
| `void add_timer_on(struct timer_list *timer, int cpu)`       | 在期望的CPU上激活定时器，必须保证定时器尚未激活且回调函数已经设置。 |
| `int try_to_del_timer_sync(struct timer_list *timer)`        | 试图同步取消定时器激活。如果定时器正在被执行，失败返回负数；否则成功取消返回非负数。 |
| `int del_timer_sync(struct timer_list *timer)`               | 同步取消激活定时器，类似 `try_del_timer_sync()` ，但等待直到取消成功为止。如果定时器不是 `TIMER_IRQSAFE` 属性，则不应该在中断上下文使用。 |
| `del_singleshot_timer_sync()`                                | 基本等效 `del_timer_sync()`，但有些版本被实现为，如果此定时器从来在回调中不激活自身，使用该函数会比 `del_timer_sync()` 更效率。 |

### 高精度定时器

> 1. `#include <linux/hrtimer.h>`
> 2. 同样是使用软中断实现，但默认上下文不是软中断。

初始化

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `hrtimer_init(struct hrtimer *timer, clockid_t which_clock, enum hrtimer_mode mode)` | 以某种时钟和模式初始化定时器。 1. 时钟类型为 `CLOCK_XXXX`，比如 `CLOCK_MONOTONIC` 单调时钟，不会因为系统时间改变而影响定时器的触发。 2. 定时器模式为 `HRTIMER_MODE_XXX`，比如 `HRTIMER_MODE_REL` 相对当前时间来触发定时器，当然还有绝对时间，是在未来某个绝对时间触发。 |

模式说明

| mode                  | comments                                                     |
| --------------------- | ------------------------------------------------------------ |
| `HRTIMER_MODE_REL`    | 触发时间相对现在的时间间隔。                                 |
| `HRTIMER_MODE_ABS`    | 触发时间在某个未来时间点。                                   |
| `HRTIMER_MODE_PINNED` | 再启动时绑定在当前cpu上，以后固定在此cpu上触发。可以与`XXX_REL` 和 `XXX_ABS` 或运算共同使用。 |
| `HRTIMER_MODE_SOFT`   | 触发时的上下文将是软中断上下文。高精度定时器默认执行上下文为进程上下文。 |

定时器操作

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `hrtimer_start(hrtimer, expire, expire_mode)`                | 启动定时器，使用 `expire_mode` 来说明 `expire` 这个时间是何种时间（相对还是绝对） |
| `hrtimer_cancel(hrtimer)`                                    | 同步的取消定时器，如果已触发，则等待其回调执行完毕。 返回0表示定时器还没有激活，否则表示定时器已激活。 |
| `hrtimer_try_cancel(hrtimer)`                                | 异步取消定时器，不会等待完成。 返回0表示定时器还没有激活，返回1表示定时器已激活，返回-1表示已触发，并且回调正在执行。 |
| `hrtimer_active(hrtimer)`                                    | 判断是否激活，当定时器已排队、回调正在被执行或定时器正在被迁移到另一CPU，则返回真，表示已经激活，否则返回假。 |
| `hrtimer_is_queued(hrtimer)`                                 | 是否已排队。 返回真假值。                                    |
| `hrtimer_callback_running(hrtimer)`                          | 回调是否正在执行。                                           |
| `hrtimer_forward_now(hrtimer, interval)`                     | 将定时器的过期值向前移动（修改过期值），一般在回调中使用。   |
| `hrtimer_start_expires(hrtimer, mode)`                       | 使用 `hrtimer_set_expires_xxx()` 系列函数设置过期时间后，启动定期器。 `mode` 用于说明已设置的过时间是绝对时间还是相对时间。 |
| `hrtimer_set_expires_xxx(timer, ktime, [delta])`             | 该系列函数用于裸设置过期时间，`ktime` 是主时间，`delta` 是副时间，两者按给定的单位相加得到最终的过期时间。该函数配合 `hrtimer_start_expires()` 使用。 |
| `enum hrtimer_restart hrtimer_callback(struct hrtimer *timer)` | 回调函数必须手动设定 `timer.function = hrtimer_callback`。 回调函数的返回值 ： 1. `HRTIMER_NORESTART` 不重启定时器。 2. `HRTIMER_RESTART` 重启定时器（回调结束后自动排队），在此之前应当修改过期时间（一般都使用 `hrtimer_forward_now()` ），否则定时器又被立即触发。 |
| `hrtimer_cb_get_time(struct hrtimer*)`                       | 根据定时器的时钟模型获取当前时钟，比如以单调时间为模型的定时器，就获取一个单调时间，单位都是纳秒。 |

### 节拍值操作

> 1. `#include <linux/timer.h>`
> 2. `#include <linux/jiffies.h>`

获取节拍和相关宏

| var or macro            | comments                                                     |
| ----------------------- | ------------------------------------------------------------ |
| `unsigned long jiffies` | 变量，机器字长的节拍时间。                                   |
| `u64 jiffies_64`        | 变量，64位的节拍时间，64位内核中与 `jiffies` 相等。          |
| `u64 get_jiffies_64()`  | 获取 64位的节拍时间，因为32位内核中，64位的读操作不是原子的。 |
| `SHIFT_HZ`              | HZ数对应的左移位数                                           |
| `TICK_NSEC`             | 1节拍等于多少个纳秒                                          |
| `TICK_USEC`             | 1节拍等于多少个微妙                                          |
| `LATCH`                 | 1节拍触发多少个PIT时钟信号，一般驱动才用到。                 |
| `MAX_JIFFY_OFFSET`      | 节拍数最大偏移。                                             |
| `HZ`                    | 每秒的节拍数。                                               |

节拍舍入

| func                                                      | comments                                                     |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `unsigned long __round_jiffies(unsigned long j, int cpu)` | 向上或向下舍入 绝对时间 `j` 节拍数到等价整秒的节拍数，然后返回（时间点）。 这一般使用在PerCPU定时器的定时上，`cpu` 可以使舍入根据不同CPU偏移一些，防止所有CPU同时触发定时器。 |
| `__round_jiffies_relative()`                              | 类似 `__round_jiffies()`，但作用于相对时间，返回相对时间（一段时间）。 |
| `unsigned long round_jiffies(unsigned long j)`            | 类似 `__round_jiffies()`，但 `cpu` 就是本地CPU。             |
| `round_jiffies_relative()`                                | 类似 `__round_jiffies_relative()`，但 `cpu` 就是本地CPU。    |
| `__round_jiffies_up(unsigned long j, int cpu)`            | 类似 `__round_jiffies()`，但强制向上舍入时间。               |
| `__round_jiffies_up_relative()`                           | 类似 `__round_jiffies_relative()`，但强制向上舍入时间。      |
| `round_jiffies_up()`                                      | 类似 `round_jiffies()`，但强制向上舍入时间。                 |
| `round_jiffies_up_relative()`                             | 类似 `round_jiffies_relative()`，但强制向上舍入时间。        |

节拍比较

| func                        | comments                                                     |
| --------------------------- | ------------------------------------------------------------ |
| `time_after(a, b)`          | 等价 `a > b`，但是可以处理节拍回绕的情况。读着 `a` 快于 `b`。对于64位的值使用 `time_after64()`。 |
| `time_before(a, b)`         | 类似 `time_after()`，等价 `a < b`。读着 `a` 慢于 `b`。对于64位的值使用 `time_before64()`。 |
| `time_after_eq(a, b)`       | 等价 `a >= b`。对于64位的值使用 `time_after_eq64()`。        |
| `time_before_eq(a, b)`      | 等价 `a <= b`。对于 64位的值使用 `time_before_eq64()`。      |
| `time_in_range(a,b,c)`      | 判断 `a` 是否在一个闭区间 `[b, c]` 之间，等价 `b <= a <= c`。对于64位的值使用`time_in_range64()`。 |
| `time_in_range_open(a,b,c)` | 判断 `a` 是否在一个开区间 `[b, c)` 之间，等价 `b <= a < c`。 |
| `time_is_before_jiffies(a)` | 等价 `time_after(jiffies, a)`，如果 `a` 是一个过去时间，则返回真。 64位仍然使用64后缀的宏，另外也有一套判断一个绝对时间是否是未来时间、 是否是过去或现在时间等等，就不一一列出了。 |

节拍转换

| func or macro                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `unsigned int jiffies_to_msecs(const unsigned long j)`       | 将节拍数转换成毫秒数，然后返回。                             |
| `jiffies_to_usecs()`                                         | 转换成微秒数。                                               |
| `u64 jiffies_to_nsecs(const unsigned long j)`                | 转换成纳秒数，纳秒数很大所以用64位整数表示。                 |
| `unsigned long __msecs_to_jiffies(const unsigned int m)`     | 将毫秒数转换为节拍数，如果 `(int) m < 0` 则返回 `MAX_JIFFY_OFFSET`。 |
| `_msecs_to_jiffies()`                                        | 类似 `__msecs_to_jiffies()` ，但不处理等价负值的毫秒数。     |
| `msecs_to_jiffies()`                                         | 类似 `__msecs_to_jiffies()` ，但对常量值的参数进行了优化。   |
| `usecs_to_jiffies()`                                         | 转换微秒数到节拍数，同样有一套`_`和`__`前缀的函数和宏，就不一一列出了。 |
| `unsigned long timespec_to_jiffies(const struct timespec *value)` | 将纳秒级系统时间转换为节拍数。64位系统时间使用 `timespec64_to_jiffies()`。 |
| `void jiffies_to_timespec(const unsigned long jiffies, struct timespec *value)` | 将节拍数转换为纳秒级系统时间。64位系统时间使用 `jiffies_to_timespec64()`。 |
| `unsigned long timeval_to_jiffies(const struct timeval *value)` | 将微秒级系统时间转换为节拍数。                               |
| `void jiffies_to_timeval(const unsigned long jiffies, struct timeval *value)` | 将节拍数转换为微秒级系统时间。                               |
| `clock_t jiffies_to_clock_t(unsigned long x)`                | 将节拍数转换为墙上时钟。                                     |
| `unsigned long clock_t_to_jiffies(unsigned long x)`          | 将墙上时钟转换为节拍数。                                     |
| `u64 jiffies_64_to_clock_t(u64 x)`                           | 将64位的节拍数转换为墙上时钟。                               |

系统时间操作

> 1. `#include <linux/time.h>`

| func                                                         | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `int timespec_equal(const struct timespec *a, const struct timespec *b)` | 比较时间是否相等，相等返回真。                               |
| `timespec_compare(x, y)`                                     | 比较操作，`x < y` 返回 `-1`；`x == y` 返回 `0`； `x > y` 返回 `1`。 |
| `timeval_compare(x, y)`                                      | 类似 `timespec_compare()` ，但比较 `timeval` 时间。          |
| `time64_t mktime64(year, mon, day, hour, min, sec)`          | 使用年月日时分秒构造一个 `time64_t` 时间（1970年开始的秒数）。 |
| `mktime()`                                                   | 类似 `mktime64()`，但返回 `unsigned long` 时间 （1970年开始的秒数）。 |
| `void set_normalized_timespec(struct timespec *ts, time_t sec, s64 nsec)` | 使用 `sec` 和 `nsec`（可以是负值，则减少 `sec` 的值）构造 `ts`。 |
| `struct timespec timespec_add_safe(struct timespec lhs, struct timespec rhs)` | 将 `lhs` 和 `rhs` 的值相加返回。可以检查溢出。               |
| `timespec_add()`                                             | 类似 `timespece_add_safe()` ，但不做检查溢出。               |
| `timespec_sub(x, y)`                                         | 类似 `timespec_add()` ，但是做 `x - y` 运算。                |
| `timespec_valid(const struct timespec *ts)`                  | 检查 `ts` 是否有效（规范），有效返回真。                     |
| `timespec_valid_strict()`                                    | 类似 `timespec_valid()` ，但进行更严格的检查。               |
| `bool timeval_valid(const struct timeval *tv)`               | 类似 `timespec_valid()` ，对 `timeval()` 的类型进行检查。    |
| `struct timespec timespec_trunc(struct timespec t, unsigned gran)` | 对 `t` 的纳秒部分裁剪 `gran`（ `0 < gran <= 10^9` ） 纳秒，然后返回新的时间。`gran` 为 1 时不做处理。 |
| `bool timeval_inject_offset_valid(const struct timeval *tv)` | 判断 `tv` 的微秒部分是否有效，有效返回真。                   |
| `timespec_inject_offset_valid()`                             | 判断 `timespec` 的纳秒部分是否有效，有效返回真。             |
| `void time64_to_tm(time64_t totalsecs, int offset, struct tm *result)` | 将 `totalsecs + offset` 的从 1970 年开始的秒数转换为 `tm` 结构。 |
| `time_to_tm()`                                               | 类似 `time64_to_tm()`，但是左右 `time_t` 类型上。            |
| `s64 timespec_to_ns(const struct timespec *ts)`              | 将 `timespec` 结构 转换为 `ns`。                             |
| `s64 timeval_to_ns(const struct timeval *tv)`                | 将 `timeval` 结构 转换为 `ns`。                              |
| `struct timespec ns_to_timespec(const s64 nsec)`             | 将 `ns` 转换为 `timespec` 结构。                             |
| `struct timeval ns_to_timeval(const s64 nsec)`               | 将 `ns` 转换为 `timeval` 结构。                              |
| `void timespec_add_ns(struct timespec *a, u64 ns)`           | 将 `ns` 加在 `timespec` 结构上。                             |

内核时间获取

> 1. `#include <linux/ktime.h>`

| func                                            | comments                                                     |
| ----------------------------------------------- | ------------------------------------------------------------ |
| `current_kernel_time()`                         | 获取系统时间，以 `timespec64` 结构返回。                     |
| `cycles_t get_cycles()`                         | 获取当前`tsc`指令周期。                                      |
| `get_seconds()`                                 | 获取当前真实时钟(从`1970-01-01`开始的秒数?)时间，一般文件是系统的时间戳使用。 |
| `void ktime_get_ts64(struct timespec64 *ts)`    | 获取单调时钟（真实时钟和墙上单调偏移）存入 `timespec64` 结构中。 |
| `void getrawmonotonic64(struct timespec64 *ts)` | 获取单调时间（流失的秒数）存入 `timespec64` 结构中。         |

延迟休眠函数

> 1. `#include <linux/delay.h>`

| func or macro                                             | comments                                                     |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `mdelay(n)`                                               | 忙等待 `n` 个毫秒。                                          |
| `udelay(n)`                                               | 忙等待 `n` 个微秒。                                          |
| `ndelay(n)`                                               | 忙等待 `n` 个纳秒。                                          |
| `void msleep(unsigned int msecs)`                         | 毫秒级休眠（调度出），不可被中断。                           |
| `unsigned long msleep_interruptible(unsigned int msecs)`  | 毫秒级休眠，可以被中断。如果被中断，返回剩余的毫秒数。       |
| `void usleep_range(unsigned long min, unsigned long max)` | 在 `[min, max]` 微秒范围内做一个不确切的休眠，可能被调度。 这个函数比起 `udelay()` 可以提高系统的响应能力。 |

## 进程虚地址管理

### 进程空间读写辅助函数

> 1. `#include <asm/uaccess.h>`
> 2. 对用户空间进行操作都可能触发缺页操作。

下列函数都使用在用户进程上下文中。

| func or macro                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `access_ok(type, addr, size)`                                | 对 长度为 `size` 的用户空间地址 `addr` 进行 `type` 类型的（读：`VERIFY_READ` 、写：`VERIFY_WRITE`）检查，如果可以读写则返回真。如果可写，则一定可读。**再进行下列读写之前必须调用此检查函数** |
| `get_user(x, ptr)`                                           | 从 用户空间地址 `ptr` 中获取简单类型（拷贝字节数同 `ptr` 指针类型一致）的值，存入 `x` 简单类型变量中（非指针）。成功返回 0，否则返回 `-EFAULT`。如果出错，`x` 被清零。可能发生缺页而休眠。 |
| `__get_user(x, ptr)`                                         | 类似 `get_user()`，但不进行检查。                            |
| `put_user(x, ptr)`                                           | 将 `x` 简单类型变量中的值 存入 用户空间简单类型的地址 `ptr` 中（拷贝字节数同 `ptr` 指针类型一致）。成功返回 0，否则返回 `-EFAULT`。可能发生缺页而休眠。 |
| `__put_user(x, ptr)`                                         | 类似 `put_user()`，但不进行检查。                            |
| `int fault_in_pages_writeable(char __user *uaddr, int size)` | 确认 长度为 `size` 的用户空间的地址 `uaddr` （页对齐）是否可写。可以返回0 ，否则返回负数的错误码。 使用了 `put_user()`，所以可能缺页休眠。 |
| `fault_in_pages_readable()`                                  | 类似 `fault_in_pages_writeable()`，但进行可读确认。使用了 `get_user()`，所以可能缺页休眠。 |
| `get_user_try { get_user_ex(x, ptr); ... } get_user_catch(err)` | 异常读用户空间简单类型的套件。如果出错，将终止读之后的流程，并将错误值存入 `err` 变量中。 |
| `put_user_try { put_user_ex(x, ptr); ... } put_user_catch(err)` | 异常写用户空间简单类型的套件。如果出错，将终止写之后的流程，并将错误值存入 `err` 变量中。 |
| `long strncpy_from_user(char *dst, const char __user *src, long count)` | 指定拷贝长度的字符串拷贝函数。将用户空间的字符串拷贝到内核空间中。成功返回拷贝长度，失败返回 `-EFAULT`。 |
| `long strlen_user(const char __user *str)`                   | 获取用户空间 `str` 的长度（包含末尾的`\0`字符。）            |
| `long strnlen_user(const char __user *str, long n)`          | 类似 `strlen_user()`，最多获取 长度为`n` 的用户空间 `str` 的长度。 |
| `clear_user(void __user *mem, unsigned long len)`            | 清零 长度 `len` 的用户空间内存 `mem`。                       |
| `__clear_user()`                                             | 类似 `clear_user()`，但不进行写检查。                        |
| `int user_atomic_cmpxchg_inatomic(uptr, ptr, old, new)`      | 原子比较交换用户空间的基础类型值，未出错返回0，出错返回 `-EFAULT`。将 `*ptr`（用户空间的地址） 与 `old` 进行比较，相等则使用 `new` 替换，并将交换前的值存入 `uptr` 中（通常是局部变量的指针）。 |
| `unsigned long copy_from_user(void *to, const void __user *from, unsigned long n)` | 从用户空间地址 `from` 拷贝 `n` 个字节到 内核空间地址 `to` 中。未出错返回0，出错返回未拷贝的字节数。进行了相关检查，并如果出错打印一些警告信息。 |
| `unsigned long copy_to_user(void __user *to, const void *from, unsigned long n)` | 类似 `copy_from_user()`，从内核空间拷贝到用户空间。          |
| `_copy_from_user()`                                          | 类似 `copy_from_user()`，但不打印警告信息。                  |
| `__copy_form_user_nocheck()`                                 | 类似 `copy_from_user()`，但不进行相关检查，也不打印警告信息。 |
| `__copy_from_user_inatomic()`                                | 类似 `copy_from_user()`，但不打印警告信息。配合 `kmap_atomic()` 在原子上下文使用。 |
| `_copy_to_user()`                                            | 类似 `copy_to_user()`，但不打印警告信息。                    |
| `__copy_to_user_nocheck()`                                   | 类似 `copy_to_user()`，但不进行相关检查，也不打印警告信息。  |
| `__copy_to_user_inatomic()`                                  | 类似 `copy_to_user()`，但不打印警告信息。配合 `kmap_atomic()` 在原子上下文使用。 |
| `unsigned long copy_user_generic(void *to, const void *from, unsigned len)` | 用户空间的两个地址的内容进行拷贝。未出错返回0，出错返回未拷贝的字节数。 |
| `copy_in_user()`                                             | 对 `copy_user_generic()` 的封装，进行了地址读写检查（`access_ok()`）。 |
| `__copy_in_user()`                                           | 对 `copy_in_user()` 的封装，但在操作基本类型（小于等于机器字长度）的长度时，效率更高。 |
| `int __copy_from_user_nocache(void *dst, const void __user *src, unsigned size)` | 将用户空间的内容拷贝到内核空间，类似 `copy_from_user()`，但不会将内容缓存在cacheline中。 |
| `__copy_to_user_nocache()`                                   | 类似 `copy_to_user()`。                                      |

## 进程管理与调度

### 进程运行状态

进程状态是互斥的，存放在 `task_struct.state` 中。

| macro                  | comments                                                     |
| ---------------------- | ------------------------------------------------------------ |
| `TASK_RUNNING`         | 进程正在执行或即将被执行。调度时，这种状态的进程将停留在运行队列里；如果内核要抢占调度一个进程，也只会考虑这种状态的进程。 |
| `TASK_INTERRUPTIBLE`   | 进程被休眠挂起，直到被主动唤醒或收到信号被唤醒。被调度时，这种状态的进程将被从运行队列里移除。 |
| `TASK_UNINTERRUPTIBLE` | 与 `TASK_INTERRUPTIBLE` 类似，但是不会被任何事件中断，直到被主动唤醒。 |
| `TASK_STOP`            | 进程被暂停，当收到 `SIGSTOP` 、 `SIGTSTP` 、 `SIGTTIN` 或 `SIGTTOU` 信号后进入这种状态，收到 `SIGCONT` 信号后恢复到 `TASK_RUNNING` 。 |
| `TASK_TRACE`           | 进程有 `debugger` 程序暂停，但被监控的时，任何信号都可以将进程置于这种状态。 |
| `EXIT_ZOMBIE`          | 进程已经被终止，但是父进程还没有调用 `wait()` 系统调用去回收其状态。 |
| `EXIT_DEAD`            | 进程已经死亡，而且父进程发出了 `wait()`，进程被内核删除。    |
| `TASK_PARKED`          | 内核线程被暂定，使用`kthread_park()`/`kthread_unpark()`操作该状态，除非解除这个状态，否则即便被唤醒，内核线程也不会被运行。 |

### 进程状态

| macro               | comments                                                     |
| ------------------- | ------------------------------------------------------------ |
| `PF_EXITING`        | 进程正在退出，刚接收到退出指令。                             |
| `PF_EXITPIDONE`     | 进程退出完成，将进程运行状态设置为 `TASK_UNINTERRUPTIBLE` ，然后调度出去，等待调度子系统回收。 |
| `PF_NOFREEZE`       | 进程不会被冻结，比如创建内核线程的辅助内核线程。             |
| `PF_NO_SETAFFINITY` | 不允许用户在设置CPU亲和性。                                  |

### 线程状态

每个进程有一个线程信息，存储在 `thread_info` 中，线程状态用于调度方面。

| macro                  | comments                                    |
| ---------------------- | ------------------------------------------- |
| `TIF_SYSCALL_TRACE`    | 正在跟踪系统调用                            |
| `TIF_NOTIFY_RESUME`    | x86不使用                                   |
| `TIF_SIGPENDING`       | 进程有信号挂起                              |
| `TIF_NEED_RESCHED`     | 必须执行调度程序，在异常和中断返回时会检查  |
| `TIF_SINGLESTEP`       | 临时返回用户态之前恢复单步执行              |
| `TIF_IRET`             | 通过iret，而不是sysexit从系统调用中强行返回 |
| `TIF_SYSCALL_AUDIT`    | 系统调用正在被审计                          |
| `TIF_POLLING_NRFLAG`   | 空闲进程正在轮询 `TIF_NEED_RESCHED` 标志    |
| `TIF_MEMDIE`           | 正在撤销进程，以回收内存（`shrink_list()`)  |
| `TIF_LAZY_MMU_UPDATES` | 正在进行惰性tlb更新                         |

### 进程组织

> 1. `#include <linux/sched.h>`
> 2. `#include <linux/pid.h>`

| macro or func                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `set_task_state(task, state)`                                | 设置指定进程的状态。                                         |
| `set_current_state(state)`                                   | 设置当前进程的状态。                                         |
| `current_thread_info()`                                      | 获取进程线程信息，调度状态。                                 |
| `current`                                                    | 指向当前运行的进程描述符。                                   |
| `get_current()`                                              | 等价 `current` 。                                            |
| `do_each_pid_task(pid, type, task) { ... } while_each_pid_task(pid, type, task);` | 通过局部变量 `task`，遍历指定的 `struct pid*` 中的 `enum pid_type` 类型的非线程组成员。 在遍历线程组成员的非 `PIDTYPE_PID` 时，要使用首进程的 `struct pid*` 。 |
| `do_each_pid_thread(pid, type, task) { ... } while_each_pid_thread(pid, type, task);` | 双循环结构，遍历指定 `pid` 中指定类型 `type` 的所有非线程组成员的线程成员，包括首进程。 |
| `for_each_process(p) { ... }`                                | 使用一个局部变量 `struct task_struct *p` 作为迭代器，遍历内核中所有的进程。遍历时使用 `rcu_read_lock()` 锁。 |
| `do_each_thread(g, t) { ... } while_each_thread(g, t);`      | 使用两个局部变量作为迭代器，遍历内核中所有进程的线程组子成员，`g` 表示每个线程组的首进程。`t` 表示 `g` 线程组中的子成员，当然包括 `p` 自身（第一个 `t` 就是 `p`）。 遍历时使用 `read_lock(&tasklist_lock)` 锁，且这时一个双循环，`break` 不会按预期执行，请使用 `goto ...` 代替。 |
| `while_each_thread(g, t) { ... }`                            | 遍历 `g` 进程所在线程组中的其他成员，循环不包含 `g` ，使用 `t` 作为迭代器。 |
| `for_each_process_thread(p, t) { ... }`                      | 与 `do_each_thread()` 类似，但是线程组成员不处理信号的话，遍历时不会出现。 |
| `for_each_thread(p, t) { ... }`                              | `for_each_process_thread()` 的辅助宏，通过指定的 `p` 进程和局部变量 `t`，遍历线程组中的能处理信号的线程，包括首进程。 |
| `bool thread_group_leader(struct task_struct *p)`            | 判断是否是线程组首进程，线程组成员进程在退出时不发出 `SIGCHLD` 信号。 |
| `int get_nr_threads(struct task_struct *tsk)`                | 获取所在线程组的线程数量，包括线程组首进程。                 |
| `bool thread_group_leader(struct task_struct *p)`            | 判断指定进程是否是线程组或进程组的首进程。                   |
| `has_group_leader_pid()`                                     | 类似 `thread_group_leader()` 。                              |
| `bool same_thread_group(struct task_struct *p1, struct task_struct *p2)` | 是否同属于一个线程组。                                       |
| `struct task_struct *next_thread(const struct task_struct *p)` | 返回指定进程在自己所在进程组中的下一个线程的进程描述符。如果没有其他线程，则返回描述符本身。 |
| `int thread_group_empty(struct task_struct *p)`              | 线程组中是否有子线程。                                       |
| `struct pid *task_pid(struct task_struct *task)`             | 获取进程本身的 `struct pid`。                                |
| `struct pid *task_tgid(struct task_struct *task)`            | 获取进程所属的线程组的首进程的 `struct pid`。                |
| `struct pid *task_pgrp(struct task_struct *task)`            | 获取进程所属的进程组的首进程的 `struct pid`。可能和 `sys_setpgid()` 系统调用参数竞争，使用时需要 `tasklist` 或 `rcu` 锁。 |
| `struct pid *task_session(struct task_struct *task)`         | 获取进程所属的会话组的首进程的 `struct pid`。使用和 `task_pgrp()` 类似，与 `sys_setsid()` 参数竞争。 |
| `pid_t pid_nr(struct pid *pid)`                              | 将 `struct pid` 转换成全局可见的 `PID` 进程号，成功返回大于0的值。 |
| `pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)` | 返回 指定命名空间中的 `PID`，不能越级查看上级命名空间中的信息。 |
| `pid_t pid_vnr(struct pid *pid)`                             | 获取当前命名空间中的 `PID`。                                 |
| `int pid_alive(const struct task_struct *p)`                 | 判断进程是否存活，不处于僵死状态。                           |
| `struct pid_namespace *task_active_pid_ns(struct task_struct *tsk)` | 获取进程所在的命名空间。                                     |
| `pid_t __task_pid_nr_ns(struct task_struct *task, enum pid_type type, struct pid_namespace *ns)` | 查找给定进程的指定类型和命名空间的 `PID` 进程号，如果 `ns` 为 `NULL` 则使用当前进程的命名空间。 `enum pid_type` 取 `PIDTYPE_PID` 进程本身、 `PIDTYPE_PGID` 所在进程组、 `PIDTYPE_SID` 所在会话组。 该函数会加 `rcu` 锁后再操作。 |
| `task_pid_nr()`                                              | 获取全局的 `PID` 。                                          |
| `task_tgid_nr()`                                             | 获取全局的 线程组首进程的 `PID` 。                           |
| `struct pid *find_vpid(int nr)`                              | 通过 `PID` 查找当前命名空间中的 `struct pid*`。              |
| `struct pid *find_pid_ns(int nr, struct pid_namespace *ns)`  | 类似 `find_vpid()` ，但可以指定 命名空间。                   |
| `struct pid *find_get_pid(pid_t nr)`                         | 类似 `find_vpid()` 但增加了 `struct pid*` 的引用，且操作时进行的 `rcu` 加锁。 |
| `struct task_struct *pid_task(struct pid *pid, enum pid_type type)` | 查看 `struct pid` 内对应 `type` 的链表中的第一个成员。`PIDTYPE_PID` 链表只有一个成员，就是自身。 |
| `struct task_struct *get_pid_task(struct pid *pid, enum pid_type type)` | 对 `pid_task()` 的封装，如果第一个成员存在，增加其引用计数，然后再返回。 且会加 `rcu` 锁后再操作。 |
| `struct pid *get_task_pid(struct task_struct *task, enum pid_type type)` | `get_pid_task()` 的反操作，如果不是得到进程自身的 `struct pid*`，且进程是线程组的非首进程成员，则获取首进程所在的进程组。 |
| `get_task_struct()`                                          | 增加进程引用。                                               |
| `put_task_struct()`                                          | 减少进程引用。                                               |
| `init_task_pid(struct task_struct *, enum pid_type, struct pid *)` | 初始化进程所属指定进程 `pid_type` 的 `struct pid*`。         |
| `attach_pid(struct task_struct *, enum pid_type)`            | 将进程关联到指定进程 `pid_type` 的 `struct pid*` 的进程组或会话组链表上。 |
| `void detach_pid(struct task_struct *task, enum pid_type type)` | 将进程从指定进程 `pid_type` 的 `struct pid*` 的进程组或会话组链表上剥离。如果没有人使用关联的 `struct pid` 则释放 |

### 内核线程

> 1. `#include <linux/kthread.h>`
> 2. 内核线程在退出时不会发出 `SIGCHLD` 信号。
> 3. 内核线程其实是按进程组织的（非线程），普通内核线程的父进程是 `kthreadd` 内核线程。
> 4. 创建的进程是处于`TASK_UNINTERRUPTIBLE`状态的，需要 `wake_up_process()` 唤醒。
> 5. 创建的内核线程默认是可以在所有CPU上运行的。
> 6. 内核线程有一个 `park` 状态，除非解除这个状态，否则内核线程的回调会阻塞在 `kthread_parkme()` 处，不会继续执行。

| func or macro                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `struct task_struct *kthread_create_on_node(int (*threadfn)(void *data), void *data, int node, const char namefmt[], ...)` | 创建 内核线程 返回进程描述符。 1. `threadfn` 内核线程运行的回调函数 2. `data` 回调函数的参数 3. `node` 内核线程将使用哪个的内存 4. `namefmt` 与 后面的可变参数 作为内核线程的命名格式化。 |
| `kthread_create()`                                           | 对 `kthread_create_on_node()` 的封装，在当前节点上创建一个内核线程。 |
| `kthread_run()`                                              | 类似 `kthread_create()` 但如果创建成功，则使用 `wake_up_process()` 唤醒。 |
| `void kthread_bind(struct task_struct *k, unsigned int cpu)` | 绑定线程到 CPU，线程将只在指定 CPU 上运行。内核线程必须要处于 `TASK_UNINTERRUPTIBLE` 状态，且没有运行才会有效。**该函数会阻塞**。 |
| `void kthread_bind_mask(struct task_struct *k, const struct cpumask *mask)` | 类似 `kthread_bind()` ，但可以绑定一组CPU。                  |
| `int kthread_stop(struct task_struct *k)`                    | 发出停止请求，同步等待回调函数的返回值；如果线程是休眠的，会先被唤醒。**禁止在回调函数中使用**。 |
| `int kthread_should_stop(void)`                              | 有其他进程发出了停止请求，**在回调函数中使用，指示回调应该返回**。 |
| `bool kthread_should_park(void)`                             | 有其他进程发出了暂停请求，**在回调函数中使用，指示回调函数应该调用 `kthread_parkme()` 暂停自己**。 |
| `int kthread_park(struct task_struct *k)`                    | 请求暂停，唤醒内核线程，然后等待（一般是回调函数）调用 `kthread_parkme()`。 **不能在回调函数中使用**。 如果成功返回 0， 失败返回 `-ENOSYS` ，内核线程已经退出了。 |
| `void kthread_parkme(void)`                                  | 执行暂停，**只能在回调函数中使用**。                         |
| `void kthread_unpark(struct task_struct *k)`                 | 解除暂停。                                                   |
| `bool kthread_freezable_should_stop(bool *was_frozen)`       | 因为需要待机，内核线程应该被冻结。`was_frozen` 如果执行了冻结，则设置为真，否则返回（也可能是解除冻结唤醒后返回）是否应该被停止。**回调函数中使用**。 |
| `struct task_struct *kthread_create_on_cpu(int (*threadfn)(void *data), void *data, unsigned int cpu, const char *namefmt)` | 类似 `kthread_create_on_node()` 但在创建成功后， 自动调用 `kthread_bind()` ，而且在待机后唤醒会自动 将内核线程绑定到指定CPU上。 |
| `smpboot_register_percpu_thread()`                           | 每一个CPU上创建一个内核线程。                                |

### 进程信号

| func                  | comments           |
| --------------------- | ------------------ |
| `signal_group_exit()` | 线程组已经开始退出 |

### 进程创建标志

`do_fork()` 使用以下标志来控制创建的子进程（线程）的行为

| flags                  | comments                                                     |
| ---------------------- | ------------------------------------------------------------ |
| `CSIGNAL`              | 为了在退出时提取里的信号的掩码                               |
| `CLONE_VM`             | 共享内存描述符和所有的页表                                   |
| `CLONE_FS`             | 共享根目录和当前工作目录，以及文件屏蔽掩码                   |
| `CLONE_FILES`          | 共享打开的文件表                                             |
| `CLONE_SIGHAND`        | 共享打开的信号处理句柄                                       |
| `CLONE_PTRACE`         | 如果父进程正在被跟踪，子进程也被跟踪                         |
| `CLONE_VFORK`          | 发出的是`vfork()`调用                                        |
| `CLONE_PARENT`         | 设置新进程的父进程为调用`fork()`进程的父进程                 |
| `CLONE_THREAD`         | 作为线程被创建，共享信号处理句柄，设置`tgid`和`group_leader`字段为进程组首进程的进程号。使用这个标志必须设置 `CLONE_SIGHAND`标志 |
| `CLONE_NEWNS`          | 子进程需要一个自己的命名空间，使用这个标志，不能设置 `CLONE_FS` |
| `CLONE_SYSVSEM`        | 共享 `System V IPC` 信号量                                   |
| `CLONE_SETTLS`         | 为新的线程创建线程局部存储段`TLS`，由参数 `tls` 提供         |
| `CLONE_PARENT_SETTID`  | 将子进程的`PID`存入父进程的用户态变量中 `ptid` 参数中。      |
| `CLONE_CHILD_CLEARTID` | 建立一种触发机制，在子进程开始执行新程序（`exec()`）或退出时，内核将清除 `ctid` 参数指向的用户态变量，并通知等待唤醒的进程。 |
| `CLONE_DETACHED`       | 没有使用                                                     |
| `CLONE_UNTRACED`       | 进程不能被跟踪，忽略 `CLONE_PTRACE`标志。                    |
| `CLONE_CHILD_SETTID`   | 将子进程的存入子进程的用户态变量中                           |
| `CLONE_STOPPED`        | 子进程将开始于 TASK_STOPPED 状态                             |

### 进程调度

1. 调度类型标志

| flags          | comments                                                     |
| -------------- | ------------------------------------------------------------ |
| `SCHED_FIFO`   | 先进先出的实时进程，如果没有其他更高优先级的程序，那么进程将一直使用CPU。 |
| `SCHED_RR`     | 时间片轮转的实时进程，该类的实时进程公平使用CPU。            |
| `SCHED_NORMAL` | 普通分时进程。                                               |

1. 调度操作

| func or macro            | comments                               |
| ------------------------ | -------------------------------------- |
| `cpu_rq(cpu)`            | 获取 `cpu` 上的运行队列。              |
| `this_rq()`              | 获取 本地CPU 上的运行队列。            |
| `task_rq(task)`          | 获取 `task` 进程所在的运行队列。       |
| `cpu_curr(cpu)`          | 获取 `cpu` 上当前运行的进程。          |
| `sched_fork()`           | 在创建新进程时，设置调度信息。         |
| `sched_clock()`          | 当前调度时间戳。                       |
| `scheduler_tick()`       | 维持最新的 `time_slice` 计数器。       |
| `try_to_wake_up()`       | 试图唤醒进程。                         |
| `schedule()`             | 调度进程，选择一个最优的进程开始运行。 |
| `load_balance()`         | 平衡运行队列上的进程负载。             |
| `set_tsk_need_resched()` | 设置进程标志，以强制被调度。           |

### 进程页的映射

| func                                                      | comments                                                     |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `int anon_vma_fork(*vma, *pvma)`                          | `vma` 是从 `pvma` 复制而来，将 `vma` 的反向匿名映射结构附加到 `pvma` 的反向匿名结构中。 |
| `void vma_interval_tree_insert_after(*vma, *pvma, *root)` | 类似 `anon_vma_fork()` ，但应用在 文件映射上。`root` 是文件地址空间的 `mapping->i_mmap`。 |
| `void page_dup_rmap(struct page *page, bool compound)`    | 增加映射计数，如果 `compound` 为真，则指示 `page` 是组合页。 |

### 虚地址管理

> 1. `#include <linux/mm.h>`

1. 操作

| func or macro                                 | comments                                                     |
| --------------------------------------------- | ------------------------------------------------------------ |
| `int copy_page_range(*dst_mm, *src_mm, *vma)` | 通过 `mm_struct.pgd` 遍历，从 `src_mm` 内存管理中将 `vma` 指示虚地址范围的页表和页 拷贝到 `dst_mm` 中。 1. 如果涉及的页已经被交换，增加交换计数。 2. 如果 `vma` 是匿名页，目标和原的页表设置为写时复制状态。 3. 如果不是交换页，则增加页的引用计数和映射计数。 |
| `bool is_cow_mapping(vm_flags_t flags)`       | 通过 `vma.flags` 判断 `vma` 管理的虚地址空间中的页在拷贝时，是否需要写时复制操作。 |

1. 标志

| flags          | comments                               |
| -------------- | -------------------------------------- |
| `VM_READ`      | 可读                                   |
| `VM_WRITE`     | 可写                                   |
| `VM_EXEC`      | 可执行                                 |
| `VM_SHARED`    | 共享多个进程                           |
| `VM_MAYREAD`   | 需要被写时复制                         |
| `VM_MAYWRITE`  | 需要被写时复制                         |
| `VM_MAYEXEC`   | 需要被写时复制                         |
| `VM_MAYSHARE`  | 需要被写时复制                         |
| `VM_GROWSDOWN` | 虚地址用于用户态堆栈，向下增长。       |
| `VM_LOCKED`    | 虚地址映射的页不会被交换或写出到文件。 |

## 辅助工具

### 数学库

> 1. `#include <linux/math64.h>`
> 2. 尽量使用这些函数做数学运算，某些情况下可以带来优化。

| func or macro                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `div64_long(x, y)`                                           | long 整数的除法运算。等价 `(long)x / y`。                    |
| `div64_ul(x,y)`                                              | 无符号 long 整数的除法运算。                                 |
| `u64 div_u64_rem(u64 dividend, u32 divisor, u32 *remainder)` | 64位无符号整数除法，返回商，余数存入 `remainder` 指针中。 等价 `*rem = x % y; return x / y;`。 |
| `div_s64_rem()`                                              | 类似 `div_u64_rem()` ，对有符号做除法运算。                  |
| `div64_u64_rem()`                                            | 类似 `div_u64_rem()`，但除数也是 64位的。                    |
| `div64_s64_rem()`                                            | 类似 `div64_u64_rem()`，对有符号做除法运算。                 |
| `u64 div64_u64(u64 dividend, u64 divisor)`                   | 等价 `x / y` 。                                              |
| `div64_s64()`                                                | 类似 `div64_u64()`。                                         |

### 位运算

> 1. `#include <linux/log2.h>`
> 2. `#include <linux/kernel.h>`

| func or macro              | comments                                         |
| -------------------------- | ------------------------------------------------ |
| `ilog2(n)`                 | 计算 `n` 的以2为底的幂次方                       |
| `roundup_pow_of_two()`     | 对齐到2为底的次方数                              |
| `round_up(x, y)`           | 将`x` 按 `y` 向上（大的方向）对齐，用于2的倍数。 |
| `round_down(x, y)`         | 将`x` 按 `y` 向下（小的方向）对齐，用于2的倍数。 |
| `roundup(x, y)`            | 类似 `round_up()` 但可用于非2的倍数。            |
| `rounddown(x, y)`          | 类似 `round_down()` 但可用于非2的倍数。          |
| `ALIGN(x, y)`              | 将 `x` 按 `y` 向上对齐（一定是`2^n`）对齐。      |
| `IS_ALIGNED(x, y)`         | 判断 `x` 是否按 `y` 对齐。                       |
| `ALIGN_DOWN(x, y)`         | 将 `x` 按 `y` 向下对齐（一定是`2^n`）对齐。      |
| `ARRAY_SIZE(ptr)`          | 求数组的大小                                     |
| `FIELD_SIZEOF(ptr, field)` | 获取`ptr`结构体中`field`字段的字节大小。         |

## 文件系统

| func or macro                   | comments                                                     |
| ------------------------------- | ------------------------------------------------------------ |
| `sturct file *get_empty_filp()` | 获取（分配）一个空的文件对象，如果超出限额，则返回空指针。   |
| `int get_unused_fd()`           | 在当前进程中分配一个文件描述符。                             |
| `fput()`                        | 减少文件描述符的引用，如果不再被任何进程引用，则释放。主要调用关联文件操作的`release()`方法。 |
| `fd_install()`                  | 将文件描述符与文件对象匹配并安装到当前进程的文件表中。       |
| `file_accessed()`               | 更新文件的访问时间。                                         |

## 套接字

> 1. 该部分暂时只支持2.6的内核API，但是绝代多数API都实用

### 套接字控制

| func or macro              | comments                   |
| -------------------------- | -------------------------- |
| `sk_stream_is_writeable()` | 测试流套接字可写           |
| `SOCK_FASYNC`              | 异步通知等待IO的进程       |
| `SOCK_LINGER`              | 四步挥手标志               |
| `TCP_NAGLE_OFF`            | 快速发送，不论数据包大小。 |

### 地址判断与宏

| func or macro               | comments                             |
| --------------------------- | ------------------------------------ |
| `sk_stream_is_writeable()`  | 测试流套接字可写                     |
| `ipv4_is_multicast()`       | 判断套接字地址是否是多播地址         |
| `ipv4_is_local_multicast()` | 判断套接字地址是否是局域网多播地址。 |
| `ipv4_is_loopback()`        | 判断套接字地址是否是环回地址。       |
| `ipv4_is_lbcast()`          | 判断套接字地址是否是广播地址。       |
| `INADDR_ANY`                | 任意地址。                           |
| `INADDR_LOOPBACK`           | 环回首地址。                         |
| `INADDR_NONE`               | 无效地址。                           |
| `INET_ADDRSTRLEN`           | IPv4地址的字符串长度                 |
| `INET6_ADDRSTRLEN`          | IPv6地址的字符串长度                 |

### sk_buff

1. 结构

```
sk_buff
+----+
|    |
+----+
|head|-------->+-----+<--+-
+----+         |     |   |
|data|-------->|     |   |
+----+         |     |  linear
|tail|-------->|     |   |
+----+         |     |   |
|end |-------->+-----+<--+-
+----+         | ref |
|    |         +-----+
+----+         |frags|----->[skb_frag][skb_frag][skb_frag]
|    | 	       +-----+      |<------- nonlinear -------->|
+----+
1234567891011121314151617
```

1. 引用计数

```
        users=2              users=1
    +----+   +----+          +----+
    | A  |   | A  | SKB      | B  | SKB
    +-+--+   +-+--+          +-+--+
      |        |               |
      |        |               |
      +----+---+               +
           |                   |
           +--------+----------+
                    |
                +---+---+
                | data  | skb_shared_info
                +-------+
                dataref=3

123456789101112131415
```

1. 操作

| func or macro       | comments                                                     |
| ------------------- | ------------------------------------------------------------ |
| `alloc_skb()`       | 分配指定长度的skb                                            |
| `kfree_skb()`       | 减少引用计数，如果为或变为0，则释放skb的缓存，包括状态（及引用对象）。 |
| `skb_copy()`        | 完全的拷贝一份skb及其包含是状态和数据。                      |
| `skb_clone()`       | 克隆一份skb的状态，并且两者共同引用相同的数据。              |
| `pskb_copy()`       | 拷贝一份skb的状态，并且两者共同引用相同的非线性数据，但各自只有一份线性数据的副本。 |
| `skb_shinfo()`      | 获取非线性部分的描述结构体。                                 |
| `skb_get()`         |                                                              |
| `skb_cloned()`      |                                                              |
| `skb_shared()`      |                                                              |
| `skb_share_check()` |                                                              |

## 编译优化

| func or macro                  | comments                                          |
| ------------------------------ | ------------------------------------------------- |
| `____cacheline_aligned_in_smp` | 按cacheline对齐一个全局（静态）的变量，加快命中。 |

## hash算法

| func or macro                                  | comments                                                     |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `hash_ptr(ptr, bucket_size_shift)`             | 计算一个指针的hash，然后根据 `bucket_size_shift` 直接定位bucket数组的下标。 |
| `hashlen_string(salt, string)`                 | 计算字符串（`\0`结尾）的hash。                               |
| `full_name_hash(salt, bufptr, size)`           | 计算指定内存长度的hash。                                     |
| `jhash_1word(val, seed)`                       | 使用一个恒定值作为种子，计算32位的数值的hash。               |
| `jhash_2word(val1, val2, seed)`                | 计算2个32位值的hash。                                        |
| `jhash_3word(val2, val2, val3, seed)`          | 计算3个32位值的hash。                                        |
| `jhash2(const u32 *k, u32 length, u32 seed)`   | 就算 32位 数组数据的hash。                                   |
| `jhash(const void *key, u32 length, u32 seed)` | 计算 内存序列的数据的hash。                                  |
| `jhash_size()`                                 |                                                              |
| `jhash_mask()`                                 |                                                              |

## rbtree

| func or macro                                                | comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `rb_parent(node)`                                            | 得到节点的父节点。                                           |
| `RB_ROOT`                                                    | 普通红黑树根节点初始化值。                                   |
| `RB_ROOT_CACHED`                                             | 缓存最左红黑树根节点初始化值。                               |
| `rb_entry(ptr, type, member)`                                | 获取包含数据结构的指针。                                     |
| `RB_EMPTY_ROOT(root)`                                        | 判断根节点是否是空。                                         |
| `RB_EMPTY_NODE(node)`                                        | 判断节点是否为空。                                           |
| `RB_CLEAR_NODE(node)`                                        | 清空节点数据。                                               |
| `void rb_link_node(struct rb_node *node, struct rb_node *parent, struct rb_node **rb_link)` | 仅将节点插入红黑树，要保证被插入节点不在红黑树中。 `rb_link` 是新节点的存放位置，它是 `node` 的 `parent` 的右节点或左节点的二级指针。 |
| `void rb_insert_color(struct rb_node *, struct rb_root *)`   | 将插入的节点作色（旋转平衡）                                 |
| `void rb_erase(struct rb_node *, struct rb_root *)`          | 删除节点，要保证被删除的节点已插入红黑树中。                 |
| `rb_insert_color_cached()`                                   | 与 `rb_insert_color()` 相似，但可以指定新插入的节点是否是树的最左节点。 |
| `rb_erase_cached()`                                          | 与 `rb_erase()` 相似，但如果删除的节点时树的最左节点，相应的最左节点值会更新。 |
| `rb_prev(node)`                                              | 得到指定节点的上一个节点（节点按某种顺序排序的），即左节点的下的最右叶枝节点。 |
| `rb_next(node)`                                              | 得到指定节点的下一个节点，即右节点下的最左叶枝节点。         |
| `rb_first(root)`                                             | 得到根节点的最左节点。                                       |
| `rb_last(root)`                                              | 得到根节点的最右节点。                                       |
| `rb_replace_node(struct rb_node *victim, struct rb_node *new, struct rb_root *root)` | 将存在于 `root` 中的 `victim` 节点替换为 `new` 节点。        |
| `rb_replace_node_cached()`                                   | 类似 `rb_replace_node()` ，但是如果必要会更新最左节点。      |
| `rbtree_postorder_for_each_entry_safe()`                     | 树的后序遍历，左右中的顺序。                                 |



> Reference: https://blog.csdn.net/CrazyHeroZK/article/details/87361308

