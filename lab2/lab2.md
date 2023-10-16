### 练习

对实验报告的要求：
 - 基于markdown格式来完成，以文本方式为主
 - 填写各个基本练习中要求完成的报告内容
 - 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）
 - 列出你认为OS原理中很重要，但在实验中没有对应上的知识点


#### 练习1：理解first-fit 连续物理内存分配算法（思考题）

#### 1.memlayout.h

```cpp
struct Page {
    int ref;             
    uint32_t flags;      
    unsigned int property;  
    list_entry_t page_link;  
};
```

一个物理页数据结构含有四个成员，分别的解释如下：

1、ref，是映射此物理页的虚拟页个数。如果这个页被页表引用了，就会把Page的ref加一；反之就会把Page的ref减一。

2、flags表示此物理页的状态标记，为1的时候代表这一页是free状态，可以被分配；如果为0，那么说明这个页已经分配了。

3、property用来记录某连续空闲页的数量，用到此成员变量的这个Page一定是连续内存块的开始地址（第一页的地址）。

4、page_link是便于把多个连续内存空闲块链接在一起的双向链表指针。

```cpp
typedef struct {
    list_entry_t free_list;   
    unsigned int nr_free;       
} free_area_t;
```

定义了一个free_area_t数据结构，作为空闲页帧表，包含了一个双向链表指针和记录当前空闲页的个数的无符号整型变量nr_free。其中的链表指针指向了空闲的物理页。

#### 2.pmm.c

```cpp
struct pmm_manager {
    const char *name;  // XXX_pmm_manager's name
    void (*init)(
        void);  // initialize internal description&management data structure
                // (free block list, number of free block) of XXX_pmm_manager
    void (*init_memmap)(
        struct Page *base,
        size_t n);  // setup description&management data structcure according to
                    // the initial free physical memory space
    struct Page *(*alloc_pages)(
        size_t n);  // allocate >=n pages, depend on the allocation algorithm
    void (*free_pages)(struct Page *base, size_t n);  // free >=n pages with
                                                      // "base" addr of Page
                                                      // descriptor
                                                      // structures(memlayout.h)
    size_t (*nr_free_pages)(void);  // return the number of free pages
    void (*check)(void);            // check the correctness of XXX_pmm_manager
};struct
```

作为物理内存管理类的基类，后续内存分配算法由该基类继承，在pmm.c中使用该结构体的指针来实现访问不同的内存分配算法。

#### 3.default_pmm.c

通过结构体可知，内存分配算法主要有五个函数组成，除去check函数，主要有四个核心函数
#### default_init：
```cpp
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```
该函数实现了初始化空闲物理页帧表，调用的过程如下：`kern/init/init.c pmm_init()`->`kern/mm/pmm.c init_pmm_manager()`->通过动态绑定`kern/mm/default_pmm.c`中的结构体地址，然后完成利用指针pmm_manager完成init函数调用。

该函数首先调用了`libs/list.h`中的list_init函数对一个双向链表进行了初始化，而起始节点指向了free_list。追溯可发现free_list其实就是`kern/mm/memlayout.h`中结构体free_area_t的成员free_list双向链表的一个实例化，它用来记录那些空闲的物理页帧。

#### default_init_memmap:
`kern/mm/pmm.c`完成了init_pmm_manager函数之后，接着调用了page_init函数。
   ```cpp
   static void page_init(void) {
       va_pa_offset = PHYSICAL_MEMORY_OFFSET;

       uint64_t mem_begin = KERNEL_BEGIN_PADDR;
       uint64_t mem_size = PHYSICAL_MEMORY_END - KERNEL_BEGIN_PADDR;
       uint64_t mem_end = PHYSICAL_MEMORY_END; //硬编码取代 sbi_query_memory()接口

       cprintf("physcial memory map:\n");
       cprintf("  memory: 0x%016lx, [0x%016lx, 0x%016lx].\n", mem_size, mem_begin,
               mem_end - 1);

       uint64_t maxpa = mem_end;

       if (maxpa > KERNTOP) {
           maxpa = KERNTOP;
       }

       extern char end[];

       npage = maxpa / PGSIZE;
       //kernel在end[]结束, pages是剩下的页的开始
       pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);

       for (size_t i = 0; i < npage - nbase; i++) {
           SetPageReserved(pages + i);
       }

       uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * (npage - nbase));

       mem_begin = ROUNDUP(freemem, PGSIZE);
       mem_end = ROUNDDOWN(mem_end, PGSIZE);
       if (freemem < mem_end) {
           init_memmap(pa2page(mem_begin), (mem_end - mem_begin) / PGSIZE);
       }
   }
```
它将前npage-nbase个页面标记为已保留，nbase表示内核使用的基本页面数量。之后计算了可用内存的起始地址freemem，确保它对齐到页面大小，并将mem_begin和men_end调整为页面边界。初始化之后如果存在空闲的内存空间，代码调用init_memmap函数，将这些内存空间标记为可用页面。
```cpp
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    if (list_empty(&free_list)) {
        list_add(&free_list, &(base->page_link));
    } else {
        list_entry_t* le = &free_list;
        while ((le = list_next(le)) != &free_list) {
            struct Page* page = le2page(le, page_link);
            if (base < page) {
                list_add_before(le, &(base->page_link));
                break;
            } else if (list_next(le) == &free_list) {
                list_add(le, &(base->page_link));
            }
        }
    }
}
```
它声明了一个Page指针指向base的地址，用于遍历base到base+n-1之间的所有元素。for循环中，首先使用断言确保指针p指向的页面是被保留的，然后令flags和property为0，并让指针p指向的页面的引用次数设置为0。之后，让base指向的Page的property为n，表示它后面有n个连续的空闲物理页。并调用宏SetPageProperty将base指向的Page的flag设置为1，将nr_free加n。

接下来，判断自由页面链表是否为空，如果为空，那么就直接把base的页面加上去，如果非空，声明一个指向自由页面链表free_list头节点的指针le，一直循环遍历直到再次回到头节点，每次循环中，都将le通过函数le2page转换为为Page类型的指针page，如果当前base指向的页面在遍历到的页面page之前，那么将它插入到le指向节点的前面并break，如果当前base指向的页面不在page之前并且le的下一个节点就是头节点了，则将base的页面放在链表末尾。

关于le2page是如何将链表指针变成Page结构体指针的，就是用le的地址减去了节点中成员变量的所占空间，得到的就是Page结构体的地址，而所占空间可以令Page=0计算得到。

#### default__alloc_pages:
```cpp
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    if (page != NULL) {
        list_entry_t* prev = list_prev(&(page->page_link));
        list_del(&(page->page_link));
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;
            SetPageProperty(p);
            list_add(prev, &(p->page_link));
        }
        nr_free -= n;
        ClearPageProperty(page);
    }
    return page;
}
```
该函数的作用是在空闲物理页帧表中找到n个连续的空闲块。

它接受了一个参数n，表示需要分配的页面数量。如果n大于空闲页面列表中nr_free的数量，则返回NULL，表示没有足够的连续空闲页可以分配。

声明一个Page指针page，声明一个用于遍历链表的指针le，在每次循环遍历中，判断当前节点p的property属性是否大于等于n，如果是，则将page的值设置为p并跳出循环，找到了可供分配的空闲页。然后获取该页面在链表中的前一个节点，并将其赋给prev，使用list_del函数将其从链表中删除。

如果当前空闲页块的大小大于所需大小，表示它可以被分割，声明一个指针p初始化为page+n，然后将p的property设置为page->property-n，即剩余的空闲页面数量，使用SetPageProperty将p的flag置1，并再把它添加到链表中，插入到prev后面。将nr_free减去n，表示已经分配了n个空闲页面。使用ClearPageProperty将page的property值置1。最后返回分配的页面page。

#### default_free_pages：
```cpp
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;

    if (list_empty(&free_list)) {
        list_add(&free_list, &(base->page_link));
    } else {
        list_entry_t* le = &free_list;
        while ((le = list_next(le)) != &free_list) {
            struct Page* page = le2page(le, page_link);
            if (base < page) {
                list_add_before(le, &(base->page_link));
                break;
            } else if (list_next(le) == &free_list) {
                list_add(le, &(base->page_link));
            }
        }
    }

    list_entry_t* le = list_prev(&(base->page_link));
    if (le != &free_list) {
        p = le2page(le, page_link);
        if (p + p->property == base) {
            p->property += base->property;
            ClearPageProperty(base);
            list_del(&(base->page_link));
            base = p;
        }
    }

    le = list_next(&(base->page_link));
    if (le != &free_list) {
        p = le2page(le, page_link);
        if (base + base->property == p) {
            base->property += p->property;
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
    }
}
```
该函数将页面重新链接到空闲物理页帧列表中，并将小的空闲块合并为大的空闲块。

它接受了两个参数，一个是Page指针base，表示要释放的起始页面，一个是n表示要释放的页面数量。它声明了一个名为p的Page指针，并初始化为base。

通过循环来遍历从base开始的n个页面，对这些页面使用断言确保它既不是reserved页面，也没有property属性，然后将flags清零表示该页面不再被占用，并将页面引用次数设为0。

之后，将释放的起始页面base的property设置为释放的页面数量n，将nr_free增加n，表示增加了n个空闲物理页帧。

如果空闲物理页帧列表是空的，那就将base的页面添加到链表上，否则，使用循环找到第一个大于base的页面page，在前面添加base，如果找不到就把它添加到链表的末尾。

最后，检查base的前一个页面和或一个页面，如果它们是连续的，那么空闲它们合并成一个更大的空闲页面。

#### 练习2：实现 Best-Fit 连续物理内存分配算法（需要编程）

与首次匹配算法的流程几乎一致，只是在alloc选择合适的页时，不再找到能用的page就中断返回，而是遍历全部的page直到找到最小的合适的页

```cpp
while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
  
        if (p->property >= n && p->property - n < min_size) {
            page = p;
            min_size = p->property - n;
        }
    }
```
![lab2.png](https://s2.loli.net/2023/10/16/4JALBI3E9ngqlRb.png)

#### 知识点

本实验中用到的知识点有分页和物理内存分配，都对应着理论课中相应的知识点。分页技术就是通过页表的映射将物理地址变成虚拟地址，在lab2中它只是一个前置条件。本实验是RISC v架构，用到了三级页表，而理论课是x86架构，用的是两级页表，但两者的思想是相同的。物理内存分配算法用于将物理内存划分成不同的分区，包括最佳匹配、最先匹配、下次匹配、伙伴系统等算法，本实验借助了一个空闲物理页帧表对物理内存资源进行了管理，在现代操作系统中，它同分页技术一道更加高效地利用了有限的物理内存资源。

没有对应到的理论课知识点：为了解决物理内存不够用的情况而使用的碎片整理和内存覆盖技术。