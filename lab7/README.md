# 前置知识:

# 同步互斥的设计与实现
互斥是指某一个资源同时只允许一个进程对其进行访问，具有唯一性和排它性，但互斥不用限制进程对资源的访问顺序，即访问可以是无序的。同步是指在进程间的执行必须严格按照规定某种先后次序来运行，即访问是有序的，这种先后次序取决于要系统完成的任务需求。在进程写资源情况下，进程间要求满足互斥条件。在进程读资源的情况下，可允许多个进程同时访问资源。 
# 同步互斥的底层支撑
由于有处理器调度的存在，且进程在访问某类资源暂时无法满足的情况下，进程会进入等待状态。这导致了多进程执行时序的不确定性和潜在执行结果的不确定性。为了确保执行结果的正确性，本实验需要设计更加完善的进程等待和互斥的底层支撑机制，确保能正确提供基于信号量的条件变量的同步互斥机制。
根据操作系统原理的知识，我们知道如果没有在硬件级保证读内存-修改值-写回内存的原子性，我们只能通过复杂的软件来实现同步互斥操作。但由于有定时器，屏蔽/使能中断，等待队列wait_queue支持test_andr_set_bit等原子操作操作机器指令的存在，使得我们在进程瞪大，同步互斥得到极大的简化。
# 定时器
在传统的操作系统中，定时器是其中一个基础而重要的功能，它提供了基于时间事件调度机制，在ucore中，时钟中断给操作系统提供了有一定间隔的时间事件，操作系统将其作为基本的调度和计时单位，
我们记两次时间中断之间的时间间隔为一个时间片,timeer splice).
基于此时间单位，操作系统得以向上提供基于时间点的事件，并实现基于时间长度的睡眠等待和唤醒机制，在每个时钟中断发生时，操作系统产生对应的时间事件。应用程序或者操作系统的其他组件可以以此构建更复杂和高级的进程管理和调度。
sched.h,sched.c定义有关timer的各种相关接口来使用timer服务，其中主要包括:
1.typedef struct{....} timer_t:定义了timer_t的基本结构，其可以用sched.h中timer_init函数对其进行初始化。
2.void timer_init(timer t *timer,struct proc_struct *proc,int expires):对某定时器进行初始化，让它在expires时间片之后唤醒proc进程
3.void add_timer(timer t *timer):向系统添加某个初始化的timer_t，该定时器在指定时间后被激活，并将对应的进程唤醒至proc进程
4.void del_timer(timer t *timer):向系统删除或取消某一个定时器，该定时器在取消后不会被系统激活并唤醒进程
5.void run_timer_list(void):更新当前系统时间点，遍历当前所有处在系统管理内的定时器，找出所有应激活的计数器，并激活他们，该过程有且只有每次定时器中断时被调用,在ucore中，其还会调用调度器事件处理程序，
一个timer_t在系统中中的存活周期可以被描述如下:
1.timer_t在某个位置被创建和初始化，并通过add_timer加入系统
管理列表中
2.系统时间被不断累加，知道run_timer_list该timer_t到期
3.run_timer_list更改对应的进程状态，并从系统管理列表中移除该timer_t.

# 屏蔽与使能中断
根据操作系统原理的知识，我们知道如果没有在硬件级保证读内存-修改值-写入内存的原子性，我们只能通过复杂的软件来实现同步互斥操作。但由于有开关中断和test_and_set_bit等原子操作机器指令的存在，使得我们在实现同步互斥原语上可以大大简化。

在ucore中提供的底层机制包括中断屏蔽/使能控制.kern/sync.c中实现的开关中断的控制函数local_intr_save(x)和local_intr_restore(x),它们是基于kern/driver文件下的intr_enable(),intr_disable()函数实现的。具体调用关系
关中断:local_intr_save --> __intr_save --> intr_disable --> cli
开中断:local_intr_restore-->__intr_restore--> intr_enable -->sti
最终的cli和sti是x86的机器指令，最终实现了关（屏蔽）中断和开(使能)中断，即设置了eflags寄存器中与中断相关的位，通过关闭中断，可以防止对当前执行的控制流被其他中断事件的处理所打断。既然不能中断，那就意味着在内核运行的当前进程无法被打断或被重新调度，即实现了对临界区的互斥操作。所以在单处理器情况下，可以通过开关中断实现对临界区的互斥保护，需要互斥的临界区代码的一般写法为:
local_intr_save(intr_flag)
{
    临界区代码
}
local_intr_restore(intr_flag)

# 等待队列
到目前为止，我们的实验中，用户进程或内核线程还没有睡眠的支持机制，在课程中提到用户进程或内核线程可以转入等待状态以等待某个特定的事件（比如睡眠，等待子进程结束,等待信号量等）,当该事件发生时这些进程能够被再次唤醒。内核实现这一功能的一个底层支撑机制就是等待队列wait_queue，等待队列和每个事件(睡眠结束，时钟到达,任务完成,资源可用)联系起来.需要等待事件的进程在转入休眠状态后插入等待队列。当事件发生后，内核遍历相应等待队列，唤醒休眠的用户线程或内核线程，并设置其状态为就绪状态(PROC_RUNNABLE),并将该进程从等待队列中清除。ucore在kern/sync/{ wait.h,wait.c}中实现了等待项wait结构和等待队列wait_queue结构以及相关的函数),这是实现ucore中信号量机制和条件变量机制和基础，进去wait_queue的进程会被设置成等待状态(PROC_SLEEEPING),直到他们被唤醒。

## 数据结构

```
typedef  struct {
    struct proc_struct *proc;     //等待进程的指针
    uint32_t wakeup_flags;        //进程被放入等待队列的原因标记
    wait_queue_t *wait_queue;     //指向此wait结构所属于的wait_queue
    list_entry_t wait_link;       //用来组织wait_queue中wait节点的连接
} wait_t;

typedef struct {
    list_entry_t wait_head;       //wait_queue的队头
} wait_queue_t;

le2wait(le, member)  
```
# 相关函数说明

与wait和wait_queue相关函数主要分为两层，底层函数是对wait_queue的初始化，插入，删除和查找操作，相关函数如下

```
void wait_init(wait_t *wait, struct proc_struct *proc);    //初始化wait结构
bool wait_in_queue(wait_t *wait);                          //wait是否在wait queue中
void wait_queue_init(wait_queue_t *queue);                 //初始化wait_queue结构
void wait_queue_add(wait_queue_t *queue, wait_t *wait);    //把wait前插到wait queue中
void wait_queue_del(wait_queue_t *queue, wait_t *wait);    //从wait queue中删除wait
wait_t *wait_queue_next(wait_queue_t *queue, wait_t *wait);//取得wait的后一个链接指针
wait_t *wait_queue_prev(wait_queue_t *queue, wait_t *wait);//取得wait的前一个链接指针
wait_t *wait_queue_first(wait_queue_t *queue);             //取得wait queue的第一个wait
wait_t *wait_queue_last(wait_queue_t *queue);              //取得wait queue的最后一个wait
bool wait_queue_empty(wait_queue_t *queue);                //wait queue是否为空
```

高层函数基于底层函数实现了让进程进入等待队列-- wait_current_set ,以及从等待队列中唤醒进程-- wakeup_wait
，相关函数如下:

```
//让wait与进程关联，且让当前进程关联的wait进入等待队列queue，当前进程睡眠
void wait_current_set(wait_queue_t *queue, wait_t *wait, uint32_t wait_state);
//把与当前进程关联的wait从等待队列queue中删除
wait_current_del(queue, wait);
//唤醒与wait关联的进程
void wakeup_wait(wait_queue_t *queue, wait_t *wait, uint32_t wakeup_flags, bool del);
//唤醒等待队列上挂着的第一个wait所关联的进程
void wakeup_first(wait_queue_t *queue, uint32_t wakeup_flags, bool del);
//唤醒等待队列上所有的等待的进程
void wakeup_queue(wait_queue_t *queue, uint32_t wakeup_flags, bool del);

```

# 调用关系举例

如下图所示，对于唤醒进程的函数wakeup_wait,可以看到它会被各种信号量的V操作函数up调用，并且它会调用wait_queue_del函数和wakeup_proc函数来完成唤醒进程的操作。

```
digraph "wakeup_wait" {
  graph [bgcolor="#F7F5F3", fontname="Arial", fontsize="10", label="", rankdir="LR"];
  node [shape="box", style="filled", color="blue", fontname="Arial", fontsize="10", fillcolor="white", label=""];
  edge [color="#CC0044", fontname="Arial", fontsize="10", label=""];
  graph [bgcolor="#F7F5F3"];
  __N1 [color="red", label="wakeup_wait"];
  __N2 [label="wait_queue_del"];
  __N3 [label="wakeup_proc"];
  __N4 [label="__up"];
  __N5 [label="up"];
  __N6 [label="phi_test_sema"];
  __N7 [label="phi_take_forks_sema"];
  __N8 [label="cond_signal"];
  __N9 [label="phi_put_forks_sema"];
  __N10 [label="cond_wait"];
  __N11 [label="unlock_mm"];
  __N12 [label="phi_take_forks_condvar"];
  __N13 [label="phi_put_forks_condvar"];
  __N14 [label="wakeup_first"];
  __N15 [label="wakeup_queue"];
  __N1 -> __N2;
  __N1 -> __N3;
  __N6 -> __N5;
  __N7 -> __N5;
  __N8 -> __N5;
  __N9 -> __N5;
  __N10 -> __N5;
  __N11 -> __N5;
  __N12 -> __N5;
  __N13 -> __N5;
  __N5 -> __N4;
  __N4 -> __N1;
  __N14 -> __N1;
  __N15 -> __N1;
}
```
如下图所示，而对于让进程进入等待状态的函数wait_current_set，可以看到它会被各种信号量的P操作函数｀down调用，并且它会调用wait_init完成对等待项的初始化，并进一步调用wait_queue_add`来把与要处于等待状态的进程所关联的等待项挂到与信号量绑定的等待队列中。

```
digraph "wait_current_set" {
  graph [bgcolor="#F7F5F3", fontname="Arial", fontsize="10", label="", rankdir="LR"];
  node [shape="box", style="filled", color="blue", fontname="Arial", fontsize="10", fillcolor="white", label=""];
  edge [color="#CC0044", fontname="Arial", fontsize="10", label=""];
  graph [bgcolor="#F7F5F3"];
  __N1 [color="red", label="wait_current_set"];
  __N3 [label="wait_init"];
  __N4 [label="list_init"];
  __N5 [label="wait_queue_add"];
  __N6 [label="list_empty"];
  __N7 [label="list_add_before"];
  __N8 [label="__down"];
  __N9 [label="down"];
  __N10 [label="phi_take_forks_sema"];
  __N11 [label="cond_signal"];
  __N12 [label="phi_put_forks_sema"];
  __N13 [label="cond_wait"];
  __N14 [label="lock_mm"];
  __N15 [label="phi_take_forks_condvar"];
  __N16 [label="phi_put_forks_condvar"];
  __N3 -> __N4;
  __N1 -> __N3;
  __N5 -> __N6;
  __N5 -> __N7;
  __N1 -> __N5;
  __N10 -> __N9;
  __N11 -> __N9;
  __N12 -> __N9;
  __N13 -> __N9;
  __N14 -> __N9;
  __N15 -> __N9;
  __N16 -> __N9;
  __N9 -> __N8;
  __N8 -> __N1;
}
```

# 信号量
信号量是一种同步互斥机制的实现，普遍存在于现在的各种操作系统内核里，相对于spinlock的应用对象，信号量的应用对象时在临界区运行的时间较长进程。等待信号量的进程需要睡眠来减少占用cpu的开销。
```
struct semaphore {
int count;
queueType queue;
};
void semWait(semaphore s)
{
s.count--;
if (s.count < 0) {
/* place this process in s.queue */;
/* block this process */;
}
}
void semSignal(semaphore s)
{
s.count++;
if (s.count<= 0) {
/* remove a process P from s.queue */;
/* place process P on ready list */;
}
}
```
基于上诉信号量实现可以认为，当多个(>1)进程可以进行互斥或同步合作时，一个进程会由于无法满足信号量设置的某条件而在某一位置停止，直到它接收到一个特定信号（表示条件满足了），为了发信号，需要使用一个称为信号量的特殊变量，为通过信号量s传送信号，信号量的V操作采用进程的可执行原语semSignal(s);为通过信号量s接收信号，信号量的P操作采用可执行的原语semWait(s),如果相应信号仍然没有发送，则进程被阻塞或睡眠，直到发送完为止

ucore中信号量参照上述原理描述，简历在开关中断机制和wait_queue的基础上进行了具体实现。信号量的数据结构定义如下
```
typedef struct {
    int value;                   //信号量的当前值
    wait_queue_t wait_queue;     //信号量对应的等待队列
} semaphore_t;
```
semaphore_t是最基本的记录型信号量结构，包含了用于计数的整数值value，和一个进程等待队列wait_queue,一个等待的进程会挂在此等待队列上.






# 练习1
## 题目描述
理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题（不需要编码）
完成练习0后，建议大家比较一下（可用meld等文件diff比较软件）个人完成的lab6和练习0完成后的刚修改的lab7之间的区别，分析了解lab7采用信号量的执行过程。执行make grade，大部分测试用例应该通过。

请在实验报告中给出内核级信号量的设计描述，并说明其大致执行流程。

请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

