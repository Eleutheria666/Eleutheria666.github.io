---
title: "Lab4: Trap"
date: 2024-08-14
categories: ["MIT6.S081"]
tags: ["OS"]
aliases: ["/lab4-trap"]
ShowToc: true
TocOpen: false
---

# Lec 5：Trap

用户态切换为内核态发生在以下**情形**：

- 系统调用
- 软件运行错误：page fault
- 设备发出中断

用户内核态的切换，要让**硬件从适合用户程序运行的状态转化为适合内核程序运行的状态**，包括：

1. 保存32个用户寄存器：内核会使用寄存器，用户程序在寄存器中的值不能被破坏
2. 保存程序计数器：内核程序执行完毕后能够从用户程序中断的位置继续执行
3. 改为supervisor mode：在内核中需要执行特权指令
4. SATP寄存器从user page table改为kernel page table
5. 堆栈寄存器指向内核地址

相比用户态，**内核态增加了以下权限**：

1. 读写控制寄存器
2. 使用PTE_U为0的页表：PTE_U设置为1用户态可使用

`trapoline`页表在用户和内核页表虚拟地址一致，映射的都是同一个物理页

`trapframe`用于保存进程在用户态的相关数据，而context用于保存进程在内核态相关数据

## 用户态➡内核态

用户进程调用`fork`系统调用，实际执行如下指令：

```assembly
.global fork
fork:
 li a7, SYS_fork
 ecall
 ret
```

1. 向a7寄存器写入系统调用编号
2. 执行ecall指令，该指令在硬件层面完成：

- 切换为supervisor mode
- 进入中断处理程序：PC➡SEPC，STVEC➡PC（STVEC事先保存了中断处理程序uservec的地址）

此时进入中断处理程序uservec（此时未切换page table，虽进入内核态但仍使用用户态的页表）

1. 保存用户态通用寄存器，用于中断后恢复上下文：`sscratch` 寄存器事先存储了`p->trapframe` ，交换 `a0` 和 `sscratch` 的值，使得 `a0` 指向进程trapframe的首地址
2. 加载内核态上下文：栈指针、内核页表、从 `TRAPFRAME` 中读取 `usertrap()` 函数的地址，并将其加载到 `t0` 寄存器

```assembly
.globl uservec
uservec:    
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #
        
		# swap a0 and sscratch, so that a0 is TRAPFRAME
        csrrw a0, sscratch, a0
        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        # .....
		# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        # restore kernel stack pointer from p->trapframe->kernel_sp
        ld sp, 8(a0)
        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)
        # restore kernel page table from p->trapframe->kernel_satp
        ld t1, 0(a0)
        # load the address of usertrap(), p->trapframe->kernel_trap
        ld t0, 16(a0)
        csrw satp, t1
        sfence.vma zero, zero

        # jump to usertrap(), which does not return
        jr t0
```

此时进入中断处理程序的第二部分

trap.c中处理

```C
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    if(which_dev == 2) {
      // pass if Set time interval task
      if(p->alarminterval != 0) {
        // Time interval tick pass
        if(--p->tickleft <= 0) {
          if(p->doing_alarm == 0) {
            p->tickleft = p->alarminterval;
            *p->alarmtrapframe = *p->trapframe;
            p->trapframe->epc = (uint64)p->alarmhandler;
            p->doing_alarm = 1;
          }
        }
      }
      // give up the CPU if this is a timer interrupt.
      yield();
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  usertrapret();
}
```

## 内核态➡用户态

- STVEC寄存器：保存uservec函数的地址，为ecall指令跳转
- SSCRATCH寄存器：保存trapframe page的虚拟地址
- trapframe的`kernel_sp`字段：存储进程的kernel stack的最顶端，下次存储在SP寄存器中
- trapframe的`kernel_satp`字段：存储内核页表的物理地址，下次存储在SATP寄存器中
- trapframe的`kernel_trap`字段：转入trap处理的内核C函数`usertrap()`的地址，下次直接跳转至该函数

```assembly
.globl userret
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        csrw satp, a1
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        # ...

		# restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```

# Lab

## 2 Backtrace

### 题目

实现`backtrace()`函数：每个堆栈帧中的帧指针用于保存调用者帧指针的地址。你的回溯应该使用这些帧指针来遍历堆栈，并在每个堆栈帧中打印保存的返回地址。

在`sys_sleep`中插入对该函数的调用，然后运行`bttest`，该命令调用`sys_sleep`，打印出当前堆栈上的函数调用列表。打印的结果如下：

```shell
backtrace:
0x0000000080002cda
0x0000000080002bb6
0x0000000080002898
```

退出qemu，使用`addr2line -e kernel/kernel`命令并输入上述结果，得到指定位置函数的函数名和所在文件

```shell
$ addr2line -e kernel/kernel
    0x0000000080002de2
    0x0000000080002f4a
    0x0000000080002bfc
    Ctrl-D
# output
kernel/sysproc.c:74
kernel/syscall.c:224
kernel/trap.c:85
```

### 思路

1. 在`kernel/riscv.c`文件中添加用于读取当前栈顶寄存器值的函数`r_fp`
2. 由于每个用户进程的栈只有一页的空间大小，使用`PGROUNDDOWN()`获取栈所在页的首位值
3. 遍历栈，对于每一个栈帧打印调用生成该栈帧的函数地址`return address`，并通过`to prev frame`获得下一个栈帧的地址（地址往Low处走）

### 源码





## 3 Alarm:x:

### 题目

### 思路

### 源码
