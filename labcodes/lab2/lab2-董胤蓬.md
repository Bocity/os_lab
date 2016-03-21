# Lab2 Report

## 练习1：实现 first-fit 连续物理内存分配算法
修改default_pmm.c中default\_alloc\_pages函数和default\_free\_pages函数，实现first\_fit内存分配算法。 

实现中所用到的数据结构为双向链表，采用实验指导书中介绍的方法，在一块连续的页空间内，使用地址最小的一页（Head Page）记录这块内存地址的大小，并通过成员变量page\_link来维护链表结构。具体的实现如下：

```c
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
        struct Page *p = page;
        for(; p != page + n; p++) {
            ClearPageProperty(p);
            SetPageReserved(p);
        }
        if (page->property > n) {
            p = page + n;
            p->property = page->property - n;
            SetPageProperty(p);
            page->property = n;
            list_add(&page->page_link, &(p->page_link));
        }
        list_del(&(page->page_link));
        nr_free -= n;
    }
    return page;
}
```

分配内存主要分为以下几个步骤：

1. 判断空闲地址空间是否大于所需空间
2. 从free_list开始，遍历链表，直到找到第一块不小于所需空间大小的内存块
3. 分配连续的n页，修改标志位
4. 从链表中删除此内存块，如果有剩余的小的内存块，重新插入链表

释放内存的函数实现为：

```c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p) && !PageProperty(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);

    list_entry_t *le = list_next(&free_list);

    while (le != &free_list) {
        p = le2page(le, page_link);
        if(p > base) break;
        le = list_next(le);
    }

    list_add_before(le, &(base->page_link));

    p = le2page(le, page_link);
    if(le != &free_list && base + base->property == p) {
        base->property += p->property;
        ClearPageProperty(p);
        list_del(&(p->page_link));
    }

    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);

    if (le != &free_list && p + p->property == base) {
        p->property += base->property;
        ClearPageProperty(base);
        list_del(&(base->page_link));
    }
    nr_free += n;
}
```

主要分为以下步骤：

1. 修改释放页的标志位
2. 找到链表中应该插入的位置并插入
3. 判断此块空余空间能否与前后空余空间合并，如果可以将其合并

### 问题1.1：你的first fit算法是否有进一步的改进空间
有，可以采用平衡树的数据结构维护空闲地址空间，这样在分配和回收空间时，查找内存的操作可以更快。

## 练习2：实现寻找虚拟地址对应的页表项

修改pmm.c中的get_pte函数，增加下面代码：

```c
	pde_t *pdep = &pgdir[PDX(la)];
    if (!(*pdep & PTE_P)) {
        struct Page *page;
        if (!create || (page = alloc_page()) == NULL) {
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0, PGSIZE);
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
```

代码主要分为以下几个步骤：

1. 根据虚地址的高十位查询页目录，找到页表项的pdep
2. 检查该页是否在内存中，如果不在，创建该页，并更新相关信息
3. 根据虚拟地址的中间十位，找到虚拟地址对应的页表项

### 问题2.1：请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

```
页目录项内容 = (页表起始物理地址 & ~ 0x0FFF) | PTE_U | PTE_W | PTE_P
页表项内容 = (pa & ~0x0FFF) | PTE_P | PTE_W
```

页目录项和页表项的高20位存储相应的物理页帧号，低12位存储标志位。标志位的定义为：

```c
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
                                                // The PTE_AVAIL bits aren't used by the kernel or interpreted by the
                                                // hardware, so user processes are allowed to set them arbitrarily.

```

### 问题2.2：如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

出现页访问异常后，硬件会触发中断，并转到中断处理入口使操作系统处理中断。

## 练习3：释放某虚地址所在的页并取消对应二级页表项的映射

修改pmm.c中的page\_remove\_pte函数。

```c
	if (*ptep & PTE_P) {
        struct Page *page = pte2page(*ptep);
        if (page_ref_dec(page) == 0) {
            free_page(page);
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);
    }
```

代码执行步骤为：

1. 找到页表项对应的物理页帧
2. 将页的对应的被引用次数减一
3. 如果ref为0，释放该页
4. 清除页表项，并更新tlb

### 问题3.1：数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

在pmm.c的page\_init函数中：

```c
	npage = maxpa / PGSIZE;
    pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);

    for (i = 0; i < npage; i ++) {
        SetPageReserved(pages + i);
    }

    uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);
```

从中可以看出，首先计算出页的总数npage，然后分配内存pages为Page数组的头指针，并对npage项初始化。可见每一项对应物理空间的每一页。

### 问题3.2：如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？鼓励通过编程来具体完成这个问题

修改memlayout.h：

```c
#define KERNBASE     0x00000000
#define VPT          0x3AC00000
```
修改kernel.ld：

```
. = 0x00100000;
```

在pmm.c中去掉置boot_pgdir[0]为0的项，因为此时虚地址与实地址一一对应：

```
//assert(boot_pgdir[0] == 0);
//boot_pgdir[0] = 0;
```

## 你的实现与参考答案的区别

练习一中的实现与答案有很大的区别。答案中的实现将物理空间的所有空闲页面都插入了双向链表，这样在分配和释放空间时所需的开销很大。所以在我的实现中，将一块连续的物理空间的第一页插入链表，并记录这块地址空间的大小，这样在分配释放空间时执行效率更高。

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

1. 练习一涉及连续物理内存分配的first-fit算法，对应原理课的联系内存分配
2. 练习二涉及线性地址与实地址的转换过程以及页表的建立和索引
3. 练习三涉及页表的释放，与原理课相对应

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

1. 物理内存分配的其他算法，如best-fit， buddy system等
2. 反置页表



    


