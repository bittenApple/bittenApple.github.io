---
layout: post
title: "Gh-ost 使用小感"
description: "Gh-ost 使用小感"
category: "Programming"
comments: true
tags: [MySQL, DDL, gh-ost]
---
[Gh-ost](https://github.com/github/gh-ost) 是 GitHub 开源的 MySQL online DDL 工具。虽然从一[开源](https://githubengineering.com/gh-ost-github-s-online-migration-tool-for-mysql/)便有所关注，但是最近才在生产环境使用该工具。

## 常见的 DDL 方式
在业务快速迭代的开发场景中，频繁的数据库表结构变更是不可避免的事情，比如增加索引和字段。如果使用 MySQL 的 alter 语句，MySQL 首先会生成原始表的拷贝，在该临时表上进行 DDL 更新，完成后，删除原始表，rename 临时表。操作临时表期间 DDL session 会对整张表加上读锁，阻塞其它 session 的写操作；在最后的 rename 阶段，会对表加上写锁，阻塞其它所有操作。如果表很小，大小只有几十 MB，阻塞时间可以忽略；但若表数据很大，阻塞时间尝尝是应用无法忍受的。尤其在最后的 rename 阶段，涉及到删除原始表的操作，如果表很大，删除时间比较长，整张表的读写操作都是被阻塞的。  

MySQL 5.6 开始引入了 Online DDL 支持，但是在 alter 过程的开始和结束还是会 block 所有的读写操作，并且不能覆盖所有的 DDL 种类。此外操作过程中不能控制速率，可能会引起较大的读写延迟；如果中途中断的话，也会有较长时间的回滚操作。另外在主从环境中，MySQL Online DDL 必须操作完 mater 后才能开始对 slave 的 DDL。

所以在生产环境中经常会引入第三方的 Online Schema Migration 工具。比较出名的是 Percona 的 [pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/2.2/pt-online-schema-change.html) 和 Facebook 的 [Facebook OSC](https://www.facebook.com/notes/mysql-at-facebook/online-schema-change-for-mysql/430801045932/)。他们都是基于数据库的 trigger 来实现 online schema migration。以 pt-online-schema-change 为例，DDL 过程包含以下步骤  

* 创建一个与原表结构相同的临时表，在临时表上执行 DDL 操作
* 创建三个 triggers，分别将原表的 UPDATE、INSERT 和 DELETE 操作同步到临时表上
* 将原表的数据根据主键顺序分批次 copy 至临时表
* 用临时表替换原表，其中包括删除 trigger 的操作。

## gh-ost 的优势
基于 trigger 的 migration 工具简单粗暴，但是有以下几个缺点： 

* DDL 期间，每个针对原表的更新操作都会因为要执行 trigger 而增加额外的写操作，所以原表的更新操作都会受影响
* 基于 trigger 的写操作也会参与临时表的 lock 竞争，比如获取 auto_increment 的值
* 不能方便的暂停，因为基于 trigger 的方式，暂停就意味着数据丢失
* Trigger 只能在 master 机器上建立，不能方便的在 slave 上进行测试

gh-ost 的一大卖点就是 triggerless，它通过监听主从同步的 binlog 实现即时变更数据的同步。在默认模式下，gh-ost 通过下面步骤完成 online DDL。

* gh-ost 连接 MySQL server（可以是主，也可以是从），获取数据库的主从拓扑信息和作必要的验证
* 建立 ghost table（也就是 DDL 后的原表）和统计表（用来统计 DDL 状态）
* 从原表同步数据至 ghost table
* 通过监听 binlog，同步变更至 ghost table
* 完成同步后，进行表切换

相较于它方式，gh-ost 只需要拷贝原表数据和重播 binlog，原表的读写不受影响。整个migration 过程可以随时暂停（MySQL online DDL 和基于 trigger 的工具都是不支持的），因为只需要记录 binlong 的执行位置即可。gh-ost 还提供充分的流量监控和控制方式，并且支持在 slave 上进行测试。  

不过 gh-ost 要求主从同步方式必须是 Row Based Replication，不支持 Mixed  和 SBR 方法，要知道 SBR 是一种效率不算高的同步方式。如果主从的 binlog 已经是 SBR 的，它会要求一台 slave 开启 RBR 的 binlog。

gh-ost migration 最后的 cut-over 阶段也是一个亮点，它的表切换是原子的。实现方式有点 tricky，基于 MySQL 内部实现，rename 的 block 优先级是高于其它 block 操作的，有兴趣可以浏览最后一个 reference 链接。

在生产环境使用 gh-ost 非常方便，并且支持 AWS 的 RDS。如果执行过程意外中断也无妨，删除掉 ghost table 和统计表后，重新开始即可。

## References
[https://www.percona.com/blog/2014/11/18/avoiding-mysql-alter-table-downtime/](https://www.percona.com/blog/2014/11/18/avoiding-mysql-alter-table-downtime/)
[https://www.percona.com/blog/2012/11/01/knowing-what-pt-online-schema-change-will-do/](https://www.percona.com/blog/2012/11/01/knowing-what-pt-online-schema-change-will-do/)
[https://github.com/github/gh-ost/blob/master/doc/why-triggerless.md](https://github.com/github/gh-ost/blob/master/doc/why-triggerless.md)    
[https://github.com/github/gh-ost/issues/82](https://github.com/github/gh-ost/issues/82)
