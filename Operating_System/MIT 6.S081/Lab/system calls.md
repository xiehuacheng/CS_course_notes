## 链接
---

[Lab: System calls (mit.edu)](https://pdos.csail.mit.edu/6.S081/2020/labs/syscall.html)

## 细节
---

1. 做实验前记得先切换git分支到syscall上

## 实现
---

### System call tracing(moderate)

- 按照实验要求修改对应文件

>Some hints:

>Add $U/_trace to UPROGS in Makefile

>Run make qemu and you will see that the compiler cannot compile user/trace.c, because the user-space stubs for the system call don't exist yet: add a prototype for the system call to user/user.h, a stub to user/usys.pl, and a syscall number to kernel/syscall.h. The Makefile invokes the perl script user/usys.pl, which produces user/usys.S, the actual system call stubs, which use the RISC-V ecall instruction to transition to the kernel. Once you fix the compilation issues, run trace 32 grep hello README; it will fail because you haven't implemented the system call in the kernel yet.

- 此外，还需要在syscall.c文件中加上如下部分

开头

```c
extern uint64 sys_trace(void);
```

`static uint64 (*syscalls[])(void)`中

```c
[SYS_trace]   sys_trace,
```

否则代码可能会这样报错

```text
3 trace: unknown sys call 22
trace:trace failed
```

- 在proc.h中为proc结构体增加一个新的参数mask来记录掩码

- syspro.c部分

```c
uint64
sys_trace(void)
{
  int mask;
  if (argint(0, &mask) < 0) // 从输入参数中获取mask，此处参照exit函数的代码
    return -1;
  myproc()->mask = mask; // 将获取到的mask传到进程结构体中进行储存
  return 0;
}
```

- syscall.c部分

```c
void syscall(void)
{
  char const *syscall_names[] = {"fork", "exit", "wait", "pipe", "read",
                                 "kill", "exec", "fstat", "chdir", "dup", "getpid", "sbrk", "sleep",
                                 "uptime", "open", "write", "mknod", "unlink", "link", "mkdir", "close", "trace", "sysinfo"};
  // 构建函数名数组，以便根据mask获取调用的函数的名称

  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if (num > 0 && num < NELEM(syscalls) && syscalls[num])
  {
    p->trapframe->a0 = syscalls[num](); // 函数的输出结果被存入了a0寄存器中
    if ((p->mask) & (1 << num))         // 将mask和1左移num位的结果进行比对，比对成功则当前的num就为函数对应的系统调用编号
    {
      printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num - 1], p->trapframe->a0); // 打印信息
    }
  }
  else
  {
    printf("%d %s: unknown sys call %d\n",
           p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

- 此外，还需要再proc.c中修改进程的释放部分（freeproc函数）的代码，以确保进程结构体中的掩码被正确重置

```c
p->mask=0;
```


### Sysinfo(moderate)

- 先按上面的hints来修改文件

- 参照对应的函数来进行编写

>Some hints:

>sysinfo needs to copy a struct sysinfo back to user space; see sys_fstat() (kernel/sysfile.c) and filestat() (kernel/file.c) for examples of how to do that using copyout().

>To collect the amount of free memory, add a function to kernel/kalloc.c

>To collect the number of processes, add a function to kernel/proc.c

- sysproc.c部分

```c
uint64
sys_sysinfo(void)
{
  struct sysinfo info;
  uint64 addr;
  struct proc *p = myproc();

  info.freemem = acquire_freemem();
  info.nproc = acquire_nproc();

  if (argaddr(0, &addr) < 0)
    return -1;
  if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
    return -1;
  return 0;
}
```

需要在前面声明对应的acquire函数

- proc.c部分（统计正在运行中的程序）

```c
uint64 acquire_nproc()
{
  struct proc *p;
  uint64 cnt = 0;

  for (p = proc; p < &proc[NPROC]; p++)
  {
    acquire(&p->lock); // 在统计时锁住当前统计的进程，避免其他程序访问？（弹幕说的）
    if (p->state != UNUSED)
    {
      cnt++;
    }
    release(&p->lock); // 解锁
  }
  return cnt;
}
```

- kalloc.c部分（统计空闲内存）

```c
uint64 acquire_freemem()
{
  struct run *r;
  uint64 cnt = 0;

  acquire(&kmem.lock);
  r = kmem.freelist;
  while (r)
  {
    r = r->next;
    cnt++;
  }
  release(&kmem.lock);
  return cnt * PGSIZE; // 乘上一个page的大小（4096bytes）
}
```