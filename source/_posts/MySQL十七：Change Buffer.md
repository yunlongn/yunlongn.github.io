title: MySQL十七：Change Buffer
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:17:00
---
在之前的文章[《InnoDB的存储结构》]()介绍的InnoDB的存储结构的组成中，我们知道Change Buffer也是用InnoDB内存结构的组成部分。
<!--more-->
![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061130778.png)

Change Buffer主要是为了在写入是减少磁盘IO而存在的，

## **一、什么是什么是Change Buffer**

**「在[《Buffer Pool》]()中介绍了buffer pool会缓存热的数据页和索引页，减少磁盘读操作，而对于磁盘的写操作，innoDB同样也有类似的策略，即通过change buffer缓解磁盘写操作产生的磁盘IO」**。

- **「Change Buffer是在【非唯一普通索引页】不在buffer pool中时，当对页进行了写操作时，在不影响数据一致性的前提下。InnoDB会将数据先写入Change Buffer中，等未来数据被读取时，再将 change buffer 中的操作merge到原数据页中」**。
- 在MySQL5.5之前，只针对insert做了优化，叫插入缓冲(insert buffer)，后面进行了优化，对delete和update也有效，叫做写缓冲(change buffer)。

## **二、Change buffer 执行过程**

**「我们知道当执行写操作时，数据页存在于Buffer Pool中时，会直接修改数据页。那如果数据页不存在于Buffer Pool中时，过程会有一些不一样，这种情况会将写操作缓存到Change Buffer中，等未来在特定条件下其合并到Buffer Pool中」**。因此当需要执行一个写入操作时，一般分为走Change buffer和不走Change buffer两种情况。

### 2.1 写入的数据页在内存中

> 这种情况在在[《Buffer Pool》](https://mp.weixin.qq.com/s?__biz=MzIxMDU5OTU1Mw==&mid=2247486351&idx=1&sn=bcafc8b2862545396401fc7c7e75c907&scene=21#wechat_redirect)的第4.3.2节 Flush链表写入过程中已经提过了，感兴趣的可以去看看，不看也没关系，这里在写一遍，凑一下字数~~

当我们在写入数据的时候，写入的数据页在内存中，MySQL不会直接更新直接更新磁盘，而是经过以下两个步骤：

- 第一步：更新Buffer Pool中的数据页，一次内存操作；
- 第二步：将更新操作顺序写Redo log，一次磁盘顺序写操作；

> 这样的效率是最高的。顺序写Redo log，每秒几万次，问题不大。

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061130796.png)

**「这种情况是被更新的数据已经别加载到Buffer Pool的前提下」**。

- **「是否会产生数据一致性问题」**

  **「因此写入的数据页在内存中这中情况不会产生数据一致性问题」**

- - 读取数据，会命中缓冲池的页（已经被修改）。
  - 缓冲池LRU数据淘汰，则会将【脏页】刷回磁盘。
  - 数据库奔溃，redo log可以恢复数据。

### 2.2 写入的数据页不在内存中

当我们修改的数据所在的数据页之前没有别读取过，或者干脆就是一条插入语句，则会经过以下两个步骤：

- 第一步：在Change buffer中记录这个写入操作，一次内存操作。
- 第二步：将写入操作顺序写Redo log，一次磁盘顺序写操作；

> 可以看到，这种方式跟上面的方式**「仅仅只是第一步写入的位置不一样而已，而且都是内存操作」**。
>
> 如果没有Change buffer，那更新可能会变成
>
> - 第一步：先从磁盘读取所在数据页加载到缓冲池，一次磁盘随机读操作；
> - 第二步：更新Buffer Pool中的数据页，一次内存操作；
> - 第三步：将更新操作顺序写Redo log，一次磁盘顺序写操作；
>
> 也就是会多一次磁盘IO，磁盘IO相比较内存操作时很慢的，并发下性能就会急剧下降。

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061130771.png)

> 这种方式的效率跟第一次差不多，写缓冲是降低磁盘IO，提升数据库写性能的一种机制。

**「是否会产生数据一致性问题」**

- 读取数据，会将Change Buffer中的数据合并到Buffer Pool中。
- 如果没有读取，Change也会被被定期刷盘到写缓冲系统表空间。
- 数据库奔溃，redo log可以恢复数据。

**「因此写入的数据页不在内存中这中情况也不会产生数据一致性问题」**。

## **三、Change Buffer大小配置**

**「从下图中可以看出，Change Buffer被包含在了Buffer Pool中的，change buffer用的是buffer pool里的内存，由于Buffer Pool的内存大小是有限制的，所以change buffer大小也是有限制的，可通过参数innodb_change_buffer_max_size设置」**。

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061130720.png)

```
show variables like '%innodb_change_buffer_max_size%';
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- innodb_change_buffer_max_size表示允许change_buffer占Buffer Pool总大小的百分比，默认值为25%，最大可设置为50%。

- - 当在系统中有大量插入，更新和删除操作时，可以增大innodb_change_buffer_max_size，以提高系统的写入性能。
  - 当在系统中有大量查询操作时，可以减小innodb_change_buffer_max_size，以减少Buffer Pool中数据页的淘汰的概率，提高系统的读取性能。

- innodb_change_buffer_max_size 设置是动态的，它允许修改设置而无需重新启动服务器。

## **四、配置Change Buffer的类型**

**「前面说到Change Buffer在MySQL5.5之后可以支持新增、删除、修改的写入，对于受I/O限制的操作（大量DML、如批量插入）有很大的性能提升价值。但是对于一些特定的场景，可以通过修改innodb_change_buffering来变更Change Buffer支持的类型，分别为插入，删除，清除启用或禁用缓冲，更新操作是插入和删除的组合」**。

```
show variables like '%innodb_change_buffering%';
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- all ：默认值，缓冲区插入，删除和清除。
- none：不缓存任何操作
- inserts：缓冲区插入操作。
- deletes：缓冲区删除标记操作。
- changes：缓冲区插入和删除标记操作。
- purges：缓冲区在后台发生的物理删除操作。

## **五、Change buffer被merge的时机**

既然Change buffer是单独内存中，写入之后会被合并到Buffer Pool中,那么是时候时候会被merge呢？

**「Change buffer会被merge触发时机」**

- 读取Change buffer中记录的数据页时，会将Change buffer合并到buffer Pool 中，然后被刷新到磁盘。
- 当系统空闲或者slow shutdown时，后台master线程发起merge。
- change buffer的内存空间用完了，后台master线程会发起merge。
- redo log写满了，但是一般不会发生。

## **六、Change buffer为什么只对非唯一普通索引页有效**

**「不知道大家有没有印象，在本文第一节就重点说了一个词【非唯一普通索引页】，Change buffer只有在非唯一普通索引页时才生效，这是为什么呢？」**

相信大家在日常工作中会经常遇到一个问题：**「主键冲突」**。

- **「主键索引，唯一索引」**

  实际上对于【唯一索引】的更新，插入操作都会**「先判断当前操作是否违反唯一性约束」**，而这个操作就必须要将索引页读取到内存中，此时既然已经读取到内存了，那直接更新即可，没有需要在用Change buffer了。

- **「非唯一普通索引」**

  **「不需要判断当前操作是否违反唯一性约束」**，也就不需要将数据页读取到内存，因此可以直接使用 change buffer 更新。

**「基于此，Change buffer只有对普通索引可以使用，对唯一索引的更新无法生效」**。

