---
title: MySQL主从复制原理
date: 2021-07-25 16:50:00
tags: MySQL
categories: 数据库
comments: true
typora-root-url: ../

---

> 关系型数据库是我们日常工作中经常使用到的存储软件，而MySQL更是其中使用最多的一个，在很多企业都能看到mysql的身影；在中小型的一些公司因为数据少，再加上设置合适的索引，往往单实例就能很好的提供服务，但对于大型的企业来讲，往往需要海量的数据服务支持，这就要求了MySQL需要为我们提供海量数据存储能力；这涉及到MySQL的主从复制、读写分离、分库分表等等高级特性。这里主要讲讲MySQL的主从复制。

### MySQL主从模式

主从复制主要操作是让一台服务器（Master）的数据和其他服务器（Slave）保持同步。一台Master角色的数据可以同步到多台备库（Slave）上，同时一台Slave本身也可以配置另外一台服务器的Master，Master库和Slave库可以有多种不同的组合。

##### 主从方式

MySQL中的主从复制有：一主一从、主主复制、一主多从、级联复制：

![图片](/images/mysql_master_slave/640.png)

##### 为什么主从 

采取MySQL主从同步，有着以下几个好处：

**数据分布**：MySQL复制可以基于地理位置来分布数据备份，例如做不同的数据中心。即使网络不稳定的环境下，远程复制仍然可以进行。但是为了低延迟的主从复制工作，还是尽量将他们的部署网络维持在一个低延迟的网络环境中。

**负载均衡：**MySQL复制可以将读操作分布到不同的节点上面，实现读密集的应用结构，而且这种模式实现比较简单，通过简单的代码修改就能实现基本的负载均衡。基于此，我们可以通过负载均衡策略将多个从服务器IP做DNS轮询，对外提供读能力。

**备份：**MySQl复制可以很好的做数据热备。

**高可用和故障切换：**MySQL复制可以很好的避免单点故障，而且Master宕机时Slave可以马上切换上去，保证业务可用性。

### MySQL复制流程

MySQL主从复制流程如下：

![图片](/images/mysql_master_slave/640-16478667616461.png)

具体步骤有：

1. Master主要处理写请求，接收到update、delete、insert请求后，Master的server层记录下一个二进制日志(binary log)[BinLog]中，也就是配置文件中`log-bin`指定的文件。
2. Slave库启动链接到Master后，会创建一个IO线程轮询请求Master库的binlog
3. Master库当有新产生的binlog时，会创建一个log dump线程，给从库同步binlog
4. Slave库通过IO线程收到Master库同步过来的binlog，将其写入到中继日志（relay log）
5. Slave库创建SQL线程，从relay log中读取数据，重做到Slave DB

这里需要申明一下，上面步骤只是简单的概述，实际上每一步都很复杂，这里再次陈述一下主从复制过程。

**第一步：**在Master库中做的修改都会记录到二进制文件binlog中，在每次提交事务之前都会记录binlog，所以binlog的记录顺序是按照事务的提交顺序来记录的。在记录了binlog之后，Master会告诉存储引擎可以提交事务了。

**第二步：**Slave备库将主库的二进制日志复制到自己的中继日志中。首先，Slave库会启动一个工作线程（IO线程），并且去Master库建立连接，然后主库会启动一个特殊的binlog dump线程，dump线程会监听Master库的二进制日志的事件。如果主库的dump线程未监听到binlog时间（不会通知Slave IO线程），或者Slave同步偏移量与Master一致（无需同步）时，Slave线程的IO线程都是睡眠状态的。

**第三步：**Slave库的IO线程收到Master库的dump线程发送过来的binlog后，会将binglog写入到Slave的中继日志（Relay log：存在于系统缓存中，所占用的内存空间开销比较低）。之后Slave的SQL线程会去读取Relay Log并在Slave备库上执行。

### MySQL复制原理

上面我们介绍了一些复制的概念和复制的流程，接下来看看复制是如何工作的。

##### 基于语句复制

MySQL5.0及之前的版本都是只支持基于语句的复制，也称为基于逻辑复制。在这个模式下，Master库会记录那些可以造成数据更改的语句，当同步到Slave库后，Slave库也就是把这些语句重新执行了一遍。

这个方式比较简单，只是记录了造成数据变更的语句，使得二进制日志更加紧凑，而且占用的空间也不大。但是，基于Slave仅仅是针对Master同步过来的binlog语句重放，会有很多风险。比如Master和Slave的系统时间不是完全一致，如果涉及到函数CURRENT_USER()、NOW()等数据，在Slave中执行时将会有不同的结果。另外一个，如果binlog的语句中存在大量的update语句时，在slave中是顺序执行的，同时需要对其加锁，很消耗性能。

##### 基于行的复制

MySQL5.1开始支持了基于行的复制，这种方式是将实际的数据记录到二进制文件中，和其他数据库的实现比较像。比如在Master中一个查询语句：

```mysql
mysql> insert into table(c1, c2, sum_c3) select c1, c2, sum(c3) from t-table group by c1, c2;
```

如果是在Master中执行，则需要扫描很多次，次数基于table表中c1、c2的组合方式。此时基于行的复制，则仅仅是三条Insert语句，同步到Slave中也是只将三条数据insert即可。

另外，基于行的复制也有弊端，比如：`update talbe set col = 1;` 这个修改全表的col值为1，那么Master给Slave同步的则是table表全表的update语句，数量是table表的总行数。

##### 总结

在MySQL中，没有哪种方式是完美的，但是我们可以在这两种模式之间进行切换，已到达比较好的同步效果。切换方式可以通过参数：`binlog_format`控制二进制格式。

### MySQL复制不一致

在主从复制过程中，如果Master记录binlog之后，在同步给Slave之前，Master宕机了怎么办？

如果遇到了Master同步binlog之前Master宕机，则会丢失数据，这在同步过程中是不被允许的。在研究解决之前，我们可以先看看MySQL主从的复制方式：

##### 异步复制

异步复制是MySQL最常见的复制方式，也是MySQL默认的复制方式，流程是Master节点在记录完当前事务内的binlog后，直接提交事务并响应客户端，并不关心Slave节点是否收到Master同步过来的binlog及之后的步骤，直接结束本次事务操作。这也是上面提到问题的产生原因，当Master宕机时如果binlog未同步给Slave，此时Slave角色升级为Master后，将丢失数据。

##### 同步复制

同步复制指的是当Master节点写入到binlog之后，会等待dump线程同步binlog给每个Slave节点，在MySQL5.7以前这里的步骤是串行的，也就是说dump线程需要一个一个去同步给Slave，同时等待Slave的响应结果；在MySQL5.7之后的Master节点多出来一个ack Thread去接收Slave的响应结果，提升复制的性能。但是整个过程我们可以看到，这种Master节点等待所有Slave接收binlog且执行完成后，才给客户端响应，这种情况在高并发场景下是性能低下的。

##### 半同步复制

半同步复制是介于异步复制和同步复制之间的一种方式，这种模式下Master只需要等待至少一个Slave节点收到binlog并且写入relay log即可，主库不再需要等待所有Slave节点的响应，这给MySQL主从同步节省了不少时间。

半同步复制相比于同步复制，提升了性能；相比于异步复制，提升了数据安全性。在同步过程中会有一定的同步时延，这需要要求Master和Slave处在一个比较好的网络连接中。另外，在Master节点给Slave节点同步binlog时，从库宕机或者网络故障，导致binlog数据无法及时同步到Slave，此时Master节点会等待一段时间（这个时间可配置），如果在这个时间内还是无法同步到Slave，则Master自动切换到异步复制模式，当从库连接正常后自动切换回半同步复制，

这里半同步复制的'半'，体现在Master同步给Slave之后，只等待Slave接收到binlog并写入relaylog就返回了。此时，Slave会启用SQL线程去读取relay log并应用到Salve节点。也就是说，在半同步复制过程中，会有一个relay log到完全应用到Slave库数据的过程，所以称这个过程为半同步复制。

半同步实现：

半同步复制是采用的MySQL插件实现的，需要在Master和Slave节点都安装插件。

1.确认MySQL是否支持动态增加插件

```mysql
mysql> select @@have_dynamic_loading
+--------------------------+
| @@have_dynamic_loading   |
+--------------------------+
| YES                     |
+--------------------------+
```

2.如上，`YES`标识可以动态安装，此时在Master和Slave上都安装插件。插件一般默认在MySQL安装目录/lib/plugin下，可以去查看一下是否存在，主库的插件是semisync_master.so，从库是semisync_slave.so。确认插件没问题就可以安装插件了。

```mysql
# 在Master节点安装 semisync_master.so
mysql>install plugin rpl_semi_sync_master soname 'semisync_master.so';

# 在Slave节点安装 semisync_slave.so
mysql>install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
```

3.安装完后，在plugin表中查看一下：

```mysql
mysql>select * from mysql.plugin;
+----------------------+---------------------+
| name                 |     dl             |
+--------------------------------------------+
| rpl_semi_sync_master | semisync_master.so |
+----------------------+---------------------+
```

4.在主从库中开启半同步复制

```mysql
# master节点开启
mysql>set global rpl_semi_sync_master_enabled=1;
mysql>set global rpl_semi_sync_master_timeout=30000;

# slave节点开启
mysql>set global rpl_semi_sync_slave_enabled=1;

# 注意：如果之前配置的是异步复制，在这里要重启一下从库的IO线程，如果是全新的半同步则不用重启.
mysql>stop slave io_thread;start slave io_thread;
```

5.在主库上查看半同步复制的状态

```mysql
mysql>show status like '%semi_sync';

# 在输出信息中，我们重点关注三个参数：
# rpl_semi_sync_master_status OFF/ON   #ON表示半同步复制打开，OFF表示关闭
# rpl_semi_sync_master_yes_tx   [number]     #这个数字表示主库当前有几个事务说通过半同步复制到从库的
# rpl_semi_sync_master_no_tx   [number]     #表示有几个事务不是通过半同步复制到从库的
```