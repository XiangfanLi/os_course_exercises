# lec2：lab0 SPOC思考题

## **提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在不同操作系统（如linux, ucore,etc.)与不同硬件（如 x86, riscv, v9-cpu,etc.)中是如何相互配合来体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？
  * 进程的切换需要硬件支持时钟中断；虚存管理需要地址映射机制，从而需要MMU等硬件；对于文件系统，需要硬件有稳定的存储介质来保证操作系统的持久性。 对应的，应当提供中断使能，触发软中断等中断相关的，设置内存寻址模式，设置页表等内存管理相关的，执行I/O操作等文件系统相关的特权指令。

- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？
  * 保护模式和实模式的根本区别是进程内存是否受保护。实模式将整个物理内存看成分段的区域，程序代码和数据位于不同区域，系统程序和用户程序没有区别对待，而且每一个指针都是指向“实在”的物理地址。这样一来，用户程序的一个指针如果指向了系统程序区域或其他用户程序区域，并改变了值，那么对于这个被修改的系统程序或用户程序，其后果就很可能是灾难性的。为了克服这种低劣的内存管理方式，处理器厂商开发出保护模式。这样，物理内存地址不能直接被程序访问，程序内部的地址（虚拟地址）要由操作系统转化为物理地址去访问，程序对此一无所知。

  * 物理地址：是处理器提交到总线上用于访问计算机系统中的内存和外设的最终地址。
  * 逻辑地址：在有地址变换功能的计算机中，访问指令给出的地址叫逻辑地址。（一般的定义是段选择子+段内偏移量是逻辑地址）
  * 线性地址：线性地址是逻辑地址到物理地址变换之间的中间层，是处理器通过段机制控制下的形成的地址空间。

- 你理解的risc-v的特权模式有什么区别？不同模式在地址访问方面有何特征？
  * risc-v有三种特权模式(工作模式) : 机器模式，监督模式和用户模式
  * 机器模式 : 具有最高级特权，可以不受限制地访问整个机器，也是risc-v硬件平台唯一必须的特权级。运行于机器模式下的代码是固有可信的，因为它可以在低层次访问机器的实现。
  * 监督模式 : 用来支持Linux,FreeBSD, Windows等操作系统的需求，特权级比用户模式高但不如机器模式。当操作系统需要处理异常或中断时，机器的控制权将被交给监督模式。监督模式还提供了虚拟存储系统。
  * 用户模式 : 用于传统应用程序，特权级模式最低，仿存空间受限，需要通过相应的接口来请求获得所需的系统服务。
  * 在地址访问方面，机器模式能不受限制地访问整个机器，在用户模式下，由于监督模式提供的虚拟存储系统，程序可使用的逻辑地址空间要大于实际可用的绝对内存空间。
  
- 理解ucore中list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```
* “:”后的数字表示每一个域在结构体中所占的位数。总的来说就是把struct的变量定义精确到了bit的程度。这个结构体是IDT中的门描述符，一个门描述符的大小为8字节。

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？

  * 假设机器为小端序(低位字节放在低地址), 这个过程是直接把intr的地址当成了一个gatedesc的地址,调用宏来填充IDT表，从代码中也可以看出来。之后事实上又把这个gatedesc转换成了unsigned类型，也就是输出了它的前半部分（因为unsigned的长度为4个字节）。
  ```
    gd_off_15_0 = 3 & 0xffff = 0x0003
    gd_ss = sel = 0x0002
    gd_args = b00000
    gd_rsv1 = b000
    gd_type = STS_TG32 = 0xf
    gd_s = b0
    gd_dpl = b00
    gd_p = b1
    gd_off_31_16 = 0x0000
 ```
  * 填充结束后，内存中从低地址到高地址依次为:
    03 00 02 00 00 8f 00 00
  * intr取前4个字节，由于是小端序，故intr的值为0x20003

### 课堂实践练习

#### 练习一

1. 请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。
  以下是 boot/bootasm.S 中的一段汇编代码
  ```
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
  ```
  这段代码的作用是加载全局描述符表, 并将系统控制寄存器cr0的最低位置为1，代表系统由实模式进入保护模式。
  lgdt gdtdesc 指令会将gdtdesc的内容加载到GDTR中
  在同一文件中可找到gdtdesc的内容:
  ```
   gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

   gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
  ```
  可以看到，gdtdesc的前两个字节为0x17，表示gdt的限长（共24个字节），后32个字节表示gdt的基址，该基址处24个字节的内容即为全局描述符表。

2. (option)请在rcore中找一段你认为难度适当的RV汇编代码，尝试解释其含义。

#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore或rcore中宏定义的用途，并举例描述其含义。
boot/bootmain.c中有如下的宏定义:
```
#define SEG_ASM(type,base,lim)                                  \
    .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
    .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
        (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```
该宏定义的作用是根据type(段的类型)、base(段的基址)、lim(段的限长)生成一个64位的段描述符
使用举例:
boot/bootasm.S中有如下汇编代码:
```
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel
```
刚进入保护模式时，gdt被初始化为24字节大小，其中依次为一个空的段描述符，一个描述大小为4GB、可读可执行的代码段的段描述符，一个描述大小为4GB、可写不可执行的数据段的段描述符


#### reference
 - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)
 - [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)
 - [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)
 - [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)
 - [IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
