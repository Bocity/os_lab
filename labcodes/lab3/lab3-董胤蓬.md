# Lab3 Report

## 练习1：给未被映射的地址映射上物理页

在kern/mm/vmm.c中完成do\_pgfault()函数，给未被映射的地址映射物理页。添加的代码为：

```c
	ptep = get_pte(mm->pgdir, addr, 1);
    if(ptep == NULL) {
        cprintf("do_pgfault failed: get_pte error!\n");
        goto failed;
    }

    if(*ptep == 0) {
        if(pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("do_pgfault failed: pgdir_alloc_page failed!\n");
            goto failed;
        }
    }
    else{
    
    }
```

执行过程主要分为以下步骤：

1. 根据虚拟地址获得pte
2. 如果pte为空，则报错
3. 如果pte内容为0，则分配相应的物理页

### 问题1.1：请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

提供虚拟地址到物理地址的转换，用于判断相应的物理地址是否合法、物理页是否在内存中以及相应的访问权限

###问题1.2：如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

硬件产生中断，再次条用缺页服务例程。

## 练习2：补充完成基于FIFO的页面替换算法

在do\_pgfault()函数中增加以下代码：

```c
	else{
        if(swap_init_ok) {
            struct Page *page = NULL;
            if(ret = swap_in(mm, addr, &page) != 0) {
                cprintf("do_pgfault failed: swap_in failed\n");
                goto failed;
            }
            page_insert(mm->pgdir, page, addr, perm);
            swap_map_swappable(mm, addr, page, 1);
            page->pra_vaddr = addr;
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
    }
```

代码执行步骤为：

1. 如果物理页在外存中，换入相应的页
2. 更新页表的映射关系
3. 设置页可交换
4. 更新页的虚拟地址

在执行缺页换入的FIFO算法时，在kern/mm/swap\_fifo.c中，修改\_fifo\_map\_swappable()函数和\_fifo\_swap\_out\_victim()函数：

```c
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && head != NULL);
    //record the page access situlation
    /*LAB3 EXERCISE 2: YOUR CODE*/ 
    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
    list_add(head, entry);
    return 0;
}
/*
 *  (4)_fifo_swap_out_victim: According FIFO PRA, we should unlink the  earliest arrival page in front of pra_list_head qeueue,
 *                            then set the addr of addr of this page to ptr_page.
 */
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
         assert(head != NULL);
     assert(in_tick==0);
     /* Select the victim */
     /*LAB3 EXERCISE 2: YOUR CODE*/ 
     //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
     //(2)  set the addr of addr of this page to ptr_page
     list_entry_t *out = head->prev;
     assert(out != head);
     struct Page *page = le2page(out, pra_page_link);
     list_del(out);
     assert(page != NULL);
     *ptr_page = page; 
     return 0;
}
```
即每次在链表头部插入相应的物理页，删除链表末尾的页，维护FIFO队列。





### 问题2.1：如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap\_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题

- 需要被换出的页的特征是什么？
- 在ucore中如何判断具有这样特征的页？
- 何时进行换入和换出操作？

现有的框架不足以支持extended clock替换算法。原因如下：现有的框架中没有动态修改访问页面的函数。需要在现有的框架中加入访问某页时的函数，动态地修改页面的访问位和修改位。

- extended clock算法的执行过程为：将被遍历的节点的访问位或修改位清零，直到找到全为0的页替换。被换出的页在相应的时间间隔中，没有被访问过或修改过。如果都被访问或修改过，则extended clock算法会遍历一遍后找到最开始的页
- 在ucore中，通过从对应位置遍历，判断相应页的标志位来找到这样的页
- 在缺页异常发生时进行换入换出

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

1. 页访问异常的几种情况，以及不同的处理方式
2. FIFO页替换算法

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

1. 没有探讨Belady现象
2. 对虚拟存储的概念的理解不够，只有页缺失的内容



    


