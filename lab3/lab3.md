### 练习

对实验报告的要求：
 - 基于markdown格式来完成，以文本方式为主
 - 填写各个基本练习中要求完成的报告内容
 - 完成实验后，请分析ucore_lab中提供的参考答案，并请在实验报告中说明你的实现与参考答案的区别
 - 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）
 - 列出你认为OS原理中很重要，但在实验中没有对应上的知识点
 

#### 练习1：理解基于FIFO的页面替换算法（思考题）
描述FIFO页面置换算法下，一个页面从被换入到被换出的过程中，会经过代码里哪些函数/宏的处理（或者说，需要调用哪些函数/宏），并用简单的一两句话描述每个函数在过程中做了什么？（为了方便同学们完成练习，所以实际上我们的项目代码和实验指导的还是略有不同，例如我们将FIFO页面置换算法头文件的大部分代码放在了`kern/mm/swap_fifo.c`文件中，这点请同学们注意）
 - 至少正确指出10个不同的函数分别做了什么？如果少于10个将酌情给分。我们认为只要函数原型不同，就算两个不同的函数。要求指出对执行过程有实际影响,删去后会导致输出结果不同的函数（例如assert）而不是cprintf这样的函数。如果你选择的函数不能完整地体现”从换入到换出“的过程，比如10个函数都是页面换入的时候调用的，或者解释功能的时候只解释了这10个函数在页面换入时的功能，那么也会扣除一定的分数

1. 在alloc_pages()里面，如果此时试图得到空闲页且没有空闲的物理页时，我们才尝试换出页面到硬盘上。
//kern/mm/pmm.c
![屏幕截图 2023-10-27 221414.png](https://s2.loli.net/2023/10/27/ZcATOoJClrV1Kfi.png)

2. 定义了一个名为swap_manager的结构体，用于描述交换管理器的接口。

//kern/mm/swap.h
![屏幕截图 2023-10-27 222911.png](https://s2.loli.net/2023/10/27/UdHt5ArkPEMqo8N.png)

name：交换管理器的名称，以字符串形式存储。init：用于进行交换管理器的全局初始化的函数指针。init_mm：用于在mm_struct结构体中初始化私有数据的函数指针。tick_event：在时钟中断发生时调用的函数指针。map_swappable：将可交换页面映射到mm_struct结构体中时调用的函数指针。set_unswappable：当一个页面被标记为共享时，从交换管理器中删除该页面的地址条目时调用的函数指针。swap_out_victim：尝试将一个页面交换出去的函数指针，返回被交换出的页面。check_swap：用于检查页面置换算法的函数指针。

3. 定义了一个名为Page的结构体，用于描述页面的属性和状态。
//kern/mm/memlayout.h

![屏幕截图 2023-10-27 223340.png](https://s2.loli.net/2023/10/27/wSUbaXsz94eQnG1.png)

ref：页面帧的引用计数，用于记录页面被引用的次数。flags：描述页面帧状态的标志位数组。visited：用于页面置换算法的辅助字段，表示页面是否被访问过。property：用于首次适应页管理器的空闲块数量。page_link：用于将页面添加到空闲页面链表中的链表项。pra_page_link：用于页面替换算法的链表项。pra_vaddr：用于页面替换算法的虚拟地址。

4. swap_out()换出一个页面.
![屏幕截图 2023-10-28 152356.png](https://s2.loli.net/2023/10/28/AGlOiszC6xPMtvE.png)

这段代码定义了一个函数swap_out，它接受三个参数：mm表示进程的内存空间，n表示需要置换的页面数量，in_tick表示是否在时钟中断中调用。在循环中，从0到n进行迭代，表示需要置换n个页面。


![屏幕截图 2023-10-28 152619.png](https://s2.loli.net/2023/10/28/WGONF7v8oyzaTDM.png)

定义了一个uintptr_t类型的变量v，用于保存页面的虚拟地址。定义了一个struct Page类型的指针page，用于保存被选择的页面。调用sm->swap_out_victim函数来选择需要置换的页面，并将选择的页面保存到page指针中。如果选择页面失败，则打印错误信息，并跳出循环.

![屏幕截图 2023-10-28 153033.png](https://s2.loli.net/2023/10/28/GRchYslJFDnIP9L.png)

将被选择的页面的虚拟地址保存到变量v中。调用get_pte函数获取页表项的地址，并将其保存到指针ptep中。使用assert宏来确保获取的页表项是有效的。
![屏幕截图 2023-10-28 153331.png](https://s2.loli.net/2023/10/28/28jKBEdQ5NyWXc1.png)

调用swapfs_write函数将页面保存到磁盘交换区。如果保存失败，则打印错误信息，并将页面重新标记为可交换。如果保存成功，则打印保存的信息，并更新页表项的值为磁盘交换区的地址。最后，释放物理页面，并调用free_page函数。

调用tlb_invalidate函数使TLB中的缓存失效，以便下次访问时重新加载页表。

这段代码实现了将进程中的页面置换到磁盘交换区的过程，实现了操作系统中的页面置换机制。


5. swapfs_write函数
![屏幕截图 2023-10-31 194558.png](https://s2.loli.net/2023/10/31/4yPe2aBlgmt3FOD.png)
swapfs_write函数将一个页面的内容写入到一个交换条目中。它使用ide_write_secs函数将与交换条目对应的扇区从页面的物理内存地址写入到交换设备中。


6. swap_in（）换入一个页面。

![屏幕截图 2023-10-28 153928.png](https://s2.loli.net/2023/10/28/iOIFAMm6py3DfVX.png)

这段代码定义了一个函数swap_in，它接受三个参数：mm表示进程的内存空间，addr表示需要加载的页面的虚拟地址，ptr_result表示加载的页面的结果。在函数中，首先调用alloc_page函数分配一页物理内存，并将结果保存到result指针中。使用assert宏来确保分配的页面不为空

![屏幕截图 2023-10-28 154108.png](https://s2.loli.net/2023/10/28/xDTyAsnvu9oNR2J.png)

调用get_pte函数获取页表项的地址，并将其保存到指针ptep中。

![屏幕截图 2023-10-28 154234.png](https://s2.loli.net/2023/10/28/olEIzknu5Gp8jRF.png)

调用swapfs_read函数从磁盘交换区读取页面，并将结果保存到result指针中。如果读取失败，则打印错误信息，并使用assert宏来确保读取成功。

打印加载的信息，包括交换区的索引和加载的页面的虚拟地址。将加载的页面的指针保存到ptr_result指针中。返回0，表示加载成功。
这段代码实现了从磁盘交换区加载页面到内存的过程。

7. swap_init()初始化
![屏幕截图 2023-10-28 155624.png](https://s2.loli.net/2023/10/28/xO7kLE5QAe3nGmg.png)

8. swapfs_read函数
![屏幕截图 2023-10-31 194814.png](https://s2.loli.net/2023/10/31/jIkwV8X1KigaRSY.png)
swapfs_read函数将交换条目的内容读取到一个页面中。它使用ide_read_secs函数将与交换条目对应的扇区从交换设备读取到页面的物理内存地址中。

9. FIFO的初始化
![屏幕截图 2023-10-28 162132.png](https://s2.loli.net/2023/10/28/yqiCmxEYPdaMznS.png)

调用list_init函数初始化一个链表pra_list_head，该链表用于保存进程的页面访问记录。然后，将链表pra_list_head的地址赋值给进程的sm_priv字段，以便在后续的页面置换操作中使用。

函数执行了上述步骤，并将链表pra_list_head的地址赋值给进程的sm_priv字段。

10. FIFO设置页面可交换
![屏幕截图 2023-10-28 162523.png](https://s2.loli.net/2023/10/28/zcnWITSaDPHebYB.png)

首先，从进程的sm_priv字段中获取链表pra_list_head的地址，并将其赋值给变量head。然后，从页面的pra_page_link字段中获取链表节点的地址，并将其赋值给变量entry。接下来，使用断言语句assert检查entry和head是否为非空。如果其中任何一个为空，将会触发断言失败，表示程序出现了错误。然后，将页面的链表节点entry添加到链表pra_list_head的末尾，表示该页面是最近访问的页面。最后，返回0表示映射成功。

函数执行了上述步骤，并将页面的链表节点添加到链表pra_list_head的末尾，表示该页面是最近访问的页面。

11. FIFO页面置换
![屏幕截图 2023-10-28 163210.png](https://s2.loli.net/2023/10/28/Tcg9z3rEeCVIin4.png)

从进程的sm_priv字段中获取链表pra_list_head的地址，并将其赋值给变量head。然后，使用断言语句assert检查head是否为非空。如果为空，将会触发断言失败，表示程序出现了错误。接下来，使用断言语句assert检查in_tick是否为0。如果不为0，将会触发断言失败，表示程序出现了错误。然后，从链表pra_list_head的最后一个节点（即最早访问的页面）开始遍历，找到一个页面作为置换的受害者。如果找到了受害者页面，将其从链表中删除，并将其地址赋值给ptr_page指针。如果没有找到受害者页面，将ptr_page指针设置为NULL。

12. _fifo_init：这个函数在FIFO页面置换算法初始化时被调用。在这个示例中，它只是简单地返回0，表示初始化成功。
13. _fifo_set_unswappable：这个函数用于将虚拟内存中的特定地址标记为不可置换。在这个示例中，它也只是简单地返回0，表示操作成功。
14. _fifo_tick_event：这个函数在每个时钟滴答事件（tick event）发生时被调用，通常是定时器中断。在这个示例中，它返回0，表示成功处理了事件
![屏幕截图 2023-10-28 165459.png](https://s2.loli.net/2023/10/28/kmdbiGg5MjlnXwp.png)

15. _fifo_check_swap：这个函数通过模拟一系列内存访问来测试FIFO页面置换算法。它向特定的虚拟地址写入值，并使用assert语句检查pgfault_num变量的值。这个函数用于测试算法的正确性。

16. 
定义了一个名为swap_manager_fifo的结构体，用于表示FIFO页面置换算法的交换管理器
![屏幕截图 2023-10-28 165934.png](https://s2.loli.net/2023/10/28/jufHEaw9DyPq8tV.png)
.init：指向_fifo_init函数，用于初始化FIFO页面置换算法。.init_mm：指向_fifo_init_mm函数，用于初始化与特定内存管理器相关的FIFO页面置换算法。.tick_event：指向_fifo_tick_event函数，用于处理时钟滴答事件。.map_swappable：指向_fifo_map_swappable函数，用于将特定页面标记为可置换。.set_unswappable：指向_fifo_set_unswappable函数，用于将特定页面标记为不可置换。.swap_out_victim：指向_fifo_swap_out_victim函数，用于选择一个页面进行置换。.check_swap：指向_fifo_check_swap函数，用于测试FIFO页面置换算法。

#### 练习2：深入理解不同分页模式的工作原理（思考题）
get_pte()函数（位于`kern/mm/pmm.c`）用于在页表中查找或创建页表项，从而实现对指定线性地址对应的物理页的访问和映射操作。这在操作系统中的分页机制下，是实现虚拟内存与物理内存之间映射关系非常重要的内容。
 - get_pte()函数中有两段形式类似的代码， 结合sv32，sv39，sv48的异同，解释这两段代码为什么如此相像。

get_pte()函数中的两段代码非常相似，因为它们都遵循相同的页表结构和地址转换逻辑，不同的地址转换模式之间的主要差异在于页表级别和地址位数的不同。在sv32模式下，使用两级页表，一级页表包含1024个二级页表项；在sv39模式下，使用三级页表，一级页表包含512个二级页表项，二级页表包含512个三级页表项；在sv48模式下，使用四级页表，一级页表包含512个二级页表项，二级页表包含512个三级页表项，三级页表包含512个四级页表项。

![屏幕截图 2023-10-29 132506.png](https://s2.loli.net/2023/10/29/cVZbOBLzaEIlkRT.png)

首先，函数接收三个参数：pgdir 表示页目录表的起始地址，la 表示需要查找页表项的虚拟地址，create 表示如果需要分配新的页表项是否创建。接下来，函数会先访问虚拟地址 la 对应的页目录表项，即 pdep1。如果该页目录表项不存在（即未被映射），则需要创建一个新的页表并将其映射到该页目录表项上。具体地，函数会首先分配一个物理页（通过调用 alloc_page() 函数），然后将该物理页清零（通过调用 memset() 函数），接着将该物理页映射到该页目录表项上（通过设置 pdep1 的值为 pte_create(page2ppn(page), PTE_U | PTE_V)）。其中，pte_create() 函数用于创建一个页表项，page2ppn() 函数用于获取物理页号。然后，函数会访问虚拟地址 la 对应的页表项，即 pdep0。同样，如果该页表项不存在，则需要创建一个新的物理页并将其映射到该页表项上。具体地，函数会首先分配一个物理页，然后将该物理页清零，接着将该物理页映射到该页表项上。同样地，函数会使用 pte_create() 函数来创建一个页表项，并将其设置为可读写和用户态可访问。最后，函数会返回虚拟地址 la 对应的页表项的地址。具体地，函数会先访问虚拟地址 la 对应的二级页表的起始地址，即 PDE_ADDR(*pdep0)，然后再访问对应的页表项，即 ((pte_t *)KADDR(PDE_ADDR(*pdep0)))[PTX(la)]。其中，PTX(la) 表示虚拟地址 la 在页表中的索引，KADDR() 函数用于将物理地址转换为内核虚拟地址。

 - 目前get_pte()函数将页表项的查找和页表项的分配合并在一个函数里，你认为这种写法好吗？有没有必要把两个功能拆开？

优点：
* 简化了代码逻辑和结构：将查找和分配合并在一个函数中，可以避免重复的代码和逻辑，使代码更加简洁和清晰。
* 减少了函数调用开销：将查找和分配合并在一个函数中，可以减少函数调用的次数，提高了代码的执行效率。

缺点：
* 功能耦合度较高：将查找和分配合并在一个函数中，使得函数的功能较为复杂，不易于理解和维护。一旦其中一个功能需要修改，可能会影响到另一个功能的实现。
* 可扩展性较差：如果需要在将来的开发中对查找和分配进行不同的优化或调整，由于两个功能被合并在一个函数中，可能需要对整个函数进行修改，而不仅仅是其中一个功能的部分。
#### 练习3：给未被映射的地址映射上物理页（需要编程）
补充完成do_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。设置访问权限 的时候需要参考页面所在 VMA 的权限，同时需要注意映射物理页时需要操作内存控制 结构所指定的页表，而不是内核的页表。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
 - 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。
 - 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
- 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

1. 首先，检查是否初始化了交换管理器（`swap_init_ok`）。如果已初始化，说明需要将从磁盘交换进入内存，所以调用 `swap_in` 函数。这个函数根据地址 `addr` 以及 `mm` 参数来加载磁盘中正确的页面到 `page` 结构体中。`swap_in_ret` 用于记录 `swap_in` 函数的返回值，如果返回 0 表示交换成功。
2. 如果交换成功，进入条件判断内部。
   a. `int page_insert_ret = page_insert(mm->pgdir, page, addr, perm);`：将 `page` 结构体中的页面插入到页表中，建立地址 `addr` 到物理页的映射。`page_insert_ret` 用于记录 `page_insert` 函数的返回值，如果返回 0 表示地址映射建立成功。
   b. `if (page_insert_ret == 0)`：如果地址映射建立成功，继续进入此条件判断内部。
   i. `swap_map_swappable(mm, addr, page, 1);`：标记页面为可交换状态，设置第四个参数为 1。这一步表示该页面可以被交换出去。`swap_map_swappable` 函数用于管理页面的可交换状态。
3. 如果在上述任何步骤中有错误，会相应地设置 `ret` 为 `-E_NO_MEM`，表示内存不足。然后跳转到 `failed` 标签。
4. 最后，无论前面的条件是否执行，都会将页面的 `pra_vaddr` 设置为 `addr`，以跟踪页面的虚拟地址。

```cpp
if (swap_init_ok) {
            struct Page *page = NULL;
            // 你要编写的内容在这里，请基于上文说明以及下文的英文注释完成代码编写
            //（1）According to the mm AND addr, try
            //to load the content of right disk page
            //into the memory which page managed.
            //(2) According to the mm,
            //addr AND page, setup the
            //map of phy addr <--->
            //logical addr
            //(3) make the page swappable.

            int swap_in_ret = swap_in(mm, addr, &page);
            if (swap_in_ret == 0) {
                // 交换成功，现在建立地址映射
                int page_insert_ret = page_insert(mm->pgdir, page, addr, perm);
                if (page_insert_ret == 0) {
                    // 建立地址映射成功，标记为swappable
                    swap_map_swappable(mm, addr, page, 1);
                } else {
                    cprintf("page_insert in do_pgfault failed\n");
                    ret = -E_NO_MEM;
                }
            } else {
                cprintf("swap_in in do_pgfault failed\n");
                ret = -E_NO_MEM;
            }

            // 设置页面的虚拟地址 pra_vaddr 为 addr
            page->pra_vaddr = addr;
  
        } else {
            cprintf("no swap_init_ok but ptep is %x, failed\n", *ptep);
            goto failed;
        }
```

1.PPN字段存储了物理页框的编号。在页替换算法中，这个字段用于确定要替换的物理页框。访问位记录了页面是否被访问过。页替换算法（例如LRU或CLOCK算法）使用这个位来确定最近被访问的页面。脏位用于指示页面是否被修改过。在页替换算法中，脏位可以用于标记需要写回磁盘的脏页，以确保数据的一致性。权限位用于确定用户和内核的访问权限。

2.页访问异常，首先产生响应的中断，并调用do_pgfault处理页访问异常。首先执行find_vma查找异常地址对应的vma，如果没有找到，则为地址访问错误；之后设置vma的权限与对齐addr，传递给pgdir_alloc_page查找与创建页面；若创建失败，则页面应该位于磁盘中，判断swap_init_ok即页面置换管理器是否初始化，若没有为页面置换初始化错误；之后调用swap_in换入页面，若失败为换页异常；之后调用page_insert建立虚拟页到物理页的映射并设置标记位，若失败为页面映射异常。

3.关系如下：

```cpp
struct Page {
    int ref;                        // 与页表项无关，确定页面是否被多个进程或线程共享
    uint_t flags;                   // 标志位，标志权限
    uint_t visited;                 // 访问位
    unsigned int property;          // 与页表项无关，空闲物理页帧数量
    list_entry_t page_link;   
    list_entry_t pra_page_link;   
    uintptr_t pra_vaddr;            // 虚拟地址
};
```

#### 练习4：补充完成Clock页替换算法（需要编程）
通过之前的练习，相信大家对FIFO的页面替换算法有了更深入的了解，现在请在我们给出的框架上，填写代码，实现 Clock页替换算法（mm/swap_clock.c）。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：
 - 比较Clock页替换算法和FIFO算法的不同。

```cpp
static int
_clock_init_mm(struct mm_struct *mm)
{     
     /*LAB3 EXERCISE 4: 2112433*/ 
     // 初始化pra_list_head为空链表
     // 初始化当前指针curr_ptr指向pra_list_head，表示当前页面替换位置为链表头
     // 将mm的私有成员指针指向pra_list_head，用于后续的页面替换算法操作
     //cprintf(" mm->sm_priv %x in fifo_init_mm\n",mm->sm_priv);
     list_init(&pra_list_head);
     curr_ptr = &pra_list_head;
     mm->sm_priv = &pra_list_head;
     return 0;
}
```
该函数实现了clock页替换算法的初始化。首先初始化pra_list_head为空链表，然后将当前指针指向pra_list_head，之后将mm的私有成员指针指向pra_list_head。

```cpp
static int
_clock_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && curr_ptr != NULL);
    //record the page access situlation
    /*LAB3 EXERCISE 4: 2112433*/ 
    // link the most recent arrival page at the back of the pra_list_head qeueue.
    // 将页面page插入到页面链表pra_list_head的末尾
    // 将页面的visited标志置为1，表示该页面已被访问
    list_add_before(mm->sm_priv,entry);
    page->visited=1;
    return 0;
}
```
该函数设置某一页面设置为可交换的。用`list_add_before(mm->sm_priv,entry)`来将页面page插入到pra_list_head的末尾，由于mm->sm_priv始终指向pra_list_head的开头，所以用了add_before函数来插入。之后，将页面的visited设为1，表示这个页面被访问过了。

```cpp
static int
_clock_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
         assert(head != NULL);
     assert(in_tick==0);
     /* Select the victim */
     //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
     //(2)  set the addr of addr of this page to ptr_page
    while (1) {
        /*LAB3 EXERCISE 4: 2112433*/ 
        // 编写代码
        // 遍历页面链表pra_list_head，查找最早未被访问的页面
        // 获取当前页面对应的Page结构指针
        // 如果当前页面未被访问，则将该页面从页面链表中删除，并将该页面指针赋值给ptr_page作为换出页面
        // 如果当前页面已被访问，则将visited标志置为0，表示该页面已被重新访问
        curr_ptr = list_next(curr_ptr);     
        if(curr_ptr==head){
            curr_ptr=list_next(curr_ptr);
        }        
        struct Page *ptr = le2page(curr_ptr, pra_page_link);
        if(ptr->visited==0){
            cprintf("curr_ptr %p\n",curr_ptr);
            list_del(curr_ptr);
            *ptr_page=ptr;
            break;
        }else{
            ptr->visited=0;
        }
    }
    return 0;
}
```
该函数实现了页面的置换。首先通过`list_next(curr_ptr)`去遍历pra_list_head，由于pra_list_head链表头是没有数据的，所以如果curr_ptr回到了头节点，就要再次执行一次`list_list`。之后，通过visited去判断当前页面是否被访问。如果被访问，那么就将visited设为0；如果未被访问，那么就将该页面从链表中删除，并把它赋给ptr_page作为换出页面。这里的删除并没有删掉curr_ptr的后继指针，因此下次置换时仍然可以调用`curr_ptr`。

clock和fifo的不同点在于：fifo维护了一个队列，通过先进入先删掉的方式进行页面置换；而clock维护了一个双向链表，通过指针在不停遍历，利用局部性原理将一个循环中没有被访问过的页面置换掉。

#### 练习5：阅读代码和实现手册，理解页表映射方式相关知识（思考题）
如果我们采用”一个大页“ 的页表映射方式，相比分级页表，有什么好处、优势，有什么坏处、风险？

好处：
* 查找效率提高，缩短时间，查找一次就可找到对应的物理地址

坏处和风险：
* 页表占用空间大，一个32位的地址空间就需要一百万项的页表
* 内存管理复杂度提高，需要维护一段连续的地址空间来存储页表，在操作系统内存紧张或者内存碎片较多时，这无疑会带来额外的开销
* 安全性可能降低，一个大页暴露了更多内存区域访问。


本实验主要知识点是页面的换入和换出，也对应了OS原理中的相关知识。实验的页面置换并非是在不忙的时候主动进行，而是出现异常之后进行的被动的置换，这同书中的知识有一点差别。同时，本实验也并没有实现OPT、LRU、LFU等其他置换算法，仅仅对其进行了理论学习。另外，对Belady现象，本实验也没有去进行探究。