---
layout: post
title: "Url 的设计"
description: ""
category: 
comments: true
tags: [Thought, Design]
---

一年多前我写了一篇关于 REST 风格 API 的[文章](https://jiang.ma/programming/2017/03/06/confusing-restful.html)，文中吐槽了遵循 REST 的不方便之处，不过如果针对一个指定资源的操作，当时的我还是愿意采用 `GET/POST domain/object_id` 这种偏 RESTful 的形式来进行设计的。通过在 URL path 中指定要操作的资源，确实会直观很多。但是现在如果让我重新写一个服务，我会采用类似 `GET/POST domain/action?object_id={id}` 的形式来设计 url，不再纠结于 Restful 的风格。

公司主要服务的 url path 因为历史原因设计的非常复杂，在 path 中包含了用户的 app 版本、平台、分辨率、具体操作资源 id 等各种变量，恨不得通过一个 path 就包含所有参数，这样作有什么坏处呢？

首先对运维极端不友好。比如访问用户资源的 http 请求可能是这样 `GET user/:user_id` 不同用户请求的 path 肯定不一样。运维同学为了搭建服务报警和性能监控，就需要维护一个额外的规则 map 来对 url 进行归类，并且随着开发同学的迭代，这个 map 也需要跟着变化。如果 url path 简单，这些开发工作都是不必要的。

此外，对程序 URL parser 的性能影响也很大。公司原有的 PHP 项目使用 Yii，因为 url 较多并且 path 形式很复杂，Yii 的 url parser 会作很多计算，包括正则运算，在使用 php 7.1 和开启 opcache 的情况下，仅仅是 url 解析这一部分的平均耗时就在 5ms 以上，后来不得不专门用 Golang 写了一个 rpc 来作这一部分工作。在 gRPC 的 [FAQ](https://grpc.io/faq/) 也提到了因为性能的考虑，gRPC 放弃了 REST 形式的 API。

不过 RESTful API 通过不同的 http code 来区分 Response 我认为还是很值得借鉴的，通过 400+ 和 500+ 的状态码来区分业务报错和服务报错这点是对运维非常友好的。