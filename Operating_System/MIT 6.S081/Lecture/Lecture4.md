# Page tables

[翻译文本链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec04-page-tables-frans)

## Virtual Memory
---

最常见的方法，同时也是非常灵活的一种方法就是使用页表（Page Tables）。页表是在硬件中通过处理器和==内存管理单元（MMU，Memory Management Unit）==实现

对于任何一条带有地址的指令，其中的地址应该认为是虚拟内存地址而不是物理地址。假设寄存器a0中是地址0x1000，那么这是一个虚拟内存地址。虚拟内存地址会被转到内存管理单元

内存管理单元会将虚拟地址翻译成物理地址。之后这个物理地址会被用来索引物理内存，并从物理内存加载，或者向物理内存存储数据

为了能够完成虚拟内存地址到物理内存地址的翻译，MMU会有一个表单，表单中，一边是虚拟内存地址，另一边是物理内存地址（这里说的表单其实就是页表）

通常来说，内存地址对应关系的表单也保存在内存中。所以CPU中需要有一些寄存器用来存放表单在物理内存中的地址。现在，在内存的某个位置保存了地址关系表单，我们假设这个位置的物理内存地址是0x10。那么在RISC-V上一个叫做**SATP**的寄存器会保存地址0x10

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/assets_-MHZoT2b_bcLghjAOPsJ_-MKKjB2an4WcuUmOlE___-MKONgZr-r8W5uRpknWQ_image.webp)

这里的基本想法是每个应用程序都有自己独立的表单，并且这个表单定义了应用程序的地址空间。所以当操作系统将CPU从一个应用程序切换到另一个应用程序时，同时也需要切换SATP寄存器中的内容，从而指向新的进程保存在物理内存中的地址对应表单

## Paging Hardware
---

xv6 运行于 Sv39 RISC-V，即在 64 位地址中只有最下面的 39 位被使用作为虚拟地址，其中底 12 位是页内偏移，高 27 位是页表索引，即 4096 字节 ($2^{12}$) 作为一个 page（每个page中的内容在物理内存中是连续的），一个进程的虚拟内存可以有 $2^{27}$ 个 page，对应到页表中就是 $2^{27}$ 个 ==page table entry (PTE)（页表记录）==。每个 PTE 有一个 44 位的 **physical page number (PPN)用来映射到物理地址上**和 10 位 flag，总共需要 54 位，也就是一个 PTE 需要 8 字节存储。即每个物理地址的高 44 位是页表中存储的 PPN，低 12 位是页内偏移，一个物理地址总共由 56 位构成（56位是硬件设计人员根据当前或者未来一段时间内的硬件水平进行估算来选取的）。

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210123111515788.png)

在实际中，页表并不是作为一个包含了 $2^{27}$ 个 PTE 的大列表存储在物理内存中的，而是采用了**三级树状**的形式进行存储，这样可以让页表分散存储。每个页表就是一页。第一级页表是一个 4096 字节的页，包含了 512 个 PTE（因为每个 PTE 需要 8 字节），每个 PTE 存储了下级页表的页物理地址，第二级列表由 512 个页构成，第三级列表由 512\*512 个页构成。因为每个进程虚拟地址的高 27 位用来确定 PTE，对应到 3 级页表就是最高的 9 位确定一级页表 PTE 的位置，中间 9 位确定二级页表 PTE 的位置，最低 9 位确定三级页表 PTE 的位置。如下图所示。第一级根页表的物理页地址存储在**satp寄存器**中，每个 CPU 拥有自己独立的`satp`寄存器。

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210123130906787.png)

PTE flag 可以告诉硬件这些相应的虚拟地址怎样被使用，比如`PTE_V`表明这个 PTE 是否存在，`PTE_R`、`PTE_W`、`PTE_X`控制这个页是否允许被读取、写入和执行，`PTE_U`控制 user mode 是否有权访问这个页，如果`PTE_U`=0，则只有 supervisor mode 有权访问这个页。

这里的Reserved是留作未来扩展的。

我对三级页表索引的理解：目的是将页表分块，如果某个一级、二级或者三级页表中的空间都没有被使用，则目前还不需要创建这个页表，理论上来将可以做很多级的页表索引，越多级的索引（使用/开辟）的比值就越接近于1，但是每一次操作的时间开销也会增大，所以三级页表是一个权衡后的选择。

开辟地址空间应该是在进程被创建的时候，而不是当一个进程尝试访问某个未被开辟的地址空间时（此时MMU会告诉操作系统或者处理器，抱歉我不能翻译这个地址，最终这会变成一个page fault。如果一个地址不能被翻译，那就不翻译）。

计算每个page的地址的方式：我们用44bit的PPN，再加上12bit的0，这样就得到了下一级page directory的56bit物理地址。这里要求每个page directory都与物理page对齐（也就是page directory的起始地址就是某个page的起始地址，所以低12bit都为0）。

这是由硬件实现的，所以3级 page table的查找都发生在硬件中。MMU是硬件的一部分而不是操作系统的一部分。在XV6中，有一个函数也实现了page table的查找，因为时不时的XV6也需要完成硬件的工作，所以XV6有这个叫做walk的函数，它在软件中实现了MMU硬件相同的功能。

page table提供了一层抽象（level of indirection）。我这里说的抽象就是指从虚拟地址到物理地址的映射。这里的映射关系完全由操作系统控制。因为操作系统对于这里的地址翻译有完全的控制，它可以实现各种各样的功能。比如，当一个PTE是无效的，硬件会返回一个page fault，对于这个page fault，操作系统可以更新 page table并再次尝试指令。

## Translation Lookaside Buffer
---

对于一个虚拟内存地址的寻址，需要读三次内存，这里代价有点高。所以实际中，几乎所有的处理器都会对于最近使用过的虚拟地址的翻译结果有缓存。这个缓存被称为：==Translation Lookside Buffer（通常翻译成页表缓存）==。你会经常看到它的缩写==TLB==。基本上来说，这就是Page Table Entry的缓存，也就是PTE的缓存。

当处理器第一次查找一个虚拟地址时，硬件通过3级page table得到最终的PPN，TLB会保存虚拟地址到物理地址的映射关系。这样下一次当你访问同一个虚拟地址时，处理器可以查看TLB，TLB会直接返回物理地址，而不需要通过page table得到结果。

如果你切换了page table，操作系统需要告诉处理器当前正在切换page table，处理器会清空TLB。因为本质上来说，如果你切换了page table，TLB中的缓存将不再有用，它们需要被清空，否则地址翻译可能会出错。

## Kernel address space
---

每个进程有一个页表，用于描述进程的用户地址空间，还有一个内核地址空间（所有进程共享这一个描述内核地址空间的页表）。为了让内核使用物理内存和硬件资源，内核需要按照一定的规则排布内核地址空间，以能够确定哪个虚拟地址对应自己需要的硬件资源地址。用户地址空间不需要也不能够知道这个规则，因为用户空间不允许直接访问这些硬件资源。

QEMU 会模拟一个从 0x80000000 开始的 RAM，一直到 0x86400000。QEMU 会将设备接口以控制寄存器的形式暴露给内核，这些控制寄存器在 0x80000000 以下。kernel 对这些设备接口控制寄存器的访问是直接和这些设备而不是 RAM 进行交互的。

![](https://fanxiao.tech/assets/img/posts/MIT_6S081/image-20210123171523476.png)

左边和右边分别是 kernel virtual address 和 physical address 的映射关系。在虚拟地址和物理地址中，kernel 都位于`KERNBASE=0x80000000`的位置，这叫做直接映射。

**用户空间的地址分配在 free memory 中**

有一些不是直接映射的内核虚拟地址：

*   trampoline page（和 user pagetable 在同一个虚拟地址，以便在 user space 和 kernel space 之间跳转时切换进程仍然能够使用相同的映射，真实的物理地址位于 kernel text 中的`trampoline.S`）
*   kernel stack page：每个进程有一个自己的内核栈 kstack，每个 kstack 下面有一个没有被映射的 guard page，guard page 的作用是防止 kstack 溢出影响其他 kstack。当进程运行在内核态时使用内核栈，运行在用户态时使用用户栈。**注意**：还有一个内核线程，这个线程只运行在内核态，不会使用其他进程的 kstack，内核线程没有独立的地址空间。

↑这部分在下面有更加详细的说明

回到最初那张图的右侧：物理地址的分布。可以看到最下面是未被使用的地址，这与主板文档内容是一致的（地址为0）。地址0x1000是boot ROM的物理地址，当你对主板上电，主板做的第一件事情就是运行存储在boot ROM中的代码，当boot完成之后，会跳转到地址0x80000000，操作系统需要确保那个地址有一些数据能够接着启动操作系统。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaeaT3eXOyG4jfKKU7%2Fimage.png?alt=media&token=a04af08d-3c8d-4c61-a63d-6376dec252ea)

接下来我会切换到第一张图的左边，这就是XV6的虚拟内存地址空间。当机器刚刚启动时，还没有可用的page，XV6操作系统会设置好内核使用的虚拟地址空间，也就是这张图左边的地址分布。

因为我们想让XV6尽可能的简单易懂，所以这里的虚拟地址到物理地址的映射，大部分是相等的关系。比如说内核会按照这种方式设置page table，虚拟地址0x02000000对应物理地址0x02000000。这意味着左侧低于PHYSTOP的虚拟地址，与右侧使用的物理地址是一样的。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKalBg8KcCxk0HNNgOr%2Fimage.png?alt=media&token=17dfd137-31cc-4262-8f7b-d00df4d3b9f1)

所以，这里的箭头都是水平的，因为这里是完全相等的映射。

除此之外，这里还有两件重要的事情：

第一件事情是，有一些page在虚拟内存中的地址很靠后，比如kernel stack在虚拟内存中的地址就很靠后。这是因为在它之下有一个未被映射的Guard page，这个Guard page对应的PTE的Valid 标志位没有设置，这样，如果kernel stack耗尽了，它会溢出到Guard page，但是因为Guard page的PTE中Valid标志位未设置，会导致立即触发page fault，这样的结果好过内存越界之后造成的数据混乱。立即触发一个panic（也就是page fault），你就知道kernel stack出错了。同时我们也又不想浪费物理内存给Guard page，所以Guard page不会映射到任何物理内存，它只是占据了虚拟地址空间的一段靠后的地址。

同时，kernel stack被映射了两次，在靠后的虚拟地址映射了一次，在PHYSTOP下的Kernel data中又映射了一次，但是实际使用的时候用的是上面的部分，因为有Guard page会更加安全。**（这里还是有点不明白）**

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKbZEkzbzbKYgRRedXU%2Fimage.png?alt=media&token=2167acef-e76c-4d0c-81b0-f5175475793f)

第二件事情是权限。例如Kernel text page被标位R-X，意味着你可以读它，也可以在这个地址段执行指令，但是你不能向Kernel text写数据。通过设置权限我们可以尽早的发现Bug从而避免Bug。对于Kernel data需要能被写入，所以它的标志位是RW-，但是你不能在这个地址段运行指令，所以它的X标志位未被设置。（注，所以，kernel text用来存代码，代码可以读，可以运行，但是不能篡改，kernel data用来存数据，数据可以读写，但是不能通过数据伪装代码在kernel中运行）

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKb_jHb6u2XMiYkH2i0%2F-MKeHhKXkn4VMBskZjDS%2Fimage.png?alt=media&token=64e16ad2-c25b-4535-bb67-cc340934c027)

在kernel page table中，有一段Free Memory，它对应了物理内存中的一段地址。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKb_jHb6u2XMiYkH2i0%2F-MKeJFYcE1NyVZ0QMc67%2Fimage.png?alt=media&token=2bdc7a95-2b87-4c3e-ab06-76120962ec67)

**XV6使用这段free memory来存放用户进程的page table**，text和data。如果我们运行了非常多的用户进程，某个时间点我们会耗尽这段内存，这个时候fork或者exec会返回错误。

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