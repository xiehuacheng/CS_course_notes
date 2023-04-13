## 链接
---

https://pdos.csail.mit.edu/6.828/2021/labs/util.html

## 细节
---

1. main函数输入参数中的argc为该函数接收的参数个数，argv为存储参数的数组
	- 系统调用名称本身视为一个参数
2. 管道的关闭需要特别注意
	- 如果不是递归的创建进程的话，不要随意关闭管道
3. 进程结束时需要调用exit()指令
4. 注意wait()的使用
5. 调用实验的测试时需要在前面加上sudo以获得更高的权限
6. 在xv6系统中调用函数之前应确保其已经在Makefile文件中进行了声明

## 关于文件描述符的关闭
---

在xv6系统中，如果你在创建一个新进程时，使用了`fork`系统调用，新进程会复制原进程的所有打开的文件描述符。因此，如果在原进程中打开了一些不需要的文件描述符，它们也会被复制到新进程中。

为了避免出现问题，你可以在创建新进程之后，使用`close`系统调用手动关闭不需要的文件描述符。这可以确保新进程只保留它需要的文件描述符，并避免出现资源泄漏的问题。注意，如果你不关闭不需要的文件描述符，可能会导致一些意想不到的问题，例如文件描述符泄露、进程崩溃等。

在使用`fork`系统调用创建子进程之后，可以使用`close`系统调用关闭不需要的文件描述符，例如：

```c
int pid = fork();
if (pid == 0)
{
    // 子进程
    close(fd); // 关闭不需要的文件描述符
    exec("some_command", args);
    exit(0);
}
else if (pid > 0)
{
    // 父进程
    wait(0);   // 等待子进程结束
    close(fd); // 关闭不需要的文件描述符
}
```

在这个例子中，如果`fd`是一个不需要的文件描述符，可以在子进程中使用`close`系统调用关闭它，以确保子进程只保留需要的文件描述符。在父进程中也可以关闭`fd`，以确保不会出现资源泄漏的问题。

## 实现
---

### sleep(easy)

- 执行系统准备好的sleep函数

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char const *argv[])
{
    if (argc != 2) // 判断参数个数是否为2
    {
        fprintf(2, "Usage: sleep seconds\n");
        exit(1);
    }
    int time = atoi(argv[1]); // 将char转为int
    sleep(time);
    exit(0);
}
```

### pingpong(easy)

- 创建管道，链接父子进程的0和1文件描述符，进行数据传输

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char const *argv[])
{
    int pid;
    int p[2];
    pipe(p); // 创建管道，0是读取，1是输出

    if (fork() == 0)
    {
        pid = getpid();
        char buf[2];
        if (read(p[0], buf, 1) != 1) // 判断读取是否成功
        {
            fprintf(2, "failed to read in child\n");
            exit(1);
        }
        close(p[0]); // 关闭不必要的管道
        printf("%d: received ping\n", pid);
        if (write(p[1], buf, 1) != 1) // 判断写入是否成功
        {
            fprintf(2, "failed to write in child\n");
            exit(1);
        }
        close(p[1]);
        exit(0);
    }
    else
    {
        pid = getpid();
        char info[2] = "a";
        char buf[2];
        buf[1] = 0; // 用于表示数组的结束标志
        if (write(p[1], info, 1) != 1)
        {
            fprintf(2, "failed to write in parent\n");
            exit(1);
        }
        close(p[1]);
        wait(0); // 等待子进程完成发送后再进行接收
        if (read(p[0], buf, 1) != 1)
        {
            fprintf(2, "failed to read in parent\n");
            exit(1);
        }
        close(p[0]);
        printf("%d: received pong\n", pid);
        exit(0);
    }
}
```

### primes (moderate)/(hard)

- 通过递归调用创建子进程，执行素数筛的操作

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void prime(int p[2])
{
    char nums[36];
    int cur = 0;
    read(p[0], nums, 36); // 通过管道进行读取nums数组

    for (int i = 0; i < 36; i++)
    {
        if (nums[i] == '1') // 找到第一个1
        {
            cur = i;
            printf("prime %d \n", i);    // 输出当前轮的结果
            for (int j = i; j < 36; j++) // 剔除自身及倍数
            {
                if (j % cur == 0)
                {
                    nums[j] = '0';
                }
            }
            break;
        }
    }

    if (cur == 0)
    {
        exit(0);
    }

    int pid = fork();
    if (pid == 0) // 子进程通过创建孙进程进行递归
    {
        prime(p);
    }
    if (pid > 0) // 子进程负责将nums数组发送给孙进程
    {
        write(p[1], nums, 36);
    }
    exit(0);
}

int main(int argc, char const *argv[])
{
    int p[2];
    pipe(p);
    char nums[36]; // 数组中的0表示该索引代表的数字不可能是一个素数，1则相反
    nums[0] = '0';
    nums[1] = '0';
    for (int i = 2; i < 36; i++) // 为每个数字添加标记
    {
        nums[i] = '1';
    }

    int pid = fork();

    if (pid == 0) // 子进程进行递归筛选
    {
        prime(p);
        wait(0);
    }
    else if (pid > 0) // 父进程通过管道向子进程发送nums数组
    {
        write(p[1], nums, 36);
        wait(0);
    }
    exit(0);
}
```

### find(moderate)

- 通过系统已经实现的ls的代码修改得到用于查找的函数，利用递归来实现对多级文件夹的查找

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char *fmtname(char *path) // 用于将文件路径转换成文件名，如：./a->a
{
    static char buf[DIRSIZ + 1];
    char *p;

    // Find first character after last slash.
    for (p = path + strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;

    // Return blank-padded name.
    if (strlen(p) >= DIRSIZ)
        return p;
    memmove(buf, p, strlen(p));
    memset(buf + strlen(p), 0, DIRSIZ - strlen(p)); // 将第二个参数由ls中的空格改为0
    return buf;
}

int recurse(char *path) // 用于判断当前文件夹是否需要进行递归
{
    if (path[0] == '.' && path[1] == 0)
    {
        return 0;
    }
    else if (path[0] == '.' && path[1] == '.' && path[2] == 0)
    {
        return 0;
    }
    else
    {
        return 1;
    }
    exit(0);
}

void find(char *path, char *target) // 用于查找当前路径下是否有符合要求的文件
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, 0)) < 0) // 不明函数1
    {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0) // 不明函数2
    {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch (st.type) // 用于判断当前访问的是一个文件还是一个文件夹
    {
    case T_FILE:                              // 文件
        fprintf(2, "Usage: find dir file\n"); // 提示用户正确的用法
        break;

    case T_DIR: // 文件夹
        if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf)
        {
            printf("find: path too long\n");
            break;
        }
        strcpy(buf, path);
        p = buf + strlen(buf);
        *p++ = '/';
        while (read(fd, &de, sizeof(de)) == sizeof(de)) // 逐个遍历目录下的文件或文件夹
        {
            if (de.inum == 0)
                continue;
            memmove(p, de.name, DIRSIZ);
            p[DIRSIZ] = 0;
            if (stat(buf, &st) < 0)
            {
                printf("find: cannot stat %s\n", buf);
                continue;
            }
            if (strcmp(target, fmtname(buf)) == 0) // 判断，符合则进行输出
            {
                printf("%s\n", buf);
            }
            if (recurse(fmtname(buf)) == 1) // 判断是否需要进行递归
            {
                find(buf, target); // 递归
            }
        }
        break;
    }
    close(fd);
}

int main(int argc, char *argv[])
{
    if (argc < 2)
    {
        printf("usage: find [path] [target]");
        exit(1);
    }

    if (argc == 2) // 若只有一个参数则默认在当前目录下进行查询
    {
        find(".", argv[1]);
        exit(0);
    }

    find(argv[1], argv[2]);
    exit(0);
}
```

### xargs(moderate)

- 通过文件描述符0读取到前一个函数的标准输出，再对输出的`\n`符号进行判断并调用子进程进行分割，最后将处理好的参数作为额外参数交由下一个函数进行处理

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"
#include "kernel/fs.h"

#define MSGSIZE 16

int main(int argc, char *argv[])
{
    sleep(10); // 确保前一个函数已经执行完毕并完成输出

    char buf[MSGSIZE];
    read(0, buf, MSGSIZE);

    char *xargv[MAXARG];
    int xargc = 0;
    for (int i = 1; i < argc; i++)
    {
        xargv[xargc] = argv[i]; // xargc是从0开始的，i是从1开始的
        xargc++;
    }
    char *p = buf;
    for (int i = 0; i < MSGSIZE; i++)
    {
        if (buf[i] == '\n')
        {
            int pid = fork();
            if (pid > 0) // 父进程进行等待即可
            {
                p = &buf[i + 1]; // 更新p指针，经过测试子进程执行结束之后p的值才发生变化
                wait(0);
            }
            else // 子进程，补充：可以看作每个子进程共用xargv的前面部分，else中的部分是每个子进程自己设置的
            {
                buf[i] = 0;       // 将原来的\n变成0，即代表buf数组的（当前）结尾
                xargv[xargc] = p; // 将已经截取的数组的（当前）头传入参数数组中
                xargc++;
                xargv[xargc] = 0;      // 对xargv数组设置结尾标志
                exec(xargv[0], xargv); // 执行调用
                exit(0);
            }
        }
    }
    wait(0);
    exit(0);
}
```

#### 注意

1. find函数的输出中不能包含空格，否则在测试时grep函数无法正常的打开文件，导致报错
2. 使用sleep函数来确保上一个函数已经完成了执行并进行了输出，否则下一个函数无法接受到输出，导致其输入为空