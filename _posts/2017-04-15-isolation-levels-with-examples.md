---
layout: post
title: "InnoDB Engine Isolation Levels with SQL Examples"
description: ""
category: "Programming" 
comments: true
tags: [MySQL,InnoDB,Isolation Level]
---

## Isolation Level
ACID (Atomicity, Consistency, Isolation, and Durability) is the set of properties describing the major guarantees of the transaction paradigm. <a href="https://en.wikipedia.org/wiki/Isolation_(database_systems)">Isolation</a> means every transaction is protected from the others while they are in progress, they cannot interfere with each other or see each other's uncommitted data. Strict isolation is hard to be achieved or at least with a high cost. Database uses different locking mechanism to guarantee different degree of isolation, which is called Isolation Level, to trade off between data protection and concurrency performance.

I always like to learn knowledge through code examples, which could help me have a better understand. In this post, I will construct several examples to show issues (such as dirty reads) users might encounter at each isolation level.

## Check and Set Isolation Level
InnoDB supports four kinds of the transaction isolation levels, ranging from the lowest to the highest: Read Uncommitted, Read Committed, Repeatable Read and Serializable. The default isolation level for InnoDB is Repeatable Read.  
To check the MySQL InnodB transaction level, we could run the following query:

```
mysql> SELECT @@GLOBAL.tx_isolation, @@tx_isolation;
+-----------------------+-----------------+
| @@GLOBAL.tx_isolation | @@tx_isolation  |
+-----------------------+-----------------+
| REPEATABLE-READ       | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.00 sec)
```
It shows the current applied global and session transaction isolation levels.  
We could also set the global and session transaction isolation levels at runtime: 

```
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

## Preparation
To simulate a concurrent environment, I use two MySQL (the version of MySQL using is latest 5.7) sessions, with each session running a separate transaction. In the following texts，symbol T1 represents the left session transaction and T2 represents the right.

Let's create the table and insert initial data first.

```
mysql> CREATE TABLE `user` 
(`id` int, `name` varchar(16), `balance` int)
ENGINE=InnoDB;
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO `user` VALUES (1, 'Alice', 100);
Query OK, 1 row affected (0.02 sec)
```

## Read Uncommitted
This is the lowest isolation level. Under this level, one transaction could see modifications which is 'dirty reads' made by other transactions that are not committed yet.
{% include image.html url="/assets/images/20170415/1.1.png" description="Figure 1.1" %}
As we can see in Figure 1.1, during T1, Alice's balance changes from 100 to 200 because of mutation made by T2 which even not committed.

## Read Commited
Uncommitted dirty read is avoided under this level. To illustrate, let's run the above SQLs under Read Committed again.
{% include image.html url="/assets/images/20170415/2.1.png" description="Figure 2.1" %}
In Figure 2.1, T1 can't see Alice's balance changed until T2 committed. To be noticed, during T1 transaction, the data of Alice is retrieved twice, but the values is different, which is the 'Non-repeatable reads' phenomenon.

Besides Non-repeatable reads, there is another problem called 'phantom row'. As Figure 2.2 showing, T1 selects data with a range search, and T2 inserts a new row concurrently, the same query produces different results in T1.
{% include image.html url="/assets/images/20170415/2.2.png" description="Figure 2.2" %}

## Repeatable Read
As the default isolation level for InnoDB, Repeatable Read prevents non-repeatable reads. Figure 3.1 demonstrates that for the duration of T1, the Alice's balance is always 100.
{% include image.html url="/assets/images/20170415/3.1.png" description="Figure 3.1" %}
In Repeatable Read, a 'snapshot' is created at the start of the transaction, which guarantees that any data read wouldn't change, if the transaction selects the same data again.

To be noted, in Repeatable Read isolation level, InnoDB uses next-key locks for searches and index scans, which prevents phantom row problem. In Figure 3.2, T1 sees the same result with range search query.
{% include image.html url="/assets/images/20170415/3.2.png" description="Figure 3.2" %}

## Serializable
This level is like Repeatable Read but more strictly. Under Serializable it will lock all records that a transaction selects. In Figure 4.1, T2 hangs when trying to insert new data, has to wait until T1 commits. 
{% include image.html url="/assets/images/20170415/4.1.png" description="Figure 4.1" %}

## Conclusion

The higher the isolation level, the more concurrency performance database will loss, just choose the appropriate isolation level for your application. But to be honest, the default Repeatable Read level of InnoDB is good enough for most situations.

                                                                                                 
## References
[https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)

