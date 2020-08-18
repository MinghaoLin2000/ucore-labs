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

● mm：内存管理的信息，包括内存映射列表、页表指针等。mm成员变量在lab3中用于虚存管理。但在实际OS中，内核线程常驻内存，不需要考虑swap page问题，在lab5中涉及到了用户进程，才考虑进程用户内存空间的swap page问题，mm才会发挥作用。所以在lab4中mm对于内核线程就没有用了，这样内核线程的proc_struct的成员变量*mm=0是合理的。mm里有个很重要的项pgdir，记录的是该进程使用的一级页表的物理地址。由于*mm=NULL，所以在proc_struct数据结构中需要有一个代替pgdir项来记录页表起始地址，这就是proc_struct数据结构中的cr3成员变量。

● state：进程所处的状态。

● parent：用户进程的父进程（创建它的进程）。在所有进程中，只有一个进程没有父进程，就是内核创建的第一个内核线程idleproc。内核根据这个父子关系建立一个树形结构，用于维护一些特殊的操作，例如确定某个进程是否可以对另外一个进程进行某种操作等等。

● context：进程的上下文，用于进程切换（参见switch.S）。在 uCore中，所有的进程在内核中也是相对独立的（例如独立的内核堆栈以及上下文等等）。使用 context 保存寄存器的目的就在于在内核态中能够进行上下文之间的切换。实际利用context进行上下文切换的函数是在kern/process/switch.S中定义switch_to。

● tf：中断帧的指针，总是指向内核栈的某个位置：当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态。当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。除此之外，uCore内核允许嵌套中断。因此为了保证嵌套中断发生时tf 总是能够指向当前的trapframe，uCore 在内核栈上维护了 tf 的链，可以参考trap.c::trap函数做进一步的了解。

● cr3: cr3 保存页表的物理地址，目的就是进程切换的时候方便直接使用 lcr3实现页表切换，避免每次都根据 mm 来计算 cr3。mm数据结构是用来实现用户空间的虚存管理的，但是内核线程没有用户空间，它执行的只是内核中的一小段代码（通常是一小段函数），所以它没有mm 结构，也就是NULL。当某个进程是一个普通用户态进程的时候，PCB 中的 cr3 就是 mm 中页表（pgdir）的物理地址；而当它是内核线程的时候，cr3 等于boot_cr3。而boot_cr3指向了uCore启动时建立好的饿内核虚拟空间的页目录表首地址。

● kstack: 每个线程都有一个内核栈，并且位于内核地址空间的不同位置。对于内核线程，该栈就是运行时的程序使用的栈；而对于普通进程，该栈是发生特权级改变的时候使保存被打断的硬件信息用的栈。uCore在创建进程时分配了 2 个连续的物理页（参见memlayout.h中KSTACKSIZE的定义）作为内核栈的空间。这个栈很小，所以内核中的代码应该尽可能的紧凑，并且避免在栈上分配大的数据结构，以免栈溢出，导致系统崩溃。kstack记录了分配给该进程/线程的内核栈的位置。主要作用有以下几点。首先，当内核准备从一个进程切换到另一个的时候，需要根据kstack 的值正确的设置好 tss （可以回顾一下在实验一中讲述的 tss 在中断处理过程中的作用），以便在进程切换以后再发生中断时能够使用正确的栈。其次，内核栈位于内核地址空间，并且是不共享的（每个线程都拥有自己的内核栈），因此不受到 mm 的管理，当进程退出的时候，内核能够根据 kstack 的值快速定位栈的位置并进行回收。uCore 的这种内核栈的设计借鉴的是 linux 的方法（但由于内存管理实现的差异，它实现的远不如 linux 的灵活），它使得每个线程的内核栈在不同的位置，这样从某种程度上方便调试，但同时也使得内核对栈溢出变得十分不敏感，因为一旦发生溢出，它极可能污染内核中其它的数据使得内核崩溃。如果能够通过页表，将所有进程的内核栈映射到固定的地址上去，能够避免这种问题，但又会使得进程切换过程中对栈的修改变得相当繁琐。感兴趣的同学可以参考 linux kernel 的代码对此进行尝试。

为了管理系统中所有的进程控制块，uCore维护了如下全局变量（位于kern/process/proc.c）：

● static struct proc *current：当前占用CPU且处于“运行”状态进程控制块指针。通常这个变量是只读的，只有在进程切换的时候才进行修改，并且整个切换和修改过程需要保证操作的原子性，目前至少需要屏蔽中断。可以参考 switch_to 的实现。

● static struct proc *initproc：本实验中，指向一个内核线程。本实验以后，此指针将指向第一个用户态进程。

● static list_entry_t hash_list[HASH_LIST_SIZE]：所有进程控制块的哈希表，proc_struct中的成员变量hash_link将基于pid链接入这个哈希表中。

● list_entry_t proc_list：所有进程控制块的双向线性列表，proc_struct中的成员变量list_link将链接入这个链表中。
```

##创建并执行内核线程
建立进程控制块(alloc_proc),现在就可以通过进程控制块来创建具体的进程/线程了。首先,考虑最简单的内核线程,它通常只是内核中的一小段代码或者函数,没有自己的专属空间.这是由于在uCore OS启动后，已经对整个内核内存空间进行了管理，通过设置页表建立了内核虚拟空间(即boot_cr3指向的二级页表描述的空间)。所以ucore os内核中的所有线程都不需要再建立各自的页表，只需共享这个内核虚拟空间就可以访问整个物理内存，从这个角度看，，内核线程被ucore os内核这个大内核进程呢所管理。

练习一.
```
static struct proc_start * alloc_proc(void)
{
    struct proc_start *proc=kmalloc(sizeof(struct proc_struct));
    if(proc!=NULL)
    {
        proc->state=PROC_UNINIT;//进程为初始化状态
        proc->pid=-1; //进程pid为-1
        proc->runs=0; //初始化时间片
        proc->kstack=0; //内核栈地址
        proc->need_resched=0;//不需要调度
        proc->parent=NULL;//父进程为空
        proc->mm=NULL; //虚拟内存为空
        memeset(&(proc->context),0,sizeof(struct context)); 初始化上下文
        proc->tf=NULL; //中断帧指针为空
        pro->cr3=boot_cr3;//页目录为内核页目录表基址
        proc->flags=0; //标志位为0
        memset(proc->name,0,PROC_NAME_LEN);//进程名为0
    }
    return proc;
}
context作用:
进程的上下文，用于进程的切换。主要保存了前一个进程的现场,在ucore中，所有的进程在内核中也是相对独立的。使用context保存寄存器的目的就在于在内核态中能够进行上下文的切换。
tf：中断帧的指针，总是指向内核栈的某个位置：当进程从用户空间跳回内核空间时，中断帧记录了进程在被中断前的状态。当内核需要跳回用户空间时，需要调整中断帧以恢复进程继续执行各寄存器的值，除此之外，ucore内核允许嵌套中断，因此为了保证嵌套中断发生时tf总是能指向当前的trapframe,ucore在内核栈上维护了tf的链

练习2 do_fork()函数的实现
```
int do_fork(uint32_t clone_flags,uintptr_t stack,struct trapframe *tf)
{
    int ret=-E_NO_FREE_PROC;
    struct proc_struct *proc; //定义新的进程
    if(nr_process>=MAX_PROCESS)
    {
        goto fork_out;
    }
    ret=-E_NO_MEM;
    if((proc=alloc_proc())==NULL) //分配内存失败
    {
        goto fork_out;
    }
    pro->parent=current;
    if(setup_kstack(proc)){  //分配内核栈
        goto bad_fork_cleanup_proc;
    }
    if(copy_mm(clone_flags,proc)!=0) //复制父进程的内存信息
    {
        goto bad_fork_cleannup_kstack;//返回
    }
    copy_thread(proc,stack,tf);//复制中断帧和上下文信息
    bool intr_flag;
    local_intr_save(intr_flag); //屏蔽中断,intr_flag=1
    {
        proc->pid=get_pid();//获取当前进程PID
        hash_proc(proc);//建立hash映射
        list_ad(&proc_list,&(proc->list_link);
        nr_process++;//进程数加一
    }
    local_intr_restore(intr_flag);//恢复中断
    wakeup_proc(proc);//唤醒新进程
    ret=proc->pid;
    fork_out:
    return ret;
    bad_fork_cleanup_kstack:
    put_kstack(proc);
    bad_fork_cleannup_proc:
    kfree(proc);

}
在使用fork或clone系统调用是，产生的进程均会有内核分配一个唯一的pid，分配过程不允许中断。
练习3：
分析schedule函数，这里使用的FIFO调度算法
```
/* 宏定义:
   #define le2proc(le, member)         \
    to_struct((le), struct proc_struct, member)*/
void
schedule(void) {
    bool intr_flag; //定义中断变量
    list_entry_t *le, *last; //当前list，下一list
    struct proc_struct *next = NULL; //下一进程
    local_intr_save(intr_flag); //中断禁止函数
    {
        current->need_resched = 0; //设置当前进程不需要调度
      //last是否是idle进程(第一个创建的进程),如果是，则从表头开始搜索
      //否则获取下一链表
        last = (current == idleproc) ? &proc_list : &(current->list_link);
        le = last; 
        do { //一直循环，直到找到可以调度的进程
            if ((le = list_next(le)) != &proc_list) {
                next = le2proc(le, list_link);//获取下一进程
                if (next->state == PROC_RUNNABLE) {
                    break; //找到一个可以调度的进程，break
                }
            }
        } while (le != last); //循环查找整个链表
        if (next == NULL || next->state != PROC_RUNNABLE) {
            next = idleproc; //未找到可以调度的进程
        }
        next->runs ++; //运行次数加一
        if (next != current) {
            proc_run(next); //运行新进程,调用proc_run函数
        }
    }
    local_intr_restore(intr_flag); //允许中断
}
```
而switch_to 函数主要完成上下文的切换，先保存上下文的寄存器值，再将下一个进程的上下文换进来。
proc_函数主要是修改内核栈