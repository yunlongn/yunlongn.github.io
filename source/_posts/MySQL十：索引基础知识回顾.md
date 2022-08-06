title: MySQL十：索引基础知识回顾
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:10:00
---
## 1、索引简介

### 1.1 什么是索引

**索引是对数据库表中一列或多列的值进行排序的一种结构，可以大大提高MySQL的检索速度**。索引在MySQL中也叫做key，当表中的数据量越来越大时，索引对于查询性能的影响非常大。
<!--more-->
那索引具体是什么呢，找几个生活中实例比较一下就清晰了：

- 新华字典：索引就相当于字典的音序表，我们可以通过音序表，快速在几百页中定位到我们要查找的字。
- 书店书架：索引就相当于书店里面的书架上的标签，可以通过标签，快速从成千上万本书中找到我们需要的书籍。

由此可知，其实索引就是一个目录，**即数据库表的一个数据目录，帮助快速定位数据在磁盘中的位置，以此达到提高查询性能的目的**。

### 1.2 索引的优缺点

- **优点**

- - 索引减小了需要扫描的数据量，从而大大加快数据的检索速度（创建索引的最主要的原因）
  - 可以加速表与表的连接
  - 可以显著的减少查询中分组和排序的时间
  - 索引可以帮助服务器避免排序和创建临时表
  - 索引可以将随机IO变成顺序IO

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

索引既然有这么多优点，那为什么不对表中每个列都建一个索引呢，这样不是更加能提升性能吗，实际上这是不可取的，索引虽然有诸多优点，但是也有很多缺点

- **缺点**

- - 对表中的数据进行增、删、改的时候，索引也要动态的维护，降低了数据的写入速度
  - 随着数据量的增加，创建索引和维护索引要耗费时间也会越来越长，影响性能
  - 索引的存储需要占物理空间，每一个索引都要占用一定的物理空间

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 2、创建索引准则

基于以上索引的介绍，我们知道索引优缺点都很明显，**我们不能在表数据中所有的列都添加索引，需要根据具体场景选择创建索引的列与类型**。那么具体应该在那些列中添加索引，那些列中不能添加索引呢？

- **能创建索引的列**

- - 主键索引，在MySQL中，主键列会默认的当成唯一性索引
  - 在业务场景中被【**当成条件查询的列**】创建索引，可以提高查询效率
  - 外键索引，比如需要【**用于JOIN的列**】创建索引，可以提高连接的速度
  - 由于索引是已经排序的，所以在经常【**用于范围查询的列**】和需要【**排序的列**】创建索引，可以避免排序，提高查询效率

- **不能创建索引的列**

  > 以上几种情况的列，一般不建议创建索引，非但不能提高查询速度，反而增加索引后提高了数据的维护时间成本和空间成本。

- - 经常用于计算的列
  - 数据值很少或者大量重复的列
  - 大字段的列
  - 经常修改的列
  - 很少使用的字段

## 3、MySQL索引的创建与分类

### 3.1MySQL索引类型

MySQL索引的类型其实只有五种，但是我们经常会听到很多种不同的索引，那其实是在不同维度划分的类型：

- **存储结构维度划分**

  B Tree索引、Hash索引、B + Tree索引

- **应用层次维度划分**

  普通索引、唯一索引、主键索引、全文索引，空间索引

  > 空间索引基本不使用，这里不做介绍

- **索引键值类型维度划分**

  主键索引、辅助索引（二级索引）

- **数据存储和索引键值逻辑关系维度划分**

  聚集索引（聚簇索引）、非聚集索引（非聚簇索引）

- **索引组成维度划分**

  组合索引（复合索引）、单一索引

本文主要以**应用层次维度**来说明索引的分类，其他维度在后续文章中描述。

### 3.2MySQL索引的创建与删除

- **索引的创建**

  索引的创建方式有三种：建表时创建索引，已存在的表上直接创建索引，已存在的表上新增列并创建索引

- - 建表时创建索引

    ```
    CREATE TABLE 表名 (
        字段名1  数据类型 [完整性约束条件…],
        字段名2  数据类型 [完整性约束条件…],
        [NORMAL | UNIQUE | FULLTEXT | SPATIAL ]   INDEX | KEY
        [索引名]  (字段名[(长度)]  [ASC | DESC]) 
    );
    ```

  - 已存在的表上直接创建索引

    ```
    CREATE  [NORMAL | UNIQUE | FULLTEXT | SPATIAL ]  INDEX  索引名  ON 表名 (字段名[(长度)]  [ASC | DESC]) ;
    ```

  - 已存在的表上新增列并创建索引（修改表结构）

    ```
    ALTER TABLE 表名 ADD  [NORMAL | UNIQUE | FULLTEXT | SPATIAL ] INDEX 索引名 (字段名[(长度)]  [ASC | DESC]) ;
    ```

- **名词解释**

- - NORMAL | UNIQUE | FULLTEXT | SPATIAL

    可选参数，Normal 普通索引，Unique 唯一索引，Full Text 全文索引，SPATIAL 空间索引

  - INDEX | KEY

    同义词，作用相同，用来指定创建索引

  - ASC | DESC

    指定升序或降序的索引值存储

- **索引的删除**

  ```
  DROP INDEX 索引名 ON 表名字;
  ```

- **查看表结构**

  ```
  desc table_name;
  ```

- **查看生成表的SQL**

  ```
  show create table table_name;
  ```

- **查看索引结构信息**

  ```
  show index from  table_name;
  ```

- **查看SQL执行时间**

  ```
  set profiling = 1;
  select * from user where id=1; 
  show profiles;
  ```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127066.png)

### 3.3 普通索引

**最基本的索引类型，基于普通字段建立的索引，没有任何限制。**

**一张表可以创建多个普通索引，一个普通索引可以包含多个字段【组合索引】，允许数据重复，允许 NULL 值插入**

- 建表时创建索引

  ```
  CREATE TABLE `user` (
      `id` int(11) NOT NULL AUTO_INCREMENT ,
      `name` char(255) CHARACTER NOT NULL ,
      `idcard` char(255) CHARACTER NOT NULL ,
      `content` text CHARACTER NULL ,
      `time` int(10) NULL DEFAULT NULL ,
      PRIMARY KEY (`id`),
      INDEX index_name (name(length))
  )
  ```

- 已存在的表上直接创建索引

  ```
  CREATE INDEX index_name ON user (name(length))
  ```

- 已存在的表上新增列并创建索引（修改表结构）

  ```
  ALTER TABLE user ADD INDEX index_name ON (name(length))
  ```

**通过以上三种方式为User表的name字段创建普通索引时，可以看到，并没有使用NORMAL关键字，这是因为在创建普通索引时，NORMAL关键字是可以省略的，直接使用Index即可**。

### 3.4 唯一索引

与**普通索引**基本相同类似，区别在于：**唯一索引字段的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一**。在创建或修改表时追加唯一约束，就会自动创建对应的唯一索引。

- 建表时创建索引

  ```
  CREATE TABLE `user` (
      `id` int(11) NOT NULL AUTO_INCREMENT ,
      `name` char(255) CHARACTER NOT NULL ,
      `idcard` char(255) CHARACTER NOT NULL ,
      `content` text CHARACTER NULL ,
      `time` int(10) NULL DEFAULT NULL ,
      PRIMARY KEY (`id`),
      INDEX index_name (name(length))
  )
  ```

- 已存在的表上直接创建索引

  ```
  CREATE UNIQUE INDEX index_name ON user (idcard(length))
  ```

- 已存在的表上新增列并创建索引（修改表结构）

  ```
  ALTER TABLE user ADD UNIQUE index_name ON (idcard(length))
  ```

**通过以上三种方式为User表的idcard（身份证号码）字段创建唯一索引时，使用UNIQUE关键字**。

### 3.5 主键索引

**是一种特殊的唯一索引，一个表只能有一个主键，不允许有空值**。一般是在建表的时候同时创建主键索引，通过PRIMARY KEY关键字指定

```
CREATE TABLE `user` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `name` char(255) CHARACTER NOT NULL ,
    `idcard` char(255) CHARACTER NOT NULL ,
    `content` text CHARACTER NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    PRIMARY KEY (`id`)
)
```

### 3.6 组合索引

**一个组合索引包含两个或两个以上的列。遵循 mysql 组合索引的【最左前缀原则】，即使用 where 时条件要按照索引建立时字段的排列方式放置索引才会生效。**

```
CREATE INDEX index_name ON user (name,idcard);
```

### 3.7 全文索引

**主要用来查找文本中的关键字，而不是直接与索引中的值相比较。**

**fulltext索引更像是一个搜索引擎，一般配合match against操作使用，而不是简单的where语句的like参数匹配。目前只有char、varchar，text 列上可以创建全文索引**。

- 建表时创建索引

  ```
  CREATE TABLE `user` (
      `id` int(11) NOT NULL AUTO_INCREMENT ,
      `name` char(255) CHARACTER NOT NULL ,
      `idcard` char(255) CHARACTER NOT NULL ,
      `content` text CHARACTER NULL ,
      `time` int(10) NULL DEFAULT NULL ,
      PRIMARY KEY (`id`),
      FULLTEXT (content)
  )
  ```

- 已存在的表上直接创建索引

  ```
  CREATE FULLTEXT INDEX index_name ON user(content)
  ```

- 已存在的表上新增列并创建索引（修改表结构）

  ```
  ALTER TABLE user ADD FULLTEXT index_name ON (content)
  ```

**通过以上三种方式为user表的content字段创建全文索引时，使用FULLTEXT关键字**。

> 在数据量较大时候，现将数据放入一个没有全局索引的表中，然后再用CREATE index创建fulltext索引，要比先为一张表建立fulltext然后再将数据写入的速度快很多。