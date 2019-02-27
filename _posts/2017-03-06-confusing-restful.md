---
layout: post
title: "让人纠结的 REST"
description: ""
category: "Programming" 
comments: true
tags: [Web Development, REST]
---
REST(Representational state transfer) 是一种 API 设计风格，按照笔者的理解，REST 的设计主要包括但不限于下面几点： 

* 服务是面向资源的，通过 URI 来定位资源，比如 https://api.example.com/v1/user/:id 指定了一个 user 资源
* 严格使用 HTTP Method 来标识对资源的操作方式，比如新增或者更新指定的资源，使用 PUT 方法，删除资源使用 DELETE 方法
* 资源之间的关系通过返回结果的 [Link header](https://tools.ietf.org/html/rfc5988) 来描述。比如在分页场景中，请求了某一页的数据，它的上一页和下一页对应的  URI 都记录在 Response 的 Link header 里   
* 通过 HTTP Status Code 标识返回结果的含义

越来越多的平台遵循 REST 来提供 API，[GitHub](https://developer.github.com/v3) 就是很好的例子。最近几年由于前段框架的兴起，前后端的交互基本上都可以抽象成针对资源的请求，这也促进了 REST 设计的流行。这篇文章不是介绍该如何设计 REST 风格的 API，而是想聊一聊在平时的 WEB 开发中，基于 REST 进行设计的蛋疼之处。

## HTML 表单不支持 PUT 和 DELETE
如果基于 REST 风格进行设计，使用 PUT 和 DELETE 方法来操作资源是不可避免的。但是在  HTML5 中，form 元素的 method 属性只支持 GET 和 POST，更不要说 PATCH 方法了(早期的 HTML5 草案是支持 PUT 和 DELETE 方法的，可惜后来被废弃了，参见 W3C 的[文档](https://www.w3.org/TR/2010/WD-html5-diff-20101019/#changes-2010-06-24))。所以如果想使用 PUT method，就不得不放弃 form 元素，自己构造 XMLHttpRequest。  

另外很多 Web 框架一般提供下面这种方式，在 form 表单里添加一个隐藏的 input 元素来告诉服务器使用了 PUT 或者 DELETE 方法，真的很无奈。  
```
<input type="hidden" name="_method" value="PUT">
```

## WEB 框架对 GET 请求不提供 CSRF 保护
[CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery) 是一种危害很大的攻击方法，攻击者通过伪装成用户身份完成攻击操作，Web 开发中对于敏感操作一定要开启对 CSRF 攻击的防护。

然而 WEB 框架一般对 GET 请求不提供 CSRF 保护，比如 [Django](https://github.com/django/django/blob/86de930f413e0ad902e11d78ac988e6743202ea6/django/middleware/csrf.py#L212) 和 [Rails](https://github.com/rails/rails/blob/bc4781583d9db237dd928b01743c6db7596a36b3/actionpack/lib/action_controller/metal/request_forgery_protection.rb#L269)。在平时的开发过程中，难免会有一些敏感数据，我们并不希望泄漏。因为是读数据，RESTful 的 api 应该使用 GET 请求，但是这就会有被 CSRF 攻击的风险，笔者在这种情况下，不得不改为使用 POST 请求获取数据。

## 有时候遵守 REST 的设计会提高复杂度
我们假设有下面的需求，一个 WIKI 系统中，每一篇文章可以被用户编辑，用户的第 n 次编辑会得到 n 个积分。  

现在需要提供一个更新指定文章的 API。因为已经知道了具体的资源，我们应该使用 PUT 请求，请求可能长这样 `curl -X PUT https://example.com/post/:id -d 'content'`。但是 PUT 请求应该是幂等的，也就是重复该请求应该有相同效果，而每次编辑得到的积分却不一样。这时候我们不得不拆分 API，在发完更新文章的请求后，再进行改变积分的操作。

在上文已经提到的分页场景中，按照严格的 REST 设计，应该先请求第一页信息，其它的分页信息通过结果的 Link header 中获取，这无疑也比通过后端框架的模板引擎直接渲染出来要麻烦很多。

此外一个 REST 风格的 API 系统应该有一个 ROOT URI，描述了其可以提供的资源。如果 client 按照从 ROOT URI 开始逐步获取资源之间的关系，即使资源名称进行改变，client 也不会受到影响。  
面向第三方的 API 平台采用这种方式无可厚非，但是中小项目的 Web 开发中，client 和 server 都是自己控制的，实现这种完全的解耦无疑会大幅增加开发成本。

## 总结
笔者是一个彻底的实用主义者，认为技术方案的选择应该基于现有的需求场景和开发条件，着重考虑开发成本和可维护性。  

但是即使不完全严格遵守 REST 风格，其中也是有很多东西可以借鉴的。比如能够针对指定资源的操作 URI 可以按照 REST 风格进行设计，获取数据的操作都使用 GET 方法等，关键是在优雅和成本之间找到一个适合自己的平衡点。
