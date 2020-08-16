练习1：

进入~/ucore_core/lab1中，make "V="

这个可以显示make过程中执行了哪些命令
```
+ cc kern/init/init.c
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
kern/init/init.c:95:1: warning: \u2018lab1_switch_test\u2019 defined but not used [-Wunused-function]
 lab1_switch_test(void) {
 ^
+ cc kern/libs/readline.c
gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/readline.c -o obj/kern/libs/readline.o
+ cc kern/libs/stdio.c
gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/stdio.c -o obj/kern/libs/stdio.o
+ cc kern/debug/kdebug.c
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/kdebug.c -o obj/kern/debug/kdebug.o
kern/debug/kdebug.c:251:1: warning: \u2018read_eip\u2019 defined but not used [-Wunused-function]
 read_eip(void) {
 ^
+ cc kern/debug/kmonitor.c
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/kmonitor.c -o obj/kern/debug/kmonitor.o
+ cc kern/debug/panic.c
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/panic.c -o obj/kern/debug/panic.o
+ cc kern/driver/clock.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/clock.c -o obj/kern/driver/clock.o
+ cc kern/driver/console.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/console.c -o obj/kern/driver/console.o
+ cc kern/driver/intr.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/intr.c -o obj/kern/driver/intr.o
+ cc kern/driver/picirq.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/picirq.c -o obj/kern/driver/picirq.o
+ cc kern/trap/trap.c
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/trap.c -o obj/kern/trap/trap.o
kern/trap/trap.c:14:13: warning: \u2018print_ticks\u2019 defined but not used [-Wunused-function]
 static void print_ticks() {
             ^
kern/trap/trap.c:30:26: warning: \u2018idt_pd\u2019 defined but not used [-Wunused-variable]
 static struct pseudodesc idt_pd = {
                          ^
+ cc kern/trap/trapentry.S
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/trapentry.S -o obj/kern/trap/trapentry.o
+ cc kern/trap/vectors.S
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/vectors.S -o obj/kern/trap/vectors.o
+ cc kern/mm/pmm.c
gcc -Ikern/mm/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/mm/pmm.c -o obj/kern/mm/pmm.o
+ cc libs/printfmt.c
gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/printfmt.c -o obj/libs/printfmt.o
+ cc libs/string.c
gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/string.c -o obj/libs/string.o
+ ld bin/kernel
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
+ cc boot/bootasm.S
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
+ cc boot/bootmain.c
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
+ cc tools/sign.c
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
+ ld bin/bootblock
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
'obj/bootblock.out' size: 472 bytes
build 512 bytes boot sector: 'bin/bootblock' success!
dd if=/dev/zero of=bin/ucore.img count=10000
\u8bb0\u5f55\u4e8610000+0 \u7684\u8bfb\u5165
\u8bb0\u5f55\u4e8610000+0 \u7684\u5199\u51fa
5120000\u5b57\u8282(5.1 MB)\u5df2\u590d\u5236\uff0c0.0769198 \u79d2\uff0c66.6 MB/\u79d2
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
\u8bb0\u5f55\u4e861+0 \u7684\u8bfb\u5165
\u8bb0\u5f55\u4e861+0 \u7684\u5199\u51fa
512\u5b57\u8282(512 B)\u5df2\u590d\u5236\uff0c0.000187718 \u79d2\uff0c2.7 MB/\u79d2
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
\u8bb0\u5f55\u4e86138+1 \u7684\u8bfb\u5165
\u8bb0\u5f55\u4e86138+1 \u7684\u5199\u51fa
```
说明:

1. 先编译生成bin/kernel的.o目标文件

2. 再将这些目标文件和库文件链接生成bin/kernel可执行文件

3. 编译 boot/bootasm.S,boot/bootmain.ctools/sign.c

4. 再根据sign规范生成bootlock可执行文件

5. dd命令将/dev/zero文件，/bin/bootlock,/bin/kernel放到ucore.img中

这里细讲三个dd命令

dd if=/dev/zero of=bin/ucore.img count=10000

创建10000扇区，每个扇区512字节

dd if=bin/bootblock of=bin/ucore.img conv=notrunc

将bin/bootblock存入第一个扇区内

dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

将kernel存入ucore.img的第二个之后的扇区中

问题二.规范

sign.c文件如下
```
1  include <stdio.h>
 2 #include <errno.h>
 3 #include <string.h>
 4 #include <sys/stat.h>
 5 
 6 int
 7 main(int argc, char *argv[]) {
 8     struct stat st;
 9     if (argc != 3) {
10         fprintf(stderr, "Usage: <input filename> <output filename>\n");
11         return -1;
12     }
13     if (stat(argv[1], &st) != 0) {
14         fprintf(stderr, "Error opening file '%s': %s\n", argv[1], strerror(errno));
15         return -1;
16     }
17     printf("'%s' size: %lld bytes\n", argv[1], (long long)st.st_size);
18     if (st.st_size > 510) {
19         fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
20         return -1;
21     }
22     char buf[512];
23     memset(buf, 0, sizeof(buf));
24     FILE *ifp = fopen(argv[1], "rb");
25     int size = fread(buf, 1, st.st_size, ifp);
26     if (size != st.st_size) {
27         fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
28         return -1;
29     }
30     fclose(ifp);
31     buf[510] = 0x55;
32     buf[511] = 0xAA;
33     FILE *ofp = fopen(argv[2], "wb+");
34     size = fwrite(buf, 1, 512, ofp);
35     if (size != 512) {
36         fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
37         return -1;
38     }
39     fclose(ofp);
40     printf("build 512 bytes boot sector: '%s' success!\n", argv[2]);
41     return 0;
42 }
```
结束标志是0x55,0xAA,总共512字节，sign.c文件会进行一波判断

 练习二.

2.1

先进入/ucore_lab1/tools/gdbinit，vim修改配置文件

set architecture i8086

target remote :1234

添加这两句，相当于和qemu连接上了，因为gdb和qemu进行通信就是通过1234端口进行的

再cd lab1文件夹下，输入make debug，进入调试模式

输入si，进行命令执行

输入x/2i $pc 查看汇编指令
![blockchain](https://img2020.cnblogs.com/blog/2021287/202007/2021287-20200717110113181-1268548378.png "xxx")

2.3
![blockchain](https://img2020.cnblogs.com/blog/2021287/202007/2021287-20200717114356661-1538677109.png "xxx")

修改gdbinit ，将断点设置在0x7c00这，反汇编的代码和bootasm.S一样的

 

2.4

gdbinit修改一下，将断点打在内核入口函数
```
file bin/kernel
set architecture i8086
target remote :1234
b kern_init
continue
```
 ![blockchain](https://img2020.cnblogs.com/blog/2021287/202007/2021287-20200717150055364-1338595851.png "xxx")

 ni 可以单步调试

 

 练习三:

3.1

挂一个博客，感觉写的非常好，只是里面的端口讲的我有点迷糊，之后问下师傅们看法

http://hengch.blog.163.com/blog/static/107800672009013104623747/

3.2
```
...
#define SEG_NULLASM                                             \
    .word 0, 0;                                                 \
    .byte 0, 0, 0, 0

#define SEG_ASM(type,base,lim)                                  \
    .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
    .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
        (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
...
lgdt gdtdesc
...
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt

```
1. lgdt gdtdesc  载入GDT表

2.  将cr0 的pe位置也就是第0位置为1  进入保护模式
```
movl %cr0, %eax //加载cro到eax
orl $CR0_PE_ON, %eax //将eax的第0位置为1
movl %eax, %cr0 //将cr0的第0位置为1
```
3.通过长跳转修改cs
```
$PROT_MODE_CSEG的值为0x80

ljmp $PROT_MODE_CSEG, $protcseg
.code32 
protcseg:
```
4.设置堆栈和段寄存器
```
movw $PROT_MODE_DSEG, %ax // 
movw %ax, %ds 
movw %ax, %es 
movw %ax, %fs 
movw %ax, %gs 
movw %ax, %ss 
movl $0x0, %ebp //设置帧指针
movl $start, %esp //设置栈指针
```
5.设置完成后，调用bootmain函数

 

