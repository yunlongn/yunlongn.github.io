title: MySQL六：InnoDB数据文件
author: yunlongn
tags:
  - Mysql
categories:
  - 数据库
  - Mysql
date: 2021-08-06 16:06:00
---

## 一、数据文件的组成

**innodb数据逻辑存储形式为表空间,而每一个独立表空间都会有一个.ibd数据文件**,ibd文件从大到小组成：

一个ibd数据文件-->Segment（段）-->Extent（区）-->Page（页）-->Row（行）![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124739.jpeg)

- **表空间(Tablesapce)**

  **表空间，用于存储多个ibd数据文件，用于存储表的记录和索引，一个文件包含多个段。**

- **段(Segment)**

  **段由数据段、索引段、回滚段组成，innodb存储引擎索引与数据共同存储，数据段即是B+树叶节点，索引段则存储非叶节点**。

- **区(Extent)**

  **区则是由连续页组成，每个区的大小为1M，一个区中一共有64个连续的页**。

- **页(Page)**

  **页是innodb存储引擎磁盘管理的最小单位，页的大小为16KB，即每次数据的读取与写入都是以页为单位**。

  > “
  >
  > 包含很多种页类型，比如数据页，undo页，系统页，事务数据页，大的BLOB对象页
<!--more-->
- **行(Row)**

  **行包含记录的字段值，事务ID（Trx id）、滚动指针（Roll pointer）、字段指针（Field pointers）等信息。**![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 二 InnoDB数据页结构

**InnoDB将数据划分为若干个页，以页作为磁盘和内存之间交互的基本单位，InnoDB中页的大小一般为 16KB**。

- **页的组成**

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

如图所示，InnoDB数据页由以下七个部分组成，

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124654.png)

### 2.1 File Header（文件头）

File Header用来记录页的一些头信息，由如下8个部分组成，共占用38个字节，如表4-3所示：

![图片](https://images-roland.oss-cn-shenzhen.aliyuncs.com/blog/mysql/202208061124637.png)

- **FIL_PAGE_SPACE_OR_CHKSUM**

- - 在MySQL4.0.14版本之前

    **该值代表该页属于哪个表空间**，当innodb_file_per_table没有开启事，共享表空间中可能存放了许多页，并且这些页属于不同的表空间。

  - MySQL4.0.14之后版本

    该值代表页的checksum值（一种新的checksum值）。

- **FIL_PAGE_OFFSET**

  表空间中页的偏移值。

- **FIL_PAGE_PREV，FIL_PAGE_NEXT**

  当前页的上一个页以及下一个页。**B+Tree特性决定了叶子节点必须是双向列表。**

- **FIL_PAGE_LSN**

  该值代表该页最后被修改的日志序列位置LSN（Log Sequence Number）。

- **FIL_PAGE_TYPE**

  页的类型。十六进制表示，0x45BF代表B+tree叶结点（存放数据的数据页）。

- **FIL_PAGE_FILE_FLUSH_LSN**

  该值仅在数据文件中的一个页中定义，代表文件至少被更新到了该LSN值。

- **FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID**

  从MySQL 4.1开始，该值代表页属于哪个表空间。

### 2.2 Page Header（页头）

用来记录数据页的状态信息，由以下14个部分组成，共占用56个字节。

> “
>
> 比如本页中已经存储了多少条记录，第一条记录的地址是什么，页目录中存储了多少个槽等。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

- **PAGE_N_DIR_SLOTS**

  在Page Directory（页目录）中的Slot（槽）数。

- **PAGE_HEAP_TOP**

  堆中第一个记录的指针。

- **PAGE_N_HEAP**

  堆中的记录数。

- **PAGE_FREE**

  指向空闲列表的首指针。

- **PAGE_GARBAGE**

  已删除记录的字节数，即行记录结构中，delete flag为1的记录大小的总数。

- **PAGE_LAST_INSERT**

  最后插入记录的位置。

- **PAGE_DIRECTION**

  最后插入的方向。取值为：

- - PAGE_LEFT（0x01）
  - PAGE_RIGHT（0x02）
  - PAGE_SAME_REC（0x03）
  - PAGE_SAME_PAGE（0x04）
  - PAGE_NO_DIRECTION（0x05）

- **PAGE_N_DIRECTION**

  一个方向连续插入记录的数量。

- **PAGE_N_RECS**

  该页中记录的数量。

- **PAGE_MAX_TRX_ID**

  修改当前页的最大事务ID，该值仅在Secondary Index定义。

- **PAGE_LEVEL**

  当前页在索引树中的位置，0x00代表叶节点。

- **PAGE_INDEX_ID**

  当前页属于哪个索引ID。

- **PAGE_BTR_SEG_LEAF**

  B+树的叶节点中，文件段的首指针位置。注意该值仅在B+树的Root页中定义。

- **PAGE_BTR_SEG_TOP**

  B+树的非叶节点中，文件段的首指针位置。注意该值仅在B+树的Root页中定义。![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 2.3 Infimun+Supremum Records

**在InnoDB存储引擎中，每个数据页中有两个虚拟的行记录，用来限定记录的边界。**

- Infimum记录是比该页中任何主键值都要小的值。
- Supremum指比任何可能大的值还要大的值。

这两个值在页创建时被建立，并且在任何情况下不会被删除，如下图 ：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 2.4 User Records（用户记录，即行记录）

**User Records即实际存储行记录的内容，InnoDB存储引擎表总是B+树索引组织的**。

> “
>
> **每当我们插入一条记录，都会从Free Space部分申请一个记录大小的空间划分到User Records部分**，当Free Space部分的空间全部被User Records部分替代掉之后，当前页就被用完了，此时如果还有新的记录插入就需要申请新的页了。

### 2.5 Free Space（空闲空间）

**Free Space指的就是空闲空间，同样也是个链表数据结构**。当一条记录被删除后，该空间会被加入空闲链表中。

### 2.6 Page Directory（页目录）

Page Directory（页目录）中存放了记录的**相对位置**

- 所有正常的记录（包括最大和最小记录，不包括为已删除的记录）会被划分为几个组。
- 每个组的最后一条记录的头信息中的n_owned属性表示该组内共有几条记录。
- 将每个组的最后一条记录的地址偏移量按顺序存储起来，每个地址偏移量也被称为一个槽。这些地址偏移量都会被存储到靠近页的尾部的地方。

**InnoDB并不是每个记录拥有一个槽，InnoDB存储引擎的槽是一个稀疏目录（sparse directory），即一个槽中可能属于（belong to）多个记录，最少属于4条记录，最多属于8条记录。**

> “
>
> Slots中记录按照键顺序存放，这样可以利用二叉查找迅速找到记录的指针。假设有（'i'，'d'，'c'，'b'，'e'，'g'，'l'，'h'，'f'，'j'，'k'，'a'），同时假设一个槽中包含4条记录，则Slots中的记录可能是（'a'，'e'，'i'），然后通过recorder header中的next_record来继续查找相关记录。

**【B+树索引本身并不能找到具体的一条记录，B+树索引能找到只是该记录所在的页】。数据库把页载入内存，然后通过Page Directory再进行二叉查找。由于二叉查找的时间复杂度很低，同时内存中的查找很快，因此通常我们忽略了这部分查找所用的时间。**

所以在一个数据页中查找指定主键值的记录的过程分为两步：

- 通过二分法确定该记录所在的槽。
- 通过记录的next_record属性组成的链表遍历查找该槽中的各个记录。

### 2.7 File Trailer（文件结尾信息）

File Trailer只有一个FIL_PAGE_END_LSN部分，占用8个字节。

- **主要作用**

  保证页能够完整地写入磁盘，校验数据完整性。

  > “
  >
  > 写入过程中磁盘损坏、机器宕机时

- **具体方法**

  前4个字节代表该页的checksum值，最后4个字节和File Header中的FIL_PAGE_LSN相同。

  通过这两个值来和File Header中的FIL_PAGE_SPACE_OR_CHKSUM和FIL_PAGE_LSN值进行比较，看是否一致，以此来保证页的完整性（not corrupted）