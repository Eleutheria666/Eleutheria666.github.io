---
title: "Lab4: traps"
date: 2024-09-14
categories: ["MIT6.S081"]
tags: ["OS"]
ShowToc: true
TocOpen: true
---

Lab链接：[Lab: Traps](https://pdos.csail.mit.edu/6.S081/2021/labs/traps.html)

Lab源码：[momo/MIT-6S081/traps - Gitee.com](https://gitee.com/Eleutheria666/mit-6s081/tree/traps/)

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



## 1 RISC-V assembly

使用`make fs.img`命令生成`call.c`文件的汇编文件`call.asm`

```assembly
int g(int x) {
   0:	1141                	addi	sp,sp,-16
   2:	e422                	sd		s0,8(sp)
   4:	0800                	addi	s0,sp,16
  return x+3;
}
   6:	250d                	addiw	a0,a0,3
   8:	6422                	ld		s0,8(sp)
   a:	0141                	addi	sp,sp,16
   c:	8082                	ret

000000000000000e <f>:

int f(int x) {
   e:	1141                	addi	sp,sp,-16
  10:	e422                	sd		s0,8(sp)
  12:	0800                	addi	s0,sp,16
  return g(x);
}
  14:	250d                	addiw	a0,a0,3
  16:	6422                	ld		s0,8(sp)
  18:	0141                	addi	sp,sp,16
  1a:	8082                	ret

000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd		ra,8(sp)
  20:	e022                	sd		s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li		a2,13
  26:	45b1                	li		a1,12
  28:	00000517          		auipc	a0,0x0
  2c:	7b850513          		addi	a0,a0,1976 # 7e0 <malloc+0xea>
  30:	00000097          		auipc	ra,0x0
  34:	608080e7          		jalr	1544(ra) # 638 <printf>
  exit(0);
  38:	4501                	li		a0,0
  3a:	00000097          		auipc	ra,0x0
  3e:	276080e7          		jalr	630(ra) # 2b0 <exit>
```

回答以下问题：

- Which registers contain arguments to functions? For example, which register holds 13 in main's call to `printf`?

  RISC - V 架构，函数调用者在调用前，将参数按顺序依次存储在 `a0 - a7` 寄存器中，然后再调用函数。

  第24行，`li a2 13`指令把立即数13存入寄存器a2，a2存入参数13，a1存入`f(8)+1`的结果，a0存入格式字符串**的地址。**

- Where is the call to function `f` in the assembly code for main? Where is the call to `g`?

  第26行， `li a1 12`指令表明main函数并未调用函数f，编译器直接计算`f(8)+1`的结果并存入寄存器a1。

- At what address is the function `printf` located?

  第34行，注释指出prinf函数位于**用户空间虚拟地址**0x638处。

- What value is in the register `ra` just after the `jalr` to `printf` in `main`?

  第30行，`auipc ra, 0x0` 把当前PC值高 20 位加载到 `ra` 寄存器，`jalr 1544(ra)` 跳转到 `printf` 函数，同时把下一条指令（地址为 0x38）的地址保存到 `ra` 寄存器。所以，在执行完 `jalr` 指令跳转到 `printf` 函数后，`ra` 寄存器的值是 0x38。



## 2 Backtrace

### 题目

实现`backtrace()`函数：遍历当前程序的调用栈，按照函数调用的先后顺序，从当前时刻起，逆向打印每个栈帧中的返回地址 

运行`bttest`，该程序调用`sys_sleep`，`sys_sleep()`调用`backtrace()`打印当前堆栈上的函数调用列表。打印的结果如下：

```shell
backtrace:
0x0000000080002cda
0x0000000080002bb6
0x0000000080002898
```

退出qemu，执行`addr2line -e kernel/kernel`命令并输入上述结果，得到地址对应的函数名和所在文件

在终端中输入 `addr2line -e kernel/kernel` 命令，并输入之前获取到的地址信息。该命令解析输入的地址，得到其所处在哪个文件中的哪个位置，为调试进一步提供关键线索。

```shell
$ addr2line -e kernel/kernel
    0x0000000080002de2
    0x0000000080002f4a
    0x0000000080002bfc
    Ctrl-D
# 上述命令执行结果：
kernel/sysproc.c:74
kernel/syscall.c:224
kernel/trap.c:85
```

### 思路

1. 在`kernel/riscv.c`文件中添加用于读取当前栈顶寄存器值的函数`r_fp`

2. 使用`PGROUNDDOWN()`获取栈所在页的顶部地址

3. 下图展现了xv6栈的结构，栈帧从内存较高地址处起始，随着函数调用朝内存较低方向生成栈帧。

   在遍历当前程序内核栈的过程中，针对每一个栈帧，打印该栈帧的函数地址，即 “返回地址（return address）”。同时，借助 “to prev frame（指向前一个栈帧）” 所指示的信息，获取调用该函数的栈帧的地址，进而推进栈的遍历流程 。

![image-20240804214358144](../../../images/image-20240804214358144.png)

### 源码

```C
void            
backtrace(void)
{
  printf("backtrace\n");

  // get current working function's frame pointer
  uint64 *fp = (uint64*)r_fp();

  uint64 limit = PGROUNDUP((uint64)fp);

  while((uint64)fp < limit) {
    printf("%p\n", *(fp-1));
    fp = (uint64*)(*(fp-2));
  }
}
```

> 指针所指向的地址虽然可表示为 64 位整数，但要获取指针所指向的数据内容，`fp` 的类型理应定义为 `uint64*`，而非 `uint64`。这是因为只有 `uint64*` 类型才能正确解引用，从而访问到目标数据。
>
> 当 `fp` 被定义为 `uint64` 类型的指针时，对该指针执行 `-1` 操作，实际上是在内存地址上向前移动了 `8 byte`。这是由于在 64 位系统中，`uint64` 类型数据占据 8 个字节，指针运算会根据其所指向的数据类型大小来调整偏移量 。

运行结果：

<img src="../../../images/image-20250401120211958.png" alt="image-20250401120211958" style="zoom:150%;" />



## 3 Alarm​​

### 题目

### 思路

### 源码
