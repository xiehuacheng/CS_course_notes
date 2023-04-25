# Calling conventions and stack frames RISC-V

[课程翻译链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec05-calling-conventions-and-stack-frames-risc-v)

## ISA & Assembly Language
---

ISA: Instruction Set（指令集）
任何一个处理器都有一个关联的ISA
每一条指令对应着一个二进制编码或者一个OPcode

编译器会将代码（高级语言）编译成汇编语言
C -> Assembly(.S/.asm) -> binary (object.o)

汇编语言没有明确的workflow（流程控制），只是一行行指令，没有循环（注，但是有基于lable的跳转）

汇编语言是==基于**寄存器**进行操作的，而不是基于内存操作==

RISC-V vs x86：

-   RISC-V：精简指令集，指令更少，更加简单，唯一开源的ISA。ARM也是RISC(Reduced Instruction Set Chip)
-   x86：复杂指令集（CISC）（Complex Instruction Set Computer），指令很多并且可以实现复杂功能，每一条指令都执行了一系列复杂的操作并返回结果

[RISC-V assembly常用指令](https://web.eecs.utk.edu/~smarz1/courses/ece356/notes/assembly/)

-   `ld/lb/lw rd, 8(rs)`：将*(rs+8)的值写入到rd寄存器，`lb`=load byte, `ld`=load double word, `lw`=load word
-   `sd/sb/sw rd, 8(rs)`：将*(rd)`的值写入到rs+8地址上
-   `add rd, rs1, rs2`：*(rd) = *(rs1) + *(rs2)
-   `addi rd, rs1, int`: *(rd) = *(rs1) + int

## Calling convention
---

调用约定(calling convention)是规定子过程如何获取参数以及如何返回的方案，调用约定一般规定了

-   参数、返回值、返回地址等放置的位置（寄存器、栈或存储器）
	- 寄存器是CPU或者处理器上，预先定义的可以用来存储数据的位置
	- RISC-V寄存器通过寄存器而非栈来传递函数参数，a0-a7是int参数，fa0-fa7是float参数
		- 如果一个函数有超过8个参数，我们就需要用内存了。从这里也可以看出，当可以使用寄存器的时候，我们不会使用内存，我们只在不得不使用内存的场景才使用它

-   如何将调用子过程的准备工作与恢复现场的工作划分到调用者(caller)与被调用者(callee)身上
   
![image-20210127123800904](https://fanxiao.tech/img/posts/MIT_6S081/image-20210127123800904.png "image-20210127123800904")

基本上来说，RISC-V中通常的指令是64bit，但是在Compressed Instruction中指令是16bit。在Compressed Instruction中我们使用更少的寄存器，也就是x8 - x15寄存器。我猜你们可能会有疑问，为什么s1寄存器和其他的s寄存器是分开的，因为s1在Compressed Instruction是有效的，而s2-11却不是

小于一个指针字(RISCV64中是8字节，RISCV32是4字节)的参数传入时将参数放在寄存器的最低位，因为RISC-V是小端系统，当2个指针字的参数传入时，低位的1个指针字放在偶数寄存器，比如a0上，高位的1个指针字放在奇数寄存器，比如a1上。当高于2个指针字的参数传入时以引用的方式传入。`struct`参数没有传到寄存器的部分将以栈的方式传入，`sp`栈指针将指向第一个没有传入到寄存器的参数

从函数返回的值，如果是整数将放在a0和a1中，如果是小数将放置在fa0和fa1寄存器中。对于更大的返回值，将放置在内存中，caller开辟这个内存，并且把指向这个内存的指针作为第一个参数传递给callee

由caller保存的寄存器==不会在函数调用之间被保存==，又名易失性寄存器，如果要在过程调用后恢复该值，则调用方有责任将这些寄存器==压入堆栈或复制到其他位置==，而callee保存的寄存器==会被保存==，称为非易失性寄存器，可以期望这些寄存器在被调用者返回后保持相同的值。比如函数A调用了函数B，所有函数A保存的caller寄存器在函数B被调用后可以被B重写覆盖

## Stack
---

![image-20210127195509729](https://fanxiao.tech/img/posts/MIT_6S081/image-20210127195509729.png "image-20210127195509729")

栈从**高地址向低地址增长**，每个大的box叫一个==stack frame（栈帧）==，栈帧由函数调用来分配，每一次我们调用一个函数，函数都会为自己创建一个Stack Frame，并且只给自己用，函数通过移动Stack Pointer（做减法）来完成Stack Frame的空间分配，每个栈帧大小不一定一样（因为栈中也保存着函数参数中第8个之后的参数），但是栈帧的最高处（底部）一定是return address

sp是stack pointer，用于指向栈顶（低地址），保存在寄存器中，代表当前栈帧的位置

fp是frame pointer，用于指向当前帧底部（高地址），保存在寄存器中

同时每个函数栈帧中保存了调用当前函数的函数（父函数）的fp（保存在to prev frame那一栏中），为了函数在执行结束时可以正确返回

这些栈帧都是由编译器编译生成的汇编文件生成的（Prologue和Epllogue语句）

leaf函数不需要栈帧，因为它们不需要调用别的函数，不需要保存返回的函数地址，也不需要保存caller寄存器

## Struct
---

基本上来说，struct在内存中是一段连续的地址，如果我们有一个struct，并且有f1，f2，f3三个字段。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MM4GlTfQa57FRIbUhP2%2F-MM8LCRNuvyydmmGuXOR%2Fimage.png?alt=media&token=42519273-f9f6-4e61-8800-4e10a67a992a)

当我们创建这样一个struct时，内存中相应的字段会彼此相邻。你可以认为struct像是一个数组，但是里面的不同字段的类型可以不一样