title: MySQL二：SQL运行机制
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:02:00
---
MySQL用了很久，但是一直也是工作的使用，对于MySQL的知识点都比较零散碎片，一直也没有整体梳理过，趁着最近不忙，梳理一下相关的知识点。

## **一、 MySQL的起源**

MySQL是一个开源的关系数据库管理系统。原开发者为瑞典的 MySQL AB公司，2008 年AB公司被Sun公司收购，并发布收购之后的首个版本 MySQL5.1。2010 年 Oracle 收购 Sun 公司，至此MySQL归入Oracle门下，之后发布了收购以后的首个版本 5.5 。

- MySQL5.1 ，该版本引入分区、基于行复制以及plugin API 。
- MySQL 5.5 ，改善集中在性能、扩展性、复制、分区以及对 windows 的支持。

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061120423.png)

<!--more-->

**「MySQL通过其【插件式的存储引擎架构】，将查询处理和其它的系统任务以及数据的存储提取分离来。并且根据不同的使用场景选择合适的存储引擎，使其在不同应用都能发挥良好作用。」**

## **二、MySQL执行过程**

在逻辑上MySQL 在执行脚本时自上而下可以分为四层，逻辑图如下：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061120413.png)

- **「sql执行流程解析」**

  首先客户端（jdbc，PHP）通过连接处理层连接mysql服务器，然后解析器通过解析树对sql语句进行解析，优化器对sql语句进行优化，最终调用第四层的存储引擎的接口，执行语句。

## **三、MySQL Server基本架构组成**

**「MySQL Server架构自顶向下大致可以分网络连接层、服务层、存储引擎层和系统文件层」**。架构图如下

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061120436.png)

### 3.1 第一层：网络连接层

**「客户端连接器，MySQL向外提供交互接口连接各种不同的客户端。」**

> 几乎支持所有主流的服务端编程技术，如常见的 Java、C、Python、.NET等，都通过各自API与MySQL建立连接。

### 3.2 第二层：服务层

MySQL Server的核心，主要包含系统**「管理和控制工具、连接池、SQL接口、解析器、查询优化器和缓存」**六个部分。

- **「连接池（Connection Pool）」**

  **「负责存储和管理客户端与数据库的连接，一个线程负责管理一个连接。」**

  > 在服务器内部，每个client连接都有自己的线程。这个连接的查询都在一个单独的线程中执行。这些线程轮流运行在某一个CPU内核(多核CPU)或者CPU中。服务器缓存了线程，因此不需要为每个client连接单独创建和销毁线程 。

- **「系统管理和控制工具（Management Services & Utilities）」**

  备份恢复、安全管理、集群管理等

- **「SQL接口（SQL Interface）」**

  用于接受客户端发送的SQL命令，并且返回查询的结果。

  > 比如DML、DDL、存储过程、视图、触发器等。

- **「解析器（Parser）」**

  负责将请求的SQL解析生成一个【解析树】。根据MySQL规则进一步检查解析树是否合法。

- **「查询优化器（Optimizer）」**

  当【解析树】通过解析器语法检查后，将交**「由优化器将其转化成执行计划与存储引擎交互」**。一般执行sql脚本会遵循【**「选取-->投影-->联接」**】的策略

  > ```
  > selectid,namefromuserwhere gender=1;
  > ```
  >
  > 执行以上sql脚本的过程：

- 1. select先根据where语句进行选取，并不是查询出全部数据再过滤
  2. select查询根据uid和name进行属性投影，并不是取出所有字段
  3. 将前面选取和投影联接起来最终生成查询结果

- **「缓存（Cache&Buffer）」**

  缓存机制是由一系列小缓存组成的。比如表缓存，记录缓存，权限缓存，引擎缓存等。

  **「如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。」**

### 3.3 第三层：存储引擎层

**「存储引擎负责MySQL中数据的存储与提取，与底层系统文件进行交互。」**

MySQL存储引擎是插件式的，服务器中的查询执行引擎通过【**「接口」**】与存储引擎进行通信，接口屏蔽了不同存储引擎之间的差异 。通过上图可以看出MySQL有好几种不同的存储引擎，最常见的是MyISAM和InnoDB。

### 3.4 第四层：系统文件层

**「主要是将数据和日志存储在运行设备的文件系统之上，并完成于存储引擎的交互，是文件的物理存储层。」** 主要包含日志文件，数据文件，配置文件，pid 文件，socket 文件等。

- **「日志文件」**

- - 【事务日志】

    包含重做日志（Redo Log）和撤销日志（Undo Logs），在后续文章中详细介绍

  - 【错误日志（Error log）】

    ```
    show variables like '%log_error%'  --默认开启
    ```

  - 【二进制日志（bin log）】**「记录对MySQL数据库执行的更改操作，包括语句的发生时间、执行时长，主要用于数据库恢复和主从复制」**。但不记录select、show等不修改数据库的SQL。

    ```
    showvariableslike'%log_bin%'; --是否开启
    showvariableslike'%binlog%'; --参数查看
    showbinarylogs;--查看日志文件
    ```

  - 【通用查询日志（General query log）】

    ```
    showvariableslike'%general%';--记录一般查询语句
    ```

  - 【慢查询日志（Slow query log）  】

    记录所有执行时间超时的查询SQL，默认是10秒。

    ```
    showvariableslike'%slow_query%'; //是否开启
    showvariableslike'%long_query_time%'; //超时时间
    ```

- **「数据文件」**

- - 【db.opt 文件】

    记录当前库默认使用的字符集和校验规则。

  - 【frm 文件】

    存储与表相关的元数据（meta）信息，包括表结构的定义信息等，每一张表都会有一个frm 文件。

  - 【MYD 文件】

    **「MyISAM 存储引擎专用」**，存放 MyISAM 表的数据（data)，每一张表都会有一个.MYD 文件。

  - 【MYI 文件】

    **「MyISAM 存储引擎专用」**，存放 MyISAM 表的索引相关信息，每一张 MyISAM 表对应一个 .MYI 文件。

  - 【ibd文件和 IBDATA 文件】

    **「存放 InnoDB 的数据文件（包括索引）」**。InnoDB 存储引擎有两种表空间方式：独享表空间和共享表空间。

    **「独享表空间」**使用 .ibd 文件来存放数据，且每张InnoDB 表对应一个 .ibd 文件。

    **「共享表空间」**使用 .ibdata 文件，所有表共同使用一个（或多个，自行配置）.ibdata 文件。

  - 【ibdata1 文件】

    系统表空间数据文件，存储表元数据、Undo日志等 。

  - 【ib_logfile0、ib_logfile1 文件】

    Redo log 日志文件。

- **「配置文件」**

  用于存放MySQL所有的配置信息文件，比如my.cnf、my.ini等。

- **「pid 文件」**

  pid 文件是 mysqld 应用程序在 Unix/Linux 环境下的一个进程文件，和许多其他 Unix/Linux 服务端程序一样，存放着自己的进程 id。

- **「socket文件」**

  socket 文件也是在 Unix/Linux 环境下才有的，用户在 Unix/Linux 环境下客户端连接可以不通过TCP/IP 网络而直接使用 Unix Socket 来连接 MySQL。