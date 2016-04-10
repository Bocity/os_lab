# Lab4 Report

## 练习1：分配并初始化一个进程控制块

在kern/process/proc.c中完成alloc\_proc()函数。代码为：

```c
   proc->state = PROC_UNINIT;
   proc->pid = -1;
   proc->runs = 0;
   proc->kstack = 0;
   proc->need_resched = 0;
   proc->parent = NULL;
   proc->mm = NULL;
   memset(&(proc->context), 0, sizeof(struct context));
   proc->tf = NULL;
   proc->cr3 = boot_cr3;
   proc->flags = 0;
   memset(proc->name, 0, PROC_NAME_LEN);
```

该函数主要对state/pid/runs/kstack/need\_resched/parent/mm/context/tf/cr3/flags/name等变量进行初始化操作。



### 问题1.1：请说明proc\_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？

context作用时在进行上下文切换的过程中，保存当前寄存器的值。其定义在kern/process/proc.h中：

```c
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```

trapframe *tf用于记录发生中断之前的栈帧的内容，其中一部分为硬件保存。定义在kern/trap/trap.h中：

```c
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```


## 练习2：为新创建的内核线程分配资源

在kern/process/proc.c中完成do\_fork()函数，增加以下代码：

```c
	proc = alloc_proc();
	if(proc == NULL) goto fork_out;

    proc->parent = current;

    if(setup_kstack(proc) != 0) goto bad_fork_cleanup_proc;
    if(copy_mm(clone_flags, proc) != 0) goto bad_fork_cleanup_kstack;
    copy_thread(proc, stack, tf);

    bool flag;
    local_intr_save(flag);
    {
    proc->pid = get_pid();
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    nr_process++;
    }
    local_intr_restore(flag);

    wakeup_proc(proc);
    ret = proc->pid;
```

代码执行步骤为：

1. 调用alloc\_proc，首先获得一块用户信息块
2. 为进程分配一个内核栈
3. 复制原进程的内存管理信息到新进程
4. 复制原进程上下文到新进程
5. 将新进程添加到进程列表
6. 唤醒新进程
7. 返回新进程号

### 问题2.1：请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

是。在do\_fork函数中，通过get\_pid函数为新进程分配一个pid。而在get\_pid的实现中，通过遍历进程链表，找到一个唯一的pid返回。


## 练习3：阅读代码，理解proc\_run函数和它调用的函数如何完成进程切换的.

代码的执行过程如下：

1. 首先判断需要切换的进程是否为当前进程
2. 关中断
3. 设置current为需要运行的进程
4. 切换栈
5. 设置CR3寄存器
6. 保存恢复通用寄存器
7. 开中断

### 问题3.1：在本实验的执行过程中，创建且运行了几个内核线程？

两个。第一个为idle线程，第二个为init线程

### 问题3.2：语句local\_intr\_save(intr\_flag);....local\_intr\_restore(intr\_flag);在这里有何作用?请说明理由

用于关中断和开中断。使在切换进程的过程中不被中断打断。

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

1. 内核线程的创建
2. 进程控制块的初始化

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

用户线程和进程未涉及，相信在之后的实验中会有涉及。



    


