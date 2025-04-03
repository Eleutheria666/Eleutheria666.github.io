---
title: "Lab9: file system"
date: 2024-10-20
categories: ["MIT6.S081"]
tags: ["OS"]
ShowToc: true
TocOpen: true
---

Lab链接：[Lab: file system (mit.edu)](https://pdos.csail.mit.edu/6.S081/2021/labs/fs.html)

Lab源码：[momo/MIT-6S081/fs - Gitee.com](https://gitee.com/Eleutheria666/mit-6s081/tree/fs/)

## 1 Large files

### 题目

当前xv6文件最大只能存储268 blocks的数据。你的任务是修改xv6的代码，将inode结点中的一个“直接索引”改为“二级索引”：该索引指向的block包含256个间接block的地址，每个间接block又包含256个数据block的地址，以扩大文件可存储数据的容量。

### 思路

将`struct inode`中的`NDIRECT`直接索引的盘块号减一，腾出一个索引用于二级索引，其他变量随之改动。**此时 struct dinode和struct inode的addrs数组大小应为[NDIRECT+2]**

```C++
// kernel/fs.h

#define NDIRECT 11 // 12->11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDOUBLEINDIRECT (NINDIRECT*NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDOUBLEINDIRECT)

struct dinode {
  // ...
  uint addrs[NDIRECT+2];   // [NDIRECT+1] -> [NDIRECT+2]
};

struct inode {
  // ...
  uint addrs[NDIRECT+2];   // [NDIRECT+1] -> [NDIRECT+2]
};
```

重点是对`bmap()`和`itruc()`的修改，两者的逻辑几乎一致。

`bn`表示当前需要读取的是`inode`的第几个block，是从0开始的，所以：

- `bn < NDIRECT`，当前block地址应由直接索引得到
- `bn-NDIRECT < NDOUBLEINDIRECT`，当前block地址由一级索引得到
- 若都不满足，当前block地址由二级索引得到

增加的是对二级索引的操作，首先通过`bn/NINDIRECT`得到一级block的地址，再通过`bn%NINDIRECT`，从一级block中得到二级block的地址，也就是目标数据块的地址。

```C++
// fs.c bmap(struct inode *ip, uint bn)

bn -= NINDIRECT;

if (bn < NDOUBLEINDIRECT) {
// inode
if((addr = ip->addrs[NDIRECT+1]) == 0)
  ip->addrs[NDIRECT+1] = addr = balloc(ip->dev);
bp = bread(ip->dev, addr);
a = (uint*)bp->data;

// block 1
if((addr = a[bn/NINDIRECT]) == 0) {
  a[bn/NINDIRECT] = addr = balloc(ip->dev);
  log_write(bp);
}
brelse(bp);
bp = bread(ip->dev, addr);
a = (uint*)bp->data;

// block 2
if((addr = a[bn%NINDIRECT]) == 0) {
  a[bn%NINDIRECT] = addr = balloc(ip->dev);
  log_write(bp);
}
brelse(bp);
```



## 2 Symbolic links

### 题目

实现xv6的符号链接功能：`symlink(char *target, char *path)`，要求`path`路径代表的文件能链接到`target`代表的文件。

### 思路

硬链接和软链接的本质区别：

- 硬链接是**不同的路径指向同一个inode文件**。
- 符号链接是**两个不同inode的文件**，目标文件的路径以字符串格式保存在符号链接的数据区域中，通过读取符号链接数据区域的内容得到目标文件。

本任务包含两个部分：①创建软链接，②根据符号链接打开目标文件

#### ① 创建软链接

这其实是操作系统提供给用户程序的一种服务，所以首先要实现系统调用。

重点是`sys_symlink(char *target, char *path)`函数的实现。

首先调用`create()`在`path`中创建一个文件项，得到该文件项的inode指针`ip`。创建文件不仅需要申请inode并写入相关元数据，还需要找到该文件所在目录，并将该inode作为文件项加入目录，`create()`已经帮我们完成这些，但调用返回时**仍持有该inode的锁**，要在创建符号链接之后释放这个锁。

其次，调用`writei()`向该inode写入目标文件的路径字符串。

```c++
// sysfile.c

uint64
sys_symlink(void) 
{
  char target[MAXPATH], path[MAXPATH];
  struct inode *ip;

  if(argstr(0,target,MAXPATH) < 0 || argstr(1,path,MAXPATH) < 0) 
    return -1;

  begin_op();

  if((ip = create(path, T_SYMLINK, 0, 0)) == 0) {
    end_op();
    return -1;
  }

  if(writei(ip, 0, (uint64)target, 0, sizeof(target)) != sizeof(target)) {
    iunlockput(ip);
    end_op();
    return -1;
  }
  iunlockput(ip);
  end_op();
  return 0;
}
```

#### ② 根据符号链接打开目标文件

`open()`已经通过`namei(char *path)`得到路径指向文件的inode。

若open模式为`O_NOFOLLOW`，**则无需调用`follow_symlink()`**，inode应当看作`F_FILE`类型的文件，进行读写操作。因此，调用`follow_symlink()`需要满足以下条件：

```C++
// sysfile.c open()

if(ip->type == T_SYMLINK && omode != O_NOFOLLOW) {
    ip = follow_symlink(ip, 0);
    if(ip == 0) {
      end_op();
      return -1;
    }
}
```

若`follow_symlink()`返回为0，表示读取失败，提前结束。

`follow_symlink()`接受两个参数：**当前需要读取文件的inode指针，当前是第几次递归调用**，其执行逻辑如下：

1. 通过读取`ip`指向的inode数据区中的内容得到目标文件的路径
2. 通过`namei()`得到目标文件的inode
3. 若当前inode类型为`F_FILE`，则得到了真正目标文件的inode，返回；否则当前inode也是符号链接，将自身的inode指针传入，继续递归

凡是对inode中`type`字段进行操作的指令，都需要获取该inode的锁才能执行。

```c++
// sysfile.c

struct inode*
follow_symlink(struct inode* ip, int times)
{
  char newpath[MAXPATH];

  if(times >= 10) {
    iunlock(ip);
    printf("follow too many times, maybe a cycle\n");
    return 0;
  }

  if(readi(ip, 0, (uint64)newpath, 0, sizeof(newpath)) == sizeof(newpath)) {
    iunlock(ip);

    if((ip = namei(newpath)) == 0)
      return ip;

    ilock(ip);
    if(ip->type == T_FILE) 
      return ip;
    return follow_symlink(ip, times+1);
  }
  return 0;
}
```

