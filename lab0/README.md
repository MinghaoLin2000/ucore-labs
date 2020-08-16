终于是写完lab0了。。我太菜了呀，感觉c语言一些东西学的浅的不行，看了好多博客，才学会。。

我是在实验楼上面搞的，上面的环境全都配好了，我就不装虚拟机了。

1. 了解汇编

这个先看源码
```
int count=1;
int value=1;
int buf[10];
void main()            
{                
   asm(    
    "cld \n\t"//将DF清零。
        "rep \n\t"//重复前缀指令
        "stosl"//将EAX中的值保存到ES:EDI指向的地址中
    :
    : "c" (count), "a" (value) , "D" (buf[0])//c：ECX a：EAX D：EDX
    :
      );//Att汇编的操作数和目的数是相对于intel是反的
}
```
执行命令 gcc -g -m32 lab0.ex2.c

-g 生成汇编文件，不进行链接和汇编

-m32 生成32位汇编文件

执行后，看ATT汇编
```

1         .file   "lab0_ex1.c"
  2         .text
  3         .globl  count
  4         .data
  5         .align 4
  6         .type   count, @object
  7         .size   count, 4
  8 count:
  9         .long   1
 10         .globl  value
 11         .align 4
 12         .type   value, @object
 13         .size   value, 4
 14 value:
 15         .long   1
 16         .comm   buf,40,32
 17         .text
 18         .globl  main
 19         .type   main, @function
 20 main:
 21 .LFB0:
 22         .cfi_startproc
 23         endbr32
 24         pushl   %ebp
 25         .cfi_def_cfa_offset 8
 26         .cfi_offset 5, -8
 27         movl    %esp, %ebp
 28         .cfi_def_cfa_register 5
 ```
汇编有点丑，可以自己调整一下，其实语法和x86差不多的，执行ecx的将eax值存入es:esi中

二.用gdb调试

就是先编译后，gdb 文件名

再layout src之后查看源代码，可以进行一波断点B 行数，运行等操作，就不写了

三. 掌握指针和类型相关的C编程

才知道有个位域的东西。。。刚开始看这个冒号，一脸懵逼啊。。

查了一下资料才知道，然后还有结构体对齐要注意，其他就还好，

放几个参考资料吧，现在觉得还行

https://blog.csdn.net/Love_Star_/article/details/105011920

https://blog.csdn.net/Love_Star_/article/details/105013457

https://blog.csdn.net/Love_Star_/article/details/105024637

四.这个结构体是和我们之前所定义的结构体差别还是很大的

不过理解了那个强制类型转换的话不难，先放代码，有个骚的点
```

#include<stdio.h>
#include<stdlib.h>
#include "list.h"
typedef struct linklist{
    list_entry_t node;
    int v;
}link;

void init(link *linknode){ //初始化链表
    linknode->=0;
    list_init(&(linknode->node));
}
void add_after(link *oldnode,link *newnode){
    list_add_after(&(oldnode->node),&(newnode->node));
}
void add_before(link *oldnode,link *newnode){
    list_add_before(&(oldnode->node),&(newnode->node));
}
void list_delete(link *linknode){  //删除某个节点
    list_del(linknode);
}
void create_linklist(link *linknode){  //建立链表
    int flag=1;
    int x;
    list_entry_t *p;
    p=&(linknode->node);
    scanf("%d",&x);
    while(x!=flag)
    {
        link *temp=(link*)malloc(sizeof(link));
        temp->v=x;
        add_after((link*)p,temp);
        p=p->next;
        scanf("%d",&x);
    }
}
void printf_linklist(link *linknode){
    list_entry_t *temp;
    temp=&(linknode->node);
    temp=temp->next;
    int count=1;
    while(temp!=&(linknode->node)){
        printf("%d",(link *)temp->v);
        temp=temp->next;
    }
}
int main()
{
    link* H=(link *)malloc(sizeof(link));
    init(H);
    create_linklist(H);
    printf_linklist(H);
}

```
 

 这里的转换成link 指针时，相等于变成了自身，我之前还奇怪这怎么遍历

这里是真强

 