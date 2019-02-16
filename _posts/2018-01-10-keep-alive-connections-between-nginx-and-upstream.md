---
layout: post
title: "Nginx proxy 和 upstream 之间的长连接"
description: "Nginx Proxy 和 upstream 之间的长连接"
category: "Programming" 
comments: true
tags: [Nginx Keep-Alive]
---

### Keep-Alive Connections
HTTP keep-alive 又叫 HTTP persistent connection，是指在一个 TCP 连接上进行多个 Http 请求。要实现 keep-alive 连接，在 HTTP 1.0 协议下，需要在 request 和 response 都设置 `Connection: keep-alive` 的 http header。在 HTTP 1.1 下，
连接默认都是 keep-alive 的，除非 Connection 的 http 头中显示地设置有其它值。
现代浏览器默认都是 keep-alive 连接，但是具体维护几个 tcp 连接，不同浏览器有不同的策略。

使用 keep-alive 连接的优势主要有以下几点：

1. 因为只使用有限的 TCP 连接，减少不必要的 TCP 握手，后续的 HTTP 响应时间会更快
2. 在使用 HTTPS 时，因为维护更少的连接，减少了 TLS 握手，CPU 负载也会减少
3. 减少了网络拥塞的可能

### Ngnix 使用 keep-alive 连接
我们先用 Golang 写一个简单的 server.go，并且运行它。

```
package main

import (
	"fmt"
	"net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello there!")
}

func main() {
	http.HandleFunc("/", handler)
	http.ListenAndServe(":8080", nil)
}
```

接着使用 Nginx 配个最简单的 proxy，笔者的 Nginx 版本是 1.12.2。

```
upstream api {
    server 127.0.0.1:8080;
}

server {
    listen       80;
    location / {
        proxy_pass http://api;
    }
}
```
然后用 curl 访问本地启动的 Nginx `curl http://127.0.0.1/ -v`，你会发现 curl 默认是使用 HTTP 1.1 协议的。同时使用 tcpdump 命令观察本地的 tcp 流量，` tcpdump  -i lo 'port 8080'`，参数`-i lo` 表示监听本地的 loopback 网关，它的地址就是 127.0.0.1。

{% include image.html url="/assets/images/20180110/1.jpg" description="Figure 1" %}

从 tcpdump 的结果可以看出 Nginx 转发请求默认使用的是 http 1.0，并且 Golang 程序在响应完请求后会主动发送 Fin 请求给 Nginx。如果我们使用 netstat 观察，就会看到一个状态是 timewait 的连接等待关闭。

如果想要 Nginx 使用 keep-alive 连接，最简单的配置如下。

```
upstream api {
    server 127.0.0.1:8080;
    keepalive 2;
}

server {
    listen       80;

    location / {
        proxy_pass http://api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```
重启 Nginx 后，使用 ab 发送批量的请求，`ab -n 10 -c 2 http://127.0.0.1/` 同时使用 netstat 进行观察 `watch "netstat -antp | grep 8080"`

{% include image.html url="/assets/images/20180110/2.jpg" description="Figure 2" %}

从图上能看出，Nginx 维护的连接状态都是 ESTABLISHED，不会看到 TIME-WAIT 状态的连接。

其中 location 中 `proxy_http_version 1.1; ` 参数会设置 Nginx proxy 和 upstream 之间使用 HTTP 1.1 协议，像上文一样通过 tcpdump 可以观察到。`proxy_set_header Connection "";` 参数是为了移除 connection header，防止 client 设置 connection 的值为 close，这样就会关闭一个 keep-alive 连接。

另外 upstream 的 keepalive 参数设置的是 Nginx 维护的空闲 keepalive 连接的最大数量。需要注意的是，该参数是针对每个 worker 进程的，并且不是限制 Nginx 和 upstream 之间能够打开的连接数。不过该值也不是越大越好，如果 Nginx 维护的空闲 keepalive 连接数过多，会影响能够同时打开的连接数量。

如果并发的请求超过 keep-alive 配置，Nginx 的行为是什么呢？笔者设置的 keep-alive 连接数是 2 个，同时发 5 个请求的话，`ab -n 10 -c 5 http://127.0.0.1/`，使用 netstat 观察，我们会发现 Nginx 会额外建立新的连接，但是不会维护 keep-alive 状态。从下图我们能看到，新建的请求还是会处于 TIME-WAIT 状态。这里有个细节很有意思，我们知道 TIME-WAIT 是先发起 FIN 请求的一方是才会有的状态，在使用 HTTP 1.1 协议的时候，是 Nginx 主动关闭连接，这个和上文在 HTTP 1.0 协议下由 Golang server 主动关闭不同。

{% include image.html url="/assets/images/20180110/3.jpg" description="Figure 3" %}


### 其它
我们在网上经常会看到内核参数 net.ipv4.tcp_tw_recycle 和 net.ipv4.tcp_tw_reuse 的设置，这两个参数和 HTTP 协议层 keep-alive 连接完全不是一回事，它们都是为了解决 tcp 连接在关闭后会处于 TIME-WAIT 状态而导致 socket 不能被尽快复用的问题。其中 `net.ipv4.tcp_tw_recycle` 配置开启可能会有[问题](http://www.pagefault.info/?p=416)，在 Linux 4.12 已经被移除了，这里主要说下 `net.ipv4.tcp_tw_reuse`  
首先为什么会有 TIME-WAIT 状态呢，主要有两个原因：  
1. 为了防止接收到延迟的数据包  
2. 确保对方也关闭 TPC 连接。主动发起关闭方发送最后一个 ACK 包后处于 TIME-WAIT 状态，如果这个 ACK 包丢失了，被动关闭方由于没有收到最后的 ACK，就知道它发送的 FIN 包或者对方最后的 ACK 丢失，然后会重发 FIN 请求，这样处于 TIME-WAIT 的一方可以重新发送 ACK 包。

RFC 1323 补充了 TCP 规范，它增加了两个字节用于记录 TCP 发送方和接收方最新收到包的时间戳。在开启 ipv4.tcp_tw_reuse 后，如果新的请求时间严格大于当前存在的 TIME-WAIT 状态的连接，内核会选择一个 socket 进行复用。由于记录了时间，针对上面第一个问题，收到过期的包就可以直接抛弃了。至于第二个，如果一个处于 TIME-WAIT 的 socket 被复用后，一旦再收到前面连接另一方的 Fin 包，就会返回一个 RST 包，这也会让另一方跳过 LAST-ACK 状态，完成连接的关闭。

上面的叙述不够直观，这篇[博客](https://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux)写的非常详细，强烈建议阅读。
