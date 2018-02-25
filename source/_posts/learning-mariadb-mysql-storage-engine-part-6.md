title: learning mariadb & mysql-storage-engine(part-6)
date: 2018-01-22 15:26:49
categories: programming
tags:
- mysql
- mariadb
- storage engine
---

相对于其它的关系型数据库，MariaDB 最大的特点就是针对于不同的业务场景，提供了不同的存储引擎。这些存储引擎存在不同的差异，例如是否支持事务、锁机制是以数据表为单位还是以行为单位。在接下来的文章中，我们将讨论在日常项目当中使用得较多的几种存储引擎。

## 存储引擎的选择
---

参考 MariaDB的官方文档：[choosing the right storage engine](https://mariadb.com/kb/en/library/choosing-the-right-storage-engine/)

<!--more-->

## Aria 存储引擎 @MariaDB
---

MyISAM 存储引擎在 Mysql 初期开始一直使用至今，其功能与结构相对简单，具有处理速度快、容易使用等有点。但由于其不支持表级锁，Mysql在引入 InnoDB 存储引擎之后，其使用频率变得越来越低。目前几乎只有内部临时表，或在非常特殊的场景下才会使用。

Aria 存储引擎作为 MyISAM 的升级版本，最大的区别在于支持事务与页面缓存的功能，广泛用于 MariaDB 中创建内部临时表。

### 事务

创建 Aria 存储引擎的数据表时，需要显式使用 TRANSACTIONAL 选项。当设置 TRANSACTIONAL=1 之后，相关数据表的所有变更内容就会先保存到日志文件（重做日志[ Redo Log ]）。

```
CREATE TABLE tb_aria_test (
    fd1 INT NOT NULL,
    fd2 VARCHAR(10) NOT NULL,
    PRIMARY KEY(fd1)
) ENGINE=Aria TRANSACTIONAL=1;
```

但是当前 Aria 的版本为 1.5 ，对事务的支持并不完善，具体可以参考 MariaDB 的[官网文档](https://mariadb.com/kb/en/library/aria-faq/)：

> Why do you use the TRANSACTIONAL keyword now when Aria is not yet transactional?
In the current development phase Aria tables created with TRANSACTIONAL=1 are crashsafe and atomic but not transactional because changes in Aria tables can't be rolled back with the ROLLBACK command. As we planned to make Aria tables fully transactional, we decided it was better to use the TRANSACTIONAL keyword from the start so so that applications don't need to be changed later.

可以官方文档当中提到的：`crash safe` ，这意味着相对于 MyISAM 存储引擎，在 MariaDB 异常退出并重启了之后，就会利用 redo-log 进行自动恢复。也就是说，目前版本的 Aria 存储引擎主要用于保护数据表不会由于异常场景而导致数据丢失。

### 页面缓存

与 MyISAM 存储引擎相比， Aria 存储引擎最大的有点是页面缓存。 MyISAM 存储引擎通过 key_buffer_size 系统设置可以使用另外的缓存，但该内存缓冲中只能缓存索引。 Aria存储引擎除了可以缓存索引之外，还可以缓存数据表的数据页，只需要创建表时，只需要设置 ROW_FORMAT 选项为 page 即可。如果将 ROW_FORMAT 设置为 fixed，则 Aria 与 MyISAM 一致，只会存储索引至缓存中。

```
CREATE TABLE tb_aria_test2 (
    fd1 INT NOT NULL,
    fd2 VARCHAR(10) NOT NULL,
    PRIMARY KEY(fd1)
) ENGINE=Aria ROW_FORMAT=page;
```

Aria 存储引擎提供了 aria_pagecache_buffer_size 系统变量、通过设置可以调整缓存可用内存空间的大小，默认为 128 MB。

### 系统变量设置

* aria_pagecache_buffer_size ：上文中已经提到，用于设置缓存空间的大小，以对索引或数据页面进行缓存。
* aria_sort_buffer_size ：用于创建 Aria 数据表或为其添加索引、或使用 REPAIR 命令是，对缓存空间进行排序处理。该系统变量用于设置排序的缓冲空间大小。
* aria_group_commit 、 aria_group_commit_interval ：如果 TRANSACTIONAL 设置为 1，会先记录 WAL 日志，同时也收集提交的事务日志进行记录。 aria_group_commit 用于设置是否使用组提交方式， aria_group_commit_interval 设置收集并记录到磁盘的时间间隔，单位为微妙。

## XtraDB 存储引擎 @MariaDB

XtraDB 是 Percona 公司在 InnoDB 源码的基础上改进并开发的高性能存储引擎，只有 PerconaServer 和 MariaDB 支持。

关于 XtraDB 和 InnoDB 选择的问题，可以参考 MariaDB 官方文档的介绍:

> XtraDB is the best choice in the majority of cases until MariaDB 10.1. It is a performance-enhanced fork of InnoDB and is MariaDB's default engine until MariaDB 10.1.
InnoDB is a good general transaction storage engine. It is the default MySQL storage engine, and default MariaDB 10.2 storage engine. For earlier releases, XtraDB is a performance enhanced fork of InnoDB and is usually preferred.

简而言之，如果你使用的 MariaDB 10.1 以及之前的版本，请使用 XtraDB ； 否则请使用 InnoDB 。

如果需要在 MariaDB 10.2 以及之上的版本中强制使用 XtraDB 存储引擎，可以按照如下方式设置 my.cnf 配置文件，并重启服务器：

```
[mysqld]
ignore_builtin_innodb = ON
plugin-load = ha_xtradb.so
```

并通过 `SHOW PLUGINS SONAME` 进行验证。

## InnoDB 存储引擎
---

### 概述

InnoDB 从 Mysql 5.5 版本开始作为默认的表存储引擎，其特点是支持行锁、支持MVCC、支持外键、提供一致性非锁定读等特性。

### 版本

Mysql 5.1 版本中，可以支持两个版本的 InnoDB ， 一个是静态编译的版本，又被称之为老版本的 InnoDB ；另一个是动态加载的 InnoDB 版本，官方称为 InnoDB Plugin ，也被称为 InnoDB 1.0.x 版本。在 Mysql 5.5 版本开始， InnoDB 升级的版本被称为 1.1.x 。 Mysql 5.6 版本中 InnoDB 升级到了 1.2.x 版本。

| 版本 | 功能     |
| :------------- | :------------- |
| 老版本 InnoDB       | 支持 ACID 、行锁设计、MVCC       |
| 1.0.x       | 在上个版本的基础上，增加了 compress 和 dynamic 页格式       |
| 1.1.x       | 在上个版本的基础上，增加了 Linux AIO 、多回滚段       |
| 1.2.x       | 在上个版本的基础上，增加了 全文索引支持、在线索引添加       |

### 体系架构

InnoDB 存储引擎有多个内存块，由这些内存块组成了一个大的内存池，这些内存块主要负责以下主要工作：
* 维护所有进程/线程需要访问的多个内部数据结构。
* 缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改之前在这里缓存。
* 重做日志（redo log）缓冲。

从下图我们可以看到，InnoDB 存储引擎主要是由 若干后台线程、内存池、物理文件组成。

![innodb-arch](https://www.zhuxiaodong.net/static/images/innodb-arch.png)

#### 后台线程

InnoDB 存储引擎时多线程的模式，由不同的后台线程组成，负责处理不同的任务。

##### Master Thread

主要负责将缓冲池的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、 Undo 页的回收。

##### IO Thread

在 InnoDB 存储引擎中大量使用了 AIO 来处理写 IO 请求，来提升数据库的性能。 IO Thread 的主要工作是负责这些 IO 请求的回调 （ call back ）处理。 一共有4种类型的 IO Thread ： write 、 read 、 insert buffer 、 log IO thread。从 InnoDB 1.0.X 版本开始， read thread 和 write thread 分别增大到了 4 个。

```
show variables like 'innodb_%io_threads';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_read_io_threads  | 4     |
| innodb_write_io_threads | 4     |
+-------------------------+-------+
```

我们可以通过 SHOW ENGINE INNODB STATUS 来查看 InnoDB 中的 IO Thread 。 从下面的部分输出上来看，除了一个 insert buffer thread 和一个 log thread 之外，还包括 4 个 read thread 和 4 个 write thread 。

```
SHOW ENGINE INNODB STATUS

FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
2410 OS file reads, 1519 OS file writes, 681 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
```

##### Purge Thread

在事务提交了之后，其对应的 Undo log 可能将不再需要，因此需要 Purge Thread 来回收清理掉已经使用并分配的 undo 页。从 InnoDB 1.1 版本开始， purge 操作可以独立到单独中进行，以减少 Master Thread 的负荷。可以通过设置 innodb_purge_threads 的值来进行控制。

```
show variables like 'innodb_purge%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| innodb_purge_batch_size              | 300   |
| innodb_purge_rseg_truncate_frequency | 128   |
| innodb_purge_threads                 | 4     |
+--------------------------------------+-------+
```

##### Page Cleaner Thread

Page Cleaner Thread 是在 InnoDB 1.2.x 版本中引入的。其作用是将之前版本中脏页的刷新操作都放入到单独的线程来完成。

#### 内存

##### 缓冲池

InnoDB 存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。由于磁盘的访问效率会远远低于内存的访问效率，因此 RDB 通常会使用缓冲池技术来提升数据库的整体性能。缓冲池简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。在数据库中读取页的操作，首先见从磁盘读到的页存放在缓冲池中，这个过程称为将页 "FIX" 在缓冲池中。如果下一次再进行读取时，首先判断页是否在缓冲池中，如果存在的话则直接读取（缓存命中），否则，读取磁盘上的页。

对于数据库页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。页从缓冲池刷新回磁盘的操作并不是在页发生更新时触发的，而是通过 checkpoint 的机制刷新回磁盘。

对于 InnoDB 存储引擎而言，其缓冲池的配置通过参数 innodb_buffer_pool_size 来进行设置。

```
show variables like 'innodb_buffer_pool_size';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| innodb_buffer_pool_size | 67108864 |
+-------------------------+----------+
```

缓冲池中缓存数据页分为了：索引页、数据页、 undo 页、插入缓冲（ insert buffer ）、自适应哈希索引（ adaptive hash index ）、锁信息（ lock info ）、数据字典信息（ data dictionary ）。

![innodb-buffer-pool-page](https://www.zhuxiaodong.net/static/images/innodb-buffer-pool-page.png)

从 InnoDB 1.0.x 版本开始，允许设置多个缓冲池实例。每个页根据 hash 值平均分配到不同缓冲池实例中。这样做的目的是减少数据库内部的资源竞争，增加数据库的并发处理能力。可以通过 innodb_buffer_pool_instances 来进行配置，默认值为 1 。

```
show variables like 'innodb_buffer_pool_instances';
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_buffer_pool_instances | 1     |
+------------------------------+-------+
```

可以通过 information_schema 数据库下的 innodb_buffer_pool_stats 表查询到缓冲池的状态。

```
select pool_id,pool_size,free_buffers,database_pages from innodb_buffer_pool_stats\G
*************************** 1. row ***************************
       pool_id: 0
     pool_size: 4096
  free_buffers: 1006
database_pages: 3089
```

##### LRU List 、 Free List 和 Flush List

参考链接：
http://www.cnblogs.com/geaozhang/p/7276802.html
http://jockchou.github.io/blog/2015/07/23/innodb-buffer-pool.html
http://blog.csdn.net/jerry____wang/article/details/53907649

通常情况下，数据库缓冲池是通过 LRU （ Latest Recent Used 最近最少使用）算法来进行管理的。即最频繁使用的页放在 LRU 列表的顶端，而最少使用的页放在 LRU 列表的末端。当缓冲池由于空间占满了之后，无法再存取新的页时，就首先会释放掉 LRU 列表末端的页。

在 InnoDB 存储引擎中，缓冲池中的页的大小默认为 16 KB，同样使用 LRU 算法进行管理。 InnoDB 对 LRU 算法做了一些优化，在列表中增加了 midpoint 位置。新读取到的页，不直接放在 LRU 列表的顶端，而是放在 midpoint 位置（又被称为： midpoint insertion strategy）。默认配置下， midpoint 的位置在 LRU 列表长度的 5/8 初，并且该位置可以由系统变量 innodb_old_blocks_pct 控制。

```
show variables like 'innodb_old_blocks_pct%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
```

从上面的 innodb_old_blocks_pct 配置可以看到，37 表示新读取的页插入到 LRU 列表尾端的 37% 的位置（ 3/8 左右）。我们把 midpoint 之后的列表称为 old 列表，而之前的称为 new 列表。 new 列表当中存储的最为活跃的热点数据。

如果直接将读取到的页放入到 LRU 列表的首部，那么某些 SQL 操作可能会使缓冲池中的页被刷新出，从而影响缓冲池的效率。例如常见的索引扫描或数据扫描类的操作，这类操作需要访问表中的很多页，某些时候甚至是全部的页数据，而这些操作某些时候仅仅只是本次查询需要，并不是活跃的热点数据。如果按照传统的 LRU 算法直接放在顶端，那么可能导致原本的热点数据页从 LRU 列表中移除，导致下一次访问时只能从磁盘中加载，效率会比较低。

为了解决这个问题，InnoDB 中引入了另外一个系统变量来管理 LRU 列表， innodb_old_blocks_time ，用于指定页读取到 mid 位置后需要等待多长时间之后才会被加入到 LRU 列表的顶端。当 innodb_old_blocks_time 设置为 0 时， 表示列表中的热点数据不会被删除掉。

```
show variables like 'innodb_old_blocks_time%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
```

同时，我们可以通过减少指定 innodb_old_blocks_pct 的值，来减少热点数据页被删除掉的概率。比如，我们预估热点数据页占整体数据的 80% ，我们可以调整 innodb_old_blocks_pct 的值为 20 。

在数据库刚启动时，LRU 列表是空的，此时页都存放在 Free 列表中。当需要从缓冲池中分页时，首先从 Free 列表中查找是否有可用的空闲页，若有则将该页从 Free 列表中删除，放入到 LRU 列表中。否则，根据 LRU 算法，淘汰 LRU 列表末尾的页，将该内存空间分配给新的页。当页从 LRU 列表的 old 部分加入到 new 部分时，被称为 page made young ，而因为 innodb_old_blocks_time 的设置而导致没有页从 old 部分移动到 new 部分的操作称为 page not made young 。

```
SHOW ENGINE INNODB STATUS\G

=====================================
Per second averages calculated from the last 27 seconds
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 13193183232
Dictionary memory allocated 4016849
Buffer pool size   786432
Free buffers       761311
Database pages     23557
Old database pages 8532
Modified db pages  851
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 7578, not young 11384
0.00 youngs/s, 0.00 non-youngs/s
Pages read 16965, created 7143, written 44129254
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 23557, unzip_LRU len: 0
I/O sum[24]:cur[0], unzip sum[0]:cur[0]
```

可以看到 Buffer pool size 共有 786432 个页，即 786432 * 16K , 总共 12 GB 的缓冲池。 Free buffers 表示当前 Free 列表中页的数量，而 Database pages 表示 LRU 列表中页的数量。可能的情况是 Free buffers + Database pages 不等于 Buffer pool size 。 原因是缓冲池中的页还可能会被分配给自适应哈希索引、Lock信息、Insert Buffer 等页，这些页都不需要 LRU 算法进行维护，因此不存在与 LRU 列表中。Pages made young 7578 表示 LRU 列表中页移动到顶端的次数为 7578, not young 11384 表示从 old 部分移动到 new 部分的次数为 11384 。

上述的 InnoDB 输出的状态信息中，还有一个非常重要的信息： Buffer pool hit rate 1000 / 1000 ，表示缓存池的命中率。在这个例子中表示命中率为 100% ，通常情况下这个值不应该小于 95% ，否则就应该观察是否出现了全表扫描引起的 LRU 列表被污染的问题。

还有一点需要注意的是， SHOW ENGINE INNODB STATUS 显示的不是当前的状态，而是过去的某个时间范围内的状态，`Per second averages calculated from the last 27 seconds` 表示在过于 27 秒内的状态。

InnoDB 存储引擎从 1.0.x 版本开始支持压缩页的功能，即将原来的 16KB 的固定页大小，压缩为 1KB\2KB\4KB\8KB。对于非 16KB 的页，使用 unzip_LRU 列表进行管理。

```
LRU len: 23557, unzip_LRU len: 0
```

可以看到 LRU 列表中一共有 1539 个页，而 unzip_LRU 列表中没有页存在。需要注意的是 LRU len 的大小是包括了 unzip_LRU len 的值的。

在 unzip_LRU 列表中对不同压缩页大小的页进行了分别管理，通过“伙伴算法”进行内存分配。假设对申请缓冲池中 4KB 的大小，其过程如下：

* 检查 4KB 的 unzip_LRU 列表，看是否有可用的空闲页；
* 若有则直接使用；
* 否则，检查 8KB 的 unzip_LRU 列表；
* 若能够得到空闲页，将页分成 2 个 4KB 页，存放到 4KB 的 unzip_LRU 列表；
* 若不能得到空闲页，从 LRU 列表中申请一个 16KB 的页，将也分为 1 个 8KB 的页、2 个 4KB 的页，分别存放到对应的 unzip_LRU 列表中。

可用通过 innodb_buffer_page_lru 表查询到 unzip_LRU 列表中的页。

```
select table_name,space,page_number,compressed_size from information_schema.innodb_buffer_page_lru where compressed_size <> 0;
```

在 LRU 列表中的页被修改后，缓冲池中的页和磁盘上的页的数据产生了不一致，这被称之为 `脏页（ dirty page ）`。这个时候数据库会通过 CHECKPOINT 机制批量将脏页刷新到磁盘上，而刷新（ Flush ）列表中的页被称为 `脏页列表`。需要注意的是，脏页即存在与 LRU 列表中，又存在于 Flush 列表中。 LRU 列表用来管理缓冲池中页的可用性， Flush 列表用来管理将页刷新回磁盘。

```
Modified db pages  851
```

上面的 InnoDB 状态信息中，显示了 dirty page 的数量。同时，可以通过 innodb_buffer_page_lru 表查询具体的 LRU 列表中脏页的具体信息。

```
select table_name,space,page_number,page_type from information_schema.innodb_buffer_page_lru where oldest_modification > 0;
+--------------+-------+-------------+-------------------+
| table_name   | space | page_number | page_type         |
+--------------+-------+-------------+-------------------+
| NULL         |   587 |           0 | FILE_SPACE_HEADER |
| NULL         |   587 |           2 | INODE             |
| `SYS_TABLES` |   587 |          35 | INDEX             |
| `SYS_TABLES` |   587 |          36 | INDEX             |
| `SYS_TABLES` |   587 |          37 | INDEX             |
| `SYS_TABLES` |   587 |          38 | INDEX             |
| `SYS_TABLES` |   587 |          39 | INDEX             |
| `SYS_TABLES` |   587 |          40 | INDEX             |
| `SYS_TABLES` |   587 |          41 | INDEX             |
| `SYS_TABLES` |   587 |          42 | INDEX             |
| `SYS_TABLES` |   587 |          43 | INDEX             |
| `SYS_TABLES` |   587 |          44 | INDEX             |
| `SYS_TABLES` |   587 |          45 | INDEX             |
| `SYS_TABLES` |   587 |          46 | INDEX             |
| `SYS_TABLES` |   587 |          47 | INDEX             |
| `SYS_TABLES` |   587 |          48 | INDEX             |
| `SYS_TABLES` |   587 |          49 | INDEX             |
| `SYS_TABLES` |   587 |          50 | INDEX             |
| `SYS_TABLES` |   587 |          51 | INDEX             |
| `SYS_TABLES` |   587 |          52 | INDEX             |
| `SYS_TABLES` |   587 |          53 | INDEX             |
| `SYS_TABLES` |   587 |          54 | INDEX             |
| `SYS_TABLES` |   587 |          55 | INDEX             |
| `SYS_TABLES` |   587 |          56 | INDEX             |
| `SYS_TABLES` |   587 |          57 | INDEX             |
| `SYS_TABLES` |   587 |          58 | INDEX             |
| `SYS_TABLES` |   587 |          59 | INDEX             |
| `SYS_TABLES` |   587 |          60 | INDEX             |
| `SYS_TABLES` |   587 |          61 | INDEX             |
| `SYS_TABLES` |   587 |          62 | INDEX             |
| `SYS_TABLES` |   587 |          63 | INDEX             |
| `SYS_TABLES` |   587 |          64 | INDEX             |
| `SYS_TABLES` |   587 |          65 | INDEX             |
| `SYS_TABLES` |   587 |          66 | INDEX             |
| `SYS_TABLES` |   587 |          67 | INDEX             |
+--------------+-------+-------------+-------------------+
```

从 InnoDB 1.2 版本开始，还可以通过表 innodb_buffer_pool_stats 表来观察缓冲池的状态。

```
select pool_id, hit_rate, pages_made_young, pages_not_made_young from information_schema.innodb_buffer_pool_stats\G
*************************** 1. row ***************************
             pool_id: 0
            hit_rate: 0
    pages_made_young: 2578
pages_not_made_young: 169
*************************** 2. row ***************************
             pool_id: 1
            hit_rate: 0
    pages_made_young: 1
pages_not_made_young: 0
*************************** 3. row ***************************
             pool_id: 2
            hit_rate: 0
    pages_made_young: 971
pages_not_made_young: 0
*************************** 4. row ***************************
             pool_id: 3
            hit_rate: 0
    pages_made_young: 1
pages_not_made_young: 0
*************************** 5. row ***************************
             pool_id: 4
            hit_rate: 0
    pages_made_young: 20
pages_not_made_young: 0
*************************** 6. row ***************************
             pool_id: 5
            hit_rate: 0
    pages_made_young: 1795
pages_not_made_young: 10406
*************************** 7. row ***************************
             pool_id: 6
            hit_rate: 0
    pages_made_young: 2212
pages_not_made_young: 809
*************************** 8. row ***************************
             pool_id: 7
            hit_rate: 0
    pages_made_young: 0
pages_not_made_young: 0
```

我们还可以通过 innodb_buffer_page_lru 来观察每一个 LRU 列表的具体信息。

```
select table_name,space,page_number,page_type from information_schema.innodb_buffer_page_lru limit 10;
+------------+-------+-------------+-----------+
| table_name | space | page_number | page_type |
+------------+-------+-------------+-----------+
| NULL       |     0 |           3 | SYSTEM    |
| NULL       |     0 |         525 | UNDO_LOG  |
| NULL       |     0 |         526 | UNDO_LOG  |
| NULL       |     0 |         527 | UNDO_LOG  |
| NULL       |     0 |         528 | UNDO_LOG  |
| NULL       |     0 |         529 | UNDO_LOG  |
| NULL       |     0 |         530 | UNDO_LOG  |
| NULL       |     0 |         531 | UNDO_LOG  |
| NULL       |     0 |         532 | UNDO_LOG  |
| NULL       |     0 |         533 | UNDO_LOG  |
+------------+-------+-------------+-----------+
```

##### 重做日志缓冲

InnoDB 存储引擎首先将重做日志信息放入到`重做日志缓冲`中，然后按一定频率将其刷新到 Redo Log 中。`重做日志缓冲` 一般不需要设置的过大，因为一般情况下每一秒钟会将重做日志缓冲刷新到日志文件中， DBA 只需要确保每秒产生的事务量在这个缓冲大小之内即可。该大小可以通过 innodb_log_buffer_size 进行控制，默认为 8MB 。

```
show variables like 'innodb_log_buffer_size';
+------------------------+----------+
| Variable_name          | Value    |
+------------------------+----------+
| innodb_log_buffer_size | 33554432 |
+------------------------+----------+
```

有如下三种情况，会将重做日志缓冲刷新到重做日志中：
* Master Thread 每一秒将重做日志缓冲刷新到重做日志中；
* 每个事物提交时会将重做日志缓冲刷新到重做日志文件；
* 当重做日志缓冲池剩余空间小于二分之一时，重做日志缓冲刷新到重做日志文件。

##### 额外的内存池

额外的内存池中存储以下的一些重要信息：
* 每一个缓冲池中的帧缓冲（ frame buffer ）
* 每一个缓冲池中的缓冲控制对象（ buffer control block ）

它们记录了 LRU 、 锁、 等待等信息。 这些对象也是需要额外的占用一些内存空间的，因此，我们在设置 Buffer Pool 的大小时，也需要考虑额外内存池的空间占用大小。

### Checkpoint

由前面的介绍当中，我们可以了解到缓冲池设计的目的是为了减少磁盘操作很慢带来的性能瓶颈。缓冲池当中的页再被 DDL 操作修改了之后，此时的数据并没有立即被刷新到硬盘中。

如果每一个页发生变化，就将新页的版本刷新到磁盘，无疑会导致性能非常差。同时，如果在缓冲池的脏页还没有被刷新到磁盘之前，数据库服务器宕机了，就会造成数据的无法恢复。因此，大部分数据可靠性要求较高的系统（例如： RDB， 部分 Nosql 系统： Hbase MongoDB 等）都采用了 `预先写日志（ Write Ahead Log ）`的策略，即当事务提交时，先写 Redo Log ，再修改缓冲池中的页。在宕机时，通过 Redo Log 来完成数据的恢复（这也是事务 ACID 中 D 【Durability 持久性】的要求）。

假设重做日志可以无限地增大，同时缓冲池也足够大，能够缓冲所有数据库的数据，那么是不需要将缓冲池页的新版本刷新回磁盘。因为发生宕机时，完全可以通过重做日志来恢复整个数据库中的数据到宕机发生的时刻。

但是对于缓冲池可以存放所有的数据，对数据库服务器的内存要求非常高，要想将超大规模的磁盘数据全部放入到内存中，成本也会非常高。对于重做日志可以无限增大，会存在同样的问题，单台服务器的伸缩性总是有限的。此外，就算假设我们能够满足这两个条件，我们必须要考虑宕机之后，数据库根据重做日志的恢复时间，假设重做日志非常大，势必会导致恢复数据的时间非常长。

Checkpoint 正是为了解决上述问题而设计的：
* 缩短数据库的恢复时间；
* 当缓冲池不够用时，将脏页刷新到磁盘；
* 重做日志不可用时，刷新脏页。

当数据库发生宕机时，数据库不需要重做所有的日志，因为在 Checkpoint 之前的数据都已经刷新回磁盘。因此数据库只需要对 Checkpoint 后的重做日志进行恢复，能够大大缩短恢复的时间。此外，当缓冲池不够用时，根据 LRU 算法会溢出最近最少使用的页，若此页为脏页，需要强制执行 Checkpoint ，将脏页刷回磁盘。

对于 InnoDB 存储引擎来说，是通过 LSN （ Log Sequence Number ）来标记版本的。而 LSN 是 8 字节的数字。每个页中有 LSN ，重做日志中也有 LSN ， Checkpoint 也有 LSN。可以通过如下的命令进行查看。

```
SHOW ENGINE INNODB STATUS\G
---
LOG
---
Log sequence number 203842722114
Log flushed up to   203842722114
Pages flushed up to 203842722114
Last checkpoint at  203842722105
0 pending log flushes, 0 pending chkp writes
7384204 log i/o's done, 0.80 log i/o's/second
----------------------
```

在 InnoDB 存储引擎中，Checkpoint 发生的时间、条件及脏页的选择都比较复杂，分为了两种 Checkpoint：

* Sharp Checkpoint
* Fuzzy Checkpoint

Sharp Checkpoint 发生在数据库关闭时，将所有的脏页都刷新回磁盘，这是默认的工作方式，即全局变量 innodb_fast_shutdown = 1 。

Fuzzy Checkpoint 是在数据库运行时进行页的刷新，即只对一部分脏页进行刷新，而不是刷新所有的脏页至磁盘。可能在以下的情况下会发生 Fuzzy Checkpoint：

* Master Thread Checkpoint
大致以每秒或每十秒的频率从缓冲池的脏页列表中刷新一定比例的页会磁盘。整个过程是异步的，即此时 InnoDB 可以进行其它操作，用户查询线程不会被阻塞。

* FLUSH_LRU_LIST Checkpoint
由于 InnoDB 存储引擎需要保证 LRU 列表中，有大概 100 个空闲页可供使用。在 InnoDB 1.1x 版本之前，需要检查 LRU 列表中是否有足够的可用空间操作发生在用户查询线程中，显然这阻塞用户的查询操作。如果没有100个可用空闲页，那么 InnoDB 存储引擎会将 LRU 列表尾端的页移除。如果这些页中有脏页，就需要进行 Checkpoint 。 从 Mysql 5.6 版本（ InnoDB 1.2.x ）开始，这个检查被放在了一个单独的 Page Cleaner 线程中执行，并可以通过参数 innodb_lru_scan_depth 控制 LRU 列表中可用页的数量，默认值为 1024 。

```
show variables like 'innodb_lru_scan_depth%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_lru_scan_depth | 1024  |
+-----------------------+-------+
```

* Async/Sync Flush Checkpoint
是指重做日志文件不可用的情况，需要强制将一些页刷新回磁盘。其目的是为了保证重做日志的循环是要用的可用性。从 InnoDB 1.2.x 版本开始，将这部分刷新操作放入到了单独的 Page Cleaner Thread 中。

* Dirty Page too much Checkpoint
在脏页太多时，也会导致强制进行Checkpoint。其目的是为了确保缓冲池中有足够可用的页。可以由全局变量 innodb_max_dirty_page_pct 控制, 默认值为 75 ，表示当缓冲池中脏页的数量为 75% 时，强制进行 Checkpoint 。

```
show variables like 'innodb_max_dirty_pages_pct';
+----------------------------+-----------+
| Variable_name              | Value     |
+----------------------------+-----------+
| innodb_max_dirty_pages_pct | 75.000000 |
+----------------------------+-----------+
```

### Master Thread 的工作方式

#### InnoDB 1.0.x 版本之前的 Master Thread

Master Thread 具有最高的线程优先级。其内部由多个循环组成，并根据数据库不同的运行状态在这些循环中进行切换。

* 主循环（ Loop ）
* 后台循环（ background loop ）
* 刷新循环（ flush loop ）
* 暂停循环（ suspend loop ）

Loop 循环（主循环），大多数操作都在这个循环中进行，其中有两大部分操作：每1秒钟的操作和每10秒的操作。其伪代码如下：

```
void master_thread() {
  loop:
  for (int i = 0; i < 10; i++) {
    do something once per seconds
    sleep 1 second if necessary
  }
  do something once per 10 seconds
  goto loop;
}
```

其中每一秒的操作包括：
* 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是）；
* 合并插入缓冲（可能）；
* 至多刷新 100 个 InnoDB 的缓冲池中的脏页到磁盘（可能）；
* 如果当前没有用户活动，则切换到 background loop（可能）；

即使某个事务还没有提交， InnoDB 存储引擎仍然每秒会将重做日志缓冲中的内容刷新到重做日志文件。

合并插入缓冲并不是每秒都会发生的。InnoDB 会判断当前一秒内发生的 IO 次数是否小于 5 次，如果小于 5 次，则认为当之前那的 IO 压力很小，可以执行合并插入缓冲的操作。

刷新 100 个脏页也不是每秒都会发生的，会判断当前缓冲池中脏页的比例（ buf_get_modified_ratio_pct ）是否超过了配置文件中 innodb_max_dirty_pages_pct 这个参数，如果超过了这个阈值， InnoDB 存储引擎认为需要做磁盘同步的操作，将 100 个脏页写入磁盘中。

总结上述操作，伪代码可以进一步优化：

```
void master_thread() {
  goto loop;
  loop:
  for (int i = 0; i < 10; i++) {
    // 休眠 1 秒
    thread_sleep(1)
    do log buffer flush to disk
    if (last_one_second_io_times < 5) {
      do merge at most 5 insert buffer
    }
    if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct) {
      do buffer pool flush 100 dirty page
    }
    if (no user activity) {
      goto background loop
    }
  }
  do something once per 10 seconds
  background loop:
    do something
  goto loop;
}
```

接下来分析每 10 秒的操作：
* 刷新 100 个脏页到磁盘（可能的情况下）；
* 合并至多 5 个插入缓冲（总是）；
* 将日志缓冲刷新到磁盘（总是）；
* 删除无用的 Undo 页（总是）；
* 刷新 100 个或者 10 个脏页到磁盘（总是）。

在以上的过程中，InnoDB 会先判断过去 10 秒之内的磁盘 IO 操作是否小于 200 次，如果是， InnoDB 认为当前有足够的磁盘 IO 操作能力，因此将 100 个脏页刷新到磁盘。接着， InnoDB 会合并插入缓冲。不同于前面提到的"每一秒操作"，这时的合并插入缓冲总是会被执行的。之后， InnoDB 会再进行一次讲日志缓冲刷新到磁盘的操作。

接下来， InnoDB 会进行 full purge 操作，即删除无用的 Undo 页。对表进行 update 、 delete 这类操作时，原先的行被标记为删除，但是因为一致性读的关心，需要保留这些行版本信息。

最后，InnoDB 会判断缓冲池中脏页的比例（ buf_get_modified_ratio_pct ）是否有超过 70% 的脏页，如果有，则刷新 100 个脏页到磁盘，否则，就只刷新 10% 的脏页到磁盘。

background loop ，若当前没有用户活动（数据库空闲时）或者数据库关闭（ shutdown ），就会切换到这个循环：
* 删除无用的 Undo 页（总是）；
* 合并 20 个插入缓冲（总是）；
* 跳回到主循环（总是）；
* 不断刷新 100 个页直到符合条件

```
void master_thread() {
  goto loop;
  // 每1秒循环
  loop:
  for (int i = 0; i < 10; i++) {
    // 休眠 1 秒
    thread_sleep(1)
    do log buffer flush to disk
    if (last_one_second_io_times < 5) {
      do merge at most 5 insert buffer
    }
    if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct) {
      do buffer pool flush 100 dirty page
    }
    if (no user activity) {
      goto background loop
    }
  }
  // 每10秒循环
  if (last_ten_second_io_times < 200) {
    do buffer pool flush 100 dirty page
  }
  do merge at most 5 insert buffer
  do log buffer flush to disk
  do full purge
  if (buf_get_modified_ratio_pct > 70%) {
    do buffer pool flush 100 dirty page
  } else {
    buffer pool flush 10 dirty page
  }
  goto loop
  // 后台运行循环
  background loop:
  do full purge
  do merge 20 insert buffer
  if not idle:
    goto loop:
  else:
    goto flush loop
  flush loop:
  do buffer pool flush 100 dirty page
  if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct) {
    goto flush loop
  }
  goto suspend loop
  suspend loop:
  suspend_thread()
  waiting event
  goto loop;
}
```

#### InnoDB 1.2.x 版本之前的 Master Thread

从 1.0.x 版本之前的 Master Thread 来看，最大的问题是将很多配置参数进行了硬编码，例如刷新 100 个脏页到磁盘，合并 20 个插入缓冲等。这对于某些写密集型的应用来说，每秒可能会产生大于 100 个的脏页，或是产生大于 20 个插入缓冲的情况， Master Thread 的负荷会非常大。

这个问题在后来的 InnoDB 版本中提供了 innodb_io_capacity 全局变量来进行了改进，innodb_io_capacity 表示磁盘的 IO 吞吐量，默认值为 200 。对于刷新到磁盘页的数量， 会按照 innodb_io_capacity 的百分比来进行控制。其规则如下：

* 在合并插入缓冲时，合并插入缓冲的数量为 innodb_io_capacity 值的 5% ；
* 在从缓冲区刷新脏页时，刷新脏页的数量为 innodb_io_capacity 。

另一个问题是，参数 innodb_max_dirty_page_pct 的默认值在 1.0.x 版本之前比较大，该值默认为 90 。这会造成在每秒刷新缓冲池和 flush loop 判断该值时，如果该值大于 innodb_max_dirty_page_pct ，才刷新 100 个脏页，如果有很大的内存，或者数据库服务器的压力很大，这时才刷新脏页的速度反而会降低。从 1.0.x 版本开始， innodb_max_dirty_page_pct 的默认值变为了 75 ，这样既可以加快刷新脏页的频率，又保证了磁盘 IO 的负载。

InnoDB 1.0.x 版本还新增了一个全局变量 innodb_adaptive_flushing （自适应地刷新），该值影响每秒刷新脏页的数量。原来的刷新规则是： 脏页在缓冲池所占的比例小于 innodb_max_dirty_page_pct 时，不刷新脏页；大于 innodb_max_dirty_page_pct 时，刷新 100 个脏页。随着 innodb_adaptive_flushing 参数的引入， InnoDB 会通过一个名为 buf_flush_get_desired_flush_rate 的函数来判断需要刷新脏页最合适的数量。其通过判断产生重做日志的速度来决定最合适的刷新脏页数量。

此外，还引入了全局变量 innodb_purge_batch_size ，该参数可以控制每次 full purge 回收的 Undo 页数量。

```
show variables like 'innodb_purge_batch_size%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_purge_batch_size | 300   |
+-------------------------+-------+
```

#### InnoDB 1.2.x 版本的 Master Thread

在 InnoDB 1.2.x 版本中，Master Thread 的伪代码如下：

```
if InnoDB is idle
  srv_master_do_idle_tasks();
else
  srv_master_do_active_tasks();
```

其中 srv_master_do_idle_tasks() 就是之前版本中每 10 秒的操作，srv_master_do_active_tasks() 就是之前每 1 秒钟的操作。同时对于刷新脏页的操作，从 Master Thread 线程分离出了一个单独的 Page Cleaner Thread ，从而减轻了 Master Thread 的工作，同时进一步提高了系统的并发性。

### InnoDB 关键特性

#### 插入缓冲

##### Insert Buffer

在 InnoDB 中，主键是行的唯一标识。通常应用程序中行记录的写入顺序是按照主键递增的顺序进行写入的。因此，插入聚集索引一般是顺序的，不需要磁盘随机读取。

```
CREATE TABLE t (
    a int auto_increment,
    b varchar(30),
    primary key(a),
    key(b)
);
```

上述表当中，对于 a 字段的写入，由于是自增长主键，因此 Insert 操作是顺序写入的，能够避免随机磁盘 IO 。b 字段上创建了二级索引，对其写入是非顺序的，随机读取会导致插入操作的性能下降。

Insert buffer 可以尽量减少这种二级索引在顺序性无法保证的前提下，产生随机 IO 对写入性能的影响。对于二级索引的插入或更新操作，并非每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，如果存在，则直接插入；否则，先会放入到一个 Insert Buffer 对象中，而并非直接写入到了这个二级索引的叶子节点。然后再根据一定的频率和情况进行 Insert Buffer 和 二级索引页子节点的 merge 操作，由于在一个索引页上，因此通常能够将多个插入合并到一个操作中，这样能够大大提升二级索引写入的性能。

Insert buffer 需要二级索引满足两个条件：`索引是二级索引` 并且 `不是唯一索引` 。 之所以不能是唯一索引，是因为判断唯一性可能又会导致额外的随机 IO 读取。

##### Change Buffer

InnoDB 从 1.0.x 版本开始引入了 Change Buffer，可将其视为 Insert Buffer 的升级。其变化是可以对 Insert ， Update ， Delete 操作都进行缓冲，它们分别是： Insert Buffer ， Purge Buffer ， Delete Buffer 。

其中对于一条记录进行 Update 操作分为两个过程：将记录标记为已删除；真正将记录删除。

因此 Delete Buffer 对应 UPDATE 操作的第一个过程，即将记录标记为删除。 Purge Buffer 对应 UPDATE 操作的第二个过程，即将记录真正删除。

innodb_change_buffering 全局变量用于设置需要开启哪些 buffer ： inserts 、deletes 、 purges 、 changes 、 all 、 none 。 changes 表示启用 inserts 和 deletes 。

```
show variables like 'innodb_change_buffering%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_change_buffering | all   |
+-------------------------+-------+
```

从 InnoDB 1.2.x 版本开始，可以通过全局变量 innodb_change_buffer_max_size 来控制 Change Buffer 最大使用内存数量。其默认值为 25 ，表示最多能够使用缓冲池 1/4 的内存空间。

```
show variables like 'innodb_change_buffer_max_size%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| innodb_change_buffer_max_size | 25    |
+-------------------------------+-------+
```

可以通过 SHOW ENGINE INNODB STATUS 查看 Change Buffer 的统计信息：

```
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
```

seg size 表示当前 Change Buffer 的大小为： 2 * 16KB； free list len 表示空闲列表的长度；size 表示已经合并的记录页数量； merges 表示合并次数，也就是实际读取页的次数。

insert 表示 Insert Buffer 的次数；delete mark 表示 Delete Buffer 的次数； delete 表示 Purge Buffer 的次数； discarded operations 表示当前 Change Buffer 发生 merge 时，表已经被删除，此时就无需再将记录合并到二级索引中了。

#### 两次写 （ Double Write ）

当数据库发生宕机时，InnoDB 存储引擎正在写入某个页到表中，而这个页只写了一部分，比如总共 16KB 的页，只写入了前 4KB ，紧接着就发生了宕机，这种情况被称为部分写失效（ partial page write ），会导致数据的丢失。 Double Write 正是用于解决部分写失效的问题。

Double Write 由两部分组成，一部分是内存当中的 doublewrite buffer ，大小为 2MB ，另一部分是物理磁盘上共享表空间中连续的 128 个页，即两个 extent ，大小也为 2MB 。在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是会通过 memcpy 将脏页先复制到内存中的 doublewrite buffer 中，再分两次，每次以 1MB 顺序地写入共享表空间的物理磁盘上，然后马上调用 fsync ，同步磁盘。

![double-write](https://www.zhuxiaodong.net/static/images/double-write.png)

Innodb_dblwr_pages_written 表示总共写入的页数量， Innodb_dblwr_writes 表示实际的写入数。

```
show global status like 'innodb_dblwr%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Innodb_dblwr_pages_written | 671   |
| Innodb_dblwr_writes        | 656   |
+----------------------------+-------+
```

如果文件系统层面已经规避了部分写失效的问题（例如: ZFS 文件系统），我们就可以通过 skip_innodb_doublewrite 来关闭。

#### 自适应哈希索引 （ Adaptive Hash Index ）

InnoDB 会监控表上各个索引页的查询，如果认为 Hash 索引能够带来速度的提升，则会自动建立 Hash 索引。由于 Hash 索引的复杂度为 O(1) ，这能够大大的提升查询性能。

可以通过 SHOW ENGINE INNODB STATUS 查看 Adaptive Hash Index 的统计信息：

```
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Hash table size 3187567, node heap has 43 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
```

#### 异步 IO

异步 IO 是指在 InnoDB 发起多个 IO 请求时，可以以异步的方式进行，不用向同步 IO 一样串行的方式来进行请求。

此外，异步 IO 还可以合并多个 IO 请求为 1 个，提高 IOPS 的性能。例如用户访问页（ space , page_no ）为：
(8, 6) (8, 7) (8, 8)

假设每个页的大小为 16KB ，同步 IO 需要3次访问才能够获取到所有的页；而采用异步 IO ，会判断这三个页是连续的，可以只通过一次 IO 请求读取 48KB 的页。

异步 IO 在 Linux 下是通过内核级别的 Native AIO 来实现的。 innodb_use_native_aio 全局变量决定是否开启 AIO 。

```
show variables like 'innodb_use_native_aio';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_use_native_aio | ON    |
+-----------------------+-------+
```

#### 刷新邻接页 （ Flush Neighbor Page ）

当刷新一个脏页时，InnoDB 存储引擎会检测该页所在区 （ extent ）的所有页，如果是脏页，那么一起进行刷新。这样能够通过 AIO 机制将多个 IO 写入操作合并为一个 IO 操作。


### 1.2.x InnoDB 版本的其它优化

#### 永久性统计信息

与之前版本不同，Mysql 5.6 的 InnoDB 采用数据表分别管理各个数据表的统计信息，而在此之前的版本中，统计信息是由各个存储引擎放在内存中进行管理，由于经常发生变更，以至于主从服务器分别制定不同的查询执行计划。在 Mysql 5.6 版本中，InnoDB 会根据数据表的全部记录或索引，分别使用 mysql 数据库的 innodb_index_stats 与 innodb_table_stats 数据表管理 `基数（ Cardinality ）` 信息，并且引入了 innodb_stats_auto_recalc 系统变量，允许用户决定是否自动更新该信息。

#### 数据读取优化

针对 Mysql 5.6 的索引条件下推与多范围读优化， Mysql 服务器的 Handler API 得到了改善。在 InnoDB 和 MyISAM 存储引擎全部都实现了 Handler API，所以可以使用索引条件下推和多范围读功能。

#### 内核互斥体

InnoDB 内部有很多共享内存对象，最具代表性的有缓冲池的各个 Block 以及 Redo-log 。由于 Mysql 的多线程架构，这些共享的内存对象由众多的客户连接竞争占用，然后释放，因此必须要对这些内存对象做同步处理。在 Mysql 当中使用锁（互斥体或信号量）来实现同步。

Mysql 5.6 版本中进行了一定程度的改进，事务的并发控制与 MVCC 等内存结构体都由单独的读写所进行控制。

#### 基于多线程的 Undo 清理

为了进行 MVCC （ Multi Version Concurrency Control ， 多版本并发控制 ）与回滚， InnoDB 单独管理着 Undo 空间 （ Undo Space 、 Rollback segment ）。

```
insert into member(m_id, m_name, m_area) values (12, 'lz14', '首尔');

update member set m_area = '京' where m_id=12;
```

执行上述 update 语句后，即使不实行 commit ， member 表的列也立即被修改为 '京' 。但是， InnoDB 存储引擎无法得知用户执行 update 语句后是执行 rollback 操作还是 commit 操作。因此，为了应对用户可能进行 rollback 操作的情形，先要将修改为 '京' 之前的值 （'首尔'） 保存到某一个地方。

![innodb-buffer-undo-log](https://www.zhuxiaodong.net/static/images/innodb-buffer-undo-log.png)

上述图描述了在执行 update 语句之后，进行 commit 或 rollback 之前 InnoDB 缓冲池与磁盘数据文件的状态。

由于 Undo 日志区域中会积累很多变更之前的信息，因此必须在某一个时间点对 Undo 日志进行删除以确保足够的磁盘空间，这样才能够保存以后的变更内容。删除 Undo 日志的操作称为 Undo 清理（ Undo purge ）。在 Mysql 5.1 之前，这一操作由 InnoDB 的主线程负责，由于 InnoDB 主线程已经承担其它更多的工作，因此有可能造成 InnoDB 主线程成为处理瓶颈，Undo Log 清理不及时的问题。

在 Mysql 5.5 版本中引入了专门负责 Undo 日志清理的线程，可以通过 innodb_purge_threads 系统变量来进行设置，其值只能设置为 0 或 1，这意味着最多能设置一个线程进行处理。 Mysql 5.6 版本改进了这一问题， innodb_purge_threads 能够设置大于 1 的值。

#### 独立的刷新线程

用户执行 DML 语句时，变更的数据会先记录到 Redo 日志中，同时永久保留在磁盘中。 InnoDB 只在内存（ InnoDB 缓冲池 ）中更改实际数据表的数据。至此， Mysql 服务器会向客户端用户返回 “查询执行完毕” 的信息。此后， InnoDB 存储引擎会再某一个时候将仅在内存（ InnoDB 缓冲池 ）中更改的数据永久记录到磁盘。我们将 InnoDB 缓冲池中更改的数据称为 “胀数据”（ Dirty ），将脏数据永久记录在磁盘上的操作称为“刷新” （ Flush ）。 InnoDB 缓冲池中管理单位，一般为 16 KB，又被称之为 “脏页” （ Dirty page ）。

在 Mysql 5.5 版本中，脏页的刷新由 InnoDB 主线程来负责执行，如果主线程负荷太大，导致刷新操作无法执行，可能会造成严重的数据一致性问题。 Mysql 5.6 版本引入了单独的线程来解决该问题。

#### 可变页大小

Mysql 5.5 以前的版本中， InnoDB 页面固定大小为 16 KB。这样会带来的问题是，若查询中每次只读取几条记录，例如记录的大小只有几十个字节，以 16 KB 页面为单位读磁盘并将 16 KB 的数据全部刷新到磁盘，这样做相当不合理。

从 Mysql 5.6 版本开始，可以将页面大小调整为 4 KB 或 8 KB 。这对于使用 SSD 硬盘做为数据库物理存储而言，会非常有用，因为能够确保每次读取传送的数据尽量的少。

#### 表空间复制

如果我们需要在不同的数据库服务器上快速的复制很大的表结构以及其数据，可以参考采用表空间复制的功能。具体可以参考这篇[文章](http://mysqlblog.fivefarmers.com/2012/11/07/smarter-innodb-transportable-tablespace-management-operations/) 。

#### 独立的 Undo 日志空间

Mysql 5.5 之前的 InnoDB 中， Undo 区域是系统表空间的一部分，此外，写缓冲或 DoubleWrite 缓冲等数据也会使用系统表空间。由于 Undo 区域主要基于随机 I/O 工作，而 DoubleWrite 缓冲主要基于顺序 I/O 工作。这意味着，很难为保存系统表空间而选定磁盘文职。

Mysql 5.6 版本引入了3个系统变量，用于将 Undo 区域放入非系统表空间的单独空间中。

* innodb_undo_directory ： 用于设置 Undo 区域的单独目录，默认为 “.” , 即直接使用系统表空间。

* innodb_undo_tablespaces ： 用于设置将 Undo 区域划分成多个数据表空间，设置的目的是减少多个线程访问 Undo 区域的互斥体竞争。

* innodb_undo_logs ： 指定 Rollback segment 的个数。

#### 只读事务优化

在 Mysql 5.5 以前的 InnoDB 存储引擎中，处理所有查询时都会自动开始/结束事务。为了保证事务的正常进行，内部要派发事务ID，并且每次都要分配内存结构体，以保持事务正常运行。

Mysql 5.6 InnoDB 中，对于下面的两种情况，将使用只读事务进行处理。在只读事务的情况下，不会派发事务 ID ，也不会分配变更数据所需要的内存空间，因此能够获得更快的性能。

* 以 "START TRANSACTION READ ONLY" 开始的事务
* Auto Commit 处于激活状态下而只进行 SELECT 查询的事务（ SELECT .. FOR UPDATE 除外 ）

在只读事务下，如果执行 UPDATE 或 INSERT 等语句时，会报出如下的错误：

```
ERROR 1792 (25006): Cannot execute statement in a READ ONLY transaction.
```

此外，我们可以通过 information_schema 数据库的 INNODB_TRX 数据表 中的 trx_is_read_only 数据列来区分是否为只读事务。

#### 缓冲池转存与加载

Mysql 5.5 之前的版本中，服务器重启后，通常刚开始的 2 ~ 30 分钟里，由于缓冲池中的数据还没有被加载并缓存，因此会导致这个阶段的性能急剧降低。

MariaDB 5.5 版本能够支持数据转存与加载的功能，用来解决上述的问题。

##### Mysql 5.6 转存与加载 （MariaDB 10.0 版本开始使用该方式进行转存和加载）

Mysql 5.6 版本中，对缓冲池页面转存的只会记录页面所属的表空间与页面ID，因此即使 buffer_pool 本身很大，转存文件也会非常小。

有 4 个系统变量对转存和加载功能进行控制：

* innodb_buffer_pool_dump_now ： 设置为 ON 会立即转存缓冲池的内容。在转存结束之后，该变量的值立即更新为 OFF 。
* innodb_buffer_pool_load_now ： 设置为 ON 时，会立即读取由 innodb_buffer_pool_filename 系统变量指定的文件，并加载数据与索引页面至 InnoDB 缓冲池。加载完了之后，该系统变量重设为 OFF 。
* innodb_buffer_pool_dump_at_shutdown ：默认值为 ON ， 服务器关闭时会将 InnoDB 缓冲池的内容转存到文件。
* innodb_buffer_pool_load_at_startup ： 默认值为 ON ， 服务器启动时会自动加载转存文件中的数据和索引文件至 InnoDB 缓冲池中。

innodb_buffer_pool_dump_now 和 innodb_buffer_pool_load_now 有一个非常有价值的应用场景，比如对于大表修改表结构修改时，我们可以使用 innodb_buffer_pool_dump_now 对缓冲池进行缓存，修改完成表结构之后，立即运行 innodb_buffer_pool_load_now 来再次加载缓冲池，以提升数据库访问的性能。

#### 重做日志大小

为了确保在服务器崩溃时保证数据安全，InnoDB 存储引擎首先会将提交的事务内容记录到日志文件，而将数据的实际修改放在后面以批处理的方式来进行处理。这被称为 Redo-Log，或 WAL （ Write Ahead Log 预写式日志）。 InnoDB 的重做日志是以一种循环的方式来使用多个文件的。

![redo-log](https://www.zhuxiaodong.net/static/images/redo-log.png)

以下3个系统变量可以设置 InnoDB 中的日志文件：

* innodb_log_file_size ： 用于设置各个日志文件的大小。服务器启动时，会按照这个系统变量的值创建日志文件，并且日志文件的大小不可变。
* innodb_log_files_in_group ： 用于设置要创建的日志文件个数。
* innodb_log_group_home_dir ： 用于设置重做日志的目录。

同时，Mysql 5.6 版本中，对于更改重做日志的大小也做出了优化，不再需要向之前非常繁琐的操作步骤，只需要直接在 my.cnf 文件当中调整 innodb_log_file_size 的大小，并重启服务器就好了。

#### 死锁记录

在 Mysql 5.5 版本以及之前的 InnoDB 中， 死锁内容只能在 SHOW ENGINE INNODB STATUS 命令查看到，并且只能查到最后一次发生的死锁内容。虽然我们可以通过一些第三方公司开发的工具来查询到所有的死锁（例如： https://www.percona.com/doc/percona-toolkit/2.2/pt-deadlock-logger.html ），但是从 Mysql 5.6 版本开始，我们可以通过设置 innodb_print_all_deadlocks 系统变量（默认为 OFF）来将产生的所有的死锁记录写入到错误日志中。
