# Page Fault

[课程翻译链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec08-page-faults-frans)

## Basics
---

当试图访问 PTE_V 为 0 的虚拟地址或 user 访问 PTE_U 为 0/kernel 访问 PTE_U 为 1 以及其他违反 PTE_W/PTE_R 等 flag 的情况下会出现 page faults。Page faults 是一个 exception，总共有 3 种 page faults：

1.  load page faults：当 load instruction 无法翻译虚拟地址时发生
2.  store page faults：当 store instruction 无法翻译虚拟地址时发生
3.  instruction page faults：当一个 instruction 的地址无法翻译时发生

page faults 种类的代码存放在`scause`寄存器中（保存了trap机制中进入到supervisor mode的原因，其中总共有3个类型的原因与page fault相关，分别是读、写和指令），无法翻译的地址（出错的地址）存放在`stval`寄存器中，出发page fault的指令的地址存放在SEPC中，并同时保存在trapflame->epc中，我们之所以关心触发page fault时的程序计数器值，是因为在page fault handler中我们或许想要修复page table，并重新执行对应的指令。

page fault可以让地址映射关系变得动态起来。通过page fault，内核可以更新page table，这是一个非常强大的功能

在 xv6 中对于 exception 一律都会将这个进程 kill 掉，但是实际上可以结合 page faults 实现一些功能。

1.  可以实现 _copy-on-write fork_。在`fork`时，一般都是将父进程的所有 user memory 复制到子进程中，但是`fork`之后一般会直接进行`exec`，这就会导致复制过来的 user memory 又被放弃掉。因此改进的思路是：子进程和父进程共享一个物理内存，但是 mapping 时将 PTE_W 置零，只有当子进程或者父进程的其中一个进程需要向这个地址写入时产生 page fault，此时才会进行 copy
2.  可以实现 _lazy allocation_。旧的`sbrk()`申请分配内存，但是申请的这些内存进程很可能不会全部用到，因此改进方案为：当进程调用`sbrk()`时，将修改`p->sz`，但是并不实际分配内存，并且将 PTE_V 置 0。当在试图访问这些新的地址时发生 page fault 再进行物理内存的分配
3.  _paging from disk_：当内存没有足够的物理空间时，可以先将数据存储在其他的存储介质（比如硬盘）上，，将该地址的 PTE 设置为 invalid，使其成为一个 evicted page。当需要读或者写这个 PTE 时，产生 Page fault，然后在内存上分配一个物理地址，将这个硬盘上的 evicted page 的内容写入到该内存上，设置 PTE 为 valid 并且引用到这个内存物理地址

## sbrk
---

sbrk是XV6提供的系统调用，它使得用户应用程序能扩大自己的heap。当一个应用程序启动的时候，sbrk指向的是heap的最底端，同时也是stack的最顶端。这个位置通过代表进程的数据结构中的sz字段表示，这里以p->sz表示，sbrk会扩展heap的上边界（也就是会扩大heap）。如果传入的值是负数的话则会缩小内存。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMSxponnGmjT-9o9zTI%2F-MMSzCvRiaVMkJF9feJx%2Fimage.png?alt=media&token=493295ce-507e-44a9-a4d4-d400b9f14aee)

在XV6中，sbrk的实现默认是eager allocation。这表示了，一旦调用了sbrk，内核会立即分配应用程序所需要的物理内存。

但是实际上，对于应用程序来说很难预测自己需要多少内存，所以通常来说，应用程序倾向于申请多于自己所需要的内存。这意味着，进程的内存消耗会增加许多，但是有部分内存永远也不会被应用程序所使用到。

## lazy allocation
---

核心思想非常简单，sbrk系统调基本上不做任何事情，唯一需要做的事情就是提升_p->sz_，将_p->sz_增加n，其中n是需要新分配的内存page数量。但是内核在这个时间点并不会分配任何物理内存。之后在某个时间点，应用程序使用到了新申请的那部分内存，这时会触发page fault，因为我们还没有将新的内存映射到page table。所以，如果我们解析一个大于旧的_p->sz_，但是又小于新的p->sz（注，也就是旧的p->sz + n）的虚拟地址，我们希望内核能够分配一个内存page，并且重新执行指令。

## zero fill on demand
---

当你查看一个用户程序的地址空间时，存在text区域，data区域，同时还有一个BSS区域（注，BSS区域包含了未被初始化或者初始化为0的全局或者静态变量）。当编译器在生成二进制文件时，编译器会填入这三个区域。text区域是程序的指令，data区域存放的是初始化了的全局变量，BSS包含了未被初始化或者初始化为0的全局变量。

在一个正常的操作系统中，如果执行exec，exec会申请地址空间，里面会存放text和data。因为BSS里面保存了未被初始化的全局变量，这里或许有许多许多个page，但是所有的page内容都为0。

通常可以调优的地方是，我有如此多的内容全是0的page，在物理内存中，我只需要分配一个page，这个page的内容全是0。然后将所有虚拟地址空间的全0的page都map到这一个物理page上。这样至少在程序启动的时候能节省大量的物理内存分配。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjOUZgcx_hv79eQ_1r%2Fimage.png?alt=media&token=9411955a-324a-4db8-b4ea-be97bcd411df)

当然这里的mapping需要非常的小心，我们不能允许对于这个page执行写操作，因为所有的虚拟地址空间page都期望page的内容是全0，所以这里的PTE都是只读的。之后在某个时间点，应用程序尝试写BSS中的一个page时，比如说需要更改一两个变量的值，我们会得到page fault。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjR5Fl20ATXIG70FFk%2Fimage.png?alt=media&token=6d3c6a13-7fa4-4ce6-b569-ceec31fa9694)

好处
1. 类似于lazy allocation。假设程序申请了一个大的数组，来保存可能的最大的输入，并且这个数组是全局变量且初始为0。但是最后或许只有一小部分内容会被使用。
2. exec中需要做的工作变少了。程序可以启动的更快，这样你可以获得更好的交互体验，因为你只需要分配一个内容全是0的物理page。所有的虚拟page都可以映射到这一个物理page上。

代价：一个page fault需要从用户空间进入一次内核（trap），同时trap机制中需要执行多次保存寄存器的操作，开销相比访问内存的指令要大

## copy on write fork(COW fork)
---

当Shell处理指令时，它会通过fork创建一个子进程。fork会创建一个Shell进程的拷贝，所以这时我们有一个父进程（原来的Shell）和一个子进程。Shell的子进程执行的第一件事情就是调用exec运行一些其他程序，比如运行echo。现在的情况是，fork创建了Shell地址空间的一个完整的拷贝，而exec做的第一件事情就是丢弃这个地址空间，取而代之的是一个包含了echo的地址空间。这里看起来有点浪费。

优化：当我们创建子进程时，与其创建，分配并拷贝内容到新的物理内存，其实我们可以直接共享父进程的物理内存page。所以这里，我们可以设置子进程的PTE指向父进程对应的物理内存page。

当然，再次要提及的是，我们这里需要非常小心。因为一旦子进程想要修改这些内存的内容，相应的更新应该对父进程不可见，因为我们希望在父进程和子进程之间有强隔离性，所以这里我们需要更加小心一些。为了确保进程间的隔离性，我们可以将这里的父进程和子进程的PTE的标志位都设置成只读的。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjvLvMe8CobFN8-afp%2Fimage.png?alt=media&token=fa092f13-afdf-4891-822d-7cab88f2872f)

在得到page fault之后，我们需要拷贝相应的物理page。假设现在是子进程在执行store指令，那么我们会分配一个新的物理内存page，然后将page fault相关的物理内存page拷贝到新分配的物理内存page中，并将新分配的物理内存page映射到子进程。这时，新分配的物理内存page只对子进程的地址空间可见，所以我们可以将相应的PTE设置成可读写，并且我们可以重新执行store指令。实际上，对于触发刚刚page fault的物理page，因为现在只对父进程可见，相应的PTE对于父进程也变成可读写的了。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjwgQ0xT6_bDgbtOid%2Fimage.png?alt=media&token=8d124d47-9789-4851-9c60-078549f42a1f)

>Frans教授：内核必须要能够识别这是一个copy-on-write场景。几乎所有的page table硬件都支持了这一点。我们之前并没有提到相关的内容，下图是一个常见的多级page table。对于PTE的标志位，我之前介绍过第0bit到第7bit，但是没有介绍最后两位RSW。这两位保留给supervisor software使用，supervisor softeware指的就是内核。内核可以随意使用这两个bit位。所以可以做的一件事情就是，将bit8标识为当前是一个copy-on-write page。

![](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMk2SxJ_cq94IMH6I6W%2Fimage.png?alt=media&token=e0dad55b-5c74-4da0-a3b0-a89d462eda58)

>当内核在管理这些page table时，对于copy-on-write相关的page，内核可以设置相应的bit位，这样当发生page fault时，我们可以发现如果copy-on-write bit位设置了，我们就可以执行相应的操作了。

当父进程退出时我们需要更加的小心，因为我们要判断是否能立即释放相应的物理page。如果有子进程还在使用这些物理page，而内核又释放了这些物理page，我们将会出问题。那么现在释放内存page的依据是什么呢？

我们需要对于每一个物理内存page的引用进行计数，当我们释放虚拟page时，我们将物理内存page的引用数减1，如果引用数等于0，那么我们就能释放物理内存page。

## demand paging
---

程序的二进制文件可能非常的巨大，将它全部从磁盘加载到内存中将会是一个代价很高的操作。又或者data区域的大小远大于常见的场景所需要的大小，我们并不一定需要将整个二进制都加载到内存中。

所以对于exec，在虚拟地址空间中，我们为text和data分配好地址段，但是相应的PTE并不对应任何物理内存page。对于这些PTE，我们只需要将valid bit位设置为0即可。

在最坏的情况下，用户程序使用了text和data中的所有内容，那么我们将会在应用程序的每个page都收到一个page fault。但是如果我们幸运的话，用户程序并没有使用所有的text区域或者data区域，那么我们一方面可以节省一些物理内存，另一方面我们可以让exec运行的更快（注，因为不需要为整个程序分配内存）。

如果内存耗尽了，一个选择是撤回page（evict page）。比如说将部分内存page中的内容写回到文件系统再撤回page。一旦你撤回并释放了page，那么你就有了一个新的空闲的page，你可以使用这个刚刚空闲出来的page，分配给刚刚的page fault handler，再重新执行指令。

什么样的page可以被撤回？并且该使用什么样的策略来撤回page？Least Recently Used，或者叫LRU。

如果你要撤回一个page，你需要在dirty page和non-dirty page中做选择。dirty page是曾经被写过的page，而non-dirty page是只被读过，但是没有被写过的page。你们会选择哪个来撤回？如果dirty page之后再被修改，现在你或许需要对它写两次了（注，一次内存，一次文件），现实中会选择non-dirty page。如果non-dirty page出现在page table中，你可以将内存page中的内容写到文件中，之后将相应的PTE标记为non-valid，这就完成了所有的工作。

在bit7，对应的就是Dirty bit。当硬件向一个page写入数据，会设置dirty bit，之后操作系统就可以发现这个page曾经被写入了。类似的，还有一个Access bit，任何时候一个page被读或者被写了，这个Access bit会被设置。

> 学生回答：没有被Access过的page可以直接撤回，是吗？
> 
>Frans教授：是的，或者说如果你想实现LRU，你需要找到一个在一定时间内没有被访问过的page，那么这个page可以被用来撤回。而被访问过的page不能被撤回。所以Access bit通常被用来实现这里的LRU策略。

> 学生提问：那是不是要定时的将Access bit恢复成0？
> 
> Frans教授：是的，这是一个典型操作系统的行为。操作系统会扫描整个内存，这里有一些著名的算法例如clock algorithm，就是一种实现方式。

## memory mapped files
---

这里的核心思想是，将完整或者部分文件加载到内存中，这样就可以通过内存地址相关的load或者store指令来操纵文件。为了支持这个功能，一个现代的操作系统会提供一个叫做mmap的系统调用。这个系统调用会接收一个虚拟内存地址（VA），长度（len），protection，一些标志位，一个打开文件的文件描述符，和偏移量（offset）。

这里的语义就是，从文件描述符对应的文件的偏移量的位置开始，映射长度为len的内容到虚拟内存地址VA，同时我们需要加上一些保护，比如只读或者读写。

当然，在任何聪明的内存管理机制中，所有的这些都是以lazy的方式实现。你不会立即将文件内容拷贝到内存中，而是先记录一下这个PTE属于这个文件描述符。相应的信息通常在VMA结构体中保存，VMA全称是Virtual Memory Area。例如对于这里的文件f，会有一个VMA，在VMA中我们会记录文件描述符，偏移量等等，这些信息用来表示对应的内存虚拟地址的实际内容在哪，这样当我们得到一个位于VMA地址范围的page fault时，内核可以从磁盘中读数据，并加载到内存中。

>学生提问：有没有可能多个进程将同一个文件映射到内存，然后会有同步的问题？
>
>Frans教授：好问题。这个问题其实等价于，多个进程同时通过read/write系统调用读写一个文件会怎么样？
>
>这里的行为是不可预知的。write系统调用会以某种顺序出现，如果两个进程向一个文件的block写数据，要么第一个进程的write能生效，要么第二个进程的write能生效，只能是两者之一生效。在这里其实也是一样的，所以我们并不需要考虑冲突的问题。
>
>一个更加成熟的Unix操作系统支持锁定文件，你可以先锁定文件，这样就能保证数据同步。但是默认情况下，并没有同步保证。