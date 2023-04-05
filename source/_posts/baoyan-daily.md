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