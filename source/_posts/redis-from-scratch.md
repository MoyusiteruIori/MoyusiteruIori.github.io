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

（注：本文中使用的伪代码语法很像 Python ，所以翻译代码段使用 Python 辅助语法高亮，实际并不是 Python）

```Python
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

本章源代码：

> - [03_client.cpp](https://build-your-own.org/redis/03/03_client.cpp.htm)
> - [03_server.cpp](https://build-your-own.org/redis/03/03_server.cpp.htm)


# 协议解析

## 总览

我们的服务器要能够处理来自客户端的多个请求，所以我们需要实现某些“协议”，至少要能把请求从 TCP 字节流中分离出来。最简单的一种提取请求的方法就是在请求头声明整个请求的长度。我们用下面这种格式：

```
+-----+------+-----+------+--------
| len | msg1 | len | msg2 | more...
+-----+------+-----+------+--------
```

这个协议包括两部分：一个 4 字节的小端序整型，表示请求的长度，以及一个变长的请求体。

在上一章的代码基础上，我们要修改一下服务器的循环，来处理多个请求：

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
    while (true) {
        int32_t err = one_request(conn_fd);
        if (err) {
            break;
        }
    }
    close(conn_fd);
}
```

`one_request` 函数只会解析一个请求，并进行响应，直到出现错误或者客户端关闭连接。在我们讲到后面的事件循环章节之前，我们的服务器一次只能接收一个连接。

## I/O 辅助函数

在编写 `one_request` 函数之前，我们先增加两个辅助函数：

```C++
static int32_t read_full(int fd, char* buf, size_t n) {
    while (n > 0) {
        ssize_t rv = read(fd, buf, n);
        if (rv <= 0) {
            return -1;  // error or unexpected EOF
        }
        assert((size_t)rv <= n);
        n -= rv, buf += rv;
    }
    return 0;
}

static int32_t write_all(int fd, const char* buf, size_t n) {
    while (n > 0) {
        ssize_t rv = write(fd, buf, n);
        if (rv <= 0) {
            return -1;
        }
        assert((size_t)rv <= n);
        n -= rv, buf += rv;
    }
    return 0;
}
```

注意两点：

1. `read()` 系统调用会返回内核中所有可读的数据，没有数据可读的话就会阻塞，若没有读到足够的数据则需要由应用自己处理。`read_full` 函数的作用就是在读到 `n` 字节的数据之前一直向内核请求读。
2. 类似地，如果内核缓冲区满了，`write` 系统调用可能只读取到部分数据，当 `write` 写入的数据比我们预期的少的时候，必须一直尝试发送直到发完 `n` 字节的数据。

## 解析器

`one_request` 函数担当了这项重任：

```C++
static int32_t one_request(int conn_fd) {
    char rbuf[4 + k_max_msg + 1] = {};
    errno = 0;
    int32_t err = read_full(conn_fd, rbuf, 4);
    if (err) {
        if (errno == 0) {
            fprintf(stderr, "EOF\n");
        }
        else {
            fprintf(stderr, "read() error");
        }
        return err;
    }
    uint32_t len = 0;   // assume little endian
    memcpy(&len, rbuf, 4);
    if (len > k_max_msg) {
        fprintf(stderr, "reading message too long: %d.\n", len);
        return -1;
    }

    // request body
    err = read_full(conn_fd, rbuf + 4, len);
    if (err) {
        fprintf(stderr, "read() error.\n");
        return err;
    }

    // do_something
    rbuf[4 + len] = 0;
    printf("Client says: %s\n", rbuf + 4);

    // reply using the same protocol
    const char reply[] = "world";
    char wbuf[4 + sizeof(reply)];   
    len = (uint32_t)sizeof(reply);
    memcpy(wbuf, &len, 4);
    memcpy(wbuf + 4, reply, len);
    
    return write_all(conn_fd, wbuf, 4 + len);
}
```

为了方便，我们给请求大小设置了一个最大值，用一个足够大的缓冲区来存储请求。在解析请求的时候，字节序是一个要考虑的因素，但和我们今天的内容不太相关，所以我们直接用 `memcpy` 来拷贝整数。

## 客户端

客户端发送请求并作出响应的代码如下：

```C++
static int32_t query(int fd, const char* text) {
    uint32_t len = (uint32_t)strlen(text);
    if (len > k_max_msg) {
        fprintf(stderr, "Sending message too long.\n");
        return -1;
    }

    char wbuf[4 + k_max_msg] = {};
    memcpy(wbuf, &len, 4);
    memcpy(wbuf + 4, text, len);
    if (int32_t err = write_all(fd, wbuf, 4 + len)) {
        return err;
    }

    // 4 bytes header
    char rbuf[4 + k_max_msg + 1] = {};
    errno = 0;
    int32_t err = read_full(fd, rbuf, 4);
    if (err) {
        if (errno == 0) {
            fprintf(stderr, "EOF\n");
        }
        else {
            fprintf(stderr, "read() error\n");
        }
        return err;
    }

    memcpy(&len, rbuf, 4);
    if (len > k_max_msg) {
        fprintf(stderr, "reading message too long: %d.\n", len);
        return -1;
    }

    // reply body
    err = read_full(fd, rbuf + 4, len);
    if (err) {
        fprintf(stderr, "read() error\n");
        return err;
    }

    // do something
    printf("Server says: %s\n", rbuf + 4);

    return 0;
}
```

## 测试

发送多个请求来测试我们的服务器：

```C++
int main() {
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

    int32_t err = query(fd, "hello1");
    if (err) {
        goto L_DONE;
    }
    err = query(fd, "hello2");
    if (err) {
        goto L_DONE;
    }
    err = query(fd, "hello3");
    if (err) {
        goto L_DONE;
    }

L_DONE:
    close(fd);
    return 0;
}
```

运行服务器和客户端：

```
# ./server 
Client says: hello1
Client says: hello2
Client says: hello3
EOF

# ./client 
Server says: world
Server says: world
Server says: world
```

## 多聊两句协议设计

在设计协议的时候要考虑很多内容，你可以看看现有的协议来学习一下各方面的权衡。

### 文本 vs 二进制

设计协议要做的第一个决定是文本 vs 二进制。文本协议的好处是人类可读，容易调试，最著名的代表就是 HTTP 协议（不是新的那个）。

文本协议的缺点是其复杂性，即使最简单的文本协议也更难解析、更易出错。你可以尝试比较一下 Redis 协议与本书的协议。

为什么文本协议不好解析呢？因为里面有变长的字符串，解析协议的代码会包含大量的长度计算，边界检查和决策。比方说你想解析一个十进制文本表示的整数“1234”，那么对每一个字节，你都得检查缓冲区末尾，以及这个整数是否完结，相比之下，一个二进制的宽度确定的整数要简单得多。

上面的例子引出了一个设计协议的小 tip：避免非必要的变长部分，变长部分越少，解析越简单，安全性的 bug 越少。假设你想在你的协议中加入一个字符串，考虑一下能否设计成定长的字符串，或者这个字符串到底有没有必要加。

### 流式数据（Streaming Data）

我们的协议在头部包含了消息的长度，然而，现实世界的协议通常使用不太明显的方法来指示消息的结束。一些应用支持连续的“流式传输”数据，而不知道输出的总长度。一个很好的例子是 HTTP 协议中的“分块传输编码（Chunked Transfer Encoding）”。

分块编码将传入数据封装为一个消息格式，该格式以消息的长度开头。客户端接收到一系列消息，直到特殊的消息指示流的结束。

也有另一种不太好的方法来实现这一点：使用一种特殊符号（分隔符）来表明流的结尾。这样做的问题在于载荷数据中就不能包含这个符号，你需要一些“转义”机制，这搞得事情很复杂。

### 进一步的考虑

解析协议的代码需要至少两次 `read()` 系统调用。我们可以通过“缓冲 I/O（Buffered I/O）”来减少系统调用的次数，意思就是一次读出尽可能多的数据到一个缓冲区中，然后尝试从缓冲区中解析多个请求。建议读者作为练习来尝试一下，这会对理解后面的章节很有帮助。

在设计和实现协议的时候，有一些新手很容易犯的错误：

> 错误 1：不处理 `read` 和 `write` 的返回值。

这两个函数读到的数据有可能会比你期望的少，看一下 `read_full` 这个函数注意点，在事件循环中这也是个常见的错误。

> 错误 2：不知道怎样表示消息的结尾。

人们经常以为 `read` 和 `write` 系统调用得到的是“消息”而不是字节流，导致协议解析不出来。早期版本的HTTP也存在这个缺陷：没有Content-Length头部或分块编码的HTTP连接无法用于多个请求。

> 错误 3：非必要的复杂性

请看协议设计的部分。

本章源代码：

> - [04_client.cpp](https://build-your-own.org/redis/04/04_client.cpp.htm)
> - [04_server.cpp](https://build-your-own.org/redis/04/04_server.cpp.htm)

# 事件循环和非阻塞 I/O

服务端网络编程中，有 3 种处理并发连接的方式，分别是：多进程（forking）、多线程（multi-threading）和事件循环（event loop）。多进程方式为每个连接创建一个新的进程来实现并发，多线程则用线程来代替进程，事件循环方式使用轮询（polling）和非阻塞 I/O ，并且一般跑在单线程上。因为线程和进程的开销，大多数现代工业级软件的网络部分都使用事件循环的方式。

我们服务器的事件循环的简化伪代码如下：

```Python
all_fds = [...]
while True:
    active_fds = poll(all_fds)
    for each fd in active_fds:
        do_something_with(fd)

def do_something_with(fd):
    if fd is a listening socket:
        add_new_client(fd)
    elif fd is a client connection:
        while work_not_done:
            do_something_to_client(fd)

def do_something_to_client(fd):
    if should_read_from(fd):
        data = read_util_EAGAIN(fd)
        process_incoming_data(data)
    while should_write_to(fd):
        write_util_EAGAIN(fd)
    if should_close(fd):
        destroy_client(fd)
```

我们不仅处理 fd （进行读、写、接收），还使用 `poll` 函数来告诉我们哪些 fd 上可以*立刻*进行操作而无需阻塞。当我们在一个 fd 上进行 I/O 操作时，这个操作应该在非阻塞模式下进行。

在阻塞模式下，`read` 会在内核中没有可读数据的时候阻塞调用者，`write` 会在内核缓冲区满的时候阻塞调用者，`accept` 会在内核队列中没有新到来的连接时阻塞调用者。在非阻塞模式下，这些操作要么不阻塞直接执行成功，要么失败并将 errno 设为 `EAGAIN` ，表示“还没准备好”。执行失败并且带有 `EAGAIN` 的操作必须在 `poll` 通知就绪之后进行重试。

`poll` 是事件循环中*唯一的*阻塞操作，除此以外的必须都是非阻塞的，因此单线程也能处理大量并发连接。所有阻塞的网络 I/O API ，如 `read`、`write` 和 `accept`， 都有非阻塞模式。有一些没有非阻塞模式的 API 则需要在线程池中执行，比如 `gethostbyname` 和磁盘 I/O ，这点会在后面的章节讲到。同样的，事件循环中必须实现定时器（timer），因为我们不能在事件循环用 `sleep` 等待。

把一个 fd 设置为非阻塞模式的系统调用是 `fcntl` ：

```C++
static void fd_set_nb(int fd) {
    errno = 0;
    int flags = fcntl(fd, F_GETFL, 0);
    if (errno) {
        die("fcntl error");
        return;
    }

    flags |= O_NONBLOCK;

    errno = 0;
    (void)fcntl(fd, F_SETFL, flags);
    if (errno) {
        die("fcntl error");
    }
}
```

在 Linux 中，除了 `poll` 系统调用，还有 `select` 和 `epoll`。古老的 `select` 系统调用和 `poll` 差不多，但是 fd 的最大数量被限制得很小，导致它被现代应用淘汰了。`epoll` API 包括 3 个系统调用：`epoll_create`，`epoll_wait`，和 `epoll_ctl`。`epoll` API 是有状态的，它并不是把 fd 的集合作为参数，而是用 `epoll_ctl` 来操作一个由 `epoll_create` 创建出来的 fd 集合，`epoll_wait` 就在该集合上执行。

下一章中我们会使用 `poll` 系统调用，因为它的代码相比有状态的 `epoll` 来说会比较少。但是真实世界的项目中通常更倾向于使用 `epoll`，因为随着 fd 数量增多，`poll` 可能会变得太大。

# 事件循环的实现

这一章会介绍一个 echo 服务器的真实 C++ 代码。

## 概览

`struct Conn` 的定义如下：

```C++
enum {
    STATE_REQ = 0,
    STATE_RES = 1,
    STATE_END = 2   // mark the connection for deletion
};

struct Conn {
    int fd = -1;
    uint32_t state = STATE_REQ;
    // buffer for reading
    size_t rbuf_size = 0;
    uint8_t rbuf[4 + k_max_msg];
    // buffer for writing
    size_t wbuf_size = 0;
    size_t wbuf_sent = 0;
    uint8_t wbuf[4 + k_max_msg];
};
```

我们需要读和写操作的缓冲区，因为是非阻塞模式编程，所以 I/O 操作经常延迟。

`state` 用来决定在这个连接上进行的是哪种操作，一个进行中的连接有两种状态。`STATE_REQ` 表示现在要从这个连接中读取请求，而 `STATE_RES` 表示要从该连接向客户端返回响应。

事件循环的代码如下：

```C++
int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        die("socket");
    }

    int val = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val));

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

    // a map of all client connections, keyed by fd
    std::unordered_map<int, Conn*> fd2conn;

    // set listen fd to non blocking mode
    set_fd_nb(fd);

    // event loop
    std::vector<struct pollfd> poll_args;
    while (true) {
        // prepare the arguments of the poll()
        poll_args.clear();
        // for convenience, the listening fd is put in the first position
        struct pollfd pfd = {fd, POLLIN, 0};
        poll_args.push_back(pfd);
        // connection fds
        for (auto& [_, conn] : fd2conn) {
            if (!conn) {
                continue;
            }
            struct pollfd pfd = {};
            pfd.fd = conn->fd;
            pfd.events = (conn->state == STATE_REQ) ? POLLIN : POLLOUT;
            pfd.events = pfd.events | POLLERR;
            poll_args.push_back(pfd);
        }

        // poll for active fds
        // the timeout argument doesn't matter here
        int rv = poll(poll_args.data(), (nfds_t)poll_args.size(), 1000);
        if (rv < 0) {
            die("poll");
        }

        // process active connections
        for (size_t i = 1; i < poll_args.size(); ++i) {
            if (poll_args[i].revents) {
                Conn* conn = fd2conn[poll_args[i].fd];
                connection_io(conn);
                if (conn->state == STATE_END) {
                    // client closed normally, or something bad happened
                    // destroy this connection
                    fd2conn[conn->fd] = nullptr;
                    (void)close(conn->fd);
                    free(conn);
                }
            }
        }

        if (poll_args[0].revents) {
            (void)accept_new_conn(fd2conn, fd);
        }
    }

    return 0;
}
```

我们事件循环的第一件事情就是为 `poll` 设置参数。监听 fd 使用 `POLLIN` 标识加入 poll 中。对于连接 fd ，其对应的 `Conn` 结构体的 state 决定了其标识。在我们的场景下，poll 标识要么是读（`POLLIN`）要么是写（`POLLOUT`），不会同时发生。如果使用 `epoll` ，那么事件循环的第一件事往往是使用 `epoll_ctl` 来更新 fd 的集合。

`poll` 还接收一个 `timeout` 参数，可以用来实现定时器（后面的章节介绍）。目前 `timeout` 并不重要，我们先随便设置一个大点的数字。

在 `poll` 返回之后，它会通知我们哪些 fd 上准备好了读/写操作，我们会相应执行。

## 新的连接

`accept_new_conn` 函数会接收一个新的连接并创建一个 `Conn` 对象：

```C++
static void conn_put(
    std::unordered_map<int, Conn*>& fd2conn, 
    struct Conn* conn
) {
    fd2conn.emplace(conn->fd, conn);
}

static int32_t accept_new_conn(
    std::unordered_map<int, Conn*>& fd2conn, 
    int fd
) {
    // accept
    struct sockaddr_in client_addr = {};
    socklen_t socklen = sizeof(client_addr);
    int connfd = accept(fd, (struct sockaddr*)&client_addr, &socklen);
    if (connfd < 0) {
        fprintf(stderr, "accept() error\n");
        return -1;
    }

    // set the new connection fd to nonblocking mode
    set_fd_nb(connfd);
    // creating the struct Conn
    struct Conn* conn = (Conn*)malloc(sizeof(struct Conn));
    if (!conn) {
        close(connfd);
        return -1;
    }
    conn->fd = connfd;
    conn->rbuf_size = 0;
    conn->wbuf_sent = 0;
    conn->wbuf_size = 0;
    conn_put(fd2conn, conn);
    return 0;
}
```

`connection_io` 函数是用于客户端连接的状态机：

```C++
static void connection_io(Conn* conn) {
    if (conn->state == STATE_REQ) {
        state_req(conn);
    }
    else if (conn->state == STATE_RES) {
        state_res(conn);
    }
    else {
        assert(0);  // not expected
    }
}
```

## 状态机：读者

`STATE_REQ` 状态用于读操作：

```C++
static void state_req(Conn* conn) {
    while (try_fill_buffer(conn));
}

static bool try_fill_buffer(Conn* conn) {
    // try to fill the buffer
    assert(conn->rbuf_size < sizeof(conn->rbuf));
    ssize_t rv = 0;
    do {
        size_t cap = sizeof(conn->rbuf) - conn->rbuf_size;
        rv = read(conn->fd, conn->rbuf + conn->rbuf_size, cap);
    } while (rv < 0 && errno == EINTR);
    if (rv < 0 && errno == EAGAIN) {
        // got EAGAIN, stop.
        return false;
    }
    if (rv < 0) {
        fprintf(stderr, "read error\n");
        conn->state = STATE_END;
        return false;
    }
    if (rv == 0) {
        if (conn->rbuf_size > 0) {
            fprintf(stderr, "unexpected EOF\n");
        }
        else {
            fprintf(stdout, "EOF\n");
        }
        conn->state = STATE_END;
        return false;
    }

    conn->rbuf_size += (size_t)rv;
    assert(conn->rbuf_size < sizeof(conn->rbuf));

    // Try to process requests one by one
    // Why is there a loop? Please read the explanation of "pipelining".
    while (try_one_request(conn));
    return (conn->state == STATE_REQ);
}
```

这里有很多东西要拆开来说。为了理解这个函数，我们先看上一章中的伪代码：

```Python
def do_something_to_client(fd):
    if should_read_from(fd):
        data = read_until_EAGAIN(fd)
        process_incoming_data(data)
    # code omitted...
```

`try_fill_buffer` 函数向缓冲区中填入数据。因为读缓冲区的大小是有限的，这个缓冲区可能在发生 `EAGAIN` 之前就满了，所以我们需要在读取之后立即处理数据来释放读缓冲区的空间，然后 `try_fill_buffer` 会循环直到发生 `EAGAIN` 。

`read` 系统调用（和其他系统调用一样），需要在发生 `EINTR` 时进行重试。`EINTR` 表示系统调用被信号打断了，即使我们的应用没有用到信号，也应该进行重试。

## 解析协议

`try_one_request` 函数会处理读取到的数据，但是为什么这里是个循环呢？是因为读缓冲区中有不止一个请求吗？正是如此。对于一个请求/响应协议，客户端不会一次只发送一个请求然后就开始等待响应，客户端可能会连续发送很多请求，而并不在每个请求之后都等待响应，这种操作模式成为“流水线（pipelining）”，因此我们不能假设读缓冲区中最多只有一个请求。

以下是 `try_one_request` 的代码：

```C++
static bool try_one_request(Conn* conn) {
    if (conn->rbuf_size < 4) {
        return false;
    }
    uint32_t len = 0;
    memcpy(&len, conn->rbuf, 4);
    if (len > k_max_msg) {
        fprintf(stderr, "too long");
        conn->state = STATE_END;
        return false;
    }
    if (4 + len > conn->rbuf_size) {
        return false;
    }

    // got one request, do something with it
    printf("Client says: %.*s\n", len, conn->rbuf + 4);

    // generating echo response
    memcpy(conn->wbuf, &len, 4);
    memcpy(conn->wbuf + 4, conn->rbuf + 4, len);
    conn->wbuf_size = 4 + len;

    // remove the request from the buffer
    size_t remain = conn->rbuf_size - 4 - len;
    if (remain) {
        memmove(conn->rbuf, conn->rbuf + 4 + len, remain);
    }
    conn->rbuf_size = remain;

    // change state
    conn->state = STATE_RES;
    state_res(conn);

    return (conn->state == STATE_REQ);
}
```

`try_one_request` 函数一次从读缓冲区中读出一个请求，生成一个响应，并且把连接状态改为 `STATE_RES` 。

## 状态机：写者

`STATE_RES` 状态的代码：

```C++
static void state_res(Conn* conn) {
    while (try_flush_buffer(conn));
}

static bool try_flush_buffer(Conn* conn) {
    ssize_t rv = 0;
    do {
        size_t remain = conn->wbuf_size - conn->wbuf_sent;
        rv = write(conn->fd, conn->wbuf + conn->wbuf_sent, remain);
    } while (rv < 0 && errno == EINTR);
    if (rv < 0 && errno == EAGAIN) {
        return false;
    }
    if (rv < 0) {
        fprintf(stderr, "write error\n");
        conn->state = STATE_END;
        return false;
    }
    conn->wbuf_sent += (ssize_t)rv;
    assert(conn->wbuf_sent <= conn->wbuf_size);
    if (conn->wbuf_sent == conn->wbuf_size) {
        // response was fully sent, change state back
        conn->state = STATE_REQ;
        conn->wbuf_size = 0;
        conn->wbuf_sent = 0;
        return false;
    }
    return true;
}
```

上面的代码尝试发送数据来清理写缓冲区，一直到遇到 `EAGAIN` 为止，如果缓冲区的数据全部发送完成，就切换回 `STATE_REQ` 状态。

## 测试

为了测试我们的服务器，我们可以直接跑第 4 章的客户端程序，因为协议是一样的。但我们也可以修改客户端代码，来展示一个流水线客户端：

```C++
static int32_t send_req(int fd, const char* text) {
    uint32_t len = (uint32_t)strlen(text);
    if (len > k_max_msg) {
        fprintf(stderr, "Sending message too long.\n");
        return -1;
    }

    char wbuf[4 + k_max_msg] = {};
    memcpy(wbuf, &len, 4);
    memcpy(wbuf + 4, text, len);
    if (int32_t err = write_all(fd, wbuf, 4 + len)) {
        return err;
    }
    return 0;
}

static int32_t read_res(int fd) {
    // 4 bytes header
    uint32_t len = 0;
    char rbuf[4 + k_max_msg + 1] = {};
    errno = 0;
    int32_t err = read_full(fd, rbuf, 4);
    if (err) {
        if (errno == 0) {
            fprintf(stderr, "EOF\n");
        }
        else {
            fprintf(stderr, "read() error\n");
        }
        return err;
    }

    memcpy(&len, rbuf, 4);
    if (len > k_max_msg) {
        fprintf(stderr, "reading message too long: %d.\n", len);
        return -1;
    }

    // reply body
    err = read_full(fd, rbuf + 4, len);
    if (err) {
        fprintf(stderr, "read() error\n");
        return err;
    }

    // do something
    printf("Server says: %s\n", rbuf + 4);

    return 0;
}

int main() {
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


    // multiple pipelined requests
    const char* query_list[] = {"hello1", "hello2", "hello3"};
    for (size_t i = 0; i < 3; ++i) {
        int32_t err = send_req(fd, query_list[i]);
        if (err) {
            goto L_DONE;
        }
    }
    for (size_t i = 0; i < 3; ++i) {
        int32_t err = read_res(fd);
        if (err) {
            goto L_DONE;
        }
    }

L_DONE:
    close(fd);
    return 0;
}
```

练习：

1. 尝试在事件循环中使用 `epoll` 代替 `poll`，这应该很简单。
2. 我们使用了 `memmove` 回收读缓冲区空间。但对每个请求都做 `memmove` 是没有必要的。请修改代码，只在 `read` 前进行 `memmove`。
3. 在 `state_res` 函数中，我们对每一个响应都调用了 `write` 。在流水线的场景下，我们其实可以缓存多个响应然后调用 `write` 一次性发送，请注意中间可能会有写缓冲区满的情况。

本章源代码：

> - [06_client.cpp](https://build-your-own.org/redis/06/06_client.cpp.htm)
> - [06_server.cpp](https://build-your-own.org/redis/06/06_server.cpp.htm)

未完待续。