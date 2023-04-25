# Isolation & system call entry/exit

[课程翻译链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec06-isolation-and-system-call-entry-exit-robert)

## trap
---

3种可能的情况使得CPU暂停对正常指令的执行：
1. syscall，移交给kernel 
2. exception，指令执行了非法操作 
3. 设备中断

以上情况合并称为*trap*（陷入）

简单的个人理解：trap用于对以上三种情况进行处理，保存当前的状态（转到内核态），执行情况对应的trap指令，（回到用户态）
最后恢复状态

trap应该对于被打断的指令是透明的，也就是说被打断的指令不应该知道这个地方产生了trap，产生trap之后现场应该得以恢复并继续执行被打断的指令

xv6对trap的处理分为四个阶段：
1. RISC-V CPU的硬件的一些动作 
2. 汇编文件为了kernel C文件进行的一些准备 
3. 用C实现的trap handler 
4. system call / device-driver service routine

通常对于user space的trap、kernel space的trap和计时器中断会有不同的trap handler

## RISC-V trap machinery
---

RISC-V CPU有一系列的控制寄存器可以通知kernel发生了trap，也可以由kernel写入来告诉CPU怎样处理trap

-   `stvec`：trap handler的地址（即，指向了内核中处理trap的指令的起始地址，也就是trampoline page的地址），由kernel写入
	- 因为ecall不会切换用户进程的page table，所以内核需要将trampoline page映射到每一个user page table中

-   `sepc`：保存trap发生时的现场program counter，因为接下来`pc`要被取代为`stvec`。`sret`是从trap回到现场的指令，将`sepc`写回到`pc`

-   `scause`：一个trap产生的原因代码，由CPU写入

-   `sscratch`：放在trap handler的最开始处

-   `sstatus`：控制设备中断是否被开启，如果`sstatus`中的SPIE位被清除，则RISC-V将推迟设备中断，若设为1则开启中断。SPP位指示这个trap是在user space中产生的还是在kernel space产生的，并将控制`sret`回到什么模式。该bit为0表示下次执行sret的时候，我们想要返回user mode而不是supervisor mode

以上寄存器只在supervisor模式下发生的trap被使用 

接下来我们先来预览一下需要做的操作：

-   首先，我们需要保存32个用户寄存器。因为很显然我们需要恢复用户应用程序的执行，尤其是当用户程序随机的被设备中断所打断时。我们希望内核能够响应中断，之后在用户程序完全无感知的情况下再恢复用户代码的执行。所以这意味着32个用户寄存器不能被内核弄乱。但是这些寄存器又要被内核代码所使用，所以在trap之前，你必须先在某处保存这32个用户寄存器。

-   程序计数器也需要在某个地方保存，它几乎跟一个用户寄存器的地位是一样的，我们需要能够在用户程序运行中断的位置继续执行用户程序。

-   我们需要将mode改成supervisor mode，因为我们想要使用内核中的各种各样的特权指令。

-   SATP寄存器现在正指向user page table，而user page table只包含了用户程序所需要的内存映射和一两个其他的映射，它并没有包含整个内核数据的内存映射。所以在运行内核代码之前，我们需要将SATP指向kernel page table。

-   我们需要将堆栈寄存器指向位于内核的一个地址，因为我们需要一个堆栈来调用内核的C函数。

-   一旦我们设置好了，并且所有的硬件状态都适合在内核中使用， 我们需要跳入内核的C代码。

操作系统的一些high-level的目标能帮我们过滤一些实现选项。其中一个目标是安全和隔离，我们不想让用户代码介入到这里的user/kernel切换，否则有可能会破坏安全性。所以这意味着，trap中涉及到的硬件和内核机制不能依赖任何来自用户空间东西。比如说我们不能依赖32个用户寄存器，它们可能保存的是恶意的数据，所以，XV6的trap机制不会查看这些寄存器，而只是将它们保存起来

在操作系统的trap机制中，我们仍然想保留隔离性并防御来自用户代码的可能的恶意攻击。同样也很重要的是，另一方面，我们想要让trap机制对用户代码是透明的，也就是说我们想要执行trap，然后在内核中执行代码，同时用户代码并不用察觉到任何有意思的事情  

当发生除了计时器中断以外的其他类型的trap时，RISC-V将执行以下步骤：

1.  如果trap是一个设备产生的中断，而SIE（也可能是SPIE位）又被清除的情况下，不做下方的任何动作
2.  清除SIE（也可能是SPIE位）来disable一切中断
3.  把`pc`复制到`sepc`
4.  把当前的模式(user / supervisor)保存到SPP
5.  设置`scause`寄存器来指示产生trap的原因
6.  将当前的模式设置为supervisor
7.  将`stvec`的值复制到`pc`
8.  开始执行`pc`指向的trap handler的代码

注意CPU并没有切换到kernel页表，也没有切换到kernel栈

## 执行流程（重要）
---

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKFsfImgYCtnwA1d2hO%2F-MKHxleUqYy-y0mrS48w%2Fimage.png?alt=media&token=ab7c66bc-cf61-4af4-90fd-1fefc96c7b5f)

函数是一一对应的：
- uservec对应userret（二者均在trampoline汇编文件中）
- usertrap对应usertrapret

## ecall
---

ecall实际上只会改变三件事情（当然还包括上面提到的第1、2、4、5步）：

第一，ecall将代码从user mode改到supervisor mode。

第二，ecall将程序计数器的值保存在了SEPC寄存器。

第三，ecall会跳转到STVEC寄存器指向的指令。

ecall是一条CPU指令

## trampoline-uservec
---

需要依靠trampoline的汇编代码来进行实现

1. 保存现场（将32个通用寄存器的数据存放到trapframe中）
2. 把内核的page table、stack以及当前执行该进程的CPU号装载到寄存器中
3. 跳转到usertrap中继续进行

## usertrap
---

1. 分情况，执行系统调用/中断/异常的处理逻辑
2. 修改了STEVC的值（将STVEC指向了kernelvec变量，这是内核空间trap处理代码的位置），还可能会修改SEPC的值（先将SEPC的值保存到了trapframe中，然后开启了系统中断，此时如果切换到了其他进程则sepc会被覆盖）

## usertrapret
---

1. 将内核部分中的相关内容（即trapframe中的前五个寄存器）填入trapframe中，以便下一次从用户空间切换到内核空间时可以正确地使用这些数据
2. 恢复STEVC（将STVEC更新到指向用户空间的trap处理代码）、SEPC的值
3. 设置SSTAUS中的SPP（控制下次执行sret时返回的模式）和SPIE位（控制执行完sret之后是否打开中断）
	- 这里指的下一次其实就是这一次，因为是先设置好了SSTAUS寄存器再进入trampoline代码中执行sret

## trampoline-userret
---

需要依靠trampoline的汇编代码来进行实现

1. 恢复现场（将trapframe中保存的32个通用寄存器的值取出）
2. 把用户空间的pagetable、stack装载到寄存器中
3. 执行sret指令

### sret

1. 程序会根据SPP位设置对应的模式
2. SEPC寄存器的数值会被拷贝到PC寄存器中
3. 根据SPIE位来设置中断状态

## trapframe
---

XV6在每个user page table映射了trapframe page，这样每个进程都有自己的trapframe page。这个page包含了很多有趣的数据，但是现在最重要的数据是用来保存用户寄存器的32个空槽位。所以，在trap处理代码中，现在的好消息是，我们在user page table有一个之前由kernel设置好的映射关系，这个映射关系指向了一个可以用来存放这个进程的用户寄存器的内存位置。这个位置的虚拟地址总是0x3ffffffe000

在进入到user space之前，内核会将trapframe page的地址保存在SSCRATCH寄存器中，也就是0x3fffffe000这个地址

通常使用csrrw指令来进行寄存器内容的交换

在内核前一次切换回用户空间时，内核会执行set sscratch指令，将这个寄存器的内容设置为0x3fffffe000，也就是trapframe page的虚拟地址

一台机器总是从内核开始运行的，当机器启动的时候，它就是在内核中。 任何时候，不管是进程第一次启动还是从一个系统调用返回，进入到用户空间的唯一方法是就是执行sret指令，在执行sret指令时SSCRATCH寄存器就已经被设置好了

trampoline page在user page table中的映射与kernel page table中的映射是完全一样的。这两个page table中其他所有的映射都是不同的，只有trampoline page的映射是一样的，因此我们在切换page table时，寻址的结果不会改变，我们实际上就可以继续在同一个代码序列中执行程序而不崩溃

## interrupt
---

XV6会在处理系统调用的时候使能中断，这样中断可以更快的服务，有些系统调用需要许多时间处理。中断总是会被RISC-V的trap硬件关闭（意思应该是系统会在执行trap前关闭系统中断以免出错），所以在这个时间点，我们需要显式的打开中断

在从内核空间返回用户空间之前，usertrapret函数会再次显式的关闭中断

在返回到用户空间之后，sret函数会再次打开中断

## Traps from user space
---

当user space中发生trap时，会将`stvec`的值复制到`pc`，而此时`stvec`的值是`trampoline.S`中的`uservec`，因此跳转到`uservec`，先保存一些现场的寄存器，恢复kernel栈指针、kernel page table到`satp`寄存器，再跳转到`usertrap`(kernel/trap.c)trap handler，然后返回`usertrapret`(kernel/trap.c)，跳回到kernel/trampoline.S，最后用`userret`(kernel/trampoline.S)通过`sret`跳回到user space

RISC-V在trap中不会改变页表，因此user page table必须有对`uservec`的mapping，`uservec`是`stvec`指向的trap vector instruction。`uservec`要切换`satp`到kernel页表，同时kernel页表中也要有和user页表中对`uservec`相同的映射。RISC-V将`uservec`保存在_trampoline_页中，并将`TRAMPOLINE`放在kernel页表和user页表的相同位置处（MAXVA)

当`uservec`开始时所有的32个寄存器都是trap前代码的值，但是`uservec`需要对某些寄存器进行修改来设置`satp`，可以用`sscratch`和`a0`的值进行交换，交换之前的`sscratch`中是指向user process的`trapframe`的地址，`trapframe`中预留了保存所有32个寄存器的空间。`p->trapframe`保存了每个进程的`TRAPFRAME`的物理空间从而让kernel页表也可以访问该进程的trapframe

当交换完`a0`和`sscratch`之后，`uservec`可以通过`a0`把所有当前寄存器的值保存到trapframe中。由于当前进程的trapframe已经保存了当前进程的kernel stack、当前CPU的hartid、`usertrap`的地址、kernel page table的地址等，`uservec`需要获取这些值，然后切换到kernel pagetable，调用`usertrap`

`usertrap`主要是判断trap产生的原因并进行处理，然后返回。因为当前已经在kernel里了，所以这时候如果再发生trap，应该交给`kernelvec`处理，因此要把`stvec`切换为`kernelvec`。如果trap是一个system call，那么`syscall`将被调用，如果是设备中断，调用`devintr`，否则就是一个exception，kernel将杀死这个出现错误的进程

回到user space的第一步是调用`usertrapret()`，这个函数将把`stvec`指向`uservec`，从而当回到user space再出现trap的时候可以跳转到`uservec`，同时设置`p->trapframe`的一些值为下一次trap作准备，比如设置`p->trapframe->kernel_sp = p->kstack + PGSIZE`。清除`SPP`为从而使得调用`sret`后能够回到user mode。设置回到user space后的program counter为`p->trapframe->epc`，最后调用跳转到TRAMPOLINE页上的`userret`回到trampoline.S，加载user page table。`userret`被`userrapret`调用返回时a0寄存器中保存了TRAPFRAME，因此可以通过这个TRAPFRAME地址来恢复之前所有寄存器的值(包括a0)，最后把TRAPFRAME保存在sscratch中，用`sret`回到user space

## Calling system calls
---

user调用`exec`执行system call的过程：把给`exec`使用的参数放到a0和a1寄存器中，把system call的代码(SYS_exec)放到a7寄存器中，`ecall`指令将陷入内核中（通过usys.pl中的entry)。`ecall`的效果有三个，包括将CPU从user mode切换到supervisor mode、将`pc`保存到`epc`以供后续恢复、将`uservec`设置为`stvec`，并执行`uservec`、`usertrap`，然后执行`syscall`。kernel trap code将把寄存器的值保存在当前进程的trapframe中。syscall将把trapframe中的a7寄存器保存的值提取出来，索引到`syscalls`这个函数数列中查找对应的syscall种类，并进行调用，然后把返回值放置在`p->trapframe->a0`中，如果执行失败，就返回-1。

syscall的argument可以用`argint`、`argaddr`、`argfd`等函数从内存中取出

## Traps from kernel space
---

当执行kernel code发生CPU trap的时候，`stvec`是指向`kernelvec`的汇编代码的。`kernelvec`将寄存器的值保存在被中断的kernel thread的栈里而不是trapframe里，这样当trap需要切换kernel thread时，再切回来之后还可以从原先的thread栈里找到之前的寄存器值。

保存完寄存器之后，跳转到`kerneltrap`这个trap handler。`kerneltrap`可以对设备中断和exception这两种trap进行处理。如果是设备中断，调用`devintr`进行处理，如果是exception就panic，如果是因为计时器中断，就调用`yield`让其他kernel thread运行

最后返回到`kernelvec`中，`kernelvec`将保存的寄存器值从堆栈中弹出，执行`sret`，将`sepc`复制到`pc`来执行之前被打断的kernel code