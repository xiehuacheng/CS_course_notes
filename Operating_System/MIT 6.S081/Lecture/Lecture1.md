# Introduction and Examples

[翻译文本链接](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec01-introduction-and-examples)

## 操作系统的共同目标
---

1. 抽象硬件
2. 硬件复用
3. 隔离
4. 共享
5. 安全系统、权限系统
6. 帮助应用程序获得更好的性能
7. 支持不同类型的使用和操作

## OS organization
---

- 在用户空间运行程序

- 内核（kernel）为运行的程序提供服务的一种特殊程序
	- 用于管理数据和计算机资源
	- 内核中也内置了一些服务，如：文件系统、进程管理
	- 计算机启动时即存在
	- 可以直接访问和操作硬件
	- 一个计算机可以拥有多个进程，但是只能有一个内核

- 程序访问内核的方式：系统调用接口（System call interface）
	- system call进入内核执行相应的服务然后返回

-   shell：一个普通的程序，其功能是让用户输入命令并执行它们，shell**不是**内核的一部分

## Processes and memory
---

- 每个运行着的程序叫做进程，每个进程的内存中存储指令、数据和堆栈
- 每个进程拥有自己的用户空间内存以及内核空间状态，当进程不再执行时xv6将存储和这些进程相关的CPU寄存器直到下一次运行这些进程。kernel将每一个进程用一个PID(process identifier)指代

## 课程用到的资源
---

- 操作系统：XV6
- 运行环境：RISC-V指令集处理器；本课程中使用QEMU模拟器代替

## System call
---

### copy

- 形式：`int copy()`
- 作用：从buffer获取一段输入并将其输出

### open

- 形式：`int open()`
- 作用：创建一个新文件，并写入一串字符

### fork

- 形式：`int fork()`
- 作用：拷贝当前进程的内存，并创建一个新的进程
- 返回值：父进程返回子进程的PID，子进程返回0

### exec

- 形式：`int exec(char *file, char *argv[])`
- 作用：从指定的文件中读取指令，并替换调用进程，丢弃当前内存
- 返回值：正常情况下不会产生返回（因为原来的exec程序已经被替换成了新的程序），除非exec在从文件中读取指令时发生错误

### forkexec

- 形式：`int forkexec(char *file, char *argv[])`
- 作用：创建一个子进程来执行exec操作，而父进程使用wait函数来等待子进程进行返回
- 优化：每次fork出来的子进程总会被exec完全替换，所以fork的复制操作就相当于是多余的，所以可以采用写入时再进行复制（copy on write）的方法来进行优化

### exit

- 形式：`int exit(int status)`
- 作用：让调用它的进程停止执行并且将内存等占用的资源全部释放
- 参数：需要一个整数形式的状态参数，0代表以正常状态退出，1代表以非正常状态退出

### wait

- 形式：`int wait(int *status)`
- 作用：等待子进程结束并返回结束的子进程的PID，当多个子进程中的一个子进程返回时，则wait结束，所以如果需要等待所有的子进程结束，则每一个fork都要有一个wait来对应
	- 子进程的退出状态存储到`int *status`这个地址中
	- 若当前进程没有子进程，则wait会返回-1（错误）
- 如果父进程不关心子进程的退出状态,则可以传递一个参数0来进行等待

### redirect

- 形式：`int redirect()`
- 作用：用于I/O重定向（结合下文的I/O部分来理解）

![](https://cdn.staticaly.com/gh/xiehuacheng/pictures@master/img/Pasted%20image%2020230405113730.png)

### 常见的系统调用

![](https://cdn.staticaly.com/gh/xiehuacheng/pictures@master/img/Pasted%20image%2020230405131007.png)

## I/O and File descriptors
---

- 用来表示一个被内核管理的、可以被进程读/写的对象的一个整数，表现形式类似于字节流，通过打开文件、目录、设备等方式获得。一个文件被打开得越早，文件描述符就越小

- 新分配的文件描述符总是当前进程中编号最小的未使用描述符

- 按照惯例，进程从文件描述符 0（标准输入）读取，将输出写入文件描述符 1（标准输出），并将错误消息写入文件描述符 2（标准错误），shell将保证总是有3个文件描述符是可用的

```c
while ((fd = open("console", O_RDWR)) >= 0)
{
    if (fd >= 3)
    {
        close(fd);
        break;
    }
}
 ```

### read和write

- 形式：`int write(int fd, char *buf, int n)`和`int read(int fd, char *bf, int n)`
- 作用：从/向文件描述符`fd`读/写n字节`bf`的内容。每个文件描述符有一个offset，`read`会从这个offset开始读取内容，读完n个字节之后将这个offset后移n个字节，下一个`read`将从新的offset开始读取字节。`write`也有类似的offset
- 返回值：成功读取/写入的字节数

```c
/* essence of cat program */
char buf[512];
int n;
  
for (;;) {
    n = read(0, buf, sizeof buf);
    if (n == 0)
        break;
    if (n < 0){
        fprintf(2, "read errot\n");
        exit(1);
    }
    if (write(1, buf, n) != n){
        fprintf(2, "write error\n");
        exit(1);
    }
}
```

### close

- 形式：`int close(int fd)`
- 作用：将打开的文件`fd`释放，使该文件描述符可以被后面的`open`、`pipe`等其他system call使用

### dup

- 形式：`int dup(int fd)`
- 作用：复制一个新的`fd`指向的I/O对象，返回这个新fd值，两个I/O对象(文件)的offset相同

```c
fd = dup(1);
write(1, "hello ", 6);
write(fd, "world\n", 6);
// outputs hello world
```

- 除了`dup`和`fork`之外，其他方式**不能**使两个I/O对象的offset相同，比如同时`open`相同的文件

Pipes
---

- 暴露给进程的一对文件描述符，一个文件描述符用来读，另一个文件描述符用来写，将数据从管道的一端写入，将使其能够被从管道的另一端读出

### pipe

- 形式：`int pipe(int p[])`
- 参数：`p[0]`为读取的文件描述符，`p[1]`为写入的文件描述符

```c
case PIPE:
pcmd = (struct pipecmd*)cmd;
if(pipe(p) < 0)
    panic("pipe");
if(fork1() == 0){
    // in child process
    close(1); // close stdout
    dup(p[1]); // make the fd 1 as the write end of pipe
    close(p[0]);
    close(p[1]);
    runcmd(pcmd->left); // run command in the left side of pipe |, output redirected to the write end of pipe
}
if(fork1() == 0){
    // in child process
    close(0); // close stdin
    dup(p[0]); // make the fd 0 as the read end of pipe
    close(p[0]);
    close(p[1]);
    runcmd(pcmd->right); //  run command in the right side of pipe |, input redirected to the read end of pipe
}
close(p[0]);
close(p[1]);
wait(0); // wait for child process to finish
wait(0); // wait for child process to finish
break;
```

## File system
---

- xv6文件系统包含了*文件*(byte arrays)和*目录*(对其他文件和目录的引用)。目录生成了一个树，树从根目录`/`开始。对于不以`/`开头的路径，认为是是相对路径

### mknod

- 创建设备文件，一个设备文件有一个major device 和一个minor device 用来唯一确定这个设备。当一个进程打开了这个设备文件时，内核会将`read`和`write`的system call重新定向到设备上。

-   一个文件的名称和文件本身是不一样的，文件本身，也叫_inode_，可以有多个名字，也叫_link_，每个link包括了一个文件名和一个对inode的引用。一个inode存储了文件的元数据，包括该文件的类型(file, directory or device)、大小、文件在硬盘中的存储位置以及指向这个inode的link的个数

### fstat

- 形式：`int fstat(int fd, struct stat *st)`
- 作用：将inode中的相关信息存储到`st`中。

### link

- 将创建一个指向同一个inode的文件名。`unlink`则是将一个文件名从文件系统中移除，只有当指向这个inode的文件名的数量为0时这个inode以及其存储的文件内容才会被从硬盘上移除

注意：Unix提供了许多在**用户层面**的程序来执行文件系统相关的操作，比如`mkdir`、`ln`、`rm`等，而不是将其放在shell或kernel内，这样可以使用户比较方便地在这些程序上进行扩展。但是`cd`是一个例外，它是在shell程序内构建的，因为它必须要改变这个calling shell本身指向的路径位置，如果是一个和shell平行的程序，那么它必须要调用一个子进程，在子进程里起一个新的shell，再进行`cd`，这是不符合常理的