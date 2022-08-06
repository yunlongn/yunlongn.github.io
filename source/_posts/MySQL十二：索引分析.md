title: MySQL(十二)：索引分析
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:12:00
---
数据库优化是一个很常见的面试题，下面就针对这一问题详细聊聊如何进行索引与sql的分析与优化。

## **一、执行计划（EXPLAIN）**

MySQL 提供了一个 EXPLAIN 命令，它**「可以对 sql语句进行分析，并输出sql执行的详细信息」**，可以让我们有针对性的优化。例如：
<!--more-->
```
explain select * from student  where id > 2;
```

这里需要注意一下版本差异

- **「MySQL 5.6.3」**

  MySQL 5.6.3以前只能 EXPLAIN SELECT ；MYSQL 5.6.3以后可以 EXPLAIN SELECT，UPDATE，DELETE

- **「MySQL 5.7」**

  MySQL 5.7以前想要显示 partitions 需要使用 explain partitions 命令；想要显示filtered 需要使用 explain extended 命令。在5.7版本后，默认explain直接显示partitions和filtered中的信息。

### 1.1执行计划详解

**「在使用索引的时候首先应该学会分析SQL的执行，使用EXPLAIN关键字可以模拟优化器执行SQL查询语句，可以知道MySQL是如何处理SQL语句」**。

使用格式：

```
#explain sql语句  如下：
explain select * from student  where id > 2;
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

从执行计划输出的结果可以看出，它有很多的字段，每个字段都有自己的含义

- **「id」**

  **「选择标识符」**：在一个查询语句中每个【SELECT】关键字都对应一个唯一的 id。两种例外的情况：

- - **「id相同」**优化器对子查询做了**「半连接（semi-jion）优化」**时，两个查询的 id 是一样的

    ```
    explain select * from student  where id in(select id from student  where id > 1);
    ```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- **「id为null」**

  ```
  explain select * from student  union select * from student  where id > 1;
  ```

  因为**「union会对结果去重，内部创建了一个 <union1,2> 名字的临时表，把查询 1 和查询 2 的结果集都合并到这个临时表中，利用唯一键进行去重，这种情况下查询 id 就为 NULL」**。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- **「select_type」**

  **「查询的类型」**，常用的值如下：

  | 查询的类型           | 类型含义                                                     |
  | :------------------- | :----------------------------------------------------------- |
  | SIMPLE               | 简单的select查询，不包含子查询或union查询，是最常见的。      |
  | PRIMARY              | 若查询中包含有子查询，最外层查询会别标记为PRIMARY            |
  | UNION                | 若第二个SELECT出现在UNION之后，则被标记为UNION；若UNION包含在FROM子句的子查询中,外层SELECT将被标记为：DERIVED |
  | SUBQUERY             | 在SELECT或WHERE列表中包含了子查询                            |
  | DERIVED              | 在FROM列表中包含的子查询被标记为DERIVED(衍生);MySQL会递归执行这些子查询, 把结果放在临时表里。 |
  | UNION RESULT         | 从UNION表获取结果的SELECT                                    |
  | DEPENDENT SUBQUERY   | 在SELECT或WHERE列表中包含了子查询,子查询基于外层             |
  | UNCACHEABLE SUBQUREY | 无法被缓存的子查询                                           |

- **「table」**

  输出结果集的表，即查询的表名

- **「partitions」**

  匹配的分区

- **「type」**

  表示存储引擎查询数据时采用的方式。它**「可以判断出查询是全表扫描还是基于索引的部分扫描」**。

  常用属性值如下，从上至下效率依次增强。

- - ALL：表示全表扫描，性能最差。
  - index：表示基于索引的全表扫描，先扫描索引再扫描全表数据。
  - range：表示使用索引范围查询。使用>、>=、<、<=、in等等。
  - ref：表示使用非唯一索引进行单值查询。
  - eq_ref：一般情况下出现在多表join查询，表示前面表的每一个记录，都只能匹配后面表的一 行结果。
  - const：表示使用主键或唯一索引做等值查询，常量查询。
  - NULL：表示不用访问表，速度最快。

- **「possible_keys」**

  表示在某个查询语句中，对某个表执行单表查询时**「可能用到的索引列表」**

- **「key」**

  表示在某个查询语句中，列表示**「实际用到的索引」**有哪些。

- **「key_len」**

  表示查询使用索引的字节数量。可以判断是否全部使用了组合索引。

  > 如果键是 NULL，则长度为 NULL。**「使用的索引的长度」**。在不损失精确性的情况下，长度越短越好 。

- **「ref」**

  当使用索引列等值匹配的条件去执行查询时，ref 列展示**「与索引列作等值匹配的对象」**。

- **「rows」**

  **「扫描出的行数(估算的行数)」**，

- - 如果查询优化器决定使用全表扫描的方式对某个表执行查询时，rows 列就代表预计需要扫描的行数；
  - 如果使用索引来执行查询时，rows 列就代表预计扫描的索引记录行数。

- **「filtered」**

  按表条件过滤的行百分比

- - 如果是全表扫描，filtered 值代表满足 where 条件的行数占表总行数的百分比
  - 如果是使用索引来执行查询，filtered 值代表从索引上取得数据后，满足其他过滤条件的数据行数的占比。

- **「Extra」**

  Extra 是 EXPLAIN 输出中另外一个很重要的列，各种操作都会在Extra提示相关信息，常见几种如下：

- - Using where：表示查询需要通过索引回表查询数据。

  - Using index：表示查询需要通过索引，索引就可以满足所需数据。

  - Using filesort：表示查询出来的结果需要额外排序，

    > 数据量小在内存排序，数据量大在磁盘排序，因此有Using filesort 建议优化。

  - Using temprorary：查询使用到了临时表，一般出现于去重、分组等操作。

## **二、回表查询**

在之前[《索引基本原理》](https://mp.weixin.qq.com/s?__biz=MzIxMDU5OTU1Mw==&mid=2247486030&idx=1&sn=ffd84e5aa3b0481d38110eb40d557dea&scene=21#wechat_redirect) 中提到InnoDB索引有聚簇索引和辅助索引。

- 聚簇索引的叶子节点存储行记录，InnoDB必须要有，且只有一个。
- 辅助索引的叶子节点存储的是主键值和索引字段值

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061128841.png)

由上图可知：**「通过辅助索引无法直接定位行记录，通常情况下，需要扫两遍索引树。先通过辅助索引定位主键值，然后再通过聚簇索引定位行记录，即回表查询」**。性能比扫一遍索引树低。

## **三、覆盖索引**

索引覆盖：**「只需要在一棵索引树上就能获取SQL所需的所 有列数据，无需回表，速度更快」**

覆盖索引形式：，搜索的索引键中的字段恰好是查询的字段

实现索引覆盖最常见的方法就是：将被查询的字段，建立到组合索引。

## **四、最左前缀原则**

在之前[《索引基本原理》](https://mp.weixin.qq.com/s?__biz=MzIxMDU5OTU1Mw==&mid=2247486030&idx=1&sn=ffd84e5aa3b0481d38110eb40d557dea&scene=21#wechat_redirect) 中提到组合索引的概念，在组合索引的使用中最关键的就是最左前缀原则。

**「组合索引使用时遵循最左前缀原则，最左前缀顾名思义，就是最左优先，即查询中使用到最左边的列， 那么查询就会使用到索引，如果从索引的第二列开始查找，索引将失效」**。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## **五、索引与排序**

### 5.1排序方式

MySQL查询支持filesort和index两种方式的排序，

- filesort是先把结果查出，然后在缓存或磁盘进行排序 操作，效率较低。
- index是指利用索引自动实现排序，不需另做排序操作，效率会比较高。

### 5.2 排序方式的选择

- **「使用index方式的排序的场景」**

- - ORDER BY 子句索引列组合满足索引最左前列

    ```
    explain select id from user order by id; //对应(id)、(id,name)索引有效
    ```

  - WHERE子句+ORDER BY子句索引列组合满足索引最左前缀

    ```
     #对应(age,name)组合索引
    explain select id from user where age=18 order by name;
    ```

- **「使用filesort方式的排序的场景」**

- - 对索引列同时使用了ASC和DESC

    ```
     #对应(age,name)组合索引
    explain select id from user order by age asc,name desc;
    ```

  - WHERE子句和ORDER BY子句满足最左前缀，但where子句使用了范围查询（例如>、<、in 等）

    ```
     #对应(age,name)组合索引
    explain select id from user where age>10 order by name;
    ```

  - ORDER BY或者WHERE+ORDER BY索引列没有满足索引最左前缀

    ```
     #对应(age,name)组合索引
    explain select id from user order by name;
    ```

  - 使用了不同的索引，MySQL每次只采用一个索引，ORDER BY涉及了两个索引

    ```
    #对应(name)、(age)两个索引
    explain select id from user order by name,age;
    ```

  - WHERE子句与ORDER BY子句，使用了不同的索引

    ```
    #对应(name)、(age)索引
    explain select id from user where name='tom' order by age;
    ```

  - WHERE子句或者ORDER BY子句中索引列使用了表达式，包括函数表达式

    ```
    #对应(age)索引
    explain select id from user order by abs(age);
    ```

### 5.3排序算法

filesort有两种排序算法：双路排序和单路排序。

- 双路排序：需要两次磁盘扫描读取，得到最终数据。第一次将排序字段读取出来，然后排序；第二 次去读取其他字段数据。
- 单路排序：从磁盘查询所需的所有列数据，然后在内存排序将结果返回。
- 如果查询数据超出缓存 sort_buffer，会导致多次磁盘读取操作，并创建临时表，最后产生了多次IO，反而会增加负担。
- 解决方案：少使用select *；增加sort_buffer_size容量和max_length_for_sort_data容量。

> 如果Explain分析SQL时Extra属性显示Using filesort，表示使用了filesort排序方式，需要优化。如果Extra属性显示Using index时，表示覆盖索引，所有操作在索引上完成。