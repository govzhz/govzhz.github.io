---
title: SELECT FOR UPDATE语句深度解析
date: 2019-06-18 22:19:04
toc: true
tags:
 - Mysql
 - 锁
categories: Mysql
---

&emsp;&emsp;`Mysql` 的 `SELECT ... FOR UPDATE` 语句是日常使用较多的用于锁定资源，确保在多个事务读取数据时始终能够读取到最新版本的数据的有效语句。那么它是怎么实现呢？在经过官网文档以及大量实践的验证之后发现网上存在大量不严谨甚至错误的信息，因此通过本文对 `SELECT FOR UPDATE` 语句作出以下总结。在具体介绍之前，先对目前网上教程或博客中会提到的几个**常见误区**进行纠正：

- ~~`SELECT FOR UPDATE` 在xx情况下会添加表级锁。~~ 

  请注意，**在任何情况下 `SELECT FOR UPDATE` 都不会添加表级锁。**事实上，在大部分情况下（DQL 语句，DML 语句，DDL 语句）都不会添加表锁，取而代之的是各种类型的行锁。

  {% blockquote %}
  {% raw %}<i class="far fa-bell"></i> {% endraw%}

  &emsp;&emsp;那么我们如何获取表锁呢？语句如下：

  {% codeblock lang:sql %}
  LOCK TABLES xx READ; # 为 xx 表添加表级 S 锁
  LOCK TABLES xx WRITE;  # 为 xx 表添加表级 X 锁
  {% endcodeblock %}

  然后我们可以通过以下语句来检测当前 Mysql 有哪些表获取了表级锁

  {% codeblock lang:sql %}
  SHOW OPEN TABLES WHERE In_use > 0
  {% endcodeblock %}

  更多的表级锁相关知识请参考[官网介绍](https://dev.mysql.com/doc/refman/8.0/en/lock-tables.html#table-lock-release)

  {% endblockquote %}

- ~~`SELECT FOR UPDATE` 在未使用索引时会"锁表"。~~

  `SELECT FOR UPDATE` 确实可以通过 `Next-key lock` 锁住所有记录和间隙来实现和表锁类似的效果。但未使用索引并非充分条件，我们判断 `SELECT FOR UPDATE` 是否锁住了所有数据和间隙还需要看它的隔离级别。

<!-- more -->

那么影响我们判断 `SELECT FOR UPDATE` 语句持有什么锁的因素有哪些呢？在这里列出以下几点：

- 隔离级别（RC/RR）
- 执行计划（聚簇索引/唯一索引/二级索引/无索引）
- 过滤条件（等值条件/范围条件）

**以下分析内容均建立在已经了解 Mysql 的行级锁的类型和作用范围的基础上，同时列出几点必要的前提论据：**

- 一般情况下，RC 级别是无法使用 `Gap Lock` 的，但在检查外键约束或者 duplicate key 检查时还是会用到的
  {% blockquote MySQL 8.0 Reference Manual https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html 15.7.1 InnoDB Locking %}
  Gap locking can be disabled explicitly. This occurs if you change the transaction isolation level to READ COMMITTED. Under these circumstances, gap locking is disabled for searches and index scans and is used only for foreign-key constraint checking and duplicate-key checking.
  {% endblockquote %}

- 一般情况下，执行计划根据某个索引查询后，会将过滤完的记录加锁后返回给 MySQL Server 进行过滤。在 RC 隔离级别下，当记录不满足条件时 MySQL Server 会调用 `handler::unlock_row()` 告诉存储引擎释放锁（破坏了 2PL 规则），RR 隔离级别下则会保持到事务提交

  {% blockquote %}
  {% raw %}<i class="far fa-bell"></i> {% endraw%}
  
  - 2PL（[两阶段加锁协议](https://en.wikipedia.org/wiki/Two-phase_locking)）是数据库中保证事务并发的控制方法，即保证多个事务在并发的情况下等同于串行的执行。它将加锁和解锁分为两个阶段。而为了在事务中能够明确的判断什么是加锁阶段，什么是解锁阶段，引入了 S2PL（Strict-2PL），即**在事务中只有提交（commit）或者回滚（rollback）时才是解锁阶段，其余时间为加锁阶段。**
  - ICP（索引条件下推）：是一种减少 server 层和 engine 层之间交互的次数的优化方式。上面提到一般情况下对于根据索引查询返回的记录将交由 MySQL Server 进行过滤，而如果过滤条件是联合索引且无法走联合索引时，如：
  
    {% codeblock lang:sql %}
    # 联合索引：(index1, index2, index3)
    # 根据最左匹配原则无法走联合索引
    select x from xx where index1 = 'xx' and index3 like '%xxxx%'
    {% endcodeblock %}
  
    正常情况下在对 index1 进行筛选后的记录就要返回。而经过 ICP 优化，由于 where 的查询列属于该联合索引，那么会将对该 where 条件记录过滤后才返回给 server 层
  
  参考：
  
  - [MySQL 加锁处理分析](http://hedengcheng.com/?p=771) 
  - [MySQL · 引擎特性 · InnoDB 事务锁系统简介](http://mysql.taobao.org/monthly/2016/01/01/) 
  - [MySQL 8.0 Reference Manual  8.2.1.5 Index Condition Pushdown Optimization](https://dev.mysql.com/doc/refman/8.0/en/index-condition-pushdown-optimization.html) 
  - [SELECT FOR UPDATE does not release locks of untouched rows in full table scans](https://bugs.mysql.com/bug.php?id=20390)
  
  {% endblockquote %}
  

## RC级别下的SELECT FOR UPDATE

虽然 Mysql 默认的事务隔离级别是 RR，但是在大多数互联网应用中 Mysql 的隔离级别会设置为 RC，因此我们也首先讨论 RC 隔离级别下的 `SELECT FOR UPDATE`。

- **在执行计划不走索引时，将只会为满足条件的记录添加 `Record Lock` **

  > 执行计划不走索引代表 sql 会走聚簇索引的全扫描，对所有记录加锁后返回给 MySQL Server 进行过滤。过滤过程中不满足条件的记录的锁会被释放，因此最终只锁住了满足条件的记录

- **在执行计划走聚簇索引时，将只为满足条件的记录添加 `Record Lock` **

- **在执行计划走唯一索引或二级索引时，将会为满足条件的记录所在的聚簇索引和二级索引添加 `Record Lock`  **

  > 为什么还需要在聚簇索引加锁呢？因为如果不锁聚簇索引意味着别的事务可以使用 `update/delete`，那么就失去了锁定资源的作用了

从上面的分析可以看出，在 RC 级别下任何情况下都不会出现"锁表"效果。但是**请注意即使 `SELECT FOR UPDATE` 的目标记录没有被锁住，也是有可能造成阻塞的。**原因在于 *Mysql 对非索引过滤（即是由 Mysql Server 过滤）的记录加锁返回的过程是不会省略的*，因此如果 `SELECT FOR UPDATE` 不走索引，那么 Mysql 会为聚簇索引的所有数据行尝试添加  `Record Lock` ，而一旦有任何一行已经被锁定，那么当前查询就会被阻塞。

## RR级别下的SELECT FOR UPDATE

Mysql 的 RR 级别为了解决幻读引入了 `Gap Lock`，这也为 `SELECT FOR UPDATE` 的加锁增加了很多可能性

- **在执行计划不走索引时，将会聚簇索引中的所有记录添加 `Next-key Lock`，相当于"锁表"**

  > RR 级别下非索引过滤的记录即使不符合过滤条件，锁也不会被释放。同时为了解决幻读，记录添加 `Next-key Lock` 来锁定间隙

- **在执行计划走聚簇索引时，若是能够命中的等值查询，将只为满足条件的记录添加 `Record Lock`；否则将覆盖范围包含过滤范围的记录添加 `Next-key Lock`**。

  > 为什么只有在等值查询是才有可能添加 `Record Lock` ？因为范围查询内的数据存在幻读问题

- **在执行计划走唯一索引时，锁住唯一索引的方式和聚簇索引相似，同时使用 `Record Lock` 锁住命中的聚簇索引**

  > 为什么只需要使用 `Record Lock` 锁住聚簇索引？因为通过唯一索引可以保证过滤范围间无法插入数据（与插入意向锁互斥），因此只需要 `Record Lock` 锁来确定目标记录不被 `update/delete` 即可

- **在执行计划走二级索引时，无论是否为等值查询都会为覆盖范围包含过滤范围的记录添加 `Next-key `，同时使用 `Record Lock` 锁住命中的聚簇索引**

  > 为什么二级索引不区分等值查询呢？因为即使是等值查询也不能唯一定位二级索引中的数据，在一棵二级索引的 B+ 树中，叶子结点由 二级索引列值 + 主键值 确定的，仅仅依靠二级索引列值还是相当于范围查询

## Serializable下的SELECT FOR UPDATE

Serializable 级别下 `SELECT FOR UPDATE` 的加锁方式基本和RR级别相同。比较特殊的是，Serializable 下是不存在快照读的，即使查询语句不添加 `for update` 也会为记录添加共享锁

## 锁分析工具

Mysql 提供了语句来查询当前持有锁的状态和类型等等，是验证我们的判断的利器。语句如下：

```sql
SELECT * FROM performance_schema.data_locks
```

它提供几个关键信息：

- LOCK_TYPE：锁类型，`RECORD` 代表行锁，`TABLE` 代表表锁
- LOCK_MODE：锁模式，`X,REC_NOT_GAP` 代表 `Record Lock` , `X, GAP` 代表 `Gap Lock` , `X` 代表 `Next-key Lock`
- INDEX_NAME：锁定索引的名称
- LOCK_DATA：与锁相关的数据，比如锁在主键上就是主键值

更多的字段解释参考 [MySQL 8.0 Reference Manual 26.12.12.1 The data_locks Table](https://dev.mysql.com/doc/refman/8.0/en/data-locks-table.html)

除此之外，Mysql 还提供了查询当前正在执行的每个事务（不包括只读事务）的信息，比如隔离级别，内存中此事务的锁结构占用的总大小等等。语句如下：

```sql
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX
```

它提供几个关键信息：

- TRX_ID：如果是非锁定的只读事务是没有该 id 的
- TRX_REQUESTED_LOCK_ID：当前事务正在等待的锁 id
- TRX_TABLES_LOCKED：当前 SQL 语句具有行锁定的表的数量
- TRX_LOCK_MEMORY_BYTES：内存中此事务的锁结构占用的总大小。
- TRX_ISOLATION_LEVEL：当前事务的隔离级别

更多的字段解释参考 [MySQL 8.0 Reference Manual 25.39.29 The INFORMATION_SCHEMA INNODB_TRX Table](https://dev.mysql.com/doc/refman/8.0/en/innodb-trx-table.html)

