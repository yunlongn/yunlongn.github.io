title: MMySQL(九)：MVCC能否解决幻读问题
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:09:00
---
**幻读【前后多次读取，数据总量不一致】**

> 同一个事务里面连续执行两次同样的sql语句，可能导致不同结果的问题，第二次sql语句可能会返回之前不存在的行。

**事务A执行多次读取操作过程中，由于在事务提交之前，事务B（insert/delete/update）写入了一些符合事务A的查询条件的记录，导致事务A在之后的查询结果与之前的结果不一致，这种情况称之为幻读**。
<!--more-->
- **MVCC能否解决幻读问题**

  首先可以明确的是，**MVCC在快照读的情况下可以解决幻读问题，但是在当前读的情况下是不能解决幻读的**。

## 1、快照读和当前读

mysql里面实际上有两种读取的方式：快照读和当前读，在之前的文章中[《MySQL(八)：MVCC多版本并发控制》](https://mp.weixin.qq.com/s?__biz=MzIxMDU5OTU1Mw==&mid=2247485973&idx=1&sn=5bf8d02d361bc36d32472aa5f2071e8e&scene=21#wechat_redirect)也有介绍，这里重新简单回顾一下。

- **快照读【Consistent Read】**

  **也叫普通读，读取的是记录数据的可见版本，不加锁，不加锁的普通select语句都是快照读，即不加锁的非阻塞读**。

  **快照读的执行方式是生成 ReadView，直接利用 MVCC 机制来进行读取，并不会对记录进行加锁**。

  如下语句：

  ```
  select * from table;
  ```

- **当前读**

  **也称锁定读【Locking Read】，读取的是记录数据的最新版本，并且需要先获取对应记录的锁**。如下语句：

  ```
  SELECT * FROM student LOCK IN SHARE MODE;  # 共享锁
  SELECT * FROM student FOR UPDATE; # 排他锁
  INSERT INTO student values ...  # 排他锁
  DELETE FROM student WHERE ...  # 排他锁
  UPDATE student SET ...  # 排他锁
  ```

## 2、MVCC能解决幻读问题的场景

**当我们在读取数据的时候是【快照读】的情况下是可以解决【幻读】的问题，其原理就是MVCC**。

下面使用案例说明：

假设表中有三条数据，以及有两个事物A/B，A读取数据，B插入数据

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

```
#事物A：
select name from user where id > 3;
#事物B：
insert into user valus('6','edwin');
```

执行过程

| 时间 | 事务A                                           | 事物C                                          |
| :--- | :---------------------------------------------- | :--------------------------------------------- |
| 1    | 开始事务                                        |                                                |
| 2    | 第一次查询：select name from user where id > 3; |                                                |
| 6    |                                                 | 开始事务                                       |
| 7    |                                                 | 执行插入：insert into user valus('6','edwin'); |
| 8    |                                                 | 提交事务                                       |
| 9    | 第二次查询：select name from user where id > 3; |                                                |
| 10   | 提交事务                                        |                                                |

**由于采用的是【快照读】的方式，在A事物开启时会产生一个版本快照，产生版本快照如下**：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127204.png)

**然后通过MVCC的【Read View】对版本快照中各个版本链中的数据进行可见性判断**，读取相应的数据版本。两次查询结果都是【id=4，5】两条数据。

> Read View具体可见性规则判断在之前的文章中[《MySQL(八)：MVCC多版本并发控制》](https://mp.weixin.qq.com/s?__biz=MzIxMDU5OTU1Mw==&mid=2247485973&idx=1&sn=5bf8d02d361bc36d32472aa5f2071e8e&scene=21#wechat_redirect)有详细的图文详解，这里就不再赘述。

**因此，即使事务B新插入了数据，由于已经生成了版本快照，也不会影响Read View的可见性规则判读，所以在【快照读】的情况下，使用MVCC不会产生幻读问题。**

## 3、MVCC不能解决幻读问题的场景

### 3.1、MVCC什么场景下不能解决幻读问题

**当我们在读取数据的时候是【当前读】的情况下，无法使用MVCC解决幻读问题**。

案例说明：还是先准备几条数据，

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

有两个事物A/B，A先读取数据，在修改数据，最后有读取数据，B插入数据，看看结果会什么

```
#事物A：
select name from user where id > 3;
#事物B：
insert into user valus('6','edwin');
#事物A：
update user set name = '彬' where id = 6;
```

执行过程

| 时间 | 事务A                                              | 事物C                                           |
| :--- | :------------------------------------------------- | :---------------------------------------------- |
| 1    | 开始事务                                           |                                                 |
| 2    | 第一次查询：select name from user where id > 3;    |                                                 |
| 6    |                                                    | 开始事务                                        |
| 7    |                                                    | 执行插入：insert into user valus ('6','edwin'); |
| 8    |                                                    | 提交事务                                        |
| 9    | 第二次查询：select name from user where id > 3;    |                                                 |
| 10   | 修改数据:update user set name = '彬' where id = 6; |                                                 |
| 11   | 第三次查询：select name from user where id > 3;    |                                                 |
| 12   | 提交事务                                           |                                                 |

- 在时间点为9的时候

  事务A的【第一次】与【第二次】查询结果与上面的快照读是一样的，基于MVCC两次查询结果都是【id=4，5】两条数据。

- 在时间点为10的时候

  事务A修改了事务B插入的数据，**由于update是当前读，所以此时会读取最新的数据(包括其他已经提交的事务)**。

- 在时间点为11的时候

  事务A执行【第三次】查询，是基于当前最新版本查询的，所以会查询到事务B插入的【id=6】的数据，一共会查询到三条数据【id=4，5，6】，与前两次查询结果不同，**从而产生了幻读。**

### 3.2、如何解决当前读的幻读问题

在可重复读(RR)的隔离级别下，执行当前读，

案例说明：还是使用上述的数据

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127163.png)

- 事务A，执行**当前读**，查询id>3的所有记录。
- 事务B，插入id=5的一条数据。

```
#事物A：
select name from user where id > 3 lock in share mode;
#事物B：
insert into user valus('6','edwin');
```

执行过程

| 时间 | 事务A                                                        | 事物B                                                        |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 1    | 开始事务                                                     |                                                              |
| 2    | 第一次查询：select name from user where id > 3 lock in share mode; |                                                              |
| 6    |                                                              | 开始事务                                                     |
| 7    |                                                              | 执行插入时发现，id>3的范围有**间隙锁**，插入阻塞，处于等待状态 |
| 8    | 第二次查询：select name from user where id > 3;              |                                                              |
| 9    | 提交事务                                                     |                                                              |
| 10   |                                                              | 事物A提交，间隙锁释放，执行插入：insert into user valus ('6','edwin'); |
| 11   |                                                              | 提交事务                                                     |

事务A在执行**当前读**【select ... lock in share mode】的时候，在【id=4，5】的记录上加了**共享锁**，并且在【id > 6】这个范围上也加了**间隙锁**，所以上图中的**事务B执行插入操作时被阻塞**了。所以事务A两次读取的数据是一样的。因此，在这种情况下是不会存在幻读问题。

- **总结**：

  RR隔离级别下，当前读执行如下语句时会带上锁之外，还会使用间隙锁+临键锁，锁住索引记录之间的范围，避免范围间插入记录，以**避免产生幻影行记录**，

  ```
  SELECT * FROM student LOCK IN SHARE MODE;  # 共享锁
  SELECT * FROM student FOR UPDATE; # 排他锁
  INSERT INTO student values ...  # 排他锁
  DELETE FROM student WHERE ...  # 排他锁
  UPDATE student SET ...  # 排他锁
  ```

- 注意

  **这种方式不能解决3.1中的幻读问题，因为在3.1中事务A执行修改数据，获取锁之前，已经读取到了事务B插入的数据，并且已经记录到Undo日志中**。