## 前置知识
内核线程是一种特殊的进程，内核进程和用户进程的区别有:
1. 内核线程只运行在内核态
2. 用户进程在会用户态和内核态交替运行
3. 所有内核线程共用ucore内核空间，不需为每个内核线程维护单独的内存空间
4. 用户进程需要维护各自的用户内存空间

为了实现内核线程，需要设计管理线程的数据结构，即进程控制块，如果要让内核线程运行，先创建对应的进程控制块，然后把这些进程控制块通过链表连在一起，便于随时进行插入，删除和查找操作等进程管理事务。然后通过调度器(scheduler)来让不同的内核线程在不同的时间段占用cpu执行，实现对cpu的分时共享。

kern_init函数中，当完成虚拟内存的初始化工作后，就调用了proc_init函数，这个函数完成了idleproc内核线程和initproc内核线程的创建或复制工作,idleproc内核线程工作就是不停地查询，看是否有其他内核线程可以执行了，如果有马上让调度器选择哪个内核线程进行执行，所以idleproc内核线程是在ucore操作系统没有其他内核线程可执行的情况下才会被调用。接着就是调用kernel_thread函数来创建initproc内核线程。initproc内核线程的工作就是显示"Hello world",表明自己存在并且能正常工作.

调度器会在特定的调度点上执行调度，完成进程切换.在lab4中，这个调度点就一出，即在cpu_idle函数中，此函数如果发现当前进程(idleproc)的need_resched为1(初始化idleproc的进程控制块时就为1了)，则调用schedule函数，完成进程调度和进程切换。进程调度的过程其实比较简单，就是在进程控制块链表中查找一个合适的内核线程，所谓合适就是指内核线程处于PROC_RUNNABLE状态。
在接下来的switch_to函数，完成具体进程切换过程.

##设计关键数据结构--进程控制块
进程管理信息用struct proc_struct，定义如下
```
struct proc_struct {
    enum proc_state state; // Process state
    int pid; // Process ID
    int runs; // the running times of Proces
    uintptr_t kstack; // Process kernel stack
    volatile bool need_resched; // need to be rescheduled to release CPU?
    struct proc_struct *parent; // the parent process
    struct mm_struct *mm; // Process's memory management field
    struct context context; // Switch here to run process
    struct trapframe *tf; // Trap frame for current interrupt
    uintptr_t cr3; // the base addr of Page Directroy Table(PDT)
    uint32_t flags; // Process flag
    char name[PROC_NAME_LEN + 1]; // Process name
    list_entry_t list_link; // Process link list
    list_entry_t hash_link; // Process hash list
};
```