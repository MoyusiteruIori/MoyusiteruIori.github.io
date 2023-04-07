---
title: 四月开始的保研挣扎日志
date: 2023-04-04 23:09:05
tags:
    - 保研
categories: 保研
keywords: '保研'
description: 菜狗怎样自我救赎
cover: cover.png
---

## 4月4日

专业课复习：
- 哈夫曼树和哈夫曼编码
- 希尔排序和归并排序
- AVL 树的调整

项目：
- 完成了 myserver 的文档
    - 写新的项目基本上到此为止了，接下来以复习之前的项目为主

算法题：
- 每日一题是 hard 区间 dp ，放弃
- 739 每日温度，单调栈
    - 求每个元素最左/右边最xxx的元素，应该能第一时间想到单调栈

- 975 奇偶跳，dp + 二分
    - `std::map::upper_bound(x)`：返回值指向首个大于 `x` 的元素
    - `std::map::lower_bound(x)`：返回值指向首个不小于 `x` 的元素


## 4月5日

**完成了初版简历**，花了一下午，但还有待改进

项目复盘：
- `myserver::Address::lookup` 的实现：
    - `getaddrinfo` 根据域名获取地址
    - 传入一个字符串 host 表示域名，一个字符串 service 表示端口（可以是 `"http"`、`"ftp"` 等固定端口的服务）
    - 传入一个 `addrinfo *` 类型的 hint，表示想获得什么样的信息，例如把 hint 的 `ai_type` 字端设为 `SOCK_DGRAM` 只获取 UDP 接口的信息
    - 传入一个 `addrinfo **` 类型的 result 指向返回的结果链表
    - 还有一个参数 `ai_flags`，传入 `AI_CANONNAME` 要求返回主机的规范名称，传入 `AI_NUMERICHOST` 要求地址是数字串
    - `addrinfo` 包含的字端和创建 socket 要设置的那些差不多，就是 `ai_family`、`ai_type`、`ai_protocol`、那些东西，另外包含一个 `ai_addrlen` 和一个 `ai_addr` 方便你取出地址，另外一个 `ai_cannoname` 在某些情况下会包含节点的正规名，然后还有一个 `addrinfo *` 用来形成链表 
    - `memchr(const void *buf, int ch, size_t count)` 从 buf 的前 count 个区域查找 ch
    - `getaddrinfo` 不区分协议，`gethostbyname` 和 `gethostbyaddr` 只支持 v4 协议

- `myserver::Address::interface_address` 的实现：
    - `getifaddrs` 获取网卡的地址信息
    - 传入一个 `ifaddrs **` 即可，返回包含一个所有网卡信息的链表
    - `ifaddrs` 包含 `ifa_name` 网卡名，`sockaddr *ifa_addr, *ifa_netmask` 网卡地址和子网掩码
    - 还包含一个 `union` ，指向一个广播地址或者一个 p2p 目的地址（没用到）

专业课复习：
- 死锁产生的必要条件：
    - 资源是互斥的
    - 线程会等待资源，并且不释放持有的资源
    - 不能抢夺资源
    - 形成循环等待

破坏以上任意一条都能解决死锁问题，但如果破坏前三条，都会改变互斥的语义，因此往往从避免循环等待入手解决

算法题：
- 2427 公因子数目，easy 题，首次尝试使用了 C++ 20 ranges ，然而 LC 并不支持。
- 48 旋转图像，应用一下线性代数，矩阵代表线性变换

## 4月6日

项目复盘：
- 关于 `myserver::Address::UnixAddress`:
    - 本地套接字的作用是进程间通信，和网络套接字类似
    - `sockaddr_un` 主要包含 108 字节的 `sun_path` 路径名，这是一个伪文件，大小永远为 0
    - 服务端 `bind` 之后，指定的 `sun_path` 中会自动生成套接字文件

<div align=center> 

{% asset_img sockaddr.png 各种socket地址 %}

</div>

- `myserver::Socket::option` 查看 socket 选项：
    - 基于 `getsockopt` 实现
    - 如果未进行 `setsockopt()` 调用，则 `getsockopt` 返回系统缺省值。
    - `getsockopt` 通过 `level` 和 `optname` 确定调用者要查看的选项，然后返回值，选项包括一些诸如 `SO_RCVBUF` 、`SO_TYPE` 之类的，具体需要查看手册
    - `setsockopt` 和 `getsockopt` 是对应的，`myserver::Socket` 中的各个设置/获取 socket 选项的方法都基于他们实现

- `myserver::Socket::bind`：ip 地址和端口号绑定到 socket

- `listen` 和 `accept`：
    - `listen` 相当于一种声明，告知内核这个 fd 是一个被动的套接字。没有 `listen` 时，内核默认假设这个套接字会调用 `connect`。
    - `accept` 用来接受客户端发起的连接，并能够获得客户端的地址
    - `connect` 由客户端调用，传入服务器的地址，向服务端发起连接

- `getpeername` 和 `getsockname`：获取对方地址和端口，获取本地地址和端口，对服务端来说 `getsockname` 没有太多卵用，但是建立 TCP 连接之后使用 `getpeername` 获取对方地址和端口可能很有用。对客户端来说，没有 `bind` 的步骤，刚创建 socket 的时候并不会分配本地地址信息，所以也不能调用 `getsockname`，但连接建立以后，可以使用 `getpeername` 获取对方的地址和端口，然而这也没什么卵用，因为你不知道对方的地址端口的话怎么调用 `connect` 去连接呢？当然，对于写框架的人来说还是有用的。
另外，对于无连接的 UDP 协议来说，不能调用 `getpeername`，但是可以调用 `getsockname`,和 TCP 一样，他的地址和端口不是在调用 `socket` 就指定了，而是在第一次调用 `sendto` 函数以后

专业课复习：
- 硬链接和软链接
    - 软链接（符号链接），`ln -s file link`
    - 硬链接，`ln file link`
    - 本质区别：软链接有自己的 inode 和 datablock，其 datablock 指向了原文件，而硬链接直接与原文件共享 inode 和 datablock。本质上硬链接和原文件是同一个文件，而软链接和原文件不是同一个文件。删除原文件后硬链接还能访问这个文件，软链接则无法访问了


<div align=center> 

{% asset_img link.png 硬链接和软链接 %}

</div>

- 希尔排序的时间复杂度为 $O(n^{\frac{3}{2}})$

- 大数定律
    - 切比雪夫大数定律：样本数量足够多时，样本平均数趋近总体平均数，不要求同分布
    - 伯努利大数定律：样本数量足够多时，事件 A 出现的频率将趋近于其概率
    - 辛钦大数定律：切比雪夫大数定律同分布情况下的特例，样本数量足够多时，样本平均数趋近其分布的数学期望

- 中心极限定理：独立同分布的样本数量足够多时，其加和近似符合正态分布

其实今天还看了半天线性代数，但是还是没有理解矩阵的行空间

算法题
- 53 最大子数组和，感觉是比较简单的 dp ，很容易看出来
- 647 回文子串，也是不太难的 dp ，注意填表顺序是从左下开始往右上填

## 4月7日

今天太忙了，做软测小组作业到现在刚回寝室，已经 4 月 8 日 00:21，来不及仔细总结，大致写一下。

项目复盘
- 关于动态链接的全局介入性，`dlfcnt.h` 中的几个函数的使用，主要是 `dlsym`

专业课没空所以没看

算法题
- 337 打家劫舍III，在二叉树上进行 dp ，想了半天做出来了但是内存不够，看了题解学到一招用哈希表作为 dp 数组的操作
- 128 最长连续序列 要求 O(n) 所以不能排序，所以空间换时间，对于数字 `n` ，哈希表记录 `n - 1` 和 `n + 1` 的情况

今天实在比较忙，做得太少了，明天一定狂补。