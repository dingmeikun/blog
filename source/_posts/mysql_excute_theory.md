---
title: MySQL执行原理
date: 2021-06-07 19:09:00
tags: MySQL
categories: 数据库
comments: true
typora-root-url: ../
---

在上一篇的 [MySQL-查询执行流程](http://mp.weixin.qq.com/s?__biz=Mzg3MTY3NzY2OQ==&mid=2247483694&idx=1&sn=3b7dc87fec6e3ac8b8fbe6708c1908c8&chksm=cefbaaa4f98c23b2e41e0a92d759fbf11d161835fb6f7b369ebf63a0d2a4b85fc940266430db&scene=21#wechat_redirect) 中，简单介绍了MySQL中的Server层和Engine层，以及Server层中的重要组成和SQL查询流程，我们知道一条SQL查询语句在MySQL中会经过连接器、会去查询缓存、会做SQL解析与优化，最终到执行引擎中执行。接下来我们针对更新语句以及一些细节来进行研究。

### MySQL执行流程

这里还是上篇文章中的MySQL执行图：

![图片](/images/mysql_excute_theory/640.png)

在一条更新语句中，如`update table set name = '张三' where id = 2;` 在由客户端发起请求后，将做以下动作：

- 客户端与MySQL的Server层建立连接，进行用户验证和权限验证，保持连接
- Server层预处理Update请求并将缓存清空，同时交由分析器处理
- 分析器对Update语句进行词法、语法分析，组成语法树，确定需要更新的表及字段等等、
- 优化器根据索引及匹配规则生成并优化执行计划，比如上述Update语句就是使用的主键索引(聚簇索引)
- 执行器根据执行计划，请求存储引擎的更新API去更新具体行的数据
- 最后到执行引擎层做最终的处理，这里涉及UndoLog、RedoLog和Binlog

关于上述步骤，我在网上找到一个更加详细的说明：

![图片](/images/mysql_excute_theory/640-16478664269201.png)

### Undolog、Redolog、Binlog

在上述图的描述中，涉及到三种日志：redolog（重做日志）、undolog（回滚日志）、binlog（归档日志），下面针对这三种日志进行说明。

#### Undo Log(保证事务一致性)

undo log主要是用来存放修改前的数据，如更新语句：`update table set name = '张三' where id = 2;` 需要将id=2数据行中的name修改为name='张三'，我们假设他之前的数据时name='李四'，则undo log就是用来存放[name='李四']的。在后续的过程中，如果遇到任何的异常或者其他情况导致的回滚操作，则需要根据undo log来进行回滚。

针对于数据的更新如Insert会有INSERT_UNDO，针对update会有UPDATE_UNDO，这是因为这两类操作都是需要事务保证的。针对事务我们都知道MySQL的MVCC多版本并发控制，它的实现就是依赖的UndoLog来实现的：

- 当一个事务A开启时，首先会记录UndoLog，接着将最大系统版本号记录到行数据D的事务版本号上（Insert事务提交后清理undolog，update事务当事务提交，或者回滚后清理undolog）。
- 后另一个事务B开启，读取行数据D的信息，发现当前数据事务版本与D事务版本不一致，说明数据被其他事务占用了，此时将读取UndoLog作为**当前读**，读取数据行D在事务A之前的数据。

#### Redo Log(保证事务持久性)

Redo log的产生，主要是由于MySQL在持久化数据时都需要进行磁盘IO，写之前的读也需要磁盘IO，这些成本都很高，所以此时引入了Redo log来提升效率。每当引擎层有数据写(更新)时，都会记录RedoLog，且针对数据的修改都是遵循：先写日志，再写磁盘。

具体操作是：

- 当执行更新语句`update table set name = '张三' where id = 2;`时，会先通过引擎查询API查询数据页放入buffer pool缓冲池
- 然后在buffer pool中拿到具体数据进行修改，此时的buffer pool与磁盘数据页将不一致，所以称此时的buffer pool为脏页dirty page，InnoDB会在合适的时候(事务commit或系统空闲)刷新脏页(将buffer pool刷新到磁盘)。

此时我们可以认为更新结束了，但是如果在刷新脏页之前MySQL宕机了怎么办？数据不是丢失了吗？那这时，就轮到Redo log出场了：在修改buffer pool的同时，把修改后的数据也记录到这个文件(Redo log顺序IO)。如果此时MySQL挂了，在重启DB时可以根据Redo log物理文件应用到磁盘，保证数据的持久性。这里我们重申一下：

- 当更新buffer pool后产生的脏页还没有刷新磁盘时，MySQL宕机，在服务重启后可以通过Redo log重做日志重新刷回到磁盘中，保证数据持久性。
- 当脏页刷新回磁盘时是一个物理磁盘的随机IO，性能较差，所以采取Redo log记录数据而不是立即写回磁盘，可以提高引擎处理效率，提升事务处理时间。

另外，RedoLog是MySQL引擎层的日志，是循环写的，主要记录了数据行做了什么更新。Redo log是固定大小，比如可以配置为4个文件，每个文件1G，那么写Redo log时就会在这四个文件中循环读写。这是什么概念呢？可以由以下的图来解释。

![图片](https://mmbiz.qpic.cn/mmbiz_png/k6UaANOJgjYiaeOAiaS2mCgojBjvdBicELqiaicnIGoTrAFIt0XGsvlGlrUcwg0CPTKibInJhQOr2fbNqZJjIvyW2GIg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

上图中有两个参数：write pos和check point。其中check poit指的是需要擦除的起始位置，之后的数据都是需要擦除的，擦除操作主要是将write pos之前写入的脏页进行刷新磁盘并更新磁盘页(清除数据)。write pos到check poit之间的数据都是可以进行脏页写入的，如果两个指针相遇，则表示Redo Log写满了需要先执行check poit、暂停write pos，进行Redo log落盘。

#### BinLog（归档）

Binlog是属于MySQL Server层的一种日志文件，我们称之为归档日志。binlog是一种记录用户操作的SQL语句(Select除外)，具体保存了binglog偏移量、操作的表，操作的列，新值、旧值、操作类型(update\insert\delete)和sql语句等等数据。

binlog有两种模式，statement格式记录sql语句，row模式记录行的内容，包括更新前和更新后的数据。一般情况下都是设置的row模式，这需要在my.cnf（或my.ini）中开启binlog并设置为row模式；这便于外部程序去获取并解析目标表的数据变更，去做一些根据数据变更后的操作，比如：采用canal模拟MySQL从库读取binlog，去做一些缓存；或者更直接的，查看当前操作数据库历史记录。

#### 对比RedoLog、Binlog、UndoLog

binlog属于MySQL的Server层自带的一种日志，不具有crash safe的能力，而redolog和undolog是InnoDB特有的两种日志

redolog是物理日志，配合buffer pool记录数据的页具体的修改；binlog是逻辑日志，记录了语句操作前后的变更与行为。

redolog是循环写的，针对redolog文件是有限的；binlog是追加的形式写入文件，空间不足则写入下一个文件(文件名为固定格式且后缀递增：master.000001)

### MySQL引擎层

从上面的分析中我们知道了在引擎层处理sql语句时，会涉及到undolog、redolog和binlog三种日志，以及事务的持久性和一致性，这里根据这三种日志进行具体的流程分析。首先，上流程图：

![图片](/images/mysql_excute_theory/640-16478664269213.png)

当执行引擎收到执行器的Update SQL后，将会有事务开启和事务提交的过程，这里针对这个情况做一下分析：

#### 事务开启

1. 引擎收到Update语句后，首先查询SQL语句中对应的数据是否在buffer pool中存在，如果不存在则通过B+Tree索引找到具体的行指针并读取磁盘页，将数据加载到buffer pool中。
2. 得到数据后，尝试对数据加锁，这是因为MySQL写锁需要确定当前数据不会被其他事务所占用，所以必须要加上排他锁。如果当前记录别其他事务占用了，则进入锁等待；如果没有被其他事务所占用，则在判断系统会不会因为自己的加锁导致死锁，如果不会则加上排它锁。
3. 写Undo Log，将当前修改前的数据写入undolog中，并在当前事务修改目标数据(填写事务编号)，最后使用回滚指针指向undo log中修改的数据，构建回滚段。用于MVCC保证和事务回滚。
4. 写Redo log buffer，此时更新ID=2的数据行，并写入Redo log buffer，同时写入Redo log。
5. 写binlog cache，修改的信息会以对应的event方式写入到binlog cache中。
6. 写change buffer，因为此时可能已经涉及了数据的修改和索引的变更，所以需要写入change buffer cache中。

#### 事务提交

1. InnoDB存储引擎事务提交分为两个阶段：prepare阶段、commit阶段，这里根据这两个阶段做一下分析：
2. 引擎层此时将redolog buffer 刷新到磁盘中，用户奔溃恢复，此时事务处于prepare预提交状态
3. 执行器接收到结果，将完整的binlog cache和redo log buffer中的XID写入到binlog中,（如果此时设计MySQL-Slave主从同步，则：发送binlog cache中的event到slave节点并等待slave ack），执行binglog fsync刷盘并清空binglog cache。
4. 执行redolog commit：此时事务产生的binglog已落盘，执行器调用引擎的提交事务接口，引擎接收到请求，commit redolog(设置redolog为commit状态)
5. 执行引擎释放排它锁，并刷新后续脏页

#### 事务回滚

​	当这些过程中出现异常，或者显示的回滚，则InnoDB会根据 Undo log 数据来进行恢复。

**引用**

https://www.cnblogs.com/wupeixuan/p/11734501.html
