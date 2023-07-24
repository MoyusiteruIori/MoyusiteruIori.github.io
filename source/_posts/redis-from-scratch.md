---
title: 用 C/C++ 写一个自己的 Redis
date: 2023-07-23 21:27:36
tags:
    - C++
    - C
    - Redis
categories: C/C++
keywords: 'Build your own, C/C++, Redis'
description: 从 0 用 C/C++ 自己写一个 Redis
cover: cover.png
---

原文链接：https://build-your-own.org/redis/
翻译：伊织

**从 0 开始写代码来学习网络编程和数据结构**

作者：James Smith

- **为什么要写 Redis ？** 因为写一个 Redis 所需要的知识比一般的应用层开发来得更广也更深，这是一种很好的培养能力的办法。（主要是在网络编程和数据结构方面）
- **为什么从 0 开始写？** Richard Feynman 有句名言：“只要我不能创造的，我就还不理解”。人类的理解充满了漏洞，即“我不知道我不知道的事情”。从 0 开始编程能够确保你的理解比较完整。
- **为什么要写 C ？** 因为 C 语言被广泛应用在系统编程和基础软件中，所以通过直接的 C 代码来学习 C 是如何在实际项目中应用的，而且也不需要 C/C++ 的预备知识。
- **为什么是以一本书的形式？** Redis 非常复杂，花费了人们很多精力，初学者直接学习会非常困难。这本书把核心的概念拆开，以便读者一步步学习。

# 介绍

## 这本书是写什么的？

这本书一步一步地演示了一个 Redis-like 的服务器的简单实现，旨在成为一个 C 语言网络编程和数据结构的实现与应用的实用指南/教程。

## 从这本书能学到什么？

Redis 可以说是现代计算机世界的*一块基石*，经受住了时间的考验。编写这样一个项目所需的知识比通常的应用层开发来得更广也更深。学习这样一个项目是程序员自我提高的一种非常好的办法。
Redis 是一个非常好的学习目标，因为它涵盖了软件工程的两个重要主题：网络编程和数据结构。

- 你能找到很多 socket API 和高级程序库的教程，但是网络编程远远不止调包这么简单。很重要的一点是要理解一些核心概念，比如事件循环（event loop）、协议（protocol）、定时器（timer）等等，这本书里都会包含。如果你不理解这些核心概念，那你就算只当调包侠，也常常会写出 fatal error。
- 很多人从书上学会了基本数据结构，但其实要继续学的东西还多着呢。真实项目中的数据结构往往有一些教科书上不会提及的实践考量。学习一个非玩具的正经项目中是怎么使用数据结构的，这会是写 Redis 带给你的一段独特体验。

正如大多数真实世界的项目一样，Redis 非常复杂，凝结了人们大量的努力，对于初学者来说直接学习会非常困难。因此这本书采取一种相反的策略：*从零开始*自己写一个。

## 为什么要从 0 开始？

原因有以下几点：

- **为了学得更快。** 从 0 开始写一样东西的过程能够循序渐进的引入各个概念。从最小的东西开始，增量地添加功能，最后搞个大的出来。
- **为了学得更深。** 虽然有很多资料可以解释现在的东西都是怎么工作的，但你从阅读资料中获得的理解通常不如自己写一个来得深刻。只读不写的话就很容易记错，或者找不到重点。
- **为了学得更多。** “从 0 开始”强迫你去了解项目的各个方面，不会有你错过的知识！而且通常不是每个方面的知识你都提前有所了解的，你会在这个过程中发现一些“我不知道我不知道的东西”。

一言以蔽之，就是 Richard Feynman 的名言：“只要我不能创造的，我就还不理解”。

## 如何使用本书？

这本书之后的章节会是循序渐进一步一步的。每一步都基于上一步，增加一个新的概念。网站会提供完整的源代码，建议读者自行调试或者尝试自己编写。

作者会把代码写得尽可能直截了当。大多数是朴素的 C 语言，少部分涉及 C++ 特性。没学过 C 的话也别担心，你刚好可以自己用另一门语言来实现。

最后的成果会是一个迷你版的 Redis ，差不多就 1200 行代码。别看 1200 很少，其中包括了这本书覆盖的很多重要方面。

这本书中使用的技术和方法并不是和真实的 Redis 完全一致的，有一部分被刻意简化了，有一些更普适化。比较这些方法的不同点能让读者学到更多东西。

这本书的代码只能在 Linux 上跑，代码可以从这里下载：

https://build-your-own.org/redis/src.tgz

书的内容和源代码可以在这里在线阅读：

https://build-your-own.org

（注：本篇翻译中的代码是译者按照教程自己写的，和原文会略有出入，但大体上是一致的，译者的代码仓库：）


# Socket 简介

这一章是 socket 编程介绍。读者应该有基本的计算机网络知识，但不是必须有网络编程经验。本书不会包含 socket API 的细节，建议读者学习本书的同时去阅读 API 手册和其他网络编程教程。(推荐一个学习 socket API 的材料： https://beej.us/)

Redis 是一个 server/client 系统。多个客户端连接到一个服务器，服务器从 TCP 连接上收到请求，并发回响应。在开始 socket 编程之前，先要学习几个 Linux 系统调用。

`socket()` 系统调用返回一个 fd 。如果你不熟悉 Unix 操作系统的话，我粗略地解释一下“fd”：一个 fd 就是一个整数，指向 Linux 内核中的某个对象，比如一个 TCP 连接，一个磁盘上的文件，一个监听端口，或者其他资源等。

`bind()` 和 `listen()` 系统调用：`bind()` 将一个地址关联到一个 socket fd 上，`listen()` 使得我们能够接受到这个地址上的连接。

`accept` 接收一个监听 fd 。当一个客户端连接到了监听地址上，`accept` 会返回一个 fd ，代表这个连接。下面是一段解释一般服务器工作流程的伪代码：

```
fd = socket()
bind(fd, address)
listen(fd)
while True:
    conn_fd = accept(fd)
    do_something_with(conn_fd)
    close(conn_fd)
```

`read()` 系统调用会从 TCP 连接上读取数据，`write()` 系统调用则发送数据。`close()` 系统调用会销毁 fd 指向的内核资源，并回收这个 fd 。

我们介绍完了服务端网络编程所需要的系统调用。对于客户端而言，`connect()` 系统调用接受一个 socket fd 和一个地址，并向这个地址发起一个 TCP 连接。下面是客户端的伪代码：

```
fd = socket()
connect(fd, address)
do_something_with(fd)
close(fd)
```

下一章开始就会使用真正的代码了。


# Hello 服务器/客户端

本章继续介绍 socket 编程。我们会编写两个程序（不完整，跑不起来）来演示上一章的系统调用。第一个程序是一个服务器，它与客户端建立连接，读取一条信息，并写回一条响应。第二个程序是一个客户端，他发起到服务器的连接，发送一条信息，并接收一条响应。我们先来写服务器。

首先，需要获取一个 socket fd：`int fd = socket(AF_INET, SOCK_STREAM, 0);`

`AF_INET` 用于 IPv4 ，对于 IPv6 或者双栈协议，可以使用 `AF_INET6` 。简便起见，整本书中，我们都使用 `AF_INET` 。

`SOCK_STREAM` 用于 TCP ，本书中我们不会用到 TCP 以外的传输层协议。书中 `socket()` 的 3 个参数一直都是固定的。

接下来，我们要介绍一个新的系统调用：

```C++
int val = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val));
```

`setsockopt()` 调用用于对 socket 进行各种配置。上面的调用例子启用了 `SO_REUSEADDR` 选项。如果不启动这个选项，服务器在重启对时候就无法绑定到同一个地址了。这里给读者留一个练习：查清楚 `SO_REUSEADDR` 到底是什么，有什么用？

> `SO_REUSEADDR` 选项意为允许服务器 bind 一个地址，即使这个地址当前已经存在已建立的连接。例如，假设服务器与一个客户端在一个地址上建立了连接，此时服务器关闭，根据 TCP 协议，和客户端的连接进入 TIME_WAIT 状态，此时服务器重启，之前的地址上的连接还未关闭，若未开启 `SO_REUSEADDR` ，就会无法绑定地址。

下一步是 `bind()` 和 `listen()` ，我们就绑定通用地址 `0.0.0.0:1234`：

```C++
struct sockaddr_in addr = {};
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port = htons(1234);
addr.sin_addr.s_addr = htonl(0);
if (bind(fd, (const struct sockaddr*)&addr, sizeof(addr))) {
    die("bind()");
}

if (listen(fd, 10)) {
    die("listen()");
}
```

循环处理各个连接并做一些事情：

```C++
while (true) {
    // accept
    struct sockaddr_in client_addr;
    socklen_t cliend_addr_len = sizeof(client_addr);
    memset(&client_addr, 0, cliend_addr_len);
    int conn_fd = accept(fd, (struct sockaddr*)&client_addr, &cliend_addr_len);
    if (conn_fd < 0) {
        continue;   // error
    }
    do_something(conn_fd);
    close(conn_fd);
}
```

`do_something` 这一函数简单进行一次读和一次写即可：

```C++
void do_something(int conn_fd) {
    char rbuf[64] = {};
    ssize_t n = read(conn_fd, rbuf, sizeof(rbuf) - 1);
    if (n < 0) {
        fprintf(stderr, "read() error");
        return;
    }
    printf("Client says: %s\n", rbuf);

    const char* wbuf = "world";
    write(conn_fd, wbuf, strlen(wbuf));
}
```

注意到 `read()` 和 `write()` 这两个调用会返回读取/写入的字节数，实际编程过程中程序员必须处理这些函数的返回值，但是在这一章里，我为了简洁而省略了许多东西，所以这一章中的代码其实不是网络编程的正确姿势。

客户端的代码会简单很多：

```C++
int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd < 0) {
    die("socket()");
}

struct sockaddr_in server_addr;
memset(&server_addr, 0, sizeof(server_addr));
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(1234);
server_addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
int rv = connect(fd, (const sockaddr*)&server_addr, sizeof(server_addr));
if (rv < 0) {
    die("connect()");
}

const char* msg = "hello";
write(fd, msg, strlen(msg));

char rbuf[64] = {};
int n = read(fd, rbuf, sizeof(rbuf) - 1);
if (n < 0) {
    die("read()");
}

printf("Server says: %s\n", rbuf);
close(fd);
```

用以下命令编译代码：

```bash
g++ -Wall -Wextra -O2 -g 03_server.cpp -o server
g++ -Wall -Wextra -O2 -g 03_client.cpp -o client
```

在一个窗口中运行 `./server` ，另一个窗口运行 `./client` ，你会见到如下输出：
```
# ./server 
Client says: hello

```

```
# ./client 
Server says: world
```

另一个给读者的练习：阅读一下本章用到的 API 的手册或者是在线教程，保证你知道如何找到这些 API 的使用帮助，因为这本书中不包含这些 API 的使用细节。

源代码：

> - [03_client.cpp](https://build-your-own.org/redis/03/03_client.cpp.htm)
> - [03_server.cpp](https://build-your-own.org/redis/03/03_server.cpp.htm)

未完待续。