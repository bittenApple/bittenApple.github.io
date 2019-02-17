---
layout: post
title: "Golang 在 IO 密集场景下的优势"
description: ""
category: 
comments: true
tags: [Golang, socket programming, epoll]
---
过去一年公司经历了服务端技术栈从 PHP 向 Golang 的转型，前不久笔者作了一次内部分享。本文是这次分享的记录，内容偏科普，主要包括以下几点

* 介绍 Linux Socket 编程和 IO 多路复用(IO Multiplexing)
* Go 调度器和对网络 IO 的封装

目的是让同学们了解 Golang 较之于 PHP、Python 这些脚本语言在面对现代开发场景中语言设计上的优势。

### Socket 编程示例
想要完全弄清楚这个问题，先要从 socket 编程讲起。下图是一个监听 9000 端口 tcp 服务示例，使用 C 语言编写。
{% include image.html url="/assets/images/20190128/block_socket.png"  description="Figure 1" %}

首先调用函数`int socket(domain, type, protocol)`创建一个代表 socket 的 fd（file description），返回值是一个 int 类型。这里额外多说几句，对 Linux 来讲，有一句话是 "Everything is a file"，具体含义是指包括进程、文件、目录、管道和 socket 等类型都可以通过 fd 来标识。我们常用的标准输入 0、标准输出 1、标准错误 2 都是最常见的 fd。对于每个 fd 代表的文件，其实就是 bytes 流，我们都可以对其进行 read 和 write，这是典型的面向接口编程。 
 
接着通过`bind()`和`listen()`函数将 socket 绑定到指定的地址和端口并进行监听。 `accept()` 函数会返回一个 fd 代表新建的 socket，通过这个 socket，我们就可以使用 `recv()` 函数读取数据了。

上述介绍的函数都是阻塞的系统调用（不了解系统调用的同学可以看这篇[文章](https://minnie.tuhs.org/CompArch/Lectures/week05.html)）。所谓的阻塞是指，当这个系统调用发生时，执行过程就交给了内核，如果内核没有返回，调用的进程就必须等待。比如上面的 recv() 函数，直到读取到 client 传过来的值，系统调用才会返回。

上面的示例代码同时只能处理一个客户端的连接，php-fpm 模式下处理 IO 的方式，背后的系统调用就和上面的一样，所以一个 php-fpm 进程一次只能处理一个请求。

### IO 多路复用(IO Multiplexing)
提到高并发，不能不提这篇写于 2000 年左右的[文章](http://www.kegel.com/c10k.html)，提出了经典的 C10K 问题。文中介绍了几种处理并发 IO 的方式，其中也提到了 IO 多路复用，包括 epoll（当时epoll 的实现还没有合进 Linux 内核）。我们现在实现一个 qps 上万的服务很方便，Nginx、Golang、Node.js 都可以很容易做到，他们背后都是使用了 epoll，kqueue 等系统调用（kqueue 是 BSD 系统的 IO Multiplexing 实现）。

这里是一个使用 epoll 创建 tcp 服务的 C 代码[示例](https://gist.github.com/bittenApple/e3675019ce8d6126e81492a0e57b3fb4)。我们需要关注这几个系统调用， `epoll_create()`，`epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event)`，`epoll_wait(epfd, &event, 1, timeout_milliseconds)`。简单来说，通过这些系统调用，可以实现在一个线程内非阻塞地对多个 fd 进行读写管理。

### Go Tcp Server
上面的 C epoll 实现里，在 for 循环中不停调用 epoll_wait，当监听的 I/O 事件发生时，调用方就可以处理内核返回的数据了，整个实现过程是异步的。  
下图是用 Golang 实现的 tcp server 示例，可以看到同上面的 C 代码实现相比简单了许多。并且在新建一个 goroutine 后，我们就能同步地编写自己的逻辑代码，那么 Golang 是如何封装实现这种效果的呢？
{% include image.html url="/assets/images/20190128/go_server.png"  description="Figure 2" %}

### Go 调度器
这里先简单介绍下 Golang 调度器，就是大家会接触到的 G-P-M 模型。其中 P 是 Golang 的逻辑处理线程；M 代表着系统线程；G 代表着 goroutine，是 Golang 的协程实现，可以理解为应用层的线程。和线程由操作系统调度一样，goroutine 是由 Golang 的调度器进行调度。

相较于多线程编程，Golang 由于作了 G-P-M 的封装抽象，在应用层可以实现任务的调度，避免了线程调度的 context-switch，cpu cache-line miss 等问题，将 io-bound 问题一定程度上转化为 cpu-bound 问题，所以适合现代的开发场景。

下面通过图片抽象展示发生非阻塞系统调用时，Go 的调度变化。下图是某个时刻的 Go 调度器的状态，G1-4 分配在 P1 上，其中 G1 正在 M1 上运行，还有三个 goroutine 正在 LRQ(Local Runnable Queue) 等待。

{% include image.html url="/assets/images/20190128/schedule1.png"  description="Figure 3" %}

当 G1 发生非阻塞系统调用时，比如网络系统调用，G1 被移动到 Net Poller，这时候 M1 可以执行其他等待的 goroutine，G2 被切换到 M1 上运行，如 Figure 4 所示，具体实现可参考[源码](https://github.com/golang/go/blob/go1.11.5/src/runtime/netpoll.go#L366)

{% include image.html url="/assets/images/20190128/schedule2.png"  description="Figure 4" %}

当异步的系统调用完成后，G1 被移动到 GRQ(Global Runnable Queue)，等待被调度器分配到其它可运行的 P 上，如 Figure 5 所示。图中的 Net Poller 可以理解为一个不停执行 poll 操作的 for 循环，实现上是由`runtime/proc.go`中的 sysmon 函数完成的，可参考[源码](https://github.com/golang/go/blob/go1.11.5/src/runtime/proc.go#L4390-L4401)

{% include image.html url="/assets/images/20190128/schedule3.png"  description="Figure 5" %}

### Summary
这几年 Golang 在当前 Web 服务端领域越发流行，笔者认为有下面几个原因：

- 我们现在面对的大部分场景都是 IO 密集型的，读写数据库，读写缓存，读写消息队列。尤其是这些年来服务端微服务模式的流行，让 IO 密集的趋势更加明显
- 操作系统提供的以 epoll 为代表的系统调用功能强大
- Golang Netpoll 模块对 epoll 等系统调用的封装，此外 Golang 调度器封装了 IO 等待的异步逻辑，使开发者能以同步的方式编写代码。

不过需要了解的是，Golang 的协程实现和基于 [CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes) 模型的 channel 通信才是 Golang 在并发编程领域的最大亮点，本文并没有涉及。这篇文章介绍了 Golang 处理网络 IO 的优势，但是覆盖的概念依然不少，很多内容都是浮光掠影，并没有深入展开，想要完全弄清楚还是需要跟踪代码实现。

