前置知识
###一.
什么是虚拟内存?简单来说是指程序员或cpu看到的内存。但有几点注意
1. 虚拟内存不一定有实际的物理单元对应，实际的物理内存单元可能不存在
2. 如果虚拟内存单元对应有实际的物理内存单元，那二者的地址一般是不相等的。
3. 通过操作系统实现的某种内存映射可建立虚拟内存与物理内存的对应关系，使得程序员或cpu访问的虚拟内存地址会自动转换成一个物理内存地址。
那么这个虚拟的作用或意义在哪里体现呢? 在操作系统中,虚拟内存其实包含多个虚拟层次，在不同的层次体现不同的作用.首先，在有了分页机制后，程序员或cpu看到地址已经不是实际的物理地址了，这已经有一层虚拟化，我们可简称为内存地址虚拟化。有了内存地址虚拟化，我们就可以通过设置页表项来限定软件运行时的访问空间，确保软件运行不越界，完成内存访问保护的功能。

通过内存地址虚拟化，可以使得软件在没有访问某虚拟内存时，不分配具体的物理内存，只有在实际访问某虚拟内存地址时，操作系统再动态地分配物理内存，建立虚拟内存到物理内存的页映射关系，这种技术称为按需分页. 把不经常访问的数据所占的内存空间临时写道硬盘上，这样可以腾出更多的空闲内存空间给经常访问的数据，当cpu访问到不经常访问的数据时，再把这些数据从硬盘读入内存中，这种技术称为页换入换出。
##页面异常
当启动分页机制以后，如果一条指令或数据的虚拟地址所对应的物理页框不在内存中或者访问有错误(比如写一个只读页或用户态程序访问内核态)，就会发生页错误异常。产生页面异常的原因主要有:
1. 目标页面不存在(页表项全为零,即该线性地址与物理地址尚未建立映射或者已经撤销)
2. 相应的物理页面不在内存中（页表项非空,但present标志位=0，比如swap分区或者磁盘文件上）
3. 访问权限不符合(此时页表项p标志=1，比如企图写只读页面)
当出现上面情况之一，那么就会产生page fault异常。产生异常的线性地址存储在cr2中，并且将是page fault的产生类型保存在error code中。

##关键数据结构
page_fault函数不知道哪些是“合法”的虚拟页，原因是ucore还缺少有一定的数据结构来描述这种不在物理内存中的“合法”虚拟页。为此ucore还通过建立mm_struct和vma_struct数据结构，描述了ucore模拟应用程序运行所需的合法内存空间。当访问内存产生page fault异常时，可获得访问的内存方式（读或写）以及具体的虚拟内存地址，这样ucore就可以查询此地址，看是否属于vma_struct数据结构中描述的合法地址范围中，如果在，则可根据具体情况进行请求调页/页换入换出处理;如果不在,则报错。
![](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab3_figs/image001.png)
##vma_struct
vma_struct描述应用程序对虚拟内存需求，以及针对vma_struct的函数操作。里把一个vma_struct结构的变量简称为vma变量。
```
struct vma_struct {  
        struct mm_struct *vm_mm;  //指向一个比vma_struct更高的抽象层次的数据结构mm_struct 
        uintptr_t vm_start;      //vma的开始地址
        uintptr_t vm_end;      // vma的结束地址
        uint32_t vm_flags;     // 虚拟内存空间的属性
        list_entry_t list_link;  //双向链表，按照从小到大的顺序把虚拟内存空间链接起来
    };  
```
VMA是描述应用程序对虚拟内存需求的变量。vm_start和vm_end描述的是一个合理地址空间范围（即严格确保vm_start< vm_end的关系）,list_link是一个双向的链表,按照从小到大顺序把一系列用vma_struct表示的虚拟内存空间链接起来，并且还要求这些链起来的vma_struct应该是不相交，即vma之间的地址空间无交集。
vm_flags表示了这个虚拟内存空间的属性，目前的属性包括
```
 #define VM_READ 0X00000001   //只读
 #define VM_WRITE 0x00000002 //可读写
 #define VM_EXEC  0x00000004 //可执行
```
vm_mm是一个指针,指向一个比vma_struct更高的抽象层次的数据结构##mm_struct
```
struct mm_struct {  
        list_entry_t mmap_list;  //双向链表头，链接了所有属于同一页目录表的虚拟内存空间
        struct vma_struct *mmap_cache;  //指向当前正在使用的虚拟内存空间
        pde_t *pgdir; //指向的就是 mm_struct数据结构所维护的页表
        int map_count; //记录mmap_list里面链接的vma_struct的个数
        void *sm_priv; //指向用来链接记录页访问情况的链表头
 };  
 ```
 mmap_list是双向链表头,链接了所有属于同一页目录表的虚拟内存空间,mmap_cache是指向当前正在使用的虚拟内存空间，由于操作系统执行的局部性原理，当前正在用到的虚拟空间在接下来的操作中可能还会用到,这时就不需要查链表，而是直接使用此指针就可找到下一次要用到的虚拟内存空间.pgdir所指向的就是mm_struct数据结构所维护的页表.通过访问pgdir可以查找某虚拟地址对应的页表项是否存在以及页表项的属性等. map_count记录mmap_List里面链接的vma_struct的个数.sm_priv指向用来链接记录页访问情况的链表头，这建立了mm_struct和后续要讲到的swap_manager之间的联系.

##错误码errorCode
产生页访问异常后,cpu把引起页访问异常的线性地址装到寄存器CR2,并给出错误码errorCode，说明了页访问异常的类型.ucore Os会把这个值保存在struct trapframe中tf_err成员变量中.而中断服务例程会调用页访问异常处理函数do_pgfault进行具体处理.这里页访问异常处理

# 练习一
完成do_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。设置访问权限 的时候需要参考页面所在 VMA 的权限，同时需要注意映射物理页时需要操作内存控制 结构所指定的页表，而不是内核的页表。注意：在LAB2 EXERCISE 1处填写代码。执行make　qemu后，如果通过check_pgfault函数的测试后，会有“check_pgfault() succeeded!”的输出，表示练习1基本正确。
请在实验报告中简要说明你的设计实现过程。请回答如下问题：

请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。
如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
```
int do_pgfault(struct mm_struct *mm,uint32_t error_code,uintptr_t addr)
{
    int ret=-E_INVAL;
    struct vma_struct *vma=find_vma(mm,addr);//查询vma
    pgfault_num++;
    if(vma==NULL||vma->vm_start>addr)
    {
        cprintf("not valid addr %x,and can not find it in vma\n,addr);
        goto faliled;
    }
    switch(error_code&3)
    {
        default:
        case 2: 
            if(!(vm->vm_flags&VM_WRITE))
            {
                cprintf("do_pgfaultfailed:error code flag= write AND not present,but the addr's vma cannot write\n");
                goto failed;
            }
            break;
        case 1:
            cprintf("do_pgfault failed:error code flag=read AND present\n");
            goto failed;
        case 0:
            if(!(vma->vm_flags&(VM_READ|VM_EXEC)))
            {
                cprintf("do_pgfault failed:error code flag=readd AND not present,but the addr's vma cannot read or exec\n");
                goto failed;
            }
    }
    uint32_t perm=PTE_U;
    if(vma->vm_flags&VM_WRITE)
    {
        perm|=PTE_W;
    }
    addr=ROUNDDOWN(addr,PGSIZE);
    ret =-E_NO_MEM;
    pte_t *ptep=NULL;
    if((ptep=get_pte(mm->pgdir,addr,1))==NULL)
    {
        cprintf("get_pte in do_pgfault failed\n");
        goto failed;
    }
    if(*ptep==0)
    {
        if(pgdir_alloc_page(mm->pagdir,addr,perm)==NULL)
        {
            cprintf("pgdir_alloc in do_pgfault failed\n");
            go failed;
        }
    }else
    {
        if(swap_init_ok)
        {
            struct Page *page=NULL; 
            if((ret=swap_in(mm,addr,&page))!=0)
            {
                cprintf("swap_in in do_pgfault failed\n");
                goto failed;
            }
            page_insert(mm->pgdir,page,addr,perm);//建立虚拟地址和物理地址之间的对应的关系
            swap_map_swappable(mm,addr,page,1);
            page->pra_vaddr=addr;
        }else
        {
            cprintf("no swap_init_ok but ptep is %x failed\n",*ptep);
            goto failed;
        }
    }
    ret=0;
    failed:
    return ret;
}
```
# 练习2
完成vmm.c中的do_pgfault函数，并且在实现FIFO算法的swap_fifo.c中完成map_swappable和swap_out_vistim函数。通过对swap的测试。注意：在LAB2 EXERCISE 2处填写代码。执行make　qemu后，如果通过check_swap函数的测试后，会有“check_swap() succeeded!”的输出，表示练习2基本正确。请在实验报告中简要说明你的设计实现过程。
请在实验报告中回答如下问题：
如果要在ucore上实现”extended clock页替换算法”请给你的设计方案，现有的swap_manager框架是否足以支持在ucore
中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下
问题

​ 需要被换出的页的特征是什么？
​ 在ucore中如何判断具有这样特征的页？
​ 何时进行换入和换出操作？

##页错误异常
页错误异常发生时,有可能是因为页面保存在swap区或者磁盘文件上造成的，所以我们需要通过页面分配解决这个问题，页面替换主要分为两个方面，页面换出和页面换入

## 页面换出部分
### 换出机制
换出页面的时机相对复杂一些，针对不同的策略有不同的时机。ucore目前有两种策略，即积极换出策略和消极换出策略。积极换出策略是指操作系统周期性地主动把某些认为不常用的页换出硬盘，上，从而确保系统中总有一定数量的空闲也存在，这样当需要空闲页，基本上能够及时满足需求；消极换出策略是指，只是当试图得到空闲页时，发现当前没有空闲的物理页可供分配，这时才开始查找“不常用”页面，并把一个或多个这样的页换出到硬盘上。在实验三中的基本练习中，支持上述的第二种情况。对于第一种积极换出策略，即每隔1秒执行一次的实现积极的换出策略，可考虑在扩展练习中实现。对于第二种消极的换出策略，则是在ucore调用alloc_pages函数获取空闲页时，此函数如果发现无法从物理内存页分配器获得空闲页，就会进一步调用swap_out函数换出某页，实现一种消极的换出策略。

## _fifo_map_swappable函数
主要作用是将最近被用到的页面添加到算法所维护的次序队列
```
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
    assert(entry != NULL && head != NULL);
    list_add(head, entry);
    return 0;
}
```
##  _fifo_swap_out_victim()函数
```a
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
     assert(head != NULL);
     assert(in_tick==0);
     list_entry_t *le = head->prev;//用le指示需要被换出的页
     assert(head!=le);
     struct Page *p = le2page(le, pra_page_link//le2page宏可以根据链表元素获得对应的Page指针p  
     list_del(le); //将进来最早的页面从队列中删除
     assert(p !=NULL);
     *ptr_page = p; //将进来最早的页面从队列中删除
     return 0;
}
```