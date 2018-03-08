---
layout: post
title: "一些 MySQL 知识和技巧"
description: "一些 MySQL 知识和技巧"
category: "Prgoramming"
comments: true
tags: [MySQL]
---

最近读了《High performance MySQL》，下面是对一些新知识的总结和摘录。

### Covering Index
如果一条 query 用到的索引能覆盖到所有要查询的字段，那么就称其为 Covering Index，尤其在 InnoDB 中，如果使用的索引不能覆盖到全部字段时候，不得不通过查询主键，通过 Cluster Index 再获取所有的字段值，这就多了几次 IO 查询（MySQL B+Tree  一般两到三层）。所以如果一个查询需要 A，B，C 三个字段，如果 C 字段取值范围有限，有的时候考虑到索引维护的开销，可能建立只包含 A，B 字段的联合索引。但是为了使用到 Covering Index，建立一个包含 A，B，C 三个字段的联合索引可能更有优势，不过损失的就是调整索引的性能。

### 获取 InnoDB 表的行数
不同于 MyISAM 表，InnoDB 并不维护每个表行数的真实值。如果使用 `SELECT COUNT(*)` 会扫描全表，数据量大的时候执行时间会很久。如果只需要知道数据数量级的话，可以采用下面的方法。执行`EXPLAIN SELECT * FROM table;` 

```
MariaDB [test]> explain SELECT * from foo;
+------+-------------+-------+------+---------------+------+---------+------+------+-------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
+------+-------------+-------+------+---------------+------+---------+------+------+-------+
|    1 | SIMPLE      | foo   | ALL  | NULL          | NULL | NULL    | NULL |  603 |       |
```
或者执行 `SHOW TABLE STATUS LIKE table`

```
MariaDB [test]> SHOW TABLE STATUS LIKE 'foo' \G;
*************************** 1. row ***************************
           Name: foo
         Engine: InnoDB
        Version: 10
     Row_format: Compact
           Rows: 603
 Avg_row_length: 2635
    Data_length: 1589248
Max_data_length: 0
   Index_length: 0
      Data_free: 141557760
 Auto_increment: NULL
    Create_time: 2018-03-08 21:50:45
    Update_time: NULL
     Check_time: NULL
      Collation: latin1_swedish_ci
       Checksum: NULL
 Create_options:
        Comment:
```
上述两个结果中的 rows 字段即为 InnoDB 给出对行数的估计值，和实际值可能差 40% 至 50%，不过对于了解表的数据量级已经足够。这些信息都维护在 MySQL 的 INFORMATION_SCHEMA.TABLES 表中，当然我们也可以执行 
`SELECT *  FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'foo' \G;` 得到类似的结果。

### 使用 Prepare 的好处
1. 安全，因为有过预解析过程，不会有 XSS 的风险
2. 提高执行效率，MySQL 服务器只需要 parse sql 语句一次，然后缓存执行计划
3. 参数的传递是通过二进制协议，比以 ASCII 方式传递更高效。例如一个 DATE 字段只需要 3 个字节传递，通过 ASCII 传递则需要 10 个字段。同时对与 Text 等字段类型可以分 chunks 传输，节约了网络带宽，也减小了客户端的内存压力，服务器也避免了从文本向二进制转换的过程。
4. 只需要传递参数，不需要传整个 query
5. MySQL 直接将参数存储至 buffer，会帮助服务器减少内存之间的拷贝

### 使用 Profiling 进行调试
可以通过下面的方法详细分析一条语句的执行细节

```
SET profiling = 1;
some sql
MariaDB [test]> SHOW PROFILES;
+----------+------------+----------------------------+
| Query_ID | Duration   | Query                      |
+----------+------------+----------------------------+
|        1 | 0.00029462 | SELECT * FROM foo LIMIT 10 |
+----------+------------+----------------------------+
1 row in set (0.00 sec)

MariaDB [test]> SHOW PROFILE FOR QUERY 1;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000067 |
| checking permissions | 0.000008 |
| Opening tables       | 0.000011 |
| After opening tables | 0.000005 |
| System lock          | 0.000003 |
| Table lock           | 0.000002 |
| After table lock     | 0.000004 |
| init                 | 0.000024 |
| optimizing           | 0.000008 |
| statistics           | 0.000013 |
| preparing            | 0.000008 |
| executing            | 0.000002 |
| Sending data         | 0.000107 |
| end                  | 0.000005 |
| query end            | 0.000004 |
| closing tables       | 0.000006 |
| freeing items        | 0.000004 |
| updating status      | 0.000013 |
| cleaning up          | 0.000002 |
+----------------------+----------+
19 rows in set (0.00 sec)

```

下面语句可读性更好

```
SET @query_id = 1;
SELECT STATE, SUM(DURATION) AS Total_R,
       ROUND(
          100 * SUM(DURATION) /
             (SELECT SUM(DURATION)
              FROM INFORMATION_SCHEMA.PROFILING
              WHERE QUERY_ID = @query_id
          ), 2) AS Pct_R,
       COUNT(*) AS Calls,
       SUM(DURATION) / COUNT(*) AS "R/Call"
    FROM INFORMATION_SCHEMA.PROFILING
    WHERE QUERY_ID = @query_id
    GROUP BY STATE
    ORDER BY Total_R DESC;

+----------------------+----------+-------+-------+--------------+
| STATE                | Total_R  | Pct_R | Calls | R/Call       |
+----------------------+----------+-------+-------+--------------+
| Sending data         | 0.000107 | 36.15 |     1 | 0.0001070000 |
| starting             | 0.000067 | 22.64 |     1 | 0.0000670000 |
| init                 | 0.000024 |  8.11 |     1 | 0.0000240000 |
| statistics           | 0.000013 |  4.39 |     1 | 0.0000130000 |
| updating status      | 0.000013 |  4.39 |     1 | 0.0000130000 |
| Opening tables       | 0.000011 |  3.72 |     1 | 0.0000110000 |
| checking permissions | 0.000008 |  2.70 |     1 | 0.0000080000 |
| optimizing           | 0.000008 |  2.70 |     1 | 0.0000080000 |
| preparing            | 0.000008 |  2.70 |     1 | 0.0000080000 |
| closing tables       | 0.000006 |  2.03 |     1 | 0.0000060000 |
| After opening tables | 0.000005 |  1.69 |     1 | 0.0000050000 |
| end                  | 0.000005 |  1.69 |     1 | 0.0000050000 |
| After table lock     | 0.000004 |  1.35 |     1 | 0.0000040000 |
| query end            | 0.000004 |  1.35 |     1 | 0.0000040000 |
| freeing items        | 0.000004 |  1.35 |     1 | 0.0000040000 |
| System lock          | 0.000003 |  1.01 |     1 | 0.0000030000 |
| executing            | 0.000002 |  0.68 |     1 | 0.0000020000 |
| Table lock           | 0.000002 |  0.68 |     1 | 0.0000020000 |
| cleaning up          | 0.000002 |  0.68 |     1 | 0.0000020000 |
+----------------------+----------+-------+-------+--------------+
```

### 其它
一条语句统计某个字段不同值的数量，比如 items 表的 color 字段存在 blue 和 red 两种值，则执行下面的语句即可

```
SELECT SUM(IF(color = 'blue', 1, 0)) AS blue,SUM(IF(color = 'red', 1, 0)) AS red FROM items;
```

拷贝表的小技巧，尤其适合在操邹原始表前建立备份表
拷贝原始表的结构以及数据到新表

```
CREATE TABLE newtable LIKE oldtable;
INSERT newtable SELECT * FROM oldtable;
```
只拷贝表字段（不包括索引）和数据。

```
CREATE TABLE newtable AS SELECT * FROM oldtable;
```
使用 Join 表确实可能很慢，甚至比全表扫描还慢，如果两个大表 Join，即使 on 的字段是 index，也可能有很多的随机硬盘 IO。



