title: MySQL三：存储引擎
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:03:00
---
## **一、MySQL存储引擎概述**

**「数据库存储引擎是数据库底层软件组织，数据库管理系统（DBMS）使用数据引擎进行创建、查询、更新和删除数据」**。不同的存储引擎提供不同的存储机制、索引、锁等功能。许多数据库管理系统都支持多种不同的数据引擎。

在关系数据库中数据的存储是以表的形式存储的，所以**「存储引擎也可以称为表类型(Table Type，即存储和操作此表的类型)」**。
<!--more-->

- **「MySQL的存储引擎」**

  **「MySQL的存储引擎在体系架构中位于第三层，负责MySQL中的数据的存储和提取，是与文件打交道的子系统，它是根据MySQL提供的文件访问层抽象接口定制的一种文件访问机制。」**

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061121197.png)

从架构图中可以看出**「mysql支持多种存储引擎， 不同版本的mysql支持的引擎会有细微差别」**

- InnoDB：支持事务，具有提交，回滚和崩溃恢复能力，事务安全
- MyISAM：不支持事务和外键，访问速度快
- Memory：利用内存创建表，访问速度非常快，因为数据在内存，而且默认使用Hash索引，但是 一旦关闭，数据就会丢失
- Archive：归档类型引擎，仅能支持insert和select语句
- Csv：以CSV文件进行数据存储，由于文件限制，所有列必须强制指定not null，另外CSV引擎也不 支持索引和分区，适合做数据交换的中间表
- BlackHole: 黑洞，只进不出，进来消失，所有插入数据都不会保存
- Federated：可以访问远端MySQL数据库中的表。一个本地表，不保存数据，访问远程表内容。
- MRG_MyISAM：一组MyISAM表的组合，这些MyISAM表必须结构相同，
- Merge表本身没有数据， 对Merge操作可以对一组MyISAM表进行操作。

我本地使用的5.7.24版本，使用以下命令可以查看当前数据库支持的引擎信息：

```
show engines
```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061121194.png)

## **二 MySQL常用存储引擎**

- **「MySQL5.7支持的存储引擎包含」**

  InnoDB 、MyISAM 、BDB、MEMORY、MERGE、EXAMPLE、NDB Cluster、ARCHIVE、CSV、BLACKHOLE、FEDERATED等。

  **「其中InnoDB和BDB提供事务安全表，其他存储引擎是非事务安全表。」**

- **「MySQL默认存储引擎」**

  **「Mysql5.5之前的默认存储引擎是MyISAM，5.5之后改为InnoDB」**。通过以下命令可以查看默认的存储引擎

  ```
  show variables like '%default_storage_engine%';
  ```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**「可以在配置文件中设置default_storage_engine修改默认的存储引擎」**

- **「常用存储引擎的特性」**

  在MySQL中常用的存储引擎：【InnoDB】【MyISAM】【MEMORY】【 MERGE】【NDB】，它们之间的一些特细如下表：

  | 特点         | InnoDB            | MyISAM | MEMORY | MERGE | NDB    |
  | :----------- | :---------------- | :----- | :----- | :---- | :----- |
  | 存储限制     | 64TB              | 265TB  | RAM    | 没有  | 384 EB |
  | 事务安全     | 支持              |        |        |       |        |
  | 锁机制       | 行锁(适合高并发)  | 表锁   | 表锁   | 表锁  | 行锁   |
  | B树索引      | 支持              | 支持   | 支持   | 支持  | 支持   |
  | 哈希索引     |                   |        | 支持   |       |        |
  | 全文索引     | 支持(5.6版本之后) | 支持   |        |       |        |
  | 集群索引     | 支持              |        |        |       |        |
  | 数据索引     | 支持              |        | 支持   |       | 支持   |
  | 索引缓存     | 支持              | 支持   | 支持   | 支持  | 支持   |
  | 数据可压缩   |                   | 支持   |        |       |        |
  | 空间使用     | 高                | 低     | N/A    | 低    | 低     |
  | 内存使用     | 高                | 低     | 中等   | 低    | 高     |
  | 批量插入速度 | 低                | 高     | 高     | 高    | 高     |
  | 支持外键     | 支持              |        |        |       |        |

虽然mysql支持的存储引擎多种多样，但是**「基本上在一般的企业和应用中大多是使用的【InnoDB】【MyISAM】两种」**。因此我们在提到存储引擎的时候也都是默认描述的这两种，那么到底什么时候使用InnoDB？什么时候使用MyISAM呢？

## **三、InnoDB和MyISAM对比**

InnoDB和MyISAM是使用MySQL时最常用的两种引擎类型，我们重点来看下两者区别。

### 3.1 事务和外键

- InnoDB支持事务和外键，具有安全性和完整性，适合大量insert或update操作
- MyISAM不支持事务和外键，它提供高速存储和检索，适合大量的select查询操作

### 3.2 锁机制

- InnoDB支持行级锁，锁定指定记录。基于索引来加锁实现。
- MyISAM支持表级锁，锁定整张表。

### 3.3 索引结构

- InnoDB使用聚集索引（聚簇索引），索引和记录在一起存储，既缓存索引，也缓存记录。

  InnoDB中叶子结点中直接存储的是索引对应的数据，如下图：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061121148.png)

- MyISAM使用非聚集索引（非聚簇索引），索引和记录分开。

  MyISAM中叶节点的data域存放的是数据记录的地址，如下图：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061121163.png)

### 3.4 并发处理能力

- MyISAM使用表锁，会导致写操作并发率低，读之间并不阻塞，读写阻塞。
- InnoDB读写阻塞可以与隔离级别有关，可以采用多版本并发控制（MVCC）来支持高并发

### 3.5 存储文件

- InnoDB表对应两个文件，一个.frm表结构文件，一个.ibd数据文件。InnoDB表最大支持64TB；
- MyISAM表对应三个文件，一个.frm表结构文件，一个MYD表数据文件，一个.MYI索引文件。从MySQL5.0开始默认限制是256TB。

### 3.6 适用场景

- **「MyISAM」**

- - 不需要事务支持（不支持）
  - 并发相对较低（锁定机制问题）
  - 数据修改相对较少，以读为主
  - 数据一致性要求不高

- **「InnoDB」**

- - 需要事务支持（具有较好的事务特性）
  - 行级锁定对高并发有很好的适应能力
  - 数据更新较为频繁的场景
  - 数据一致性要求较高
  - 硬件设备内存较大，可以利用InnoDB较好的缓存能力来提高内存利用率，减少磁盘IO

## **四、InnoDB与MyISAM如何选择**

- 需要事务选择InnoDB
- 存在并发修改选择InnoDB
- 追求快速查询，且数据修改少，选择MyISAM
- 在绝大多数情况下，推荐使用InnoDB