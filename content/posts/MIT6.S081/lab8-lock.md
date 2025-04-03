---
title: "Lab8: locks"
date: 2024-10-11
categories: ["MIT6.S081"]
tags: ["OS"]
ShowToc: true
TocOpen: true
---

Lab链接：[Lab: locks](https://pdos.csail.mit.edu/6.S081/2021/labs/lock.html)

Lab源码：[momo/MIT-6S081/lock - Gitee.com](https://gitee.com/Eleutheria666/mit-6s081/tree/lock/)

## 1 Memory allocator:white_check_mark:

### 题目

当前xv6，不同CPU核调用`kalloc`/`kfree`函数都是针对同一内存空间进行操作。为了保证数据的一致性，调用`kalloc`/`kfree`需要先获取`kmem.lock`锁。但是这个锁的颗粒度较大，锁住的是所有可分配的内存空间，这样会导致更多CPU竞争这个锁，更多的线程因为无法获取锁而原地等待，不能及时获取/回收相应的内存空间。

你的任务是缩小锁的颗粒度，基本思想是为每个CPU维护一个空闲列表，每个列表都有自己的锁。这样不同CPU上的分配和释放可以并行运行，不会产生锁争用，因为每个CPU将在不同的列表上运行。

若一个CPU的空闲列表为空，在这种情况下，通过“窃取”另一个CPU空闲列表的一部分进行分配，这会引入锁争用，但很少发生。

### 思路

1. 创建一个长度为`NCPU`的`kmem`列表，列表的每一项对应一个CPU，包含了锁核空闲列表的头节点
2. `kinit`初始化内存空间时，对于每一个CPU，只能在【`end+id*INTERVAL`，`end+id*INTERVAL+PGSIZE`】分配内存，所以`freerange()`的范围为【`end+id*INTERVAL`，`end+id*INTERVAL+PGSIZE`】
3. 对于`kfree`函数，如果当前释放的地址`pa`不属于该CPU可控的内存空间之内，则通过`pa`定位当前该物理地址所属的CPU
4. 对于`kalloc`函数，如果当前CPU的空闲列表为0，则遍历所有CPU，从第一个空闲列表不为空的CPU所控制的内存区域进行分配

通过减少锁锁住的共享资源，减少锁竞争的概率，从而提高执行效率，实现锁的优化。

## 2 Buffer cache:x:

这个实验目标是对XV6的磁盘缓冲区进行优化。初始的XV6磁盘缓冲区中是使用一个LRU链表来维护的，而这就导致了每次获取、释放缓冲区时就要对整个链表加锁，也就是说缓冲区的操作是完全串行进行的。



