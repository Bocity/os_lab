# Lab6 Report

## 练习1：使用 Round Robin 调度算法

### 问题1.1：请理解并分析`sched_calss`中各个函数指针的用法，并结合Round Robin调度算法描ucore的调度执行过程。

`sched_class`的定义在`kern/schedule/sched.h`中：

```c
struct sched_class {
    // the name of sched_class
    const char *name;
    // Init the run queue
    void (*init)(struct run_queue *rq);
    // put the proc into runqueue, and this function must be called with rq_lock
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    // get the proc out runqueue, and this function must be called with rq_lock
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    // choose the next runnable task
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    // dealer of the time-tick
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
    /* for SMP support in the future
     *  load_balance
     *     void (*load_balance)(struct rq* rq);
     *  get some proc from this rq, used in load_balance,
     *  return value is the num of gotten proc
     *  int (*get_proc)(struct rq* rq, struct proc* procs_moved[]);
     */
}
```
在`sched.c`中，实例化一个`sched_class`对象为`sched_class = &default_sched_class`，并进行初始化。在后续进行进程选择和切换的时候，分别调用`sched_class_enqueue` `sched_class_pick_next` `sched_class_dequeue`函数调用`sched_class`类中的函数指针，执行相应的功能。

在Round Robin调度算法中，通过将`default_sched_class`指向为相应的实现，即可使用RR调度算法。

```c
struct sched_class default_sched_class = {
    .name = "RR_scheduler",
    .init = RR_init,
    .enqueue = RR_enqueue,
    .dequeue = RR_dequeue,
    .pick_next = RR_pick_next,
    .proc_tick = RR_proc_tick,
};
```

### 问题1.2：请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

1. 首先维护多个链表，每个链表保存优先级不同的就绪队列
2. 进程在初始化时首先插入优先级最高的队列等待
3. 在调度时，从优先级最高的队列中开始寻找进程，若为空则进入下一优先级队列寻找
4. 对于某个进程，如果在规定的时间片内没有完成，则将其插入下一优先级队列的链表中


## 练习2：实现 Stride Scheduling 调度算法

实现在`kern/schedule/default_sched.c`中的五个函数，实现如下：

- `stride_init`

```c
static void
stride_init(struct run_queue *rq) {
     /* LAB6: YOUR CODE 
      * (1) init the ready process list: rq->run_list
      * (2) init the run pool: rq->lab6_run_pool
      * (3) set number of process: rq->proc_num to 0       
      */
      list_init(&(rq->run_list));
      rq->lab6_run_pool = NULL;
      rq->proc_num = 0;
}
```
进行初始化操作，将运行队列置空，同时将`proc_num`置零。

- `stride_enqueue`

```c
static void
stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE 
      * (1) insert the proc into rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_insert: insert a entry into skew_heap
      *         list_add_before: insert  a entry into the last of list   
      * (2) recalculate proc->time_slice
      * (3) set proc->rq pointer to rq
      * (4) increase rq->proc_num
      */
#if USE_SKEW_HEAP
    rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
#else
    list_add_before(&(rq->run_list), &(proc->run_link));
#endif
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num++;
}
```
这里需要根据是否使用优先级队列来进行不同的操作，如果使用，直接插入，否则插入到链表末端。将`proc->rq`置为当前`rq`，同时将`proc_num`加一。

- `stride_dequeue`

```c
static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE 
      * (1) remove the proc from rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_remove: remove a entry from skew_heap
      *         list_del_init: remove a entry from the  list
      */
#if USE_SKEW_HEAP
    rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
#else
    list_del_init(&(proc->run_link));
#endif
    rq->proc_num--;
}
```

将进程从数据结构中删除，同时`proc_num`减一。

- `stride_pick_next`

```c
static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
     /* LAB6: YOUR CODE 
      * (1) get a  proc_struct pointer p  with the minimum value of stride
             (1.1) If using skew_heap, we can use le2proc get the p from rq->lab6_run_poll
             (1.2) If using list, we have to search list to find the p with minimum stride value
      * (2) update p;s stride value: p->lab6_stride
      * (3) return p
      */
#if USE_SKEW_HEAP
    if (!rq->lab6_run_pool) {
        return NULL;
    }
    struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
#else
    list_entry_t *le = list_next(&(rq->run_list));
    if(le == &(rq->run_list)) {
        return NULL;
    }

    struct proc_struct *p = le2proc(le, run_link);
    for (le = list_next(le); le != &(rq->run_list); le = list_next(le)) {
        struct proc_struct *p1 = le2proc(le, run_link);
        if((int32_t)(p->lab6_stride - p1->lab6_stride) < 0) {
            p = p1;
        }
    }
#endif
    if(p->lab6_priority == 0) {
        p->lab6_stride += BIG_STRIDE;
    }
    else {
        p->lab6_stride += BIG_STRIDE / p->lab6_priority;
    }
    return p;
}
```
首先需要针对不同的数据结构，找到下一个选择的进程。如果是优先级队列，堆顶端的进程即为结果，否则需要遍历链表，找到`lab6_stride`最小的进程。然后根据算法将`lab6_stride`加上`BIG_STRIDE / p->lab6_priority`.

- `stride_proc_tick`

```c
static void
stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE */
    if(proc->time_slice > 0) {
        proc->time_slice--;
    }
    if(proc->time_slice == 0) {
        proc->need_resched = 1;
    }
}
```
判断当前进程的时间片个数是否大于零，如果是则减一，否则将`need_resched`置1，在下次时钟中断时进行切换进程。

执行`make run-priority`，得到的结果如下：

```
kernel_execve: pid = 2, name = "priority".main: fork ok,now need to wait pids.child pid 6, acc 740000, time 1001child pid 7, acc 912000, time 1002child pid 4, acc 380000, time 1003child pid 5, acc 560000, time 1003child pid 3, acc 196000, time 1004main: pid 3, acc 196000, time 1005main: pid 4, acc 380000, time 1005main: pid 5, acc 560000, time 1005main: pid 6, acc 740000, time 1005main: pid 7, acc 912000, time 1005main: wait pids overstride sched correct result: 1 2 3 4 5all user-mode processes have quit.```
可以看到，程序可以正常运行。

## 实现与参考答案的区别
参考答案在时钟中断时没有调用`sched_class_proc_tick`，所以存在一定的问题。我的视线中通过`run_timer_list`接口实现了调用的功能。


## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

1. 调度算法的视线
2. 优先级队列的作用





    


