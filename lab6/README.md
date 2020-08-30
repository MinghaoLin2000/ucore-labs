## 前置知识
### 1.进程状态：
在此次实验中，进程的状态之间的转换需要有一个更为清晰的表述，在 ucore中，runnable的进程会被放在运行队列中。值得注意的是，在具体实现中，ucore定义的进程控制块struct proc_struct包含了成员变量state,用于描述进程的运行状态，而running和runnable共享同一个状态(state)值(PROC_RUNNABLE。不同之处在于处于running态的进程不会放在运行队列中。进程的正常生命周期如下：

进程首先在 cpu 初始化或者 sys_fork 的时候被创建，当为该进程分配了一个进程控制块之后，该进程进入 uninit态(在proc.c 中 alloc_proc)。
当进程完全完成初始化之后，该进程转为runnable态。
当到达调度点时，由调度器 sched_class 根据运行队列rq的内容来判断一个进程是否应该被运行，即把处于runnable态的进程转换成running状态，从而占用CPU执行。
running态的进程通过wait等系统调用被阻塞，进入sleeping态。
sleeping态的进程被wakeup变成runnable态的进程。
running态的进程主动 exit 变成zombie态，然后由其父进程完成对其资源的最后释放，子进程的进程控制块成为unused。
所有从runnable态变成其他状态的进程都要出运行队列，反之，被放入某个运行队列中。
### 2.内核抢占点
ucore操作系统也可以看成一个特殊的内核进程或多个内核线程的集合，其实ucore是可抢占的，cpu控制权可是被强制剥夺，比如以下几种固定的情况是例外：
1.进行同步互斥操作，比如争抢一个信号量，锁
2.进行磁盘读写等耗时的异步操作，由于等待完成的耗时太长，ucore会调用shecedule让其他就绪进程执行
这几种情况其实都是由于当前进程所需的某个资源无法得到满足，无法继续执行下去，从而不得不主动放弃对cpu的控制权的情况，如果参照用户进程任意位置都可被内核打断并放弃CPU控制权的情况，这些在内核中放弃CPU控制权的执行地点是固定而不是任意的，不能体现内核任意位置都可抢占性的特点。
### 设计思路
总体就三个步骤，选择出合适的进程，并入就绪进程队列，离开就绪进程队列。
然后在调度器中形成了一个timer时间事件感知操作。在运行过程中，调度器可以调整进程控制块中与进程调度的属性（消耗时间片，进程优先级），有可能重排队列。

### RR 调度算法实现
RR调度算法的调度思想 是让所有runnable态的进程分时轮流使用CPU时间。RR调度器维护当前runnable进程的有序运行队列。当前进程的时间片用完之后，调度器将当前进程放置到运行队列的尾部，再从其头部取出进程进行调度。RR调度算法的就绪队列在组织结构上也是一个双向链表，只是增加了一个成员变量，表明在此就绪进程队列中的最大执行时间片。而且在进程控制块proc_struct中增加了一个成员变量time_slice，用来记录进程当前的可运行时间片段。这是由于RR调度算法需要考虑执行进程的运行时间不能太长。在每个timer到时的时候，操作系统会递减当前执行进程的time_slice，当time_slice为0时，就意味着这个进程运行了一段时间（这个时间片段称为进程的时间片），需要把CPU让给其他进程执行，于是操作系统就需要让此进程重新回到rq的队列尾，且重置此进程的时间片为就绪队列的成员变量最大时间片max_time_slice值，然后再从rq的队列头取出一个新的进程执行。下面来分析一下其调度算法的实现。

RR_enqueue的函数实现如下表所示。即把某进程的进程控制块指针放入到rq队列末尾，且如果进程控制块的时间片为0，则需要把它重置为rq成员变量max_time_slice。这表示如果进程在当前的执行时间片已经用完，需要等到下一次有机会运行时，才能再执行一段时间。
```
static void
RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list), &(proc->run_link));
    if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
        proc->time_slice = rq->max_time_slice;
    }
    proc->rq = rq;
    rq->proc_num ++;
}
```
RR_pick_next的函数实现如下表所示。即选取就绪进程队列rq中的队头队列元素，并把队列元素转换成进程控制块指针。
```
static struct proc_struct *
FCFS_pick_next(struct run_queue *rq) {
    list_entry_t *le = list_next(&(rq->run_list));
    if (le != &(rq->run_list)) {
        return le2proc(le, run_link);
    }
    return NULL;
}
```
RR_dequeue的函数实现如下表所示。即把就绪进程队列rq的进程控制块指针的队列元素删除，并把表示就绪进程个数的proc_num减一。
```
static void
FCFS_dequeue(struct run_queue *rq, struct proc_struct *proc) {
    assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
    list_del_init(&(proc->run_link));
    rq->proc_num --;
}
```
RR_proc_tick的函数实现如下表所示。即每次timer到时后，trap函数将会间接调用此函数来把当前执行进程的时间片time_slice减一。如果time_slice降到零，则设置此进程成员变量need_resched标识为1，这样在下一次中断来后执行trap函数时，会由于当前进程程成员变量need_resched标识为1而执行schedule函数，从而把当前执行进程放回就绪队列末尾，而从就绪队列头取出在就绪队列上等待时间最久的那个就绪进程执行。
```
static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
    if (proc->time_slice > 0) {
        proc->time_slice --;
    }
    if (proc->time_slice == 0) {
        proc->need_resched = 1;
    }
```
## 练习一:使用Round Robin 调度算法
完成练习0后，建议大家比较一下（可用kdiff3等文件比较软件）个人完成的lab5和练习0完成后的刚修改的lab6之间的区别，分析了解lab6采用RR调度算法后的执行过程。执行make grade，大部分测试用例应该通过。但执行priority.c应该过不去。
请在实验报告中完成：

请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描ucore的调度执行过程。

请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计。
1.初始化进程队列(RR_init函数)
```
static void RR_init(struct run_queue *rq)
{
    list_init(&(rq->run_list));//初始化运行队列
    rq->proc_num=0; //初始化进程数为0
}
```
其中的run_queue结构体
```
struct run_queue{
    //运行队列
    list_entry_t run_list;
    //内部进程总数
    unsigned int proc_num;
    //每个进程一轮占用的最多时间片
    int max_time slice;
    //优先队列形式的进程容器
    skew_heap_entry_t *lab6_run_pool;
}
结构体中的skew_heap_entry结构体
struct skew_heap_entry{
    struct skew_heap_entry *parent,*left,*right;
}
typed struct skew_heap_entry skew_heap_entry_t;
从代码可以看出RR_init函数，函数比较简单，完成了对进程队列初始化。
2.将进程加入就绪队列(RR_enqueue函数)
```
static void RR_enqueue(struct run_queue *rq,struct proc_struct *proc)
{
    //进程控制块指针非空
    assert(list_empty(&(proc->run_link)));
    //把进程的PCB指针放入到rq队列末尾
    list_add_before(&(rq->run_list),&(proc->run_link));
    //PCB的时间片为0或者进程的时间片大于分配给进程的最大时间片
    if(proc->time_slice==0||proc->time_slice>rq->max_time_slice)
    {
        proc->time_slice=rq->max_time_slice;//修改时间片
    }
    pro->rq=rq; //加入进程池
    rq->proc_num ++; //就绪进程数加一

}
3 将进程从就绪队列中移除(RR_dequeue函数)
```
static void RR_dequeue(struct run_queue *rq,struct proc_struct *proc)
{
    //PCB指针非空并且进程在就绪队列中
    assert(!list_empty(&(proc->run_link))&&proc->rq==rq);
    //将PCB指针从就绪队列中
    list_del_init(&(proc->run_link));
    rq->proc_num--; //就绪进程数减一
}
```
4.选择下一调度进程(RR_pick_next函数)
```
static void proc_struct **RR_pick_next(struct run_queue *rq)
{
    //选取就绪进程队列rq中的队头
    list_entry_t *le=list_next(&(rq->run_list));
    if(le!=&(rq->run_list)) //取得就绪进程
    {
        return le2proc(le,run_link);//返回PCB指针
    }
    return NULL;
}
```
这里感觉是暗示不可以一个进程使用这个函数，因为是先取当前进程的下一个进程，再判断是否是同一个进程，如果不是，则就是下一个队头
5 时间片（RR_proc_tick函数）

```
static void RR_proc_tick(struct run_queue *rq,struct proc_struct *proc)
{
    if(proc->time_slice>0)
    {
        proc->time_slice--;
    }
    if(pro->time_slice==0)
    {
        //设置此进程成员变量need_resched标识为1，进程需要调度
        proc->need_resched=1;
    }
}
```
6 sched_class 
struct sched_class default_sched_class={
    .name="RR_scheduler",
    .init=RR_init,
    .enqueue=RR_enqueue,
    .dequeue=RR_dequeue,
    .pick_next=RR_pick_next,
    .proc_tick=RR_proc_tick,
}
提供了调度算法的切换接口

7.问题
sched_class
```
struct sched_class{
    //调度器名字
    const char* name;
    //初始化运行队列
    void (*init)(struct run_queue *rq);
    //将进程p插入队列rq
    void ((*enqueue)(struct run_queue *rq,struct proc_struct *p);
    //将进程p从队列rq中删除
    void ((*dequeue)(struct run_queue *rq,struct proc_struct *p);
    //返回运行队列中下一个可执行的进程
    struct proc_struct* (*pick_next) (struct run_queue *rq);
    //timetick处理函数
    void (*proc_tick)(struct run_queue* rq,struct proc_struct *p);
};
调度执行过程:
即每次timer到时后，trap函数将会间接调用此函数来把当前执行进程的时间片time_slice减一。如果time_slice降到零，则设置此进程成员变量need_resched标识为1，这样在下一次中断来后执行trap函数时，会由于当前进程程成员变量need_resched标识为1而执行schedule函数，从而把当前执行进程放回就绪队列末尾，而从就绪队列头取出在就绪队列上等待时间最久的那个就绪进程执行。
练习2：实现Stride Scheduling 调度算法
相比于RR调度,Stride Scheduling函数定义了一个比较器
static int proc_stride_comp(void *a,void *b)
{
    //通过PCB指针取得进程a
    struct proc_struct *p=le2proc(a,lab6_run_pool);
    //通过PCB指针取得进程b
    struct proc_struct *q=le2proc(b,la6_run_pool);
    //步数相减，通过正负比较大小关系
    int32_t c=p->lab6_stride-q->lab6_stride;
    if(c>0) return 1;
    else if(c==0) return 0;
    else return -1;
}
1 初始化运行队列
static void stride_init(struct run_queue *rq)
{
    list_init(&(rq->run_list));//初始化调度器类
    rq->lab6_run_pool=NULL;//初始化当前进程运行队列为空
    rq->proc_num=0;//设置运行队列为空
}
2 将进程加入就绪队列
static void stride_enqueue(struct run_queue *rq,struct proc_struct *proc)
{
    #if USER_SKEW_HEAP
    rq->lab6_run_pool=skew_heap_insert(rq->lab6_run_pool,&(proc->lab6_run_
    #else
    assert(list_empty(&(proc->run_link)));
    list_add_before(&(rq->run_list),&(proc->run_link));
    #endif
    if(proc->time_slice==0||proc->time_slice>rq->max_time_slice)
    {
        proc->time_slice=rq->max_time_slice;
    }
    proc->rq=rq;
    rq->proc_num++;
}
在ucore中USE_SKEW_HEAP定义为1，因此# else与#endif代码会被忽略.
其中skew_heap_insert函数
static inline skew_heap_entry_t* skew_heap_insert(skew_heap_entry_t *q,skew_heap_entry_t *b,compare_f comp)
{
   skew_heap_init(b);//初始化进程b
   return skew_heap_merge(a,b,comp);//返回a与b进程结合的结果
}
函数 skew_heap_init函数
static inline void skew_heap_init(skew_heap_entry_t *a)
{
    a->left=a->right=a->parent=NULL;
}
函数中的skew_heap_merge函数
static inline skew_heap_entry_t *
skew_heap_merge(skew_heap_entry_t *a, skew_heap_entry_t *b,
                compare_f comp)
{
     if (a == NULL) return b; 
     else if (b == NULL) return a;

     skew_heap_entry_t *l, *r;
     if (comp(a, b) == -1) //a进程的步长小于b进程
     {
          r = a->left; //a的左指针为r
          l = skew_heap_merge(a->right, b, comp);

          a->left = l;
          a->right = r;
          if (l) l->parent = a;

          return a;
     }
     else
     {
          r = b->left;
          l = skew_heap_merge(a, b->right, comp);

          b->left = l;
          b->right = r;
          if (l) l->parent = b;

          return b;
     }
}
3 将进程从就绪队列中移除
static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE */
     rq->lab6_run_pool =
          skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
     rq->proc_num --;
}
4 选择进程调度
static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
     /* LAB6: YOUR CODE */
#if USE_SKEW_HEAP
     if (rq->lab6_run_pool == NULL) return NULL;
     struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
#else
     list_entry_t *le = list_next(&(rq->run_list));

     if (le == &rq->run_list)
          return NULL;

     struct proc_struct *p = le2proc(le, run_link);
     le = list_next(le);
     while (le != &rq->run_list)
     {
          struct proc_struct *q = le2proc(le, run_link);
          if ((int32_t)(p->lab6_stride - q->lab6_stride) > 0)
               p = q;
          le = list_next(le);
     }
#endif
     if (p->lab6_priority == 0) //优先级为0
          p->lab6_stride += BIG_STRIDE; //步长设置为最大值
   //步长设置为优先级的倒数
     else p->lab6_stride += BIG_STRIDE / p->lab6_priority;
     return p;
}
5 时间片部分
static void
stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE */
    if (proc->time_slice > 0) {  //到达时间片
        proc->time_slice --; //执行进程的时间片time_slice减一
    }  
    if (proc->time_slice == 0) { //时间片为0
     //设置此进程成员变量need_resched标识为1,进程需要调度
        proc->need_resched = 1; 
    }  
}