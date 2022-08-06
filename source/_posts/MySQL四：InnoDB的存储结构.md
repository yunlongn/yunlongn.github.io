title: MySQL四：InnoDB的存储结构
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:04:00
---
**「MySQL存储引擎最大的特点就是【插件化】，可以根据自己的需求使用不同的存储引擎，innodb存储引擎支持行级锁以及事务特性，也是多种场合使用较多的存储引擎。」**

> 当官方的存储引擎不足以满足时，我们通过抽象的API接口实现自己的存储引擎。
>
> 抽象存储引擎API接口是通过抽象类handler来实现，handler类提供诸如打开/关闭table、扫表、查询Key数据、写记录、删除记录等基础操作方法。
>
> 每一个存储引擎通过继承handler类，实现以上提到的方法，在方法里面实现对底层存储引擎的读写接口的转调。

<!--more-->

**「InnoDB是为处理巨大数据量时的最大性能设计。它的CPU效率可能是任何其它基于磁盘的关系数据库引擎所不能匹敌的」**。这是官网给出的一句话，可见InnoDB在mysql中的地位。

- 在MYSQL5.5版本，具体是在5.5.8版本之后,，**「InnoDB代替MYISAM称为MYSQL的默认存储引擎」**。
- InnoDB存储引擎支持事务，具有自动崩溃恢复的特性，特点是行锁设计、支持外键，并支持类似于Oracle的非锁定读，即默认读取操作不会产生锁，在日常开发中使用非常广泛。

## **一、InnoDB架构组成**

InnoDB的存储结构分为**「内存结构(左)和磁盘结构(右)两大部分」**，

官方的InnoDB引擎架构图如下：

- MySQL 5.7以前的版本

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- MySQL 5.7 版本

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061122170.png)

由上面两张架构图可以看出，**「InnoDB存储结构在MySQL 5.7 版本之后做了一些调整」**

- 将 Undo日志表空间从共享表空间 ibdata 文件中分离出来，可以在安装MySQL 时由用户自行指定文件大小和数量。
- 增加了 temporary 临时表空间，里面存储着临时表或临时查询结果集的数据。
- Buffer Pool 大小可以动态修改，无需重启数据库实例。

## **二、 InnoDB内存结构**

从架构图中可以看出【**「内存结构主要包括Buffer Pool、Change Buffer、Adaptive Hash Index和Log Buffer四大组件」**】。

### 2.1 Buffer Pool

**「即【缓冲池，简称BP】。BP以Page页为单位，默认大小16K，BP的底层采用链表数据结构管理Page」**。

在InnoDB访问表记录和索引时会在Page页中缓存，以后使用可以减少磁 盘IO操作，提升效率。

- **「Page管理机制」**

  Page根据状态可以分为三种类型：

  **「InnoDB通过三种链表结构来维护和管理上述三种page类型」**

- - 【free list】：空闲缓冲区，管理free page

  - 【flush list】：刷新到磁盘的缓冲区，管理dirty page

    内部page按修改时间排序。脏页即存在于flush链表，也在LRU链表中，两种互不影响，**「LRU链表负责管理page的可用性和释放，而flush链表负责管理脏页的刷盘操作」** 。

  - 【lru list】：正在使用的缓冲区，管理clean page和dirty page

    缓冲区以midpoint为基点：

    前面链表称为new列表区，存放经常访问的数据，占63%；

    后面的链表称为old列表区，存放使用较少数据，占37%。

  - 【free page】 ：空闲page，未被使用

  - 【clean page】：被使用page，数据没有被修改过

  - 【dirty page】：脏页，page的数据被修改过，页中数据和磁盘的数据产生了不 一致

- **「改进型LRU算法维护」**

  > **「每当有新的page数据读取到buffer pool时，InnoDb引擎会判断是否有空闲页，是否足够，如果有就将free page从free list列表删除，放入到LRU列表中。没有空闲页，就会根据LRU算法淘汰LRU链表默认的页，将内存空间释放分配给新的页。」**

- - **「普通LRU」**

    末尾淘汰法，新数据从链表头部加入，释放空间时从末尾淘汰

  - **「改性LRU」**

    **「链表分为new和old两个部分，加入元素时并不是从表头插入，而是从中间midpoint位置插入」**，如果数据很快被访问，那么page就会向new列表头部移动，如果数据没有被访问，会逐步向old尾部移动，等待淘汰。

- **「Buffer Pool配置参数」**

  ```
  show variables like '%innodb_page_size%';
  ```

- - 查看page页大小

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- 查看lru list中old列表参数

```
show variables like '%innodb_old%';
```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061122065.png)

- 查看buffer pool参数

```
show variables like '%innodb_buffer%';
```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061122030.png)

**「一般我们将innodb_buffer_pool_size设置为总内存大小的60%-80%， innodb_buffer_pool_instances可以设置为多个避免缓存争夺。」**

### 2.2 Change Buffer

**「写【缓冲区，简称CB】。在进行DML操作时，如果BP没有其相应的Page数据， 并不会立刻将磁盘页加载到缓冲池，而是在CB记录缓冲变更，等未来数据被读取时，再将数据合并恢复到BP中。」**

- **「ChangeBuffer占用BufferPool空间」**，默认占25%，最大允许占50%，可以根据读写业务量来进行调整。

调整参数为：innodb_change_buffer_max_size

> **「当更新一条记录时，该记录在BufferPool存在，直接在BufferPool修改，一次内存操作。」**
>
> **「如果该记录在BufferPool不存在（没有命中），会直接在ChangeBuffer进行一次内存操作，不用再去磁盘查询数据，避免一次磁盘IO。」**
>
> **「当下次查询记录时，会先进行磁盘读取，然后再从 ChangeBuffer中读取信息合并，最终载入BufferPool中。」**

- **「写缓冲区仅适用于非唯一普通索引页」**

> 如果在索引设置唯一性，在进行修改时，InnoDB必须要做唯一性校验，因此必须查询磁盘， 做一次IO操作。会直接将记录查询到BufferPool中，然后在缓冲池修改，不会在 ChangeBuffer操作。

### 2.3 Adaptive Hash Index

**「即【自适应哈希索引】，用于优化对BP数据的查询」**。

**「InnoDB存储引擎会监控对表索引的查找，如果观察到建立哈希索引可以带来速度的提升，则建立自适应哈希索引，所以称之为自适应」**。InnoDB存储引擎会自动根据访问的频率和模式来为某些页建立哈希索引。

实现本质上就是一个从某个检索条件到某个数据页的【哈希表】。

```
mysql> show variables like '%hash%';
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| innodb_adaptive_hash_index       | ON    |
| innodb_adaptive_hash_index_parts | 8     |
+----------------------------------+-------+
2 rows in set (0.00 sec)
```

- **「innodb_adaptive_hash_index」**

  控制innodb自适应哈希索引特性是否开启参数

- **「innodb_adaptive_hash_index_parts」**

  凡是缓存都会涉及多个缓存消费者间的锁竞争。

  MySQL通过设立多个AHI分区，每个分区使用独立的锁，来减少锁竞争。

### 2.4 Log Buffer

**「即【日志缓冲区】，用来保存要写入磁盘上log文件（Redo/Undo）的数据」**。

日志缓冲区刷盘时机：

- 日志缓冲区的内容**「定期刷新」**到磁盘log文件中。
- **「日志缓冲区满时会自动将其刷新」**到磁盘，可以改变innodb_log_buffer_size参数大小，减少磁盘IO频率。

当遇到BLOB类型或多行更新的大事务操作时，增加日志缓冲区可以节省磁盘I/O。

LogBuffer主要是用于记录InnoDB引擎日志，在DML操作时会产生Redo和Undo日志。

## **三、 InnoDB磁盘结构**

InnoDB磁盘主要包含【Tablespaces，InnoDB Data Dictionary，Doublewrite Buffer、Redo Log 和Undo Logs】五部分组成。

### 3.1 表空间（Tablespaces）

innodb存储引擎在存储设计上模仿了Oracle的存储结构，其数据是按照表空间进行管理的。**「表空间用于存储表结构和数据」**。表空间又分为系统表空间、独立表空间、 通用表空间、临时表空间、Undo表空间等多种类型。

#### 3.1.1 表空间组成

- **「物理结构组成」**

  **「在系统表空间由于所有的表公用一个.ibdatat1数据文件，所以针对每个表只有一个.frm表结构文件」**。

  **「在独立表空间中，每个表分别都有一个.frm表结构文件，一个.ibd数据文件」**。

  innodb存储引擎物理组织形式可以理解为其在磁盘的存储形式，表现为各种文件，其分类大概为

  | 文件          | 功能           | 描述                                               |
  | :------------ | :------------- | :------------------------------------------------- |
  | ibdatat1      | 共享表空间文件 | 系统/共享表空间，存储各种缓冲数据                  |
  | .frm          | 表定义文件     | 记录表的定义，列名以及列的数据类型                 |
  | .ibd          | 表数据存储文件 | 独立表空间，存储数据表的数据，按行存储             |
  | ib_logfile0/1 | redo日志文件   | 重做日志文件，一共两个循环使用，一个写完即写另一个 |

**「表空间是innodb存储引擎逻辑结构的最高层，所有的数据都存储在表空间中，默认innodb有一个共享表空间，所有的数据都存储在共享表空间中，可以通参数innodb_per_table设置每张表单独存放在一个表空间中」**。

- **「独立表空间内存储的只是数据、索引、插入缓冲页」**。
- **「回滚日志、插入缓冲索引页、事务信息、二次写缓冲等其他数据还是存放在共享表空间」**。

#### 3.1.2 表空间的五种类型

- **「系统表空间」**（The System Tablespace）

  > innodb_data_file_path用来指定innodb tablespace文件，如果我们不在My.cnf文件中指定innodb_data_home_dir和innodb_data_file_path那么默认会在datadir目录下创建ibdata1 作为innodb tablespace。

  ```
  #默认值:
  innodb_data_file_path = ibdata1:12M:autoextend
  # ibdata1 : 文件名为ibdata1
  # 12M : 大小为12M
  # autoextend : 自动扩展
  ```

- - 包含InnoDB数据字典，Doublewrite Buffer，Change Buffer，Undo Logs的**「存储区域」**。
  - 系统表空间也默认包含任何用户在**「系统表空间创建」**的表数据和索引数据。
  - 系统表空间是一个共享的表空间因为它是被多个表共享的。

- **「独立表空间」**（File-Per-Table Tablespaces）

  **「默认开启，独立表空间是一个单表表空间，该表创建于自己的数据文件中，而非创建于系统表空间中」**。

  开启独立表空间参数为：innodb_file_per_table

  ```
  innodb_file_per_table = ON：独立表空间：tablename.ibd
  innodb_file_per_table = OFF：系统表空间：ibdataX
  ```

- - 【innodb_file_per_table = ON】

    **「新建表被创建于【表空间】中,每一个表建立ibd的扩展文件,文件名为：表名.ibd」**，该文件默认被创建于数据库目录中,表空间的表文件支持动态和压缩行格式。

  - 【innodb_file_per_table = OFF】

    **「innodb将被创建于【系统表空间】中，即ibdataX中」**。X代表从1开始的一个数字

  - 【查看表的存储表空间的存储方式值】

    ```
    show variables like 'innodb_file_per_table';
    ```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061122027.png)

- 【修改存储表空间的存储方式值为OFF】

  ```
  set global innodb_file_per_table=off;
  ```

- **「通用表空间」**（General Tablespaces）

  通用表空间为通过create tablespace语法创建的共享表空间。通用表空间可以创建于 mysql数据目录外的其他表空间，其可以容纳多张表，且其支持所有的行格式。

  ```
  #创建表空间tablespaces1
  CREATE TABLESPACE tablespaces1 ADD DATAFILE tablespaces1.ibd Engine=InnoDB; 
  #将表添加到test1表空间
  CREATE TABLE test1 (c1 INT PRIMARY KEY) TABLESPACE tablespaces1;
  ```

- **「撤销表空间」**（Undo Tablespaces）

  **「撤销表空间由一个或多个包含Undo日志文件组成。」**

  > 在MySQL 5.7版本之前Undo占用的是System Tablespace共享区，从5.7开始将Undo从System Tablespace分离了出来。

- - 【innodb_undo_tablespaces】

    innodb_undo_tablespaces = 0 ：默认值，表示使用系统表空间ibdata1

    innodb_undo_tablespaces = 1：大于0表示使用undo表空间undo_001、 undo_002等

- **「临时表空间」**（Temporary Tablespaces）

  **「mysql服务器正常关闭或异常终止时，临时表空间将被移除，每次启动时会被重新创建」**。

  临时表空间分为两种：

- - 【session temporary tablespaces】

    存储的是用户创建的临时表和磁盘内部的临时表

  - 【global temporary tablespace】

    储存用户临时表的回滚段（rollback segments ）

#### 3.1.3 系统表空间与独立表空间怎么选择

系统表空间与独立表空间都是**「存储表结构和数据」**，只是存储位置和方法不同，那么应该怎么选择呢

- 系统表空间与独立表空间差异

- - 系统表空间由于所有的数据都放ibdataX文件中，**「容易产生IO瓶颈」**
  - 独立表空间单表存储，可以同时刷新多个文件数据

**「基于此，我们一般都是使用独立的表空间进行管理，独立表空间也是默认的配置」**

- 如何把系统表空间中的表转移到独立表空间中

- - 使用mysqldump导出所有数据库表的数据(备份)。

  - 停止MYSQL服务器，修改my.conf配置，删除原来innodb表空间的相关文件

    ```
    #修改my.conf配置
    innodb_file_per_table = ON
    ```

  - 重启MYSQL服务,并重建Innodb系统表空间

  - 重新导入备份的数据

### 3.2数据字典（InnoDB Data Dictionary）

**「InnoDB数据字典由内部系统表组成」**。这些表包含用于查找表、索引和表字段等对象的元数据。

元数据物理上位于InnoDB系统表空间中。数据字典元数据在一定程度上 与InnoDB表元数据文件（.frm文件）中存储的信息重叠。

### 3.3 双写缓冲区（Doublewrite Buffer）

**「位于系统表空间，是一个存储区域」**。

> 在BufferPage的page页刷新到磁盘真正的位置前，会先将数据存在Doublewrite 缓冲区。如果在page页写入过程中出现操作系统、存储子系统或 mysqld进程崩溃，InnoDB可以在崩溃恢复期间从Doublewrite缓冲区中找到page页备份。在大多数情况下，默认情况下启用双写缓冲区。

- innodb_doublewrite = 0 ：禁用Doublewrite 缓冲区

- innodb_flush_method = O_DIRECT  ：数据文件写入操作会通知操作系统不要缓存数据，也不要用预读

  **「innodb_flush_method控制innodb数据文件及redo log的打开、 刷写模式」**。

### 3.4 重做日志（Redo Log）

- **「重做日志是一种基于磁盘的数据结构，用于在崩溃恢复期间修正不完整事务写入的数据」**。
- MySQL以循环方式写入重做日志文件，记录InnoDB中所有对Buffer Pool修改的日志。

> 当出现实例故障，导致数据未能更新到数据文件，则数据库重启时须redo，重新把数据更新到数据文件。读写事务在执行的过程中，都会不断的产生redo log。默认情况下，重做日志在磁盘上由两个名为ib_logfile0和ib_logfile1的文件物理表示。

### 3.5 撤销日志（Undo Logs）

**「撤消日志是在事务开始之前保存的被修改数据的备份，用于回滚事务」**。

撤消日志属于逻辑日志，根据每行记录进行记录。

撤消日志存在于系统表空间、撤消表空间和临时表空间中。