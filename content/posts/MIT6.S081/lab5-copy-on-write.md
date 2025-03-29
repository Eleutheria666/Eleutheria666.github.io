---
title: "Lab4: Trap"
date: 2024-08-14
categories: ["MIT6.S081"]
tags: ["OS"]
aliases: ["/lab4-trap"]
ShowToc: true
TocOpen: false
---

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

