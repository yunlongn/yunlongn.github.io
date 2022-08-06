title: MySQL(八)：读懂MVCC多版本并发控制
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:08:00
---

mysql在并发的情况下，会引起脏读，幻读，不可重复读等一系列的问题，为解决这些问题，引入了mvcc的机制。本文就详细看看mvcc是怎么解决脏读，幻读等问题的。

## 1、 数据库事务

### 1.1 事务

**事务是操作数据库的最小单元，将【多个任务作为单个逻辑工作单元】执行的一系列数据库操作，他们作为一个整体一起向数据库提交，要么都执行、要么都不执行。**
<!--more-->
> 大白话解释：
>
> 事务就是当要完成一件事件，这件事又包含多个任务的时候，只有当所有的任务都执行成功，则认为这个事情是成功；只要有其中一个任务没有执行成功，则认为这件事执行失败，其他的执行成功的任务也要回滚到未执行的状态。

- **开启事务**【开始记录一个事情中的多个任务】
- **执行事务**【正常情况下，一条语句就是一个任务】
- **提交事务**【成功】| **回滚事务**【失败】

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127996.png)

事务的作用：**保证数据的最终一致性**。

### 1.2 事务四大特性

**事务四大特性即ACID：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）**。

- **原子性（Atomicity）**

  事务是操作数据库的最小单元，作为一个整体被执行，包含一个事务中的所有操作要么全部都执行，要么全部失败回滚。

- **一致性（Consistency）**

  事务必须使数据库从一个一致性状态转换到另一个一致性状态，即在事务开始之前和事务结束以后，数据不会被破坏，保持一致性。

  > 假如A账户给B账户转100块钱，不管事务是否成功，A账户给B账户的总金额是不变的。

- **隔离性（Isolation）**

  当多个事务并发访问数据库时，一个事务不应该被其他事务干扰，多个并发事务之间是相互隔离的。

- **持久性（Durability）**

  事务一旦完成后被提交，该事务对数据库所作的操作更改，将持久地保存在数据库之中。

### 1.3 并发下的事务问题

虽然事务能保持数据最终一致性，但是在并发下执行事务，发会引起**脏读、不可重复读、幻读**等问题。

- **脏读【读取未提交数据】**

  **如果一个事务读取到了另一个未提交事务修改过的数据，称发生了脏读**。

  一般事务的脏读都是拿转账的案例说明，这里也转账和取款为案例：

  | 时间 | 事务A：转账                                        | 事务B:取款                                 |
  | :--- | :------------------------------------------------- | :----------------------------------------- |
  | 1    |                                                    | 开始事务                                   |
  | 2    | 开始事务                                           |                                            |
  | 3    |                                                    | 查询账户余额为10000元                      |
  | 4    |                                                    | 执行取款操作，取款3000元，余额更改为7000元 |
  | 5    | 查询账户余额为7000元（产生脏读）                   |                                            |
  | 6    |                                                    | 取款失败，回滚事务，余额还原为10000元      |
  | 7    | 转入5000元，余额被更改为12000元（脏读的7000+5000） |                                            |
  | 8    | 提交事务                                           |                                            |

  > 从上述执行过程的结果，最后账户余额为12000元，但是实际上B取款失败，余额为10000，加上A转入的5000元，账户最终的余额应该为15000元，平白无故少了3000元，这就是脏读。银行肯定是不允许这种事情发生的，不然就没人敢在银行存钱了......

- **不可重复读【前后多次读取，数据内容不一致】**

  **同一个事务内，前后多次读取，读取到的数据内容不一致，称之为不可重复读**。

  还是以转账的案例：

  | 时间 | 事务A：查询                     | 事务B:取款                                 |
  | :--- | :------------------------------ | :----------------------------------------- |
  | 1    | 开始事务                        |                                            |
  | 2    | 第一次查询，账户的余额为10000元 |                                            |
  | 3    |                                 | 开始事务                                   |
  | 4    |                                 | 执行取款操作，取款3000元，余额更改为7000元 |
  | 5    |                                 | 提交事务                                   |
  | 6    | 第二次查询，账户的余额为7000元  |                                            |
  | 7    | 提交事务                        |                                            |

  > 从上述案例描述中可以看出，事务A执行的过程中，事务B修改了账户余额，导致事务A中的两次查询结果不一致，这就是不可重复读，对于事务A而言莫名其妙的余额变少了，那肯定不干......

- **幻读【前后多次读取，数据总量不一致】**

  **事务A执行多次读取操作过程中，由于在事务提交之前，事务B（insert/delete/update）写入了一些符合事务A的查询条件的记录，导致事务A在之后的查询结果与之前的结果不一致，这种情况称之为幻读**。

  以student表中的数据为例：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

依次执行下面这两个语句

```
#查询语句
select * from student  where id > 2;
#写入语句
insert into student(id,c_id,name,sex,score) value(6,2,'吕布','男',89);
```

| 时间 | 事务A：读取                           | 事务B:写入                        |
| :--- | :------------------------------------ | :-------------------------------- |
| 1    | 开始事务                              |                                   |
| 2    | 第一次执行查询语句，结果为3条数据结果 |                                   |
| 3    |                                       | 开始事务                          |
| 4    |                                       | 执行写入语句，插入一条ID为6的数据 |
| 5    |                                       | 提交事务                          |
| 6    | 第二次执行查询语句，结果为4条数据结果 |                                   |
| 7    | 提交事务                              |                                   |

> 从上述案例描述中可以看出，事务A在前后两次执行的过程中，由于事务B插入了满足查询语句的数据，导致事务A两次查询结果的总数不一样，这就是幻读。

- **总结**

  一般我们再理解幻读与不可重复读的时候，容易混淆，其实只需要分清一点就可以，

  **一般而言：幻读是指查询数据的【条数总量】不一致，不可重复读是指查询数据的数据内容不一致**。

### 1.4 事务的四大隔离级别

数据库设计了四种隔离级别：**串行化(Serializable)、可重复读(Repeatable read)、读已提交(Read committed)、读未提交(Read uncommitted)**，用来解决并发事务存在的**脏读、不可重复读、幻读**等问题。

- **读未提交(Read uncommitted)**

  **在读未提交的隔离级别下，所有事务能够读取【其他事务未提交】的数据。**

  读取其他事务未提交的数据，会造成脏读。因此在该种隔离级别下，不能解决脏读、不可重复读和幻读。

- **读已提交(Read committed)**

  **在读已提交的隔离级别下，所有事务只能读取【其他事务已经提交】的数据**。Oracle和SQL Server的默认的隔离级别。

  读已提交能够解决脏读的现象，但是还是会有**不可重复读、幻读**的问题

  > 读已提交会有一个事务的前后多次的查询中却返回了不同内容的数据的现象。

- **可重复读(Repeatable read)**

  **在可重复读的隔离级别下，限制了读取数据的时候，不可以进行修改，所有事务前后多次的读取到的数据内容是不变的**。mysql的默认事务隔离级别

  这种隔离级别解决了**重复读**的问题，但是读取范围数据的时候，是可以add数据的，所以还是会造成某个事务前后多次读取到的数据总量不一致的现象，从而产生幻读。

  > 针对以上问题，一般我们也可以使用**间隙锁**和**临键锁**来解决幻读问题，这个以后再讲

- **串行化(Serializable)**

  **事务最高的隔离级别，在串行化的隔离级别下，所有的事务顺序执行，不存在任何冲突，可以避免脏读、不可重复读与幻读所有并发问题**。

  但是串行化的隔离级别，会导致大量的操作超时和锁竞争，从而大大降低数据库的性能，一般不使用这样事务隔离级别。

**四种隔离级别存在的并发问题如下：**

【 ×】表示未解决，【√】表示已解决

| 隔离级别                   | 脏读 | 不可重复读 | 幻读 |
| :------------------------- | :--- | :--------- | :--- |
| 读未提交(Read uncommitted) | ×    | ×          | ×    |
| 读已提交(Read committed)   | √    | ×          | ×    |
| 可重复读(Repeatable read)  | √    | √          | ×    |
| 串行化(Serializable)       | √    | √          | √    |

## 2、MVCC基础概念

> 数据库通过**加锁**，可以实现事务的隔离性，**串行化隔离级别就是加锁实现的**，但是加锁会**降低数据库性能**。
>
> 因此，数据库引入了**MVCC多版本并发控制**，在读取数据不用加锁的情况下，实现读取数据的同时可以修改数据，修改数据时同时可以读取数据。

### 2.1 什么是MVCC

**MVCC(Mutil-Version Concurrency Control)，多版本并发控制。是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问。用于支持读已提交(RC）和可重复读(RR）隔离级别的实现**。

> **MVCC**在**MySQL InnoDB**引擎中的实现主要是为了在处理读-写冲突时提高数据库并发性能，记录读已提交和可重复读这两种隔离级别下事务操作版本连的过程。

- **数据库并发场景一般有三种：**

- - **读-读**：不存在任何问题，不需要并发控制
  - **读-写**：有线程安全问题，可能会造成事务隔离性问题，可能会有脏读，幻读，不可重复读
  - **写-写**：有线程安全问题，可能会存在更新丢失问题。

- MVCC主要是用来解决【**读-写**】冲突的**无锁并发控制**，可以解决以下问题：

- - **在并发读写数据时，可以做到在读操作时不用阻塞写操作，写操作不用阻塞读操作，提高数据库并发读写的性能**。
  - **可以解决脏读，幻读，不可重复读等事务隔离问题，但不能解决【写-写】引起的更新丢失问题**。

- **MVCC与锁的组合**：

  **一般数据库中都会采用以上MVCC与锁的两种组合来解决并发场景的问题，以此最大限度的提高数据库性能**。

- - **MVCC + 悲观锁**MVCC解决读-写冲突，悲观锁解决写-写冲突。
  - **MVCC + 乐观锁**MVCC解决读-写冲突，乐观锁解决写-写冲突。

> 通过上述描述，MVCC的作用可以概括为就是为了解决【读写冲突】，提高数据库性能的，而MVCC的实现又依赖于六个概念：【隐式字段】【undo日志】【版本链】【快照读和当前读】【读视图】。

### 2.2 隐式字段

**在InnoDB存储引擎，针对每行记录都有固定的两个隐藏列【DB_TRX_ID】【DB_ROLL_PTR】以及一个可能存在的隐藏列【DB_ROW_ID】**。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

| 隐式字段    | 描述                                                         | 是否必须存在 |
| :---------- | :----------------------------------------------------------- | :----------- |
| DB_TRX_ID   | 事物Id，也叫事物版本号，占用6byte的标识，**事务开启之前，从数据库获得一个自增长的事务ID，用其判断事务的执行顺序** | 是           |
| DB_ROLL_PTR | 占用7byte，**回滚指针，指向这条记录的上一个版本的undo log记录，存储于回滚段（rollback segment）中** | 是           |
| DB_ROW_ID   | 隐含的自增ID（隐藏主键），如果表中没有主键和非NULL唯一键时，则会生成一个**单调递增的行ID作为聚簇索引** | 否           |

表中的数据会因此分为两种形式：

- **有主键或唯一非空字段**

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- **没有主键且没有唯一非空字段**

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 2.3 undo日志

**一种用于撤销回退的日志，在事务开始之前，会先记录存放到 Undo 日志文件里，备份起来，当事务回滚时或者数据库崩溃时用于回滚事务**。undo日志的详细介绍在之前的[《MySQL(七)：一文详解六大日志》](https://mp.weixin.qq.com/s?__biz=MzIxMDU5OTU1Mw==&mid=2247485806&idx=1&sn=ed2de04421546b4187a8bbf9da821648&scene=21#wechat_redirect)中有详细介绍。

**undo日志的主要作用是事务回滚和实现MVCC快照读**。

**undo log日志分为两种**：

- **insert undo log****代表事务在`insert`新记录时产生的`undo log`, 仅用于事务回滚，并且在事务提交后可以被立即丢弃**。
- **update undo log****事务在进行`update`或`delete`时产生的`undo log`; 不仅在事务回滚时需要，在实现MVCC快照读时也需要**；所以不能随便删除，只有在快速读或事务回滚不涉及该日志时，对应的日志才会被清理线程统一清除。

MVCC实际上是使用的`update undo log` 实现的快照读。

> **InnoDB 并不会真正地去开辟空间存储多个版本的行记录，只是借助 undo log 记录每次写操作的反向操作。所以B+ 索引树上对应的记录只会有一个最新版本，InnoDB 可以根据 undo log 得到数据的历史版本，从而实现多版本控制。**

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127029.png)

### 2.4 版本链

> 一致性非锁定读是通过 **MVCC** 来实现的。但是MVCC 没有一个统一的实现标准，所以各个存储引擎的实现机制不尽相同。InnoDB 存储引擎中 MVCC 的实现是通过 **undo log** 来完成的

**当事务对某一行数据进行改动时，会产生一条Undo日志，多个事务同时操作一条记录时，就会产生多个版本的Undo日志，这些日志通过回滚指针（DB_ROLL_PTR）连成一个链表，称为版本链**。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

> 只要有事务写入数据时，就会产生一条对应的 undo log，一条 undo log 对应这行数据的一个版本，当这行数据有多个版本时，就会有多条 undo log 日志，undo log 之间通过回滚指针（DB_ROLL_PTR）连接，这样就形成了一个 undo log 版本链。

### 2.5 快照读和当前读

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

### 2.6 读视图【Read View】

**Read View提供了某一时刻事务系统的快照，主要是用来做`可见性`判断, 里面保存了【对本事务不可见的其他活跃事务】**。

**当事务在开始执行的时候，会产生一个读视图（Read View），用来判断当前事务可见哪个版本的数据，即可见性判断**。

**实际上在innodb中，每个SQL语句执行前都会生成一个Read View**。

#### 2.6.1 读视图的四个属性

MySQL`5.7`源码中对`Read View`定义了四个属性，如下：

```
class ReadView {
	private:
		/** The read should not see any transaction with trx id >= this
		value. In other words, this is the "high water mark". */
		trx_id_t	m_low_limit_id;

		/** The read should see all trx ids which are strictly
		smaller (<) than this value.  In other words, this is the
		low water mark". */
		trx_id_t	m_up_limit_id;

		/** trx id of creating transaction, set to TRX_ID_MAX for free
		views. */
		trx_id_t	m_creator_trx_id;

		/** Set of RW transactions that was active when this snapshot
		was taken */
		ids_t		m_ids;

		/** The view does not need to see the undo logs for transactions
		whose transaction number is strictly smaller (<) than this value:
		they can be removed in purge if not needed by other views */
		trx_id_t	m_low_limit_no;

		/** AC-NL-RO transaction view that has been "closed". */
		bool		m_closed;

		typedef UT_LIST_NODE_T(ReadView) node_t;

		/** List of read views in trx_sys */
		byte		pad1[64 - sizeof(node_t)];
		node_t		m_view_list;
};
```

- **creator_trx_id**

  创建当前read view的事务ID

- **m_ids**

  当前系统中所有的活跃事务的 id，活跃事务指的是当前系统中开启了事务，但还没有提交的事务;

- **m_low_limit_id**

  表示在生成ReadView时，当前系统中活跃的读写事务中最小的事务id，即m_ids中的最小值。

- **m_up_limit_id**

  当前系统中事务的 id 值最大的那个事务 id 值再加 1，也就是系统中下一个要生成的事务 id。

**ReadView 会根据这 4 个属性，结合 undo log 版本链，来实现 MVCC 机制，决定一个事务能读取到数据那个版本**。

> 假设现在有事务 A 和事务 B 并发执行，事务 A 的事务 id 为 10，事务 B 的事务 id 为 20。
>
> 事务A的ReadView ：m_ids=[10,20]，m_low_limit_id=10，m_up_limit_id=21，creator_trx_id=10。
>
> 事务B的ReadView ：m_ids=[10,20]，m_low_limit_id=10，m_up_limit_id=21，creator_trx_id=20。

#### 2.6.2 读视图可见性判断规则

将Read View中的活跃事务Id按照大小放在坐标轴上表示的话，如下图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

当一个事务读取某条数据时，会**通过DB_TRX_ID【Uodo日志的事务Id】在坐标轴上的位置**来进行可见性规则判断，如下：

- **DB_TRX_ID < m_low_limit_id**

  表示DB_TRX_ID对应这条数据【Undo日志】是在当前事务开启之前，其他的事务就已经将该条数据修改了并提交了事务(事务的 id 值是递增的)，所以当前事务【开启Read View的事务】能读取到。

- **DB_TRX_ID >= m_up_limit_id**

  表示在当前事务【creator_trx_id】开启以后，有新的事务开启，并且新的事务修改了这行数据的值并提交了事务，因为这是【creator_trx_id】后面的事务修改提交的数据，所以当前事务【creator_trx_id】是不能读取到的。

- **m_low_limit_id =< DB_TRX_ID < m_up_limit_id**

- - **DB_TRX_ID  在 m_ids 数组中**

-   表示DB_TRX_ID【写Undo日志的事务】 和当前事务【creator_trx_id】是在同一时刻开启的事务

- - - **DB_TRX_ID  不等于creator_trx_id**

      

      **DB_TRX_ID事务修改了数据的值，并提交了事务，所以当前事务【creator_trx_id】不能读取到。**

      

    - **DB_TRX_ID  等于creator_trx_id**

- ​      表明数据【Undo日志】 是自己生成的，因此是**可见**的

- - **DB_TRX_ID  不在 m_ids 数组中**

    表示的是在当前事务【creator_trx_id】开启之前，其他事务【DB_TRX_ID】将数据修改后就已经提交了事务，所以当前事务能读取到。

#### 2.6.3 读视图可见性判断规则案例说明

了解了读视图可见性判断规则，下面通过一个场景案例图解的方式来详细逐条验证上述规则。一般来说，我们的行数据结构都为一下模式：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127038.png)

假设有一个事物【DB_TRX_ID = 10】在表中插入了一条数据，则它的数据结构为为：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127017.png)

- 【第一步】：假设现在有事务 A【DB_TRX_ID = 20】 和事务 B 【DB_TRX_ID = 30】并发执行

  ```
  #事物A：
  select name from user where id = 1;
  #事物B：
  update user set name = 'edwin' where id = 1;
  ```

  事物开始后分别生成ReadView

- - 事务A的ReadView ：m_ids=[20,30]，m_low_limit_id=20，m_up_limit_id=31，creator_trx_id=20。
  - 事务B的ReadView ：m_ids=[20,30]，m_low_limit_id=20，m_up_limit_id=31，creator_trx_id=30。

- 【第二步】：事物A开启事物之后通过版本链**第一次**读取数据，版本链中的DB_TRX_ID = 10，小于事物A的【DB_TRX_ID = 20】，说明DB_TRX_ID = 10这条数据是事物A开启之前就已经写入，并提交了事物，所以事物A可以读取到。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- 【第三步】：事务 B 【DB_TRX_ID = 30】修改数据，将name修改为Edwin，修改后写入Undo Log日志，**此时还没有提交事务B**。示意图如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- 【第四步】：事务A【DB_TRX_ID = 20】**第二次**去读取数据

  **在 undo log版本链中，数据最新版本的事务id为30，这个值处于事务A的 ReadView 里 m_low_limit_id 和 m_up_limit_id 并且存在于m_ids 数组中，表示这个版本的数据是和自己同一时刻启动的事务修改的，因此这个版本的数据，数据 A 读取不到**。

此时需要沿着 undo log 的版本链向前找，接着会找到该行数据的上一个版本db_trx_id=10，由于db_trx_id=10小于 m_low_limit_id的值，因此事务 A 能读取到该版本的值，即事务 A 读取到的值是星之码。

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127069.png)

- 【第五步】：现在事务 B 提交，此时系统中活跃的事务只有事物A，事物A**第三次**读取，读取到内容就有两种可能性：

  > 这里留一个问题一：造成这两种情况的原因是什么？
  >
  > 我们留到本文第三节【不同隔离级别MVCC实现原理】中说明，继续案例

- - **读已提交（RC）隔离级别：读取到是事物B提交的Edwin**。
  - **可重复读（RR）隔离级别：读取到是原始数据提交的星河之码**。

- 【第六步】：新的事物C【DB_TRX_ID = 40】修改数据，将name修改为彬

```
#事物C：
update user set name = '彬' where id = 1;
```

执行脚本前生成的ReadView如下，执行脚本后，提交事物C。

**事务C的ReadView ：m_ids=[20,40]，m_low_limit_id=20，m_up_limit_id=41，creator_trx_id=40**。

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127042.png)

- 【第七步】：事务 A【DB_TRX_ID = 20】**第四次**读取数据，

  此时由于事物A，由于事物A的m_up_limit_id=31，而日志中的DB_TRX_ID=40，根据可见性判断规则可以知到，事物A不能读取到DB_TRX_ID=40的记录，按照版本链的DB_POLL_PTR继续往上找，找到DB_TRX_ID=30的记录，虽然30在事物A的的m_ids=[20,30]，但是**DB_TRX_ID=30不等于事物A的creator_trx_id=20**，所以还是不能读取，继续往上找，最终读取到了DB_TRX_ID=10的记录，name=星河之码

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

> 实际上，这里事务A在不同场景下也是可以读取到DB_TRX_ID=40得数据的。
>
> 这里也留一个问题二：在什么场景下能够读取到DB_TRX_ID=40得数据name=彬呢？
>
> 我们留到本文第三节【不同隔离级别MVCC实现原理】中说明，继续案例

- 【第八步】：事务 A【DB_TRX_ID = 20】开始修改数据，将name 修改为 '法外狂徒张三'

  ```
  #事物A：
  update user set name = '法外狂徒张三' where id = 1;
  ```

  此时事务A还没有提交，但是已经写入了Undo 日志，新的版本链如下

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127345.png)

- 【第九步】：事务 A**第五次**读取数据

  由于Undo日志中的最新数据DB_TRX_ID=20等于事物A的creator_trx_id=20，说明是自己修改的数据，可以查到，name=法外狂徒张三

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127351.png)

通过以上九个步骤图解的方式，对读视图可见性判断规则做了分析，通过ReadView 和 undo log分析了MVCC 的实现原理，接下来结合事务的隔离级别，看看MVCC是怎么读取数据的。

## 3、不同隔离级别MVCC实现原理

### 3.1 MVCC实现原理

通过上述对【Read View】的分析可以总结出：**InnoDB 实现MVCC是通过` Read View与Undo Log` 实现的，Undo Log 保存了历史快照，形成版版本链，Read View可见性规则判断当前版本的数据是否可见**。

**InnnoDB执行查询语句的具体步骤为**：

- 执行语句之前获取查询事务自己的事务Id，即事务版本号。
- 通过事务id获取Read View
- 查询存储的数据，将其事务Id与Read View中的事务版本号进行比较
- 不符合Read View的可见性规则，则读取Undo log中历史快照数据
- 找到当前事务能够读取的数据返回

**而在实际的使用过程中，Read View在不同的隔离级别下是得工作方式是不一样**。

### 3.2 读已提交（RC）MVCC实现原理

**在读已提交(Read committed)的隔离级别下实现MVCC，同一个事务里面，【每一次查询都会产生一个新的Read View副本】，这样可能造成同一个事务里前后读取数据可能不一致的问题（不可重复读并发问题）**。

还是按照上述案例来说明一下：

- 【第一步】：准备一条原始数据

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127017.png)

- 【第二步】：假设现在有事务 A【DB_TRX_ID = 20】 和事务 B 【DB_TRX_ID = 30】并发执行

  ```
  #事物A：
  select name from user where id = 1;
  #事物B：
  update user set name = 'edwin' where id = 1;
  ```

  执行过程为

  | 时间 | 事务A                                           | 事务B                                                  |
  | :--- | :---------------------------------------------- | :----------------------------------------------------- |
  | 1    | 开始事务                                        |                                                        |
  | 2    | 第一次查询：select name from user where id = 1; |                                                        |
  | 3    |                                                 | 开始事务                                               |
  | 4    |                                                 | 执行修改：update user set name = 'edwin' where id = 1; |
  | 5    |                                                 | 提交事务                                               |
  | 6    | 第二次查询：select name from user where id = 1; |                                                        |
  | 7    | 提交事务                                        |                                                        |

  版本链为：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061127382.png)

**案例结果分析**：

上述案例在**在读已提交(Read committed)的隔离级别下实现，同一个事务里面，【每一次查询都会产生一个新的Read View副本】**。所以第二步实际上产生了三个Read View

|                            | m_ids   | m_low_limit_id | m_up_limit_id | creator_trx_id |
| :------------------------- | :------ | :------------- | :------------ | :------------- |
| 事务A：第一次查询Read View | [20,30] | 20             | 31            | 20             |
| 事务B：Read View           | [20,30] | 20             | 31            | 30             |
| 事务A：第二次查询Read View | [20]    | 20             | 31            | 20             |

通过可见性判断：

- **事务A第一次查询时**

  日志事务Id【DB_TRX_ID = 10】 < 最小活跃事务ID【m_low_limit_id=20】，因此可以读取到DB_TRX_ID = 10这条版本链中的数据。即name = 星河之码。

- **事务A第二次查询时**

  此时事务B已经提交，版本链中最新版本为DB_TRX_ID = 30，而可见性规则中虽然满足

  【m_low_limit_id=20】=<【DB_TRX_ID=30】<【m_up_limit_id=20】但是【DB_TRX_ID=30】不在m_ids集合[20]中，因此事务A的第二次查询可以读取【DB_TRX_ID=30】的数据，即name = edwin。

**案例总结**：

通过上述案例说明，**同一个事务A的两个相同查询，第一次结果为星河之码，第二次结果为edwin，因此在读已提交（RC）隔离级别下，存在不可重复读并发问题**。

> 此处也就解答了2.6.3中【第五步】的问题一中的第一种情况：读已提交（RC）隔离级别：读取到是事物B提交的Edwin。同样也解答了【第七步】的问题二，为什么能读取DB_TRX_ID=40得数据name=彬。

### 3.3 可重复读（RR）MVCC实现原理

**在可重复读(Repeatable read)的隔离级别下实现MVCC，【同一个事务里面，多次查询，都只会产生一个共用Read View】，以此不可重复读并发问题**。

案例与3.2一样，这里就不重复赘述，可以再看一遍3.2的【第一步】【第二步】，直接进行案例分析

**案例结果分析**：

由于同一个事物只会产生一个共用Read View，所以可重复读的隔离级别下第二步只产生了两个Read View

上述案例在**可重复读(Repeatable read)，【每一次查询都会产生一个新的Read View副本】**。所以第二步实际上产生了三个Read View

|                  | m_ids   | m_low_limit_id | m_up_limit_id | creator_trx_id |
| :--------------- | :------ | :------------- | :------------ | :------------- |
| 事务A：Read View | [20,30] | 20             | 31            | 20             |
| 事务B：Read View | [20,30] | 20             | 31            | 30             |

通过可见性判断：

- **事务A第一次查询时**

  日志事务Id【DB_TRX_ID = 10】 < 最小活跃事务ID【m_low_limit_id=20】，因此可以读取到DB_TRX_ID = 10这条版本链中的数据。即name = 星河之码。

- **事务A第二次查询时**

  此时事务B已经提交，版本链中最新版本为DB_TRX_ID = 30，而可见性规则中虽然满足

  【m_low_limit_id=20】=<【DB_TRX_ID=30】<【m_up_limit_id=20】并且【DB_TRX_ID=30】也在m_ids集合[20，30]中，但是【DB_TRX_ID=30】不等于事物A的【creator_trx_id=20】，说明**DB_TRX_ID=30是同一时刻其他事物提交的，事物A不能读取到**，因此事物A只能按照版本链继续往上找，最终读取到【DB_TRX_ID=10】的数据，即name = 星河之码。

**案例总结**：

通过上述案例说明，**同一个事务A的两个相同查询，结果都为星河之码，因此在可重复读（RR）隔离级别下，解决了不可重复读并发问题**。

> 其实读已经提交与可重复读的可见性判断的区别就在于事务A第二次查询时使用的Read View不通。
>
> 此处也就解答了2.6.3中【第五步】的问题一中的第二种情况：可重复读（RR）隔离级别：读取到是原始数据提交的星河之码。同样也解释了【第七步】，为什么能读取到的是DB_TRX_ID=10得数据name=星河之码。