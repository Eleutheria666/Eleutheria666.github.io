---
title: "Lab6: multithreading"
date: 2024-09-28
categories: ["MIT6.S081"]
tags: ["OS"]
ShowToc: true
TocOpen: true
---

Lab链接：[Lab: Multithreading](https://pdos.csail.mit.edu/6.S081/2021/labs/thread.html)

Lab源码：[momo/MIT-6S081/thread - Gitee.com](https://gitee.com/Eleutheria666/mit-6s081/tree/thread/)

## 1 Uthread: switching between threads:o:

### 题目

当前xv6的线程切换需要用户线程进入内核空间后，再调用CPU调度器线程完成线程切换。

本lab需要你在用户空间完成线程切换。

### 思路

Q：为什么只需要保存callee-saved register？

A：调用`swtch()`前会保存caller-saved reg至栈帧，而`swtch()`是callee，该函数需保存callee-saved reg。对于`trapframe`，由于系统会在用户进程不经意间打断，所以需要保存所有的寄存器

栈从高地址到低地址增长，因此初始化线程的sp指针应该是`stack+STACK_SIZE-1`

## 2 Using threads​​

## 3 Barrier​​

实现多个线程的同步。
