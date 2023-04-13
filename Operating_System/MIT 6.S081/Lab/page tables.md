## 链接
---

https://pdos.csail.mit.edu/6.S081/2020/labs/pgtbl.html

## 细节
---

1. 修改和添加函数需要将声明补充或更新到defs.h文件中
2. 位运算中的 | （或）相当于无进位相加
3. 位运算中的&（与）相当于获取有效位

## 实现
---

### Print a page table(easy)

- 运用递归打印xv6系统的三级页表以及对应的物理地址

- vm.c部分

```c
void vmprint(pagetable_t pagetable)
{
  vmprintwalk(pagetable, 0);
}

// 辅助函数，用于进行遍历操作
void vmprintwalk(pagetable_t pagetable, int depth)
{

  if (depth == 0)
  {
    printf("page table %p\n", pagetable);
  }

  // 遍历每个PTE
  for (int i = 0; i < 512; i++)
  {
    pte_t pte = pagetable[i];

    // 如果PTE无效，则跳过
    if ((pte & PTE_V) == 0)
    {
      continue;
    }

    // 打印缩进
    for (int j = 0; j <= depth; j++)
    {
      printf("..");
      if (j != depth)
      {
        printf(" ");
      }
    }

    // 打印PTE索引和PTE值
    printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));

    // 如果PTE指向的是页表，则递归打印页表
    if ((pte & PTE_R) == 0 && (pte & PTE_W) == 0 && (pte & PTE_X) == 0)
    {
      vmprintwalk((pagetable_t)PTE2PA(pte), depth + 1);
    }
  }
}
```

- ↓proc.c部分↓

- 修改scheduler的内容，使得切换进程时将切换到对应进程的内核页表

```c
w_satp(MAKE_SATP(p->kpagetable));
sfence_vma();
```

- 在进程移出CPU时切换回原来的全局内核页表

```c
kvminithart();
```

- 定义(vm.c部分)

```c
void kvminithart()
{
  w_satp(MAKE_SATP(kernel_pagetable));
  sfence_vma();
}
```

- 在allocproc函数中的进程创建部分对内核页表进行创建

```c
p->kpagetable = kvm_map_pagetable();
```

- 在freeproc函数中对进程的内核页表进行释放

```c
kvmfree(p->kpagetable);
```

### A kernel page table per process(hard)

- 为每个进程维护一个单独的内核页表

- 需要在proc.h的proc定义中增加一个成员属性

```c
pagetable_t kpagetable;
```

- vm.c部分

```c
// 该函数用于将对应的页表的地址映射到对应物理地址上
void kvmmapkern(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if (mappages(pagetable, va, sz, pa, perm) != 0)
    panic("kvmmap");
}

// 该函数用于生成一个用户进程的内核页表（由kvminit修改而来）
pagetable_t kvm_map_pagetable()
{
  // 初始化一个页表，并将所有项置为0
  pagetable_t pagetable = uvmcreate();
  // 拷贝内核页表中的内容
  for (int i = 1; i < 512; i++)
  {
    pagetable[i] = kernel_pagetable[i];
  }

  // 映射用户进程需要的地址

  // uart registers
  kvmmapkern(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmapkern(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmapkern(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmapkern(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // kvminit后面的三行映射只与内核有关，故不在用户页表中进行映射

  return pagetable;
}

// 该函数用于递归的释放页表项
void kvmfree(pagetable_t kpagetale)
{
  pte_t pte = kpagetale[0];
  pagetable_t level1 = (pagetable_t)PTE2PA(pte);
  for (int i = 0; i < 512; i++)
  {
    pte_t pte = level1[i];
    if (pte & PTE_V)
    {
      uint64 level2 = PTE2PA(pte);
      kfree((void *)level2);
      level1[i] = 0;
    }
  }
  kfree((void *)level1);
  kfree((void *)kpagetale);
}
```

### Simplify copyin/copyinstr(hard)

- 将用户进程中的用户页表中的映射复制到它的内核页表中，以便内核可以直接将用户进程的指针进行解引用操作以获得对应的物理地址（通过copyin以及copyinstr函数来进行）

- vm.c部分

```c
// 该函数用于将用户进程中的用户页表中的映射写入到内核页表中
void kvmmapuser(int pid, pagetable_t kpagetable, pagetable_t upagetable, uint64 newsz, uint64 oldsz)
{
  uint64 va;
  pte_t *upte;
  pte_t *kpte;

  if (newsz >= PLIC) // 判断内存是否越界
    panic("kvmmapuser: newsz too large");

  for (va = oldsz; va < newsz; va += PGSIZE)
  {
    upte = walk(upagetable, va, 0); // 不需要分配新的页表项
    kpte = walk(kpagetable, va, 1); // 需要分配新的页表项
    *kpte = *upte;                  // 拷贝
    // because the user mapping in kernel page table is only used for copyin
    // so the kernel don't need to have the W,X,U bit turned on
    *kpte &= ~(PTE_U | PTE_W | PTE_X); // 删除对应的标志位，原因如上
  }
}
```

- 权限部分可以参考hints
	-   What permissions do the PTEs for user addresses need in a process's kernel page table? (A page with PTE_U set cannot be accessed in kernel mode.)

- 修改fork( ),exec( ),sbrk( ),userinit( )的实现，增加以下语句

```c
kvmmapuser(p->pid, p->kpagetable, p->pagetable, p->sz, 0);
```