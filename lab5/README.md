相关知识
## 内核线程与用户进程的区别
```
内核线程只运行内核态，用户进程会在用户态和内核态交替运行
所有内核线程共用ucore内核内存空间，不需为每个内核线程维护内存空间，但是用户进程是需要各自的内存空间的
```
---
## 用户态向内核态的转换

通常操作系统会采用软中断或者叫做trap的方式完成，实际上，发生中断时，已经实现了从用户态切换到内核态，为了实现这种切换，我们需要建立好中断们，中断门的中断描述表，制定出了中断发生后的跳转至何处，并且发生中断时我们必须保存ss，esp等信息。 但是，中断会根据保存的这些信息返回到用户态中，为了实现停留在内核态，我们对cs进行修改，将其指向内核态的代码段，其次，我们将CS的CPL设为0，在此处还需要根据要执行的指令修改EIP，这样最后执行IRET指令时，CPU会将堆栈信息取出，并返回到EIP以及CS所指内容去执行，从而便实现了从ring3到ring0的转换
![blockchain](https://img-blog.csdnimg.cn/20190908121745220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpbmdkaW5nZG9kbw==,size_16,color_FFFFFF,t_70 "xxx")
为了实现特权级的切换，实际上还需要访问TSS任务状态段，简单来说，任务状态段就是内存中的一个数据结构。这个结构中保存着和任务相关的信息。当发生任务切换的时候会把当前任务用到的寄存器内容(cs/eip/ds/ss/EFLAGS...)保存在TSS中以便任务切换回来时候继续使用。
为了访问TSS，还需要访问全局描述表，全局描述表GDT保存着TSS的地址，TSS最终会被加载进内存中，其中有一个Task Register的cache缓存，最终通过基址加上偏移来确定Task所在的具体位置

## 练习1：加载应用程序并执行
do_execv函数调用load_icode(位于/kern/process/proc.c中)来加载并解析一个处于内存中的ELF文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段，数据段，要设置好proc_struct结构中成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。

了解一波两个函数的功能
do_execv函数的主要功能和实现:完成用户进程的创建工作
1.加载新的执行码做好用户态内存空间清空准备。如果mm不为NULL，则设置页表为内核空间页表，进一步判断mm的引用数减一后是否为0，如果为0，则表明没有进程再需要此进程所占用的内存空间，为此根据mm中的记录，释放进程所占内存空间和进程页表本身所占的空间，为此根据mm中的记录，释放进程所占用户空间内存和进程页表本身所占的空间，最后把当前的进程的mm内存管理指针为空，由于此处的initproc是内核线程，所以mm为NULL，整个处理都不会做。
2.加载应用程序执行码到当前进程的新创建的1用户态虚拟空间中，这里涉及到读ELF格式，申请内存空间，建立用户态虚拟空间，加载应用程序执行码，load_icode函数完成了整个复杂的工作
//主要目的在于清理原来进程的内存空间，为新进程执行准备好空间和资源
```
int do_execve(const char *name, size_t len, unsigned char *binary, size_t size) 
{
    struct mm_struct *mm = current->mm;
    if (!user_mem_check(mm, (uintptr_t)name, len, 0)) {
        return -E_INVAL;
    }
    if (len > PROC_NAME_LEN) {
        len = PROC_NAME_LEN;
    }

    char local_name[PROC_NAME_LEN + 1];
    memset(local_name, 0, sizeof(local_name));
    memcpy(local_name, name, len);
//如果mm不为NULL，则不执行该过程
    if (mm != NULL) 
    {
        //将cr3页表基址指向boot_cr3,即内核页表
        lcr3(boot_cr3);
        if (mm_count_dec(mm) == 0) 
        {  
            //下面三步实现将进程的内存管理区域清空
            exit_mmap(mm);
            put_pgdir(mm);
            mm_destroy(mm);
        }
        current->mm = NULL;
    }
    int ret;
    //填入新的内容，load_icode会将执行程序加载，建立新的内存映射关系，从而完成新的执行
    if ((ret = load_icode(binary, size)) != 0) {
        goto execve_exit;
    }
    //给进程新的名字
    set_proc_name(current, local_name);
    return 0;

execve_exit:
    do_exit(ret);
    panic("already exit: %e.\n", ret);
}
```
## load_icode函数的主要功能和实现
调用mm_create函数来申请进程的内存管理数据结构mm所需内存空间，并对mm进行初始化；

调用setup_pgdir来申请一个页目录表所需的一个页大小的内存空间，并把描述ucore内核虚空间映射的内核页表（boot_pgdir所指）的内容拷贝到此新目录表中，最后让mm->pgdir指向此页目录表，这就是进程新的页目录表了，且能够正确映射内核虚空间；

根据应用程序执行码的起始位置来解析此ELF格式的执行程序，并调用mm_map函数根据ELF格式的执行程序说明的各个段（代码段、数据段、BSS段等）的起始位置和大小建立对应的vma结构，并把vma插入到mm结构中，从而表明了用户进程的合法用户态虚拟地址空间；

调用根据执行程序各个段的大小分配物理内存空间，并根据执行程序各个段的起始位置确定虚拟地址，并在页表中建立好物理地址和虚拟地址的映射关系，然后把执行程序各个段的内容拷贝到相应的内核虚拟地址中，至此应用程序执行码和数据已经根据编译时设定地址放置到虚拟内存中了；

需要给用户进程设置用户栈，为此调用mm_mmap函数建立用户栈的vma结构，明确用户栈的位置在用户虚空间的顶端，大小为256个页，即1MB，并分配一定数量的物理内存且建立好栈的虚地址<—>物理地址映射关系；

至此,进程内的内存管理vma和mm数据结构已经建立完成，于是把mm->pgdir赋值到cr3寄存器中，即更新了用户进程的虚拟内存空间，此时的initproc已经被hello的代码和数据覆盖，成为了第一个用户进程，但此时这个用户进程的执行现场还没建立好；

先清空进程的中断帧，再重新设置进程的中断帧，使得在执行中断返回指令“iret”后，能够让CPU转到用户态特权级，并回到用户态内存空间，使用用户态的代码段、数据段和堆栈，且能够跳转到用户进程的第一条指令执行，并确保在用户态能够响应中断；
此时initproc将按产生系统调用的函数调用路劲原路返回，执行中断返回指令iret后，将切换到用户进程程序的第一句语句位置_start处开始执行。
proc_struct结构中tf结构体变量的设置，从而实现tf从内核态切换到用户态然后执行程序
代码如下：
```
tf->tf_cs=USER_CS;
tf->tf_ds=tf->tf_es=tr->tf_ss=USER_DS;
tf->tf_esp=USTACKTOP;
tf->tf_eip=elf->e_entry;
tf->tf_eflags=FL_IF;//FL_IF为中断打开状态
ret=0;
```
问题1.1 用户进程执行

1.调用schedule函数，调度器占用cpu资源之后，用户态进程调用了exec系统调用，从而转入到了系统调用的处理例程。
2.之后进行正常的中断处理例程，然后控制权转移到了syscall.c中的syscall函数，然后根据系统调用号，转移给了sys_exec函数，函数中调用了do_execve函数来完成指定应用程序的加载。
3.在do_execve中进行若干设置，包括推出当前进程的页表，换用内核的PDT，调用load_icode函数完成对整个用户进程的初始化，包括堆栈的设置以及将ELF可执行文件的加载，之后通过current->tf指正，修改了当前系统调用的trapframe，使其中断返回时，切换成用户态，并且控制权转移到应用程序的入口点。

## 练习2：父进程复制自己的内存空间给子进程
```
//将实际的代码段和数据段搬到新的子进程里面去，再设置好页表的相关内容
int
copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share) {
    //确保start和end可以整除PGSIZE
   	assert(start % PGSIZE == 0 && end % PGSIZE == 0);
    assert(USER_ACCESS(start, end));
    //以页为单位进行复制
    do {
     //得到A&B的pte地址
        pte_t *ptep = get_pte(from, start, 0), *nptep;
        if (ptep == NULL) 
        {
            start = ROUNDDOWN(start + PTSIZE, PTSIZE);
            continue ;
        }

        if (*ptep & PTE_P) {
            if ((nptep = get_pte(to, start, 1)) == NULL) {
                return -E_NO_MEM;
            }
        uint32_t perm = (*ptep & PTE_USER);
        //get page from ptep
        struct Page *page = pte2page(*ptep);
        //为B分一个页的空间
        struct Page *npage=alloc_page();
        assert(page!=NULL);
        assert(npage!=NULL);
        int ret=0;
        /* LAB5:EXERCISE2 YOUR CODE
         * (1) find src_kvaddr: the kernel virtual address of page
         * (2) find dst_kvaddr: the kernel virtual address of npage
         * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
         * (4) build the map of phy addr of  nage with the linear addr start
         */
       //1.找寻父进程的内核虚拟页地址
        void * kva_src = page2kva(page);
       //2.找寻子进程的内核虚拟页地址   
        void * kva_dst = page2kva(npage);
        //3.复制父进程内容到子进程 
        memcpy(kva_dst, kva_src, PGSIZE);
       //4.建立物理地址与子进程的页地址起始位置的映射关系
        ret = page_insert(to, npage, start, perm);
        assert(ret == 0);
        }
        start += PGSIZE;
    } while (start != 0 && start < end);
    return 0;
}
```
问题1.2 如何设计实现"Copy on Write"

以下为个人观点，这个思想其实就是一个共享的概念，多个用户共享一个进程，但是会出现一个问题，如果一个用户将要修改这个进程的内存空间，这会影响到其他的用户，所以就有了需要修改时，将这部分内存空间，拷贝一份，只修改这份拷贝的不就结束了嘛，其他用户是感觉不到的，大大节省了内存空间。

## 练习3 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现

**fork：**完成进程的拷贝，由do_fork函数完成，主要过程如下：

首先检查当前总进程数目是否到达限制，如果到达限制，那么返回E_NO_FREE_PROC；
调用alloc_proc来创建并初始化一个进程控制块；
调用setup_kstack为内核进程（线程）建立栈空间、分配内核栈；
调用copy_mm拷贝或者共享内存空间；
调用copy_thread复制父进程的中断帧和上下文信息；
调用get_pid()为进程分配一个PID；
将进程控制块加入哈希表和链表，并实现相关进程的链接；
最后返回进程的PID
**exec：**完成用户进程的创建工作，同时使用户进程进入执行。由do_exec函数完成，主要过程如下：

检查进程名称的地址和长度是否合法，如果合法，那么将名称暂时保存在函数栈中，否则返回E_INVAL；
将cr3页表基址指向内核页表，然后实现对进程的内存管理区域的释放；
调用load_icode将代码加载进内存并建立新的内存映射关系，如果加载错误，那么调用panic报错；
调用set_proc_name重新设置进程名称。
wait: 完成对子进程的内核栈和进程控制块所占内存空间的回收。由do_wait函数完成，主要过程如下：

首先检查用于保存返回码的code_store指针地址位于合法的范围内；
根据PID找到需要等待的子进程PCB，循环询问正在等待的子进程的状态，直到有子进程状态变为ZOMBIE：
如果没有需要等待的子进程，那么返回E_BAD_PROC；
如果子进程正在可执行状态中，那么将当前进程休眠，在被唤醒后再次尝试；
如果子进程处于僵尸状态，那么释放该子进程剩余的资源，即完成回收工作。
exit: 完成当前进程执行退出过程中的部分资源回收。 由do_exit函数完成，主要过程如下：

释放进程的虚拟内存空间；
设置当期进程状态为PROC_ZOMBIE即标记为僵尸进程
如果父进程处于等待当期进程退出的状态，则将父进程唤醒；
如果当前进程有子进程，则将子进程设置为initproc的子进程，并完成子进程中处于僵尸状态的进程的最后的回收工作
主动调用调度函数进行调度，选择新的进程去执行