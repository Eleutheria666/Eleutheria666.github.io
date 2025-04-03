---
title: "Lab7: networking"
date: 2024-10-4
categories: ["MIT6.S081"]
tags: ["OS"]
ShowToc: true
TocOpen: true
---

Lab链接：[Lab: networking](https://pdos.csail.mit.edu/6.S081/2021/labs/net.html)

Lab源码：[momo/MIT-6S081/net - Gitee.com](https://gitee.com/Eleutheria666/mit-6s081/tree/net/)

## network:o:

### 题目

完成一个网卡驱动程序中传输网络包处理程序`e1000_transmit()`和接受网络包处理程序`e1000_recv()`

### 思路

传输数据包程序`e1000_transmit()`

- 在内存开辟一个暂存待发送数据包的缓冲区，`struct tx_desc`中的addr字段指向内核将待发送数据包写入缓冲区的地址
- 维护一个`struct tx_desc`的环状数组

接受网络包处理程序`e1000_recv()`

- 在内存开辟一个暂存接收到的数据包的缓冲区，`struct rx_desc`中的addr字段指向E1000将接受到的数据包写入缓冲区的地址
- 维护一个`struct rx_desc`的环状数组
