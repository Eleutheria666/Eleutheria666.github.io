---
title: "Lab2: System call"
date: 2024-09-01
categories: ["MIT6.S081"]
tags: ["OS"]
aliases: ["/lab2-syscall"]
ShowToc: true
TocOpen: false 
---

# Lab2： System call

实验原址：[Lab: System calls](https://pdos.csail.mit.edu/6.S081/2021/labs/syscall.html)

源码地址：

## 1 System call tracing

> 在操作系统开发与调试过程中，对程序进行系统调用进行跟踪有助于深入了解程序运行时的系统行为

### 题目

创建一个新的系统调用 `trace` ，该系统调用接收一个整数参数 `mask` 。通过对 `mask` 二进制位的设置，指定需要跟踪的系统调用。例如，若要跟踪 `fork` 系统调用，程序需调用 `trace(1 << SYS_fork)` ，其中 `SYS_fork` 是在 `kernel/syscall.h` 中定义系统调用编号。当每个系统调用即将返回时，若该系统调用的编号在 `mask` 中被设置，则打印一行信息，包含进程 ID、系统调用名称以及返回值，无需打印系统调用的参数。

`trace` 系统调用应仅对调用它的进程及其后续通过 `fork` 创建的子进程启用跟踪功能，而不影响其他进程。

（一）部分系统调用跟踪

执行 `trace 32 grep hello README` ，其中 `32` 对应 `1<<SYS_read` ，表示仅跟踪 `grep` 执行过程中 `read` 系统调用，输出：

```sh
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
```

（二）全系统调用跟踪

执行 `trace 2147483647 grep hello README` ，`2147483647` 的低 31 位全为 1，即跟踪 `grep` 执行过程中所有系统调用，输出：

```plaintext
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
4: syscall close -> 0
```

（三）无跟踪情况：执行 `grep hello README` ，此程序未启用跟踪，因此无跟踪输出。

（四）特定程序的系统调用跟踪

执行 `trace 2 usertests forkforkfork` ，将跟踪 `usertests` 中 `forkforkfork` 测试的所有后代进程的 `fork` 系统调用，输出示例：

```plaintext
usertests starting
test forkforkfork: 407: syscall fork -> 408
408: syscall fork -> 409
409: syscall fork -> 410
410: syscall fork -> 411
409: syscall fork -> 412
410: syscall fork -> 413
409: syscall fork -> 414
411: syscall fork -> 415
...
```

### 思路

<span style="color:#CC0000;">系统调用实现：提供系统调用入口 + 实现系统调用函数</span>

提供系统调用入口

- 在`user/user.h`文件中声明系统调用函数原型
- make 编译时会将`user/usys.pl`文件中的内容转化为`user/usys.S`，这个汇编文件指明了用户调用系统函数所执行的指令：
  - 依次向寄存器a0、a1存储系统调用参数
  - ecall指令，进行系统调用

<img src="./../../../../../knowledge-note/assets/images/image-20250328232938435.png" alt="image-20250328232938435" style="zoom:50%;" />

- 在内核 `kernel/syscall.h`声明系统调用编号， `kernel/syscall.c`声明系统调用函数

实现trace系统调用：

- 在进程结构体中添加mask参数字段
- trace系统调用实现函数，从寄存器中获取系统调用参数mask的值放入该结构体字段中

### 源码

**kernel/proc.h**：设置进程trace系统调用的mask参数字段

```C++
struct proc:
  int tracemask;   // System Trace Call mask
```

**kernel/sysproc.c**：将trace系统调用参数mask放入进程结构体的tracemask字段中

```C++
uint64
sys_trace(void)
{
  int mask;
  if(argint(0, &mask) < 0) {
    return -1;
  }
    
  struct proc *p = myproc();
  p->tracemask = mask;
  return 0;
}
```

**kernel/syscall.c**：系统调用时，判断当前进程traceemask字段是否把包含该系统调用，如果包含打印调用信息

```C++
static char* syscalls_name[] = {
  "",
  "fork",
  "exit",
  "wait",
   .....
  "trace",
};

int syscall(void) 
{
    if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();
        if(((p->tracemask >> num) & 1) == 1) 
            printf("%d: syscall %s -> %d\n", p->pid, syscalls_name[num], p->trapframe->a0);
    } else {
        printf("%d %s: unknown sys call %d\n", p->pid, p->name, num);
        p->trapframe->a0 = -1;
    }
}
```

**kernel/proc.c**：子进程复制父进程mask参数

```C++
void fork():
  // copy trace mask
  np->tracemask = p->tracemask;
```



## 2 Sysinfo

### 题目

添加一个新的系统调用 `sysinfo`，旨在为用户态程序提供便捷获取系统运行信息的途径，有助于系统监控、调试以及性能优化等操作

### 思路

该系统系统调用需接收一个参数，该参数是指向 `struct sysinfo` 结构体的指针。此结构体在 `kernel/sysinfo.h` 中定义，用于存储系统信息。

### （一）系统调用设计

1. **参数设定**：新增的 `sysinfo` 
2. **信息填充**：内核在处理 `sysinfo` 系统调用时，需对 `struct sysinfo` 结构体的特定字段进行填充。其中，`freemem` 字段要设置为系统当前空闲内存的字节数，这反映了系统当前可用于分配的内存资源量；`nproc` 字段需设置为状态不为 `UNUSED` 的进程数量，以此体现系统中正在运行或处于就绪等活跃状态的进程总数。

实验提供了测试程序 `sysinfotest`，用于验证 `sysinfo` 系统调用的正确性。若 `sysinfotest` 程序最终输出 “sysinfotest: OK”，则表明实验成功完成。

### 思路

在`proc.c`、`kalloc.c`文件中分别实现返回当前系统非空闲进程数和空闲内存大小的两个函数proccnt、kfreecnt

sysinfo系统调用实现：

- 从寄存器a0获取写入结果用户空间地址指针
- 创建sysinfo结构体，调用proccnt、kfreecnt获取当前系统运行参数并写入该结构体
- 调用copyout函数将内核空间的系统数据写回

### 源码

**kernel/proc.c**：返回当前系统中，状态不为UNUSED的进程的数目

```C++
uint64
proc_cnt(void) 
{
  uint64 cnt = 0;
  struct proc *p;
  
  for(p = proc; p < &proc[NPROC]; p++) {
    if(p->state != UNUSED) 
      ++cnt;
  }
  return cnt;
}
```

**kernel/kalloc.h**：返回当前系统中，空闲内存大小（空闲页数 * 页大小）

```C++
uint64
kcnt(void) 
{
  uint64 cnt = 0;

  struct run *r;
  r = kmem.freelist;

  while(r != 0) {
    ++cnt;
    r = r->next;
  }
  
  return cnt * PGSIZE;
}
```

**kernel/sysinfo.c**：实现sysinfo系统调用

```C++
uint64 
sys_sysinfo(void)
{   
  uint64 addr;
  argaddr(0, &addr);

  struct sysinfo info;
  struct proc *p = myproc();

  info.freemem = kcnt();
  info.nproc = proc_cnt();

  if(copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
    return -1;
  return 0;
}
```



