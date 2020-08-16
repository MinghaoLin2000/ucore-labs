练习0

参考

https://www.jianshu.com/p/9051fffb37b9

将三个文件直接复制替换就好了

练习1， first-fist

先看初始化部分
```
free_area_t这个结构体里面就两个属性,

typedef struct {
    list_entry_t free_list;         // the list header
    //列表的头
    unsigned int nr_free;           // # of free pages in this free list
    //空闲页的数量
} free_area_t;
free_area_t free_area;
 #define free_list (free_area.free_list) //列表的头
 #define nr_free (free_area.nr_free) //空闲页的数目
static void
default_init(void) {
    list_init(&free_list); //初始化链表
    nr_free = 0; //空闲页数初始化为零
}

static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;//起始的空闲页表地址
    for (; p != base + n; p ++) {
        assert(PageReserved(p));检查是否为保留页
        p->flags = p->property = 0; 表示可分配
        set_page_ref(p, 0); 
    }
    base->property = n; 因为base页是起始页，所以property设置为n，表示有n个连续的空闲块
    SetPageProperty(base);
    nr_free += n;
    list_add(&free_list, &(base->page_link));
}
```
借用下其他师傅的图
分配内存:

 ![blockchain](https://img2020.cnblogs.com/blog/2021287/202008/2021287-20200814012753837-793242678.png "xxx")

```
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0); //n值大于0
    if (n > nr_free) { //当n大于Nf_free说明不够分配，就结束
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list; 空闲块的头指针
    while ((le = list_next(le)) != &free_list) { //直到把链表都给遍历了
        struct Page *p = le2page(le, page_link); //找到连续的内存块数
        if (p->property >= n) { 如果大于意味着，可以分配内存，break退出
            page = p;
            break;
        }
    }
    if (page != NULL) {
        list_del(&(page->page_link));
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;
　　　　　　　　SetPageProperty(p); 
            list_add(&free_list, &(p->page_link));
    }
list_del(&(page->page_link));  //在空闲链表中删除刚刚分配出去的
        nr_free -= n;
        ClearPageProperty(page);
    }
    return page;
}
释放内存：



//释放掉n个页块，释放后也要考虑释放的块是否和已有的空闲块是紧挨着的，也就是可以合并的
//如果可以合并，则合并，否则直接加入双向链表

static void default_free_pages(struct Page *base, size_t n) 
{
    assert(n > 0);  //n必须大于0
    struct Page *p = base;  
    //首先将base-->base+n之间的内存的标记以及ref初始化
    for (; p != base + n; p ++)
     {
        assert(!PageReserved(p) && !PageProperty(p));
        //将flags和ref设为0
        p->flags = 0; 
        set_page_ref(p, 0);
    }
    //释放完毕后先将这一块的property改为n
    base->property = n;
    SetPageProperty(base);
    list_entry_t *le = list_next(&free_list);

    // 检查能否将其合并到合适的页块中
    while (le != &free_list)
    {
        p = le2page(le, page_link);
        le = list_next(le);
        //如果这个块在下一个空闲块前面，二者可以合并
        if (base + base->property == p) 
        {
            //让base的property等于两个块的大小之和
            base->property += p->property;
            //将另一个空闲块删除即可
            ClearPageProperty(p);
            list_del(&(p->page_link));
        }
        //如果这个块在上一个空闲块的后面，二者可以合并
        else if (p + p->property == base) 
        {
            //将p的property设置为二者之和
            p->property += base->property;
            //将后面的空闲块删除即可
            ClearPageProperty(base);
            base = p;
            //注意这里需要把p删除，因为之后再次去确认插入的位置
            list_del(&(p->page_link));
        }
    }
    //整体上空闲空间增大了n
    nr_free += n;
    le = list_next(&free_list);
    // 将合并好的合适的页块添加回空闲页块链表
    //因为需要按照内存从小到大的顺序排列列表，故需要找到应该插入的位置
    while (le != &free_list) 
    {
        p = le2page(le, page_link);
        //找到正确的位置：
        if (base + base->property <= p)
        {
            break;
        }
        //否则链表项向后，继续查找
        le = list_next(le);
    }
    //将base插入到刚才找到的正确位置即可
    list_add_before(le, &(base->page_link));
}
```
 练习2：

主要实验目的就是填充get_pte函数以此来获取页表项目的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。
get-pte函数调用关系如下:

 ![](https://upload-images.jianshu.io/upload_images/15226386-66fbf32e5eb9dd63.png?imageMogr2/auto-orient/strip|imageView2/2/w/386/format/webp)

```
不得不说，注释是真的贴心，基本看注释就能写了
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    pde_t *pdep = &pgdir[PDX(la)]; // 找到 PDE 这里的 pgdir 可以看做是 页目录表的基址
    if (!(*pdep & PTE_P)) {         // 看看 PDE 指向的页表 是否存在
        struct Page* page = alloc_page(); // 不存在就申请一页物理页
        /* 通过 default_alloc_pages() 分配的页 的地址 并不是真正的页分配的地址
            实际上只是 Page 这个结构体所在的地址而已 故而需要 通过使用 page2pa() 将 Page 这个结构体
            的地址 转换成真正的物理页地址的线性地址 然后需要注意的是 无论是 * 或是 memset 都是对虚拟地址进行操作的
            所以需要将 真正的物理页地址再转换成 内核虚拟地址
            */
        if (!create || page == NULL) {
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0, PGSIZE); // 将这一页清空 此时将 线性地址转换为内核虚拟地址
        *pdep = pa | PTE_U | PTE_W | PTE_P; // 设置 PDE 权限
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
}
```

PDE
![fsa](https://upload-images.jianshu.io/upload_images/15226386-5d59af8cf898b370.png?imageMogr2/auto-orient/strip|imageView2/2/w/432/format/webp)
```
P:表示该页保存在物理内存中
R:表示该页可读可写
U:表示该页可以被任何权限用户访问
W:表示cpu可以直写回内存
D:表示不需要被cpu缓存
A:表示该页被写过
S:表示一个页4MB
9-11位保留给os使用
12-31位指明PTE基地址
```
PTE
![](https://upload-images.jianshu.io/upload_images/15226386-1a3deb904d21a562.png?imageMogr2/auto-orient/strip|imageView2/2/w/432/format/webp)
```
0-3和PDE相同
C:和PDED位一样
A:同PDE
D:表示该页被写过
G:表示在cr3寄存器更新时无需刷新TLB中关于该页的地址
9-11位保留给os使用
12-31位指明物理页基址
```

练习3：

static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {

    if(*ptep & PTE_P){
        struct Page *page = pte2page(*ptep); //查看二级页pte映射的物理页地址
        if(page_ref_dec(page) == 0){
            free_page(page);
        }
        *ptep = NULL; //清空PTE
        tlb_invalidate(pgdir, la);刷新TLB
    }

}
```