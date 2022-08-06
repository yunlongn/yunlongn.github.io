title: MySQL五：InnoDB线程模型
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:05:00
---

## **一、InnoDB线程模型的组成**

在Innodb存储引擎中，后台线程的主要作用是**「负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据」**。此外它会将已经修改的数据文件刷新到磁盘文件中，保证在发生异常的情况下，Innodb能够恢复到正常的运行状态。
<!--more-->
![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061123754.png)

**「InnoDB存储引擎是多线程的模型，所以有多个不同的后台线程，负责处理不同的任务」**。主要有：

Master Thread、IO Thread、Purge Thread、Page Cleaner Thread四种。![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## **二 Master Thread**

**「Master thread是InnoDB的主线程，负责调度其他各线程，优先级最高」**。

- **「主要作用」**

  将缓冲池中的数据一步刷新到磁盘，保证数据的一致性。

- **「主要工作」**

  脏页的刷新（page cleaner thread）、undo页回收（purge thread）、redo日志刷新（log thread）、合并写缓冲等。

Master thread内部有两个主要处理时机，分别是每隔1秒和10秒处理。

- 每隔1秒

- - innodb_max_dirty_pages_pct

  - innodb_io_capacity

    合并插入缓冲时,每秒合并插入缓冲的数量为 innodb_io_capacity值的5%，默认就是 200*5%=10

    在从缓冲区刷新脏页时,每秒刷新脏页的数量就等于innodb_io_capacity的值，默认200

  - 刷新日志缓冲区，刷到磁盘

  - 合并写缓冲区数据，根据IO读写压力来决定是否操作

  - 刷新脏页数据到磁盘，根据脏页比例达到75%才操作，此处涉及两个参数

    > 脏页比例通过innodb_max_dirty_pages_pct配置，innodb_max_dirty_pages_pct参数值保存在变量srv_max_buf_pool_modified_pct 里面，这是一个全局变量，初始值为 75.0

- 每隔10秒

- - 刷新脏页数据到磁盘
  - 合并写缓冲区数据
  - 刷新日志缓冲区
  - 删除无用的undo页

## **三、 IO Thread**

**「为了提高数据库的性能，在InnoDB中使用了大量的AIO（Async IO）来做读写处理」**。

一共有4种总共10个IO Thread：

- **「read thread」**（4个）

  负责读取操作，将数据从磁盘加载到Buffer Pool的Page页。

- **「write thread」**（4个）

  负责写操作，将Buffer Pool的dirty page刷新到磁盘。

  ```
   show variables like "%innodb%io_threads%";
  ```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- **「log thread」**（1个）

  负责将Log Buffer内容刷新到磁盘。

- **「insert buffer thread」**（1个）

  负责将Change Buffer内容刷新到磁盘。

  ```
  #查看当前IO线程的工作状态
  show engine innodb status;
  ```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061123716.png)

## **四 Purge Thread**

事务提交之后，其使用的undo日志将不再需要，因此需要Purge Thread回收已经分配的undo 页。

> 早前的版本只支持一个Purge Thread，目前mysql 5.7版本支持多个Purge Thread，目的是为了进一步加快undo数据页的回收速度。

```
show variables like '%innodb_purge_threads%';
```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061123708.png)

## **五 Page Cleaner Thread**

**「作用是将脏数据放入到单独的线程中刷新到磁盘，脏数据刷盘后相应的redo log也就可以覆盖，即可以同步数据，又能达到redo log循环使用的目的」**。

减轻原来的Master Thread的工作，同时可以缓解用户查询线程的阻塞，进一步提高Innodb 存储引擎的性能。

```
 show variables like '%innodb_page_cleaners%';
```

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061123712.png)