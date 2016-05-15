# Lab7 Report

## 练习1：理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

### 问题1.1：给出内核级信号量的设计描述，并说其大致执行流流程。

内核级信号量的定义在`kern/sync/sem.h`中：

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```
`value`用于表示信号量中资源的整数值，`wait_queue`表示等待队列。
对于信号量存在以下几种操作：

1. `sem_init(semaphore_t *sem, int value)`: 初始化信号量，设置`value`并新建一个等待队列
2. `void up(semaphore_t *sem)`: V操作，调用`__up()`函数实现。
3. `void down(semaphore_t *sem)`: P操作，调用`__down()`函数实现。
4. `bool try_down(semaphore_t *sem)`: 非阻塞的P操作，如果信号量的`value`大于0，则直接减一。

从以上的操作中可以看出，关于信号量的核心的实现为`__up()`和`__down()`函数。下面对这两个函数做进一步分析。

```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```
V操作的执行过程分为以下步骤：

- 关中断
- 判断等待队列是否为空，若为空，将`value`值加一
- 否则，调用`wake_up`将睡眠的进程唤醒
- 开中断返回

```c
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```
P操作的执行过程分为以下步骤：

- 关中断
- 判断信号量的`value`是否大于0，如果是，则将`value`减一后开中断返回
- 否则，将当前进程加入到等待队列，开中断。执行`schedule`进行进程调度
- 如果被唤醒，则关中断
- 从等待队列中删除此进程
- 开中断并返回

### 问题1.2：给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

由于实现信号量机制需要包含开关中断的操作，所以在用户态无法直接执行，需要系统调用来完成用户态的信号量机制。可以增加与信号量相关的系统调用，比如`SYS_SEMINIT`,`SYS_UP`,`SYS_DOWN`等。

相同点：实现机制相同
不同点：用户态需要系统调用



## 练习2：完成内核级条件变量和基于内核级条件变量的哲学家就餐问题

### 问题2.1：给出内核级条件变量的设计描述，并说其大致执行流流程。

ucore中管程数据结构`monitor_t`的定义如下：

```c
typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```
其成员变量的含义及功能为：

- `mutex`: 二值信号量，实现每次只允许一个进程进入管程的信号量，确保了互斥访问性质.
- `cv`: 管程中的条件变量`cv`通过执行`wait_cv`，会使得等待某个条件C为真的进程能够离开管程并睡眠，且让其他进程进入管程继续执行；而进入管程的某进程设置条件C为真并执行`signal_cv`时，能够让等待某个条件C为真的睡眠进程被唤醒，从而继续进入管程中执行。
- `next`/`next_count`: 配合进程对条件变量`cv`的操作而设置的，这是由于发出`signal_cv`的进程A会唤醒睡眠进程B，进程B执行会导致进程A睡眠，直到进程B离开管程，进程A才能继续执行，这个同步过程是通过信号量`next`完成的；而`next_count`表示了由于发出singal_cv而睡眠的进程个数。

条件变量的数据结构`condvar_t`定义如下：

```c
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```
其成员变量的含义及功能为：

- `sem`: 用于让发出`wait_cv`操作的等待某个条件C为真的进程睡眠，而让发出`signal_cv`操作的进程通过这个sem来唤醒睡眠的进程。
- `count`: 表示等在这个条件变量上的睡眠进程的个数.
- `owner`: 表示此条件变量的宿主是哪个管程。

对于管程的两个重要的函数为`cond_wait`和`cond_signal`

```c
void
cond_wait (condvar_t *cvp) {
    //LAB7 EXERCISE1: YOUR CODE
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
   /*
    *         cv.count ++;
    *         if(mt.next_count>0)
    *            signal(mt.next)
    *         else
    *            signal(mt.mutex);
    *         wait(cv.sem);
    *         cv.count --;
    */
    cvp->count++;
    if (cvp->owner->next_count > 0) {
      up(&(cvp->owner->next));
    }
    else {
      up(&(cvp->owner->mutex));
    }
    down(&(cvp->sem));
    cvp->count--;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```
执行过程如下：

- 条件变量的`count`加一
- 如果`monitor.next_count`大于0，表示有进程执行`cond_signal`函数睡眠了，这些进程构成一个链表，需要唤醒链表中的一个进程。
- 否则，需要唤醒由于互斥而不能进入管程的进程链表中的一个进程
- 对条件变量的信号量执行P操作，请求访问资源
- 条件变量的`count`减一

```c
void 
cond_signal (condvar_t *cvp) {
   //LAB7 EXERCISE1: YOUR CODE
   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);  
  /*
   *      cond_signal(cv) {
   *          if(cv.count>0) {
   *             mt.next_count ++;
   *             signal(cv.sem);
   *             wait(mt.next);
   *             mt.next_count--;
   *          }
   *       }
   */
  if (cvp->count > 0) {
    cvp->owner->next_count++;
    up(&(cvp->sem));
    down(&(cvp->owner->next));
    cvp->owner->next_count--;
  }
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```
执行过程如下：

- 判断是否存在等待此条件的进程
- 如果有，将此进程睡眠在`cvp->owner->next`上，等待其他进程将本进程再次唤醒

### 问题2.2：给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

同信号量一样，需要封装为系统调用接口。在此不再赘述。

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

1. 信号量的基本原理与实现
2. 管程、条件变量的基本原理与实现
3. 哲学家就餐问题的模拟方法







    


