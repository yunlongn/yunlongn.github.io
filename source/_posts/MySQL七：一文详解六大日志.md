title: MySQL(七)：一文详解六大日志
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:07:00
---
日志一般分为逻辑日志与物理日志两类

- **「逻辑日志」**：即执行过的事务中的sql语句，执行的sql语句（增删改）**「反向」**的信息
- **「物理日志」**：`mysql` 数据最终是保存在数据页中的，物理日志记录的就是数据页变更 。

**「`mysql`数据库中日志是重要组成部分，记录着数据库运行期间各种状态信息」**。主要有6类：

- 二进制日志
- 重做日志
- 撤销日志
- 错误日志
- 查询日志
- 中继日志
<!--more-->
**「而我们一般比较关注的是二进制日志( `binlog` )和事务日志(包括重做日志`redo log` 和撤销日志 `undo log` )」**。

## **一、二进制日志(binlog)**

### 1.1 什么是binlog

**「记录对MySQL数据库执行的更改操作，包括语句的发生时间、执行时长，主要用于数据库恢复和主从复制」**。但不记录select、show等不修改数据库的SQL。

**「`binlog`是逻辑日志，在 `mysql Server` 层进行记录，属于`mysql`全局日志，无论使用什么存储引擎都会记录 `binlog` 日志」**。

- **「binlog产生的时机」**

  **「事务提交的时候，一次性将事务中的sql语句（一个事物可能对应多个sql语句）按照一定的格式记录到binlog中」**。

- **「binlog记录方式」**

  **「`binlog` 是通过追加的方式记录数据库执行的写操作(不包括读操作)信息，以二进制保存在磁盘中。」**

- **「开启binlog」**

  在my.inf主配置文件中添加如下配置

  ```
  #打开binlog日志 默认是OFF
  log_bin=ON
  #binlog日志的基本文件名，后面会追加标识来表示每一个文件 例如：mysql-bin.000094
  log_bin_basename=/var/lib/mysql/mysql-bin
  #指定的是binlog文件的索引文件  管理所有的binlog文件的目录 记录有多少个binlog文件等
  log_bin_index=/var/lib/mysql/mysql-bin.index
  
  #或者将以上三个配置合二为一
  log-bin=/var/lib/mysql/mysql-bin
  #mysql会根据这个配置自动设置log_bin为on状态，自动设置log_bin_index文件为指定文件名后跟.index
  ```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- **「binlog大小设置与过期时间」**

  **「`binlog` 是通过追加的方式进行写入的，有两个参数需要控制binlog大小设置与过期时间」**

  ```
  [mysqld]
  expire_logs_days = 7
  max_binlog_size = 104857600
  
  set global max_binlog_size=104857600;#100M
  show variables like '%max_binlog_size%';
  show variables like '%expire_logs_days%';
  expire_logs_days 
  ```

  ```
  set global max_binlog_size=104857600;#100M
  set global expire_logs_days = 7;
  show variables like '%max_binlog_size%';
  show variables like '%expire_logs_days%';
  ```

- - **「查看配置」**

    ```
    show variables like '%max_binlog_size%';
    show variables like '%expire_logs_days%';
    ```

  - **「命令配置」**

    及时生效，重启失效，

  - **「max_binlog_size」**

    **「`max_binlog_size` 代表每个 `binlog`文件的大小，当文件大小达到给定值之后，会生成新的文件来保存日志，默认值是1GB」**。

  - **「expire_logs_days」**

    **「mysql自动清除过期binlog日志的天数。默认值为0,表示没有自动删除」**。

  - **「配置文件配置」**

    永久性，需要重启，修改my.conf：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124907.png)

### 1.2 binlog使用场景

在实际应用中， `binlog` 的主要使用场景有两个，分别是 **「主从复制」** 和 **「数据恢复」** 。

1. **「数据复制」** ：

   在 `Master` 端开启 `binlog` ，然后将 `binlog`发送到各个 `Slave` 端， `Slave` 端执行 `binlog` 从而达到主从数据一致。

2. **「数据恢复」**

   通过使用 `mysqlbinlog` 工具来恢复数据。

### 1.3 binlog刷盘时机

**「对于 `InnoDB` 存储引擎而言，只有在事务提交时才会记录`biglog` 日志」**，此时记录还在内存中，那么 `biglog`是什么时候刷到磁盘中的呢？

**「`mysql` 通过 `sync_binlog` 参数控制 `biglog` 的刷盘时机，取值范围是 `0-N`：」**

- 0：不去强制要求，由系统自行判断何时写入磁盘；
- 1：每次 `commit` 的时候都要将 `binlog` 写入磁盘；
- N：每N个事务，才会将 `binlog` 写入磁盘。

```
show variables like '%sync_binlog%';
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**「`MySQL 5.7.7`之后版本的 `sync_binlog` 默认值是 `1` ，配置为1也是数据最不容易丢失的，但是设置一个大一些的值可以提升数据库性能，因此实际情况下也可以将值适当调大，牺牲一定的一致性来获取更好的性能。」**

### 1.4 binlog日志格式

`binlog` 日志格式通过 `binlog_format` 指定，有三种格式，分别为 `STATMENT` 、 `ROW` 和 `MIXED`。日志。

> 在 `MySQL 5.7.7` 之前，默认的格式是 `STATEMENT` ， `MySQL 5.7.7` 之后，默认值是 `ROW`。

```
mysql> show variables like '%binlog_format%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set, 1 warning (0.00 sec)
```

- **「STATMENT」**

  基于`SQL` 语句的复制( `statement-based replication, SBR` )，每一条修改数据的sql语句会记录到`binlog` 中  。

- - 缺点：在某些情况下会导致主从数据不一致，比如执行sysdate() 、  slepp()  等 。
  - 优点：不需要记录每一行数据的变化，减少了 binlog 日志量，节约 IO  , 从而提高了性能；

- **「ROW」**

  基于行的复制(`row-based replication, RBR` )，不记录每条sql语句的上下文信息，仅需记录哪条数据被修改了 。

- - 缺点：会产生大量的日志，尤其是`alter table` 的时候会让日志暴涨
  - 优点：能清楚记录每一个行数据的修改细节，能完全实现主从数据同步和数据的恢复；

- **「MIXED」**

  基于`STATMENT` 和 `ROW` 两种模式的混合复制(`mixed-based replication, MBR` )，一般的复制使用`STATEMENT` 模式保存 `binlog` ，对于 `STATEMENT` 模式无法复制的操作使用 `ROW` 模式保存 `binlog`

## **二、重做日志（Redo Log）**

**「重做日志（Redo Log）是InnoDB存储引擎所特有的日志」**。了解Redo Log从以下几个方向来看：

- Redo Log解决了什么问题
- 为什么需要redo log
- redo log效率为什么快
- redo log file刷盘
- Redo Log和BinLog的比较

### 2.1Redo Log解决了什么问题

**「InnoDB作为MySQL的存储引擎，数据是存放在磁盘中的，如果每次读写数据都需要磁盘I/O，效率会很低。」**

为提高读写效率，InnoDB添加了缓存池（Buffer Pool）作为访问数据库的缓冲，其包含了磁盘中部分数据页的映射。

- **「当从数据库读取数据时，会首先从Buffer Pool中读取，如果Buffer Pool中没有，则需要从磁盘中读取然后放入 Buffer Pool；」**
- **「当向数据库写入数据时，会首先写入Buffer Pool，Buffer Pool中修改的数据会定期刷新到磁盘中（即：刷脏）」**。

*Buffer Pool的使用大大提高了读写数据的效率，但是也带了新的问题：如果MySQL宕机，而此时Buffer Pool中修改的数据还没有刷新到磁盘，就会导致数据的丢失，事务的持久性无法保证*。而Redo Log 解决了这个问题。

### 2.1为什么需要redo log

- 事务的四大特性ACID中有一个是 **「持久性」**

  **「只要事务提交成功，那么对数据库做的修改就被永久保存下来了，不可能因为任何原因再回到原来的状态」** 。

- **「`mysql`是如何保证一致性」**

最简单的做法是在每次事务提交的时候，将该事务涉及修改的数据页全部刷新到磁盘中。但是这么做会有性能瓶颈，主要体现在两个方面：

- 因为 `Innodb` 是以 `页` 为单位进行磁盘交互的，而一个事务很可能只修改一个数据页里面的几个字节，这个时候将完整的数据页刷到磁盘的话，太浪费资源了！
- 一个事务可能涉及修改多个数据页，并且这些数据页在物理上并不连续，使用随机I/O写入性能太差！

因此 `mysql` 设计了 `redo log` ， **「redo log是物理日志，只记录事务对数据页做了哪些修改，而不是某一行或某几行修改成什么样，它用来恢复提交后的物理数据页(恢复数据页，且只能恢复到最后一次提交的位置)，相对而言文件更小并且是顺序IO」**。

### 2.3 redo log效率为什么快

Redo Log 默认是在事务提交的时候将日志写入磁盘，为什么它比直接将 Buffer Pool中修改的数据写入磁盘要快呢？

- **「磁盘刷脏操作是随机I/O，因为每次修改的数据的位置都是随机的，但是写redo log是追加操作，顺序I/O」**。
- **「磁盘刷脏是以数据页为单位的（Page），MySQL默认页的大小是16KB，一个page上一个小修改都要整页写入；而 redo log中只包含真正修改的部分，不会有无效I/O」**。

### 2.4 redo log基本概念

- **「重做日志是一种基于磁盘的数据结构，用于在崩溃恢复期间修正不完整事务写入的数据」**。
- MySQL以循环方式写入重做日志文件，记录InnoDB中所有对Buffer Pool修改的日志。

> 当出现实例故障，导致数据未能更新到数据文件，则数据库重启时须redo，重新把数据更新到数据文件。读写事务在执行的过程中，都会不断的产生redo log。默认情况下，重做日志在磁盘上由两个名为ib_logfile0和ib_logfile1的文件物理表示。

`redo log` 包括两部分

- 一个是内存中的日志缓冲( `redo log buffer` )，临时性
- 一个是磁盘上的日志文件( `redo log file`)，永久性。

### 2.5 redo log写磁盘的方式

`mysql` 每执行一条 `DML` 语句，先将记录写入 `redo log buffer`，后续**「某个时间点再一次性将多个操作记录写到 `redo log file`」**。这种 **「先写日志，再写磁盘」** 的技术就是 **「「预写式日志」」**（`Write-Ahead Logging` ，缩写 WAL）。

**「在计算机操作系统中，用户空间( `user space` )下的缓冲区数据是无法直接写入磁盘的，中间必须经过操作系统内核空间( `kernel space` )的缓冲区( `OS Buffer` )」**。因此， `redo log buffer` 写入 `redo log file` 实际上是**「先写入 `OS Buffer` ，然后再通过系统调用 `fsync()` 将其刷到 `redo log file`中」**。

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124947.png)

- **「redo log file刷盘时机」**

  **「`mysql` 支持三种将 `redo log buffer` 写入 `redo log file` 的时机」**，可以通过 `innodb_flush_log_at_trx_commit` 参数配置：

  ```
  mysql> show variables like '%innodb_flush_log_at_trx_commit%';
  +--------------------------------+-------+
  | Variable_name                  | Value |
  +--------------------------------+-------+
  | innodb_flush_log_at_trx_commit | 1     |
  +--------------------------------+-------+
  1 row in set, 1 warning (0.01 sec)
  ```

- - 速度较快，比0安全，只有在**「操作系统崩溃或者系统断电」**的情况下，上一秒钟所有事务数据才可能丢失。

  - 最安全，但也是最慢，即使mysql挂掉也不会丢失数据，但是IO频繁，性能下降。

  - 速度最快，不太安全，在mysql挂掉的时候，会丢失一秒钟的数据。

  - 当`innodb_flush_log_at_trx_commit` = 0

    【**「延迟写」**】：**「提交事务时不会立即将数据`redo log buffer` 写入到 `OS Buffer` ，而是通过 InnoDB 的主线程每秒写入 `OS Buffer`并调用 `fsync()` 将其刷到 `redo log file`中」**。

  - 当`innodb_flush_log_at_trx_commit` = 1

    【**「实时写实时刷」**】：**「每次提交事务时都会将数据`redo log buffer` 写入到 `OS Buffer` 并立即调用 `fsync()` 将其刷到 `redo log file`中」**。

  - 当`innodb_flush_log_at_trx_commit` = 2

    【**「实时写延迟刷」**】：**「每次提交事务时都会将数据`redo log buffer` 写入到 `OS Buffer` 中，然后每间隔一秒调用 `fsync()` 将其刷到 `redo log file`中」**

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 2.6 redo log写入形式

**「在innodb中，既有`redo log` 需要刷盘，还有 `数据页` 也需要刷盘， `redo log`存在的意义主要就是降低对 `数据页` 刷盘的要求」** 。

**「`redo log` 实际上记录数据页的变更，而这种变更记录是没必要全部保存，因此 `redo log`实现上采用了【大小固定，循环写入】的方式，当写到结尾时，会回到开头循环写日志」**。如下图：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124978.png)

- `write pos` ：表示 `redo log` 当前记录的 `LSN` (逻辑序列号)位置
- `check point` ：表示 **「数据页更改记录」** 刷盘后对应 `redo log` 所处的 `LSN`(逻辑序列号)位置。

`write pos` 到 `check point` 之间的部分是 `redo log` 空着的部分，用于记录新的记录；

`check point` 到 `write pos` 之间是 `redo log` 待落盘的数据页更改记录。

当 `write pos`追上`check point` 时，会先推动 `check point` 向前移动，空出位置再记录新的日志。

> 启动 `innodb` 的时候，无论上次是正常关闭或异常关闭，都会进行恢复操作。
>
> 重启`innodb` 时，首先会检查磁盘中数据页的 `LSN` ，如果数据页的`LSN` 小于日志中的 `LSN` ，则会从 `checkpoint` 开始恢复。
>
> 如果在宕机前正处于`check point` 的刷盘过程，且**「数据页的刷盘进度超过了日志页的刷盘进度」**，此时会出现数据页中记录的 `LSN` 大于日志中的 `LSN`，这时超出日志进度的部分将不会重做，因为这本身就表示已经做过的事情，无需再重做。

### 2.7 Redo Log和BinLog的比较

redo log 主要用来。binlog是用来进行归档的。

**「一个更新的sql先执行到redo log内为预提交状态，binlog写入，写入之后通知redo log改提交状态」**

- **「作用不同」**

- - redo log是用于【崩溃恢复】的，保证MySQL宕机也不会影响持久性。

    > redo log 主要用来数据库宕机恢复数据的

  - binlog是【用于时间点恢复】的，保证服务器可以基于时间点恢复数据或者用于主从复制。

    > binlog是用来进行归档的

- **「层次不同」**

- - redo log是InnoDB存储引擎实现的，innodb独享
  - binlog是MySQL的服务器层实现的服务，全局引擎共享

- **「内容不同」**

- - redo log是物理日志，内容基于磁盘的数据Page，**「记录该数据页更新状态内容」**

    > 比如：「将第6页、第8行、第7个位置改成aaaa」这种

  - bin log是逻辑日志，内容是二进制的，根据binlog＿format参数的不同，分为不同的模式，可能基于SQL 语句、基于数据本身、或者二者的混合，**「记录更新过程」**。

    > 比如：insert into t values (null, 4, '2022-03-24');  也跟bin log日志格式有关

- **「写入时机不同」**

- - bin log是在事务提交完成后进行一次写入
  - redo log的写入时机有三种（具体在上文中有介绍）

- **「写入方式不同」**

- - binlog在写满或者重启之后，会生成新的binlog文件，旧的日志数据会一直保留。
  - redo log是循环使用，会清理旧的日志数据。

## **三 撤销日志（Undo Logs）**

> 世间没有后悔药，但是在MySQL中实现了重头开始，撤销日志（Undo Logs）就是MySQL的后悔药。

**「撤消日志是在事务开始之前保存的被修改数据的备份，用于回滚事务」**。

撤消日志属于逻辑日志，根据每行记录进行记录。

撤消日志存在于系统表空间、撤消表空间和临时表空间中。

### 3.1 什么是Undo log

Undo：意为撤销或取消，undo即返回指定某个状态的操作

- **「undo log」**

  **「一种用于撤销回退的日志，在事务开始之前，会先记录存放到 Undo 日志文件里，备份起来，当事务回滚时或者数据库崩溃时用于回滚事务」**。

- **「undo log记录的是什么」**

  **「undo log中记录的是当前事务操作中的相反操作」**。

  undo日志属于逻辑日志，记录的是一个操作过程，sql执行delete或者update操作都会记录一条undo日志

  ```
    一条insert语句在undo log中会对应一条delete语句，
    一条delete语句在undo log中会对应一条insert语句，
    update语句会在undo log中对应相反的update语句，
  ```

### 3.2 Undo log存储方式

**「Undo log的存储由InnoDB存储引擎实现」**，数据保存在InnoDB的数据文件中，innodb存储引擎对undo的管理采用回滚段（rollback segment）的数据结构。

回滚段（rollback segment）中有1024个undo log segment，

- MySQL5.5版本之前

  只支持1个rollback segment，即只能存储1024个undo log segment

- MySQL5.5版本之后

  支持128个rollback segment（innodb_undo_logs配置项），即能存储128*1024个undo log segment

### 3.3 Undo log相关的变量

```
mysql> show variables like '%innodb_undo%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_undo_directory    | .\    |
| innodb_undo_log_truncate | OFF   |
| innodb_undo_logs         | 128   |
| innodb_undo_tablespaces  | 0     |
+--------------------------+-------+
4 rows in set, 1 warning (0.03 sec)
```

- innodb_undo_directory

  定义存储的目录路径，默认值.\，表示datadir，

  datadir参数在my.in中配置

  ```
  [mysql]
  # 设置mysql数据库的数据的存放目录
  datadir="D:\mysql-8.0.13-winx64\data"
  ```

- innodb_undo_log_truncate

  开启1(ON)/关闭0(OFF)在线回收（收缩）undo log日志文件，支持动态设置，默认关闭

- innodb_undo_logs

  回滚段rollback segment的数量，Mysql5.5版本后默认设置为128

- innodb_undo_tablespaces

  默认值为0，表示undo log全部写入一个表空间文件，可以设置这个变量，平均分配到多少个文件中。

### 3.4 Undo log的工作原理

**「undo log在事务开启之前产生，当事务提交后，InnoDB会将事务对应的undo日志保存在删除list中，后台通过清除线程进行回收处理」**。

以一条sql执行update、select过程，如图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- 执行update操作，事务A提交时候（事务还没提交），会将数据进行备份，备份到对应的undo buffer，
- Undo Log保存了未提交之前的操作日志，User表数据肯定就是持久保存到InnoDB的数据文件IBD，默认情况。
- 此时事务B进行查询操作，直接读undo buffer缓存，事务A还没提交事务，如需要回滚，不读磁盘，先直接从undo buffer缓存读取

### 3.5 Undo log作用说明

- **「实现事务的原子性」**undo log可以用于实现事务的原子性， 如果事务处理过程中要执行回滚（rollback）操作，可以利用undo log将数据恢复到事务开始之前
- **「实现多版本并发控制（MVCC）」**Undo Log 在 MySQL InnoDB 存储引擎中用来实现多版本并发控制，事务没提交之前，undo日志可以作为高并发情况下其它并发事务进行快照读。

### 3.6 Bin log、redo log、Undo 如何协同

**「先来回顾一下几个日志的主要作用」**

- Buffer Pool 是MySQL的一个非常重要的组件，因为针对数据库的增删改操作都是在Buffer Pool完成的
- Undo log 记录的是数据操作前的样子
- Redo log 记录的是数据被操作后的样子
- Bin log 记录的是整个操作记录

**「准备更新一条数据到事务的提交的流程描述：」**

- 首先执行器根据MySQL的执行计划来查询数据，先是从缓存池(Buffer Pool)中查询数据，如果没有就会去 数据库中查询，如果查询到了就将其放到缓存池中。

- 在数据被缓存到缓存池的同时，会写入 undo log 日志文件

- 更新的动作是在BufferPool中完成的，同时会将更新后的数据添加到redo log buffer中

- 完成以后就可以提交事务，在提交的同时会做以下三件事

- - 将redo log buffer中的数据刷入到redo log 文件中
  - 将本次操作记录写入到bin log文件中
  - 将bin log 文件名字和更新内容在bin log中的位置记录到redo log中，同时在redo log 最后添加 commit 标记

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124896.png)

### 3.7 Undo log的清理

#### 3.7.1 Undo log类型

**「在回滚段中，每个 undo log 段都有一个类型字段，共有两种类型」**：

- **「insert undo log」****「代表事务在`insert`新记录时产生的`undo log`, 其回滚段类型为 insert undo logs，仅用于事务回滚，并且在事务提交后可以被立即丢弃」**。
- **「update undo log」****「事务在进行`update`或`delete`时产生的`undo log`，其回滚段类型为 update undo logs; 不仅在事务回滚时需要，在实现MVCC快照读时也需要」**

#### 3.7.2 Undo log清理类型

Undo log的清理也分为两种情况

- **「事务 rollback」**

  如果事务rollback，innodb 通过执行 undo log中的所有反向操作，实现事务回滚，随后就会删除该事务关联的所有 undo log 段。

- **「事务 commit」**

- - 对于 insert undo logs，事务回滚后，innodb会直接清除该事务关联的所有 undo log 段。
  - 对于 update undo logs，只有当前没有任何活跃事务存在时，innodb 的 purge 线程才会清理这些 undo log 段

#### 3.7.3 purge 线程

上述提到的 **「purge 线程，是一个周期运行的垃圾收集线程，主要用来收集 undo log 段」**。

- **「innodb 会将所有需要清理的任务添加到 purge 队列中，」**

  > 可以通过 innodb_max_purge_lag 配置项设定 purge 队列的大小

- **「purge 线程会在周期执行时，对 purge 队列中的任务进行清理，」**

  > 可以通过innodb_max_purge_lag_delay 配置项设定 purge 线程的执行周期间隔

- **「尽量缩短使用中每个事务的持续时间，可以让 purge 线程有更大概率回收已经没有存在必要的 undo log 段，从而尽量释放磁盘空间的占用」**

## **四、错误日志(error log)**

**「错误日志(error log)：记录mysql服务的启停时正确和错误的信息，还记录启动、停止、运行过程中的错误信息。」**

- **「指定错误日志文件」**

  **「在my.ini的[mysqld]下：添加代码：log-error=file_name.txt。」**

  ```
  log-error=E:\log-error.txt。
  ```

  **「如果没有指定file_name，则默认的错误日志文件为datadir目录下的 `hostname`.err ，hostname表示当前的主机名。」**

- **「查看错误日志位置」**

  ```
  show variables like 'log_error';
  ```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124974.png)

- **「错误日志的产生」**

- - **「MySQL 5.5.7之前」**

    刷新日志操作(如flush logs)会备份旧的错误日志(以_old结尾)，并创建一个新的错误日志文件并打开。

  - **「在MySQL 5.5.7之后」**

    执行刷新日志的操作时，错误日志**「会关闭并重新打开」**，如果错误日志不存在，则会先创建。

  - **「MySQL正在运行状态下」**

    **「在运行状态下删除错误日志后，不会自动创建错误日志，只有在刷新日志的时候才会创建一个新的错误日志文件」**。

## **五、查询日志**

**「查询日志分为一般查询日志和慢查询日志」**。通过查询是否超出变量**「long_query_time」**指定时间的值来判定的。

**「在超时时间内完成的查询是一般查询，可以将其记录到一般查询日志中，超出时间的查询是慢查询，可以将其记录到慢查询日志中。」**

- **「long_query_time」**

  ```
  # 指定慢查询超时时长，超出此时长的属于慢查询，会记录到慢查询日志中
  long_query_time = 10 
  # 定义一般查询日志和慢查询日志的输出格式，不指定时默认为file
  log_output={TABLE|FILE|NONE}  
  ```

- - **「TABLE：表示记录日志到表中」**
  - **「FILE：表示记录日志到文件中」**
  - **「NONE：表示不记录日志」**

### 5.1 一般查询日志(general log )

**「记录了服务器接收到的每一个查询或是命令」**，无论这些查询或是命令是否正确甚至是否包含语法错误，general log 都会将其记录下来 。

开启General logmysql服务器需要不断地记录日志，会产生一定的系统开销。 所有Mysql默认关闭一般查询日志。

- **「开启general log」**

  ```
  set global general_log=on;#为全局变量
  ```

- **「开启general log」**

  ```
  set global general_log=off;
  ```

- **「查看general log是否开启」**

  ```
  show global variables like 'general_log';
  ```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124908.png)

- **「设置日志文件路径」**

  默认是库文件路径下主机名加上.log

  ```
  set global general_log_file='/tmp/general.log';
  ```

- **「查看日志输出格式」**

  ```
  show variables like 'log_output';
  ```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124055.png)

### 5.2 慢查询日志(slow log)

**「慢查询日志记录执行时间超过long_query_time和没有使用索引的查询语句，并且只会记录执行成功的语句。」**

**「查询超出变量 long_query_time 指定时间值的为慢查询。不包含查询获取锁(包括锁等待)的时间」**。

- **「查看慢查询指定时间」**

  **「默认10s」**

  ```
  mysql> show variables like  'long_query_time';
  +-----------------+-----------+
  | Variable_name   | Value     |
  +-----------------+-----------+
  | long_query_time | 10.000000 |
  +-----------------+-----------+
  1 row in set, 1 warning (0.00 sec)
  ```

- **「查看慢查询的条数」**

  ```
  mysql> show status like "%slow_queries%";
  +---------------+-------+
  | Variable_name | Value |
  +---------------+-------+
  | Slow_queries  | 4     |
  +---------------+-------+
  1 row in set (0.00 sec)
  ```

  > 以上Slow_queries = 4 说明查询超过10秒的查询有4个

- **「启用慢查询日志」**

  1与ON等价，0与OFF等价。

  ```
  mysql> set @@global.slow_query_log=on;
  #或者  
  mysql> set @@global.slow_query_log=1;
  ```

- **「查看是否启用慢查询日志」**

  ```
  show variables like 'slow_query%';
  ```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124161.png)

- **「slow_query_log」**

```
slow_query_log={1|ON|0|OFF} 与  log_slow_queries={``yes``|no} 都是表示是否启用慢查询日志，两个同时变化。
```

- **「slow_query_log_file」**

  日志文件位置，默认路径为库文件目录下主机名加上-slow.log

**「【MySQL记录慢查询日志是在查询执行完毕且已经完全释放锁之后记录的】，因此慢查询日志记录的顺序和执行的SQL查询语句顺序可能会不一致（先执行完先记录）」**。

## **六、中继日志(relay log)**

**「主要作用：主从复制」**

**「【从服务器I/O线程】将主服务器的【二进制日志】读取过来记录到从服务器本地文件，然后【从服务器SQL线程】会读取relay-log日志的内容并应用到从服务器，从而使从服务器和主服务器的数据保持一致」**

- **「查看relay log配置」**

```
show variables like '%relay%';
```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124222.png)