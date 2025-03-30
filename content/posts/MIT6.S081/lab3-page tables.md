---
title: "Lab3: page tables"
date: 2024-09-21
categories: ["MIT6.S081"]
tags: ["OS"]
aliases: ["/lab3-pagetables"]
ShowToc: true
TocOpen: false
---

# Lab3: page tables

## 1 Speed up system calls

### 题目

创建进程时，在虚拟地址`USYSCALL`映射一个只读页面。在此页面的开头，存储一个结构体`usyscall`（也在`memlayout.h`中定义），并以当前进程PID初始化该结构体。

### 思路

1. 在`proc.h`中增加`struct usyscall *usyscall`字段，用于存储共享页指针
2. `allocproc()`（进程创建时向内存申请空闲页）：申请新页，并将该页的指针地址存入`p->usyscall`字段中
3. `proc_pagetable()`（初始化进程虚拟内存空间，即申请用户内存空间的一级页表并实现已申请的特殊页的映射）：map USYSCALL页的物理地址和`USYSCALL`虚拟地址，形成的页表项添加到当前函数的页表中，设置该页权限为只读+**允许用户访问**，<span style="color:#CC0000;">map失败要unmap之前成功map的页</span>
4. `proc_freepagetable()`（释放用户内存空间的二级页表）：<span style="color:#CC0000;">unmap usyscall页</span>
5. `freeproc()`（进程销毁时清空该进程所占资源）：使用`kfree()`清除`p->usyscall`指向的物理页，对应的p->usyscall设置为0

> 报错：【panic：freewalk：leaf】
>
> 定位：只有uvmfree调用了freewalk；多处调用uvmfree之前使用uvunmap unmap页表项
>
> 释放页表之前，要先unmap页表的对应关系，即清除pte中的PTE_V为0：
>
> - 三、二级页表：(pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0
> - 一级页表：pte & PTE_V == 1
>
> freewalk在释放一级页表时，如果一级页表项是有效的（pte & PTE_V == 1），则发生上述错误
>
> 所以调用freewalk之前，需要unmap一级页表页表项

### 源码

**kernel/proc.h**

```C
// struct proc
struct usyscall *usyscall;
```

**kernel/proc.c**

```C
void allocproc()
{
  // Allocate a USYSCALL page for kernel to share with
  if((p->usyscallpage = (pagetable_t)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  p->usyscall->pid = p->pid;
}

void proc_pagetable()
{
    // map the usyscall just below USYSCALL
    if(mappages(pagetable, USYSCALL, PGSIZE,
              (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
        uvmunmap(pagetable, USYSCALL, 1, 0);
        uvmfree(pagetable, 0);
        return 0;
    }
}

void freeproc()
{
  if(p->usyscall)
    kfree((void*)p->usyscall);
  p->usyscall = 0; 
}


// proc_freepagetable
  uvmunmap(pagetable, USYSCALL, 1, 0);
```



## 2 Print a page table​​

### 题目

定义`vmprint(pagetable_t)`函数，打印`pagetable_t`页表及子页表中所有合法页表项的内容

### 思路

定义两个函数：`vmprint`、`vmsprint`

`vmsprint(pagetable_t)`

1. 遍历`pagetable_t`页表所有页表项
2. 若当前页表项`pte_t`是合法的，打印相关信息
3. 版本一：若当前页表项合法并且不存在RWX在权限，说明该页表项所指向的页表不是零级页表，递归调用`vmsprint`函数，打印当前页表项所指向的页表
4. 版本二：无论当前页表项是不是指向零级页表，都进行递归，但是`vmsprint`函数需要增加一个`level`c参数，表示当前页表是第几级页表。若是第三级，表示当前页表是零级页表，退出，不需要进行递归。

### 源码

**kernel/vm.c**

```c++
void
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmsprint(pagetable, 1);
}

void 
vmsprint(pagetable_t pagetable, int depth)
{
  if(depth == 4)
    return;
        
  pte_t pte;
  uint64 child;

  for(int i = 0; i < 512; ++i) {
    pte = pagetable[i];
    if(pte & PTE_V) {
      printf("..");
      for(int k = 1; k < depth; ++k) {
        printf(" ..");
      }
      child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      vmsprint((pagetable_t)child, depth+1);
    }
  }
}
```

C语言中，使用`%p`占位符打印指针变量的值，也就是虚拟地址。

```C
int *p;
printf("the address is %p\n", p);
```



## 3 Detecting which pages have been accessed​

### 题目

实现`pgaccess`系统调用，汇报哪些page在上次该系统调用之前被修改或访问过。包含以下三个参数：

- va：要检查的第一个user page的虚拟地址
- checknum：待检查页数
- dstva：位于用户空间，用于存储结果的缓冲区的虚拟地址（最低位对应第一页）

### 思路

获取`pgaccess()`系统调用的参数

对于每一待检查的页：

1. 计算该页虚拟首地址va
2. 基于walk函数，获取虚拟地址va在一级页表中的页表项<span style="color:#CC0000;">（第29行代码错误，未获取最后一次PTE）</span>
3. 依据页表项的具体数值，对`PTE_A`进行检查。若`PTE_A`已被置位，就把该页的标识符写入掩码（mask）里。同时，为避免对后续检查产生干扰，需清除`PTE_A`标记
4. 使用`copyout()`将结果写回用户空间

### 源码

```C++
uint64
sys_pgaccess(void) {
	uint64 va, dstva, mask = 0L;
	int checknum;
    
	if(argaddr(0, &va) < 0)
	  return -1;
	if(va >= MAXVA) {
		panic("pgaccess virtual address no more than MAXVA");
		return -1;
	}
	if(argint(1, &checknum) < 0) 
	  return -1;
	if(checknum > 64)
	  checknum = 64;
	if(argaddr(2, &dstva) < 0)
	  return -1;
    
	struct proc *p = myproc();
    
	for (int i = 0; i < checknum; ++i) {
		pte_t *pte;
		pagetable_t pagetable = p->pagetable;
        
		for (int level = 2; level > 0; --level) {
			pte = &pagetable[PX(level, va)];
			pagetable = (pagetable_t)PTE2PA(*pte);
		}
		pte = &pagetable[PX(0, va)];
        
		if(*pte & PTE_A) {
			mask |= 1 << i;
			*pte &= (~PTE_A);
		}
		va += PGSIZE;
	}
    
	if(copyout(p->pagetable, dstva, (char *)&mask, (uint64)(sizeof(mask))) < 0)
	  return -1;
    
	return 0;
}
```

