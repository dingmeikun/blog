---
title: MySQL查询执行流程
date: 2021-06-05 20:09:00
tags: MySQL
categories: 数据库
comments: true
typora-root-url: ../
---

在MySQL组织结构中，主要有两层：Server层、Engine层。其中Server层有：连接器、查询缓存、分析器、优化器、执行器；而执行引擎层主要负责数据提取、更新、事务执行、事务提交等动作，MySQL支持InnoDB、MyISAM、Memory等多种存储引擎。

那一个简单的如：`select * from table where name = '张三'`是如何执行的？接下来让我们一步一步分析。

### MySQL执行流图

![img](/images/mysql_query_flow/0.png)

#### 连接器

连接器存在于MySQL的Server层，主要是对接我们的程序，或者说是对接客户端/服务端等需要通过用户名密码以及对应的链接参数进行请求连接MySQL，进行url和用户名密码参数校验、安全校验、权限校验以及获取用户权限等。

#### 查询缓存

建立连接后，首先会去确认缓存中是否存在。这里的缓存是使用key-value的形式存储，key为查询的sql语句，value为结果集。如果命中了缓存则直接返回给客户端，如果没有命中则进行后面的阶段[分析器]，并且把最终的结果缓存起来。

这里值得注意的是：`MySQL8.0之后去除缓存查询功能`。这是因为MySQL缓存很容易失效，只要数据被更新了就会删除缓存；其次缓存的命中率低下，主要MySQL查询语句中的条件调换位置就会miss缓存。所以在很多情况下，使用缓存并不能很高效的提升查询性能，但是如果数据更新的很不频繁则可以考虑开启缓存。

#### 分析器

分析器的主要作用是将客户端发过来的sql语句进行分析，这将包括预处理与解析过程，在这个阶段会解析sql语句的语义，并进行关键词和非关键词进行提取、解析，并组成一个解析树。

其次就是针对SQL语句进行词法、语法分析。第一步：词法分析：SQL语句有多个关键字组成，首先是提取关键字，对使用的关键字进行词法分析，是否符合MySQL的关键字描述；第二步：就是语法分析，根据输入的关键字，分析查询的哪个表、哪些字段，并校验他们对不对等等。

#### 优化器

生成执行计划，并根据执行计划和索引设置优化执行计划，比如我们常见的最左侧匹配原则，当有联合索引使用时，如果按照最左匹配出来数据，优化器会将下一个索引也放进去判断。从而提升性能，但是这个优化不一定是最好的。比如用了两个单独索引，优化器可能将其位置调换，不一定是顺序的。

#### 执行器

在执行器的阶段,此时会调用存储引擎的API,API会调用存储引擎。存储引擎具体分类有InnoDB、MyISAM、Memory等，常用的是myisam和innodb。

### MySQL引擎层

在解析和优化阶段，mysql将生成查询对应的执行计划，mysql的查询执行引擎则根据这个执行计划来完成整个查询。这里执行计划是一个数据结构，而不是和很多其他的关系型数据库那样对应的字节码。mysql简单的根据执行计划给出的指令逐步执行。在根据执行计划逐步执行的过程中，有大量的操作需要通过调用存储引擎实现的接口来完成。为了执行查询，mysql只需要重复执行计划中的各个操作，直到完成所有的数据查询。