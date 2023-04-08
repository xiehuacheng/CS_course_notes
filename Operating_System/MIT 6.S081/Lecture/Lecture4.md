# Page tables

[翻译文本链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec04-page-tables-frans)

## Paging Hardware
---

xv6 运行于 Sv39 RISC-V，即在 64 位地址中只有最下面的 39 位被使用作为虚拟地址，其中底 12 位是页内偏移，高 27 位是页表索引，即 4096 字节 ($2^{12}$) 作为一个 page，一个进程的虚拟内存可以有 $2^{27}$ 个 page，对应到页表中就是 $2^{27}$ 个 ==page table entry (PTE)（页表记录）==。每个 PTE 有一个 44 位的 physical page number (PPN)用来映射到物理地址上和 10 位 flag，总共需要 54 位，也就是一个 PTE 需要 8 字节存储。即每个物理地址的高 44 位是页表中存储的 PPN，低 12 位是页内偏移，一个物理地址总共由 56 位构成。

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210123111515788.png)

在实际中，页表并不是作为一个包含了 $2^{27}$ 个 PTE 的大列表存储在物理内存中的，而是采用了**三级树状**的形式进行存储，这样可以让页表分散存储。每个页表就是一页。第一级页表是一个 4096 字节的页，包含了 512 个 PTE（因为每个 PTE 需要 8 字节），每个 PTE 存储了下级页表的页物理地址，第二级列表由 512 个页构成，第三级列表由 512\*512 个页构成。因为每个进程虚拟地址的高 27 位用来确定 PTE，对应到 3 级页表就是最高的 9 位确定一级页表 PTE 的位置，中间 9 位确定二级页表 PTE 的位置，最低 9 位确定三级页表 PTE 的位置。如下图所示。第一级根页表的物理页地址存储在**satp寄存器**中，每个 CPU 拥有自己独立的`satp`寄存器



![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210123130906787.png)

PTE flag 可以告诉硬件这些相应的虚拟地址怎样被使用，比如`PTE_V`表明这个 PTE 是否存在，`PTE_R`、`PTE_W`、`PTE_X`控制这个页是否允许被读取、写入和执行，`PTE_U`控制 user mode 是否有权访问这个页，如果`PTE_U`=0，则只有 supervisor mode 有权访问这个页。

## Kernel address space
---

每个进程有一个页表，用于描述进程的用户地址空间，还有一个内核地址空间（所有进程共享这一个描述内核地址空间的页表）。为了让内核使用物理内存和硬件资源，内核需要按照一定的规则排布内核地址空间，以能够确定哪个虚拟地址对应自己需要的硬件资源地址。用户地址空间不需要也不能够知道这个规则，因为用户空间不允许直接访问这些硬件资源。

QEMU 会模拟一个从 0x80000000 开始的 RAM，一直到 0x86400000。QEMU 会将设备接口以控制寄存器的形式暴露给内核，这些控制寄存器在 0x80000000 以下。kernel 对这些设备接口控制寄存器的访问是直接和这些设备而不是 RAM 进行交互的。

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210123171523476.png)

左边和右边分别是 kernel virtual address 和 physical address 的映射关系。在虚拟地址和物理地址中，kernel 都位于`KERNBASE=0x80000000`的位置，这叫做直接映射。

用户空间的地址分配在 free memory 中

有一些不是直接映射的内核虚拟地址：

*   trampoline page（和 user pagetable 在同一个虚拟地址，以便在 user space 和 kernel space 之间跳转时切换进程仍然能够使用相同的映射，真实的物理地址位于 kernel text 中的`trampoline.S`）
*   kernel stack page：每个进程有一个自己的内核栈 kstack，每个 kstack 下面有一个没有被映射的 guard page，guard page 的作用是防止 kstack 溢出影响其他 kstack。当进程运行在内核态时使用内核栈，运行在用户态时使用用户栈。**注意**：还有一个内核线程，这个线程只运行在内核态，不会使用其他进程的 kstack，内核线程没有独立的地址空间。

## Code: creating an address space
---

xv6 中和页表相关的代码在`kernel/vm.c`中。最主要的结构体是`pagetable_t`，这是一个指向页表的指针。`kvm`开头的函数都是和 kernel virtual address 相关的，`uvm`开头的函数都是和 user virtual address 相关的，其他的函数可以用于这两者

几个比较重要的函数：

*   `walk`：给定一个虚拟地址和一个页表，返回一个 PTE 指针
*   `mappages`：给定一个页表、一个虚拟地址和物理地址，创建一个 PTE 以实现相应的映射
    
*   `kvminit`用于创建 kernel 的页表，使用`kvmmap`来设置映射
*   `kvminithart`将 kernel 的页表的物理地址写入 CPU 的寄存器`satp`中，然后 CPU 就可以用这个 kernel 页表来翻译地址了
*   `procinit`(kernel/proc.c) 为每一个进程分配 (`kalloc`)kstack。`KSTACK`会为每个进程生成一个虚拟地址（同时也预留了 guard pages)，`kvmmap`将这些虚拟地址对应的 PTE 映射到物理地址中，然后调用`kvminithart`来重新把 kernel 页表加载到`satp`中去。

每个 RISC-V **CPU** 会把 PTE 缓存到 _Translation Look-aside Buffer (TLB)_ 中，当 xv6 更改了页表时，必须通知 CPU 来取消掉当前的 TLB，取消当前 TLB 的函数是`sfence.vma()`，在`kvminithart`中被调用

## Physical memory allocation for kernel
---

xv6 对 kernel space 和 PHYSTOP 之间的物理空间在运行时进行分配，分配以页 (4096 bytes) 为单位。分配和释放是通过对空闲页链表进行追踪完成的，分配空间就是将一个页从链表中移除，释放空间就是将一页增加到链表中

kernel 的物理空间的分配函数在`kernel/kalloc.c`中，每个页在链表中的元素是`struct run`，每个`run`存储在空闲页本身中。这个空闲页的链表`freelist`由 spin lock 保护，包装在`struct kmem`中。

*   `kinit()`：对分配函数进行初始化，将 kernel 结尾到 PHYSTOP 之间的所有空闲空间都添加到 kmem 链表中，这是通过调用`freerange(end, PHYSTOP)`实现的
*   `freerange()`对这个范围内所有页都调用一次`kfree`来将这个范围内的页添加到`freelist`链表中

## User space memory
---

每个进程有自己的用户空间下的虚拟地址，这些虚拟地址由每个进程自己的页表维护，用户空间下的虚拟地址从 0 到 MAXVA

当进程向 xv6 索要更多用户内存时，xv6 先用`kalloc`来分配物理页，然后向这个进程的页表增加指向这个新的物理页的 PTE，同时设置这些 PTE 的 flag

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210123220911443.png)

图 3.4 是一个进程在刚刚被`exec`调用时的用户空间下的内存地址，stack 只有一页，包含了`exec`调用的命令的参数从而使`main(argc, argv)`可以被执行。stack 下方是一个 guard page 来检测 stack 溢出，一旦溢出将会产生一个 page fault exception

`sbrk`是一个可以让进程增加或者缩小用户空间内存的 system call。`sbrk`调用了`growproc`(kernel/proc.c) 来改变`p->sz`从而改变 **heap** 中的 program break，`growproc`调用了`uvmalloc`和`uvmdealloc`，前者调用了`kalloc`来分配物理内存并且通过`mappages`向用户页表添加 PTE，后者调用了`kfree`来释放物理内存

## Code: exec
---

`exec`是一个 system call，为以 ELF 格式定义的文件系统中的可执行文件创建用户空间。

`exec`先检查头文件中是否有 ELF_MAGIC 来判断这个文件是否是一个 ELF 格式定义的二进制文件，用`proc_pagetable`来为当前进程创建一个还没有映射的页表，然后用`uvmalloc`来为每个 ELF segment 分配物理空间并在页表中建立映射，然后用`loadseg`来把 ELF segment 加载到物理空间当中。注意`uvmalloc`分配的物理内存空间可以比文件本身要大。

接下来`exec`分配 user stack，它仅仅分配一页给 stack，通过`copyout`将传入参数的 string 放在 stack 的顶端，在 ustack 的下方分配一个 guard page

如果`exec`检测到错误，将跳转到`bad`标签，释放新创建的`pagetable`并返回 - 1。`exec`必须确定新的执行能够成功才会释放进程旧的页表 (`proc_freepagetable(oldpagetable, oldsz)`)，否则如果 system call 不成功，就无法向旧的页表返回 - 1

## Real world
---

xv6 将 kernel 加载到 0x8000000 这一 RAM 物理地址中，但是实际上很多 RAM 的物理地址都是随机的，并不一定存在 0x8000000 这个地址

实际的处理器并不一定以 4096bytes 为一页，而可能使用各种不同大小的页