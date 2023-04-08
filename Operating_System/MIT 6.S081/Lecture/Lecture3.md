#  OS organization and system calls

[翻译文本链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec03-os-organization-and-system-calls)

## Isolation（隔离性）
---

1. 程序之间
	- 一个程序的错误不会影响到其他程序
2. 程序与操作系统之间
	- 当你的应用程序出现问题时，你会希望操作系统不会因此而崩溃。比如说你向操作系统传递了一些奇怪的参数，你会希望操作系统仍然能够很好的处理它们（能较好的处理异常情况）
	- 如果没有操作系统，应用程序会直接与硬件交互，从隔离性的角度来看，这并不是一个很好的设计。从内存的角度来说，如果应用程序直接运行在硬件资源之上，那么每个应用程序的文本，代码和数据都直接保存在物理内存中。因为两个应用程序的内存之间没有边界，那么就有可能出现一个程序会覆盖另一个程序中的内容的情况

- 在应用程序和硬件之间增加一层抽象层，使得应用程序无法直接的访问和操作硬件

- 使用操作系统的一个原因，甚至可以说是主要原因就是为了实现multiplexing和内存隔离。如果你不使用操作系统，并且应用程序直接与硬件交互，就很难实现这两点。系统为应用程序提供接口，接口通过抽象硬件资源，从而使得提供强隔离性成为可能

## Defensive（防御性）
---

操作系统需要确保所有的组件都能工作，所以它需要做好准备抵御来自应用程序的攻击。如果说应用程序无意或者恶意的向系统调用传入一些错误的参数就会导致操作系统崩溃，那就太糟糕了

另一个需要考虑的是，应用程序不能够打破对它的隔离。应用程序非常有可能是恶意的，它或许是由攻击者写出来的，攻击者或许想要打破对应用程序的隔离，进而控制内核

两种实现强隔离性的方法
1. ==user/kernel mode== 在RISC-V中被称为Supervisor mode但是其实是同一个东西
2. page table或者==虚拟内存（Virtual Memory）==

## User mode and supervisor mode
---

为了实现进程隔离，RISC-V CPU 在硬件上提供 3 种执行命令的模式：_machine mode_, _supervisor mode_, _user mode_

1.  machine mode 的权限最高，CPU 以 machine mode 启动，machine mode 的主要目的是为了配置电脑，之后立即切换到 supervisor mode，在本课程中不重要
2.  supervisor mode 运行 CPU 执行特殊权限指令（privileged instructions），比如中断管理、对存储页表地址的寄存器进行读写操作、执行 system call。运行在 supervisor mode 也称为在 _kernel space_ 中运行
3.  应用程序只能执行 user mode 指令，即普通权限的指令（unprivileged instructions），比如改变变量、执行 util function。运行在 user mode 也称为在 _user space_ 中运行。要想让 CPU 从 user mode 切换到 supervisor mode，RISC-V 提供了一个特殊的==`ecall`指令==，要想从 supervisor mode 切换到 user mode，调用`sret`指令

在处理器里面有一个flag。在处理器的一个bit，当它为1的时候是user mode，当它为0时是kernel mode

内核夺回（恶意或死循环）程序控制权的机制：内核会通过硬件设置一个定时器，定时器到期之后会将控制权限从用户空间转移到内核空间，之后内核就有了控制能力并可以重新调度CPU到另一个进程中

### eacll指令

- 当一个用户程序想要将程序执行的控制权转移到内核，它只需要执行ECALL指令，并传入一个数字。这里的数字参数代表了应用程序想要调用的System Call

- 在内核侧，有一个位于syscall.c的函数syscall，每一个从应用程序发起的系统调用都会调用到这个syscall函数，syscall函数会检查ECALL的参数

## Virtual memory
---

基本上来说，处理器包含了page table，而page table将虚拟内存地址与物理内存地址做了对应（映射）

==每一个进程都会有自己独立的page table==，这样的话，每一个进程只能访问出现在自己page table中的物理内存。操作系统会设置page table，使得每一个进程都有**不重合**的物理内存，这样一个进程就不能访问其他进程的物理内存，因为其他进程的物理内存都不在它的page table中。一个进程甚至都不能随意编造一个内存地址，然后通过这个内存地址来访问其他进程的物理内存。这样就给了我们内存的强隔离性

内核也有自己的独立地址空间

## Kernel organization
---

内核有时候也被称为可被信任的计算空间（Trusted Computing Base）（TCB）

_monolithic kernel_（宏内核）：整个操作系统在 kernel 中，所有 system call 都在 supervisor mode 下运行。xv6 是一个 monolithic kernel
- 优点：操作系统中的各个部分位于同一个程序中，它们可以紧密的集成在一起，这样的集成提供很好的性能
- 缺点：代码出现bug的可能性更大

_micro kernel_（微内核）：将需要运行在 supervisor mode 下的操作系统代码压到最小，保证 kernel 内系统的安全性，将大部分的操作系统代码执行在 user mode 下
- 优缺点相反

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210121175322692.png)

如 2.1 所示，文件系统是一个 user-level 的进程，为其他进程提供服务，因此也叫做 server。这里工作的方式是，Shell会通过内核中的IPC系统发送一条消息，内核会查看这条消息并发现这是给文件系统的消息，之后内核会把消息发送给文件系统。文件系统会完成它的工作之后会向IPC系统发送回一条消息说，这是你的exec系统调用的结果，之后IPC系统再将这条消息发送给Shell。

现在，对于任何文件系统的交互，都需要分别完成2次用户空间<->内核空间的跳转。与宏内核对比，在宏内核中如果一个应用程序需要与文件系统交互，只需要完成1次用户空间<->内核空间的跳转，所以微内核的的跳转是宏内核的两倍。

xv6 kernel source file 如下所示

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210121185952327.png)

## 内核的编译过程
---

首先，Makefile（XV6目录下的文件）会读取一个C文件，例如proc.c；之后调用gcc编译器，生成一个文件叫做proc.s，这是RISC-V 汇编语言文件；之后再走到汇编解释器，生成proc.o，这是汇编语言的二进制格式；Makefile会为所有内核文件做相同的操作，比如说pipe.c，会按照同样的套路，先经过gcc编译成pipe.s，再通过汇编解释器生成pipe.o；之后，系统加载器（Loader）会收集所有的.o文件，将它们链接在一起，并生成内核文件

这里生成的内核文件就是我们将会在QEMU中运行的文件。同时，为了你们的方便，Makefile还会创建kernel.asm，这里包含了内核的完整汇编语言，你们可以通过查看它来定位究竟是哪个指令导致了Bug

## QEMU
---

我们来看传给QEMU的几个参数：

-   -kernel：这里传递的是内核文件（kernel目录下的kernel文件），这是将在QEMU中运行的程序文件。
-   -m：这里传递的是RISC-V虚拟机将会使用的内存数量
-   -smp：这里传递的是虚拟机可以使用的CPU核数
-   -drive：传递的是虚拟机使用的磁盘驱动，这里传入的是fs.img文件

当你想到QEMU时，你不应该认为它是一个C程序，你应该把它想成是一个真正的主板

在内部，在QEMU的主循环中，只在做一件事情：

-   读取4字节或者8字节的RISC-V指令
-   解析RISC-V指令，并找出对应的操作码（op code）
-   之后，在软件中执行相应的指令

这基本上就是QEMU的全部工作了，对于每个CPU核，QEMU都会运行这么一个循环

为了完成这里的工作，QEMU的主循环需要维护寄存器的状态。所以QEMU会有以C语言声明的类似于X0，X1寄存器等等

## Process
---

隔离的单元叫做==进程==，一个进程不能够破坏或者监听另外一个进程的内存、CPU、文件描述符，也不能破坏 kernel 本身。

为了实现进程隔离，xv6 提供了一种机制让程序认为自己拥有一个独立的机器。一个进程为一个程序提供了一个私有的内存系统，或 _address space_，其他的进程不能够读 / 写这个内存。xv6 使用 ==_page table_(页表)== 来给每个进程分配自己的 address space，页表再将这些 address space，也就是进程自己认为的虚拟地址 (_virtual address_) 映射到 RISC-V 实际操作的物理地址 (_physical address_)

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210121190517403.png)

虚拟地址从 0 开始，往上依次是指令、全局变量、栈、堆。RISC-V 上的指针是 64 位的，xv6 使用低 38 位，因此最大的地址是 $2^{38}-1$=0x3fffffffff=MAXVA

进程最重要的内核状态：1. 页表 `p->pagetable` 2. 内核堆栈`p->kstack` 3. 运行状态`p->state`，显示进程是否已经被分配、准备运行 / 正在运行 / 等待 IO 或退出

每个进程中都有==线程 (_thread_)==，是执行进程命令的最小单元，可以被暂停和继续

每个进程有两个堆栈：用户堆栈 (_user stack_) 和内核堆栈 (_kernel stack_)。当进程在 user space 中进行时只使用用户堆栈，当进程进入了内核 (比如进行了 system call) 使用内核堆栈

## Starting the first process
---

[参考链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec03-os-organization-and-system-calls/3.9-xv6-qi-dong-guo-cheng)

RISC-V 启动时，先运行一个存储于 ROM 中的 bootloader 程序`kernel.ld`来加载 xv6 kernel 到内存中，然后在 machine 模式下从`_entry`开始运行 xv6。bootloader 将 xv6 kernel 加载到 0x80000000 的物理地址中，因为前面的地址中有 I/O 设备

在`_entry`中设置了一个初始 stack，`stack0`来让 xv6 执行`kernel/start.c`。在`start`函数先在 machine 模式下做一些配置，然后通过 RISC-V 提供的`mret`指令切换到 supervisor mode，使 program counter 切换到`kernel/main.c`

`main`先对一些设备和子系统进行初始化，然后调用`kernel/proc.c`中定义的`userinit`来创建第一个用户进程。这个进程执行了一个`initcode.S`的汇编程序，这个汇编程序调用了`exec`这个 system call 来执行`/init`，重新进入 kernel。`exec`将当前进程的内存和寄存器替换为一个新的程序 (`/init`)，当 kernel 执行完毕`exec`指定的程序后，回到`/init`进程。`/init`(`user/init.c`) 创建了一个新的 console device 以文件描述符 0,1,2 打开，然后在 console device 中开启了一个 shell 进程，至此整个系统启动了