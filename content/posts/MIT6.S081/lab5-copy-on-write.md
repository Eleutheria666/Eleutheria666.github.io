---
title: "Lab5: copy on Write"
date: 2024-09-21
categories: ["MIT6.S081"]
tags: ["OS"]
ShowToc: true
TocOpen: true
---

Lab链接：[Lab: Copy-on-Write Fork for xv6](https://pdos.csail.mit.edu/6.S081/2021/labs/cow.html)

Lab源码：[momo/MIT-6S081/cow - Gitee.com](https://gitee.com/Eleutheria666/mit-6s081/tree/cow/)

## 基于Page faults的附加功能

### 恢复page fault中断

恢复page fault，需要三个要素：

- 引起page fault的虚拟地址（STVAL）
- 引起page fault原因的类型（SCAUSE）
- 发生page fault时程序计数器值，即造成page fault的指令在用户虚拟空间的位置（SEPC）

理想情况下，操作系统应能够在发生page fault中断时，对其进行响应的处理，继续运行后续指令

Lazy page allocation

eager allocation：一旦调用了sbrk，内核会**立即分配应物理内存**。但会存在过多申请的内存过少使用的情况。

lazy allocation：sbrk系统调不做任何事情，唯一需要做的事情就是提升`p->sz`，将`p->sz`增加n，其中n是需要新分配的内存page数量。但是内核在这个时间点并**不会分配任何物理内存**。之后某个时间点程序**使用到了新申请的那部分内存，会触发page fault**，因为我们还没有分配实际的物理内存（即新的内存未映射到page table）。所以，当我们看到了一个page fault，相应的虚拟地址小于当前`p->sz`，同时大于stack，那么我们就知道这是一个来自于heap的地址，但是内核还没有分配任何物理内存。

### Copy on write fork

调用`fork`时，先创建4个新的page，再将父进程page的内容拷贝至4个新的分配给子进程的page中。

之后调用`exec`又会释放这些page，并分配新的page来包含echo相关的内容。

优化：创建子进程时，与其创建分配拷贝内容到新的物理内存，不如**直接共享父进程的物理内存page并设置子进程的PTE指向父进程对应的物理内存page**。但是父进程和子进程之间的隔离性，要保证：**子进程对这些内存的修改应该对父进程不可见**，所以这里的**父进程和子进程的PTE的标志位都设置成只读的**。

父进程和子进程继续运行，当父进程或子进程执行store指令更新一些全局变量时会触发page fault，因为现在在向一个只读PTE写数据。

在得到page fault之后，先分配一个新的page，然后将父进程相应的page拷贝到新page，并将新page映射到子进程的页表中。这时，新分配的page只对子进程可见，相应的PTE设置成可读写，并且重新执行指令。对于触发page fault对应的物理page，因为现在只对父进程可见，相应的PTE对于父进程也变成可读写的了。

释放：当我们释放page时，我们将物理内存page的引用数减1。**只有引用数为0时释放物理内存page**。

### Least Recently Used（LRU)

应用程序启动时，不会把全部指令加载到内存，也不会分配远超于需要的data内存区域。

回收：当系统发生OOM，物理内存耗尽时，将部分内存page中的内容写回到文件系统再收回这个page

回收哪些page：access位为0，回收最近未被访问过的page。此外，dirty位为0，表明当前page被修改过

### Memory Mapped Files

memory mapped files：将完整或者部分文件加载到内存中，这样就可以通过内存地址相关的load或者store指令来操纵文件。

mmap系统调用：虚拟内存地址，长度，protection，标志位，一个**打开的文件描述符**，偏移量。从文件描述符对应的文件的偏移量位置开始，映射长度为len的内容到虚拟内存地址VA，同时加上一些保护，比如只读或者读写。

mmap系统调用的过程：先记录一下这个PTE属于这个文件描述符。相应的信息保存在VMA结构体（Virtual Memory Area）。例如对于这里的文件f，会有一个VMA，在VMA中我们会记录文件描述符，偏移量等等，这些信息用来**表示对应的内存虚拟地址的实际内容在哪**，这样当我们得到一个位于VMA地址范围的page fault时，内核可以从磁盘中读数据，并加载到内存中。



## 1 Implement copy-on write

xv6 中`fork`系统调用会将父进程的所有内存数据复制给子进程。若父进程内存占用大，复制耗时久，且若子进程后续执行`exec`，复制的内存大概率被丢弃，造成资源浪费。

copy-on write机制直到父子进程中的一方需写入页面时才进行实际的复制操作，以此提升系统效率 。

