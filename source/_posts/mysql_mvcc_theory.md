---
title: MySQL之多版本并发控制
date: 2021-07-20 23:58:00
tags: MySQL
categories: 数据库
comments: true
typora-root-url: ../
---

> MVCC，全称是Multi-Version Concurrency Control，就是多版本并发控制。MVCC是一种并发控制方法，一般在数据库中做数据库的并发访问限制，实现编程过程中的事务实现。

MVCC在MySQL中主要是为了提升数据库的并发访问能力，让MySQL提供更好的读-写能力，解决读-写冲突，做到不加锁也能非阻塞并发读。

#### 当前读和快照读

在了解MVCC之前，我们需要了解一下什么是MySQL InnoDB的`当前读`和`快照读`。

##### 当前读

在InnoDB中存在很多当前读，比如 `selectlock in share mode(共享锁)`,`select for update(排他锁)`，`update/insert/delete（排它锁）`都是是一种当前读。之所以称之为`当前读`，是因为读取的是该记录的**最新版本**，当`当前读`开始时，将会保证其他事务不能修改当前记录，会对当前记录进行加锁。

##### 快照读

在InnoDB中，我们称**不加锁**的select操作就是快照读，也就是不加锁的读取操作。但是这里有一个前提是隔离级别不能是串行级别的，串行级别下的快照读会退化成当前读。InnoDB使用快照读是为了提升MySQL的并发性能，快照读基于MVCC多版本控制避免加锁的操作，在读取数据时并不一定是该记录的**最新版本**，可能是该记录的**历史版本**。

所以，MVCC为了实现读写不冲突、提升性能，而设计的是读不加锁（**快照读**），和需要加锁以获得最新版本数据的（**当前读**）。

#### MVCC的好处

数据库并发场景有：

- 读 - 读：不存在问题，天然幂等，不需要并发控制
- 读 - 写：有线程安全问题，可能会造成事务隔离性问题，可能造成脏读、幻读、不可重复读
- 写 - 写：有线程安全问题，可能会存在数据丢失。

针对上面的三种并发场景，MVCC可以很好的解决**读 - 写**的无锁并发控制，具体一点的做法是InnoDB在为某行数据做事务操作时，为其分配一个单向递增的时间戳（系统版本号）作为数据的最新版本号，并关联到当前的事务中。此时另外一个线程过来查询，快照读读到的数据是第一个事务开始前的数据，当前读读到的是第一个事务修改后的数据。

- 在并发读写数据库时，可以做到读操作时不需要阻塞写，写操作也不会阻塞读。
- 可以解决脏读、幻读、不可重复读的事务隔离性问题。但不可以解决数据丢失的问题。

#### MVCC原理

MVCC多版本并发控制的具体实现是由多种方案实现的，主要由：**3个隐式字段、undo日志、Read view来实现的。**

##### 隐式字段

InnoDB在设计表时，除了我们设计的字段外，还会给我们新增隐式的三个字段：DB_TRX_ID`,`DB_ROLL_PTR`,`DB_ROW_ID

- DB_TRX_ID：6byte，记录的是**最新操作（修改/删除）的事务ID**
- DB_ROLL_PTR：7byte，记录的是**回滚指针**，执行这条记录的上一个版本号（存储在rollback segment中[涉及undo log]）
- DB_ROW_ID：8byte，隐藏的自增ID，也就是当表没有设置主键时，InnoDB会自动为表生成DB_ROW_ID作为主键索引。

当事务产生时，会对被加事务的数据增加**最新的事务版本号DB_TRX_ID**，同时增加**回滚指针DB_ROLL_PTR** 用以回滚时找到undo log做具体的回滚操作，回退到上一个版本。

##### undo日志

在InnoDB中，undo log分为`insert undo log`和`update undo log`，具体介绍如下：

- insert undo log：事务在insert时产生undo log，在事务提交后删除undo log。
- update undo log：事务在update时产生undo log，在事务提交后删除，或者在回滚时找到undo log。

其中update 的undo log在实际操作中起主要作用，undo log 保存在一个rollback segment回滚段文件中，下面以一个例子讲解MVCC的执行流程：假设有一个表T(String name, int age)

1. 事务1修改第一行数据，首先对其增加排它锁，
2. 事务1流程中，InnoDB把当前数据拷贝到undo log最为历史版本数据，
3. 事务1修改数据name为tom，并且修改该数据隐藏事务**DB_TRX_ID**为1（默认为1，之后递增），并修改回滚指针**DB_ROLL_PTR**为undo log地址。
4. 事务1提交，释放排它锁。

![图片](/images/mysql_mvcc_theory/640.png)

  5.事务2修改第一行数据，并对其增加排它锁，

  6.事务2中把当前数据拷贝到undo log最为历史版本数据，并且最新的数据放在链表的表头，插在undo log最前面

  7.事务2修改age=30，并且修改该数据隐藏事务**DB_TRX_ID**为2，并修改回滚指针**DB_ROLL_PTR**为undo log地址。

  8.事务2提交，释放排它锁

![图片](/images/mysql_mvcc_theory/640-16478666662881.png)

##### Read View

Read View就是事务进行**快照读**操作时产生的读视图，在事务执行快照读的那一刻，会生成数据库当前的一个快照，记录着当前系统当前活跃事务的ID（即最新事务版本号）。所以，当某个事务进行快照读的时候，会对该记录创建一个**read view**读视图，并且以这个视图来判断当前事务可以看到的目标数据是哪个版本的：有可能是最新的，有可能是在undo log中。

具体判断规则是：

1. 首先，取出当前要被修改的数据的事务 DB_TRX_ID，并且与当前其他活跃的事务ID最对比（保持在其他事务的Read view在）
2. 如果取出的事务 DB_TRX_ID能够匹配到当前其他活跃事务的 DB_TRX_ID，则DB_TRX_ID就是当前修改事务的所能看见的Read View视图
3. 如果不能匹配到或者其他不可见性（事务进行中-不可见），则需要通过DB_ROLL_PTR 回滚指针知道undo log的DB_TRX_ID再做比较，从链表首部遍历到尾部，一直找到符合条件的DB_TRX_ID，则这个DB_TRX_ID就是当前事务可以看见的Read View视图

#### MVCC相关问题

##### 1、RR是如何在RC的基础上解决不可重复读的？

当前读和快照读在RR下的区别

| **事务1**                   | **事****务****2**                          |
| --------------------------- | ------------------------------------------ |
| 开启事务                    | 开启事务                                   |
| 快照读(无影响)查询金额为500 | 快照读查询金额为500                        |
| 更新金额为400               |                                            |
| 提交事务                    |                                            |
|                             | select `快照读`金额为500                   |
|                             | select lock in share mode`当前读`金额为400 |

我们可以看到，事务2在事务1提交后的快照读(500)和当前读(400)的结果不一样。让我们换个查询位置再看看：

| **事务1**                     | **事****务****2**                          |
| ----------------------------- | ------------------------------------------ |
| 开启事务                      | 开启事务                                   |
| 快照读（无影响）查询金额为500 |                                            |
| 更新金额为400                 |                                            |
| 提交事务                      |                                            |
|                               | select `快照读`金额为400                   |
|                               | select lock in share mode`当前读`金额为400 |

这里我们看到，事务2在事务1提交后的快照读(400)和当前读(400)的结果一样了，这是为什么呢？

这是因为：InnoDB的快照读是**非常依赖事务首次出现的快照读的地方**，这直接决定了事务后续快照读结果的能力。值得注意的是，不管是update，其他如delete和update一样，如果事务2的快照读是在事务1提交之后进行的，也能读到最新的数据。

##### 2、RC、RR级别下的InnoDB快照读的异同

RC和RR级别下的InnoDB快照读的主要区别在于他们生成`Read View`的时机的不同。

- 在RR级别下某个事务的开始，都是会去创建一个快照和Read view，并将当前系统活跃的其他事务ID记录进去，此后再次调用快照读时，调用的是同一个Read view，所以主要当前事务是在其他事务提交后使用的快照读，那么当前事务之后的快照读都是使用的同一个Read view的，所以其他事务的修改对于当前事务都是不可见的。
- 在RR级别下某个事务的开始、每次的快照读都会生成一个快照和Read view，所以我们在RC级别下的事务中每次操作都是可以看到其他事务提交的更新的。