## 链接
---

[Lab: Traps (mit.edu)](https://pdos.csail.mit.edu/6.S081/2020/labs/traps.html)

## 细节
---

1. 在用户态下不要尝试重新存储trapframe中用来保存内核有关数据的寄存器，否则可能会引发错误
2. 从寄存器中获取地址时需要将其从uint64类型转成uint64 \*类型，这样做才可以对对应地址的目标进行操作

## 实现
---

### RISC-V assembly (easy)

- 热身

- 通过观察汇编代码回答一些小问题

### Backtrace (moderate)

- 打印函数嵌套调用中各级函数的栈帧指针（地址）

- 需要对函数调用栈的中栈帧结构有一定的了解[[Lecture5#Stack]]

- 在defs.h中加入定义

- 在sysproc.c/sys_sleep中加上backtrace( )来进行测试

- kernel/printf.c部分

```c
void backtrace()
{
	printf("backtrace:\n");
	uint64 frame_pointer = r_fp();			  // 获取当前函数的指针的地址
	uint64 *frame = (uint64 *)frame_pointer;  // 将地址转换成指针的形式，即当前函数的栈帧的指针
	uint64 up = PGROUNDUP(frame_pointer);	  // 获取栈页表的上边界
	uint64 down = PGROUNDDOWN(frame_pointer); // 获取栈页表的下边界

	while (frame_pointer > down && frame_pointer < up)
	{
		printf("%p\n", frame[-1]);		 // 该位置保存了当前函数的返回地址
		frame_pointer = frame[-2];		 // 该位置保存了当前函数的上一个函数的栈帧指针（一串地址）
		frame = (uint64 *)frame_pointer; // 将地址转换成指针的形式
	}
}
```

- 当backtrace能够正常运行之后，将其加入到kernel/printf.c的panic中

### Alarm (hard)

- 实现一个函数使得进程在执行时每经过设定的某个时间之后就会执行相应的函数

- 这个中断的机制相当于trap机制

- 进行相关的设置（将alarmtest写入Makefile中，在对应的文件中增加sigalarm和sigreturn的系统调用）

-   在`struct proc`(proc.h)中新增新的成员属性

```c
int ticks;  
int ticks_cnt;  
uint64 handler;  
int handlerRuning;  
  
uint64 saved_epc; // saved user program counter  
uint64 saved_ra;  
uint64 saved_sp;  
uint64 saved_gp;  
uint64 saved_tp;  
uint64 saved_t0;  
uint64 saved_t1;  
uint64 saved_t2;  
uint64 saved_t3;  
uint64 saved_t4;  
uint64 saved_t5;  
uint64 saved_t6;  
uint64 saved_a0;  
uint64 saved_a1;  
uint64 saved_a2;  
uint64 saved_a3;  
uint64 saved_a4;  
uint64 saved_a5;  
uint64 saved_a6;  
uint64 saved_a7;  
uint64 saved_s0;  
uint64 saved_s1;  
uint64 saved_s2;  
uint64 saved_s3;  
uint64 saved_s4;  
uint64 saved_s5;  
uint64 saved_s6;  
uint64 saved_s7;  
uint64 saved_s8;  
uint64 saved_s9;  
uint64 saved_s10;  
uint64 saved_s11;
```

- 在proc.c的`allocproc()`中初始化`proc`字段

-   每经过设定好的那么多个ticks，硬件时钟就会强制一个中断，这个中断在kernel/trap.c中的`usertrap()`中处理

- sysproc.c部分

```c
uint64
sys_sigalarm(void)
{
	int ticks;
	uint64 handler;
	// 从入参中获取对应的值
	argint(0, &ticks);
	argaddr(1, &handler);
	struct proc *p = myproc();
	// 初始化变量
	p->ticks = ticks;
	p->handler = handler;
	p->ticks_cnt = 0;
	return 0;
}

uint64
sys_sigreturn(void)
{
	struct proc *p = myproc();
	// 将之前保存的各个寄存器进行恢复
	p->trapframe->epc = p->saved_epc;
	p->trapframe->ra = p->saved_ra;
	p->trapframe->sp = p->saved_sp;
	p->trapframe->gp = p->saved_gp;
	p->trapframe->tp = p->saved_tp;
	p->trapframe->a0 = p->saved_a0;
	p->trapframe->a1 = p->saved_a1;
	p->trapframe->a2 = p->saved_a2;
	p->trapframe->a3 = p->saved_a3;
	p->trapframe->a4 = p->saved_a4;
	p->trapframe->a5 = p->saved_a5;
	p->trapframe->a6 = p->saved_a6;
	p->trapframe->a7 = p->saved_a7;
	p->trapframe->t0 = p->saved_t0;
	p->trapframe->t1 = p->saved_t1;
	p->trapframe->t2 = p->saved_t2;
	p->trapframe->t3 = p->saved_t3;
	p->trapframe->t4 = p->saved_t4;
	p->trapframe->t5 = p->saved_t5;
	p->trapframe->t6 = p->saved_t6;
	p->trapframe->s0 = p->saved_s0;
	p->trapframe->s1 = p->saved_s1;
	p->trapframe->s2 = p->saved_s2;
	p->trapframe->s3 = p->saved_s3;
	p->trapframe->s4 = p->saved_s4;
	p->trapframe->s5 = p->saved_s5;
	p->trapframe->s6 = p->saved_s6;
	p->trapframe->s7 = p->saved_s7;
	p->trapframe->s8 = p->saved_s8;
	p->trapframe->s9 = p->saved_s9;
	p->trapframe->s10 = p->saved_s10;
	p->trapframe->s11 = p->saved_s11;
	p->handlerRuning = 0;
	return 0;
}
```

- trap.c/alarmHelper辅助函数部分

```c
void alarmHelper(void)
{
	struct proc *p = myproc();
	// 保存当前的trapframe
	p->saved_epc = p->trapframe->epc;
	p->saved_ra = p->trapframe->ra;
	p->saved_sp = p->trapframe->sp;
	p->saved_gp = p->trapframe->gp;
	p->saved_tp = p->trapframe->tp;
	p->saved_t0 = p->trapframe->t0;
	p->saved_t1 = p->trapframe->t1;
	p->saved_t2 = p->trapframe->t2;
	p->saved_t3 = p->trapframe->t3;
	p->saved_t4 = p->trapframe->t4;
	p->saved_t5 = p->trapframe->t5;
	p->saved_t6 = p->trapframe->t6;
	p->saved_a0 = p->trapframe->a0;
	p->saved_a1 = p->trapframe->a1;
	p->saved_a2 = p->trapframe->a2;
	p->saved_a3 = p->trapframe->a3;
	p->saved_a4 = p->trapframe->a4;
	p->saved_a5 = p->trapframe->a5;
	p->saved_a6 = p->trapframe->a6;
	p->saved_a7 = p->trapframe->a7;
	p->saved_s0 = p->trapframe->s0;
	p->saved_s1 = p->trapframe->s1;
	p->saved_s2 = p->trapframe->s2;
	p->saved_s3 = p->trapframe->s3;
	p->saved_s4 = p->trapframe->s4;
	p->saved_s5 = p->trapframe->s5;
	p->saved_s6 = p->trapframe->s6;
	p->saved_s7 = p->trapframe->s7;
	p->saved_s8 = p->trapframe->s8;
	p->saved_s9 = p->trapframe->s9;
	p->saved_s10 = p->trapframe->s10;
	p->saved_s11 = p->trapframe->s11;
	// 将当前的sepc寄存器设置为目标函数的地址，当程序继续执行时下一步将会跳转到对应的函数
	p->trapframe->epc = p->handler;
}
```

- trap.c/usertrap部分

```c
// give up the CPU if this is a timer interrupt.
if (which_dev == 2)
{
	if (p->ticks > 0)
	{
		p->ticks_cnt++;
		if (p->handlerRuning == 0 && p->ticks_cnt > p->ticks)
		{
			p->handlerRuning = 1; // 设置对应的属性以确保在执行目标函数时不会继续执行alarm的操作
			p->ticks_cnt = 0;	  // 重置计数
			alarmHelper();
		}
	}
	yield();
}
```