title: learning mariadb & mysql transaction(part-10)
date: 2018-02-08 09:52:46
categories: programming
tags:
- mysql
- mariadb
- innodb
- transaction
- ACID
---

# ACID
---

## A （ Atomicity ）原子性

原子性是指整个数据库事务是不可分割的工作单位。只有使事务中所有的数据库操作都执行成功，才算整个事务成功。事务中任何一个 SQL 语句执行失败，已经执行成功的 SQL 语句也必须撤销，数据库状态应该回退到执行事务前的状态。

<!--more-->

## C （ consistency ）一致性

一致性是指事务将数据库从一种状态变为下一种一致的状态。在事务开始之前和事务结束以后，数据库的完整性约束没有被破坏。例如，在表中有一个字段为姓名，为唯一约束，即在表中姓名不能重复。如果一个事务对姓名字段进行了修改，但是在事务提交或事务操作发生回滚后，表中的姓名变得非唯一了，这就破坏了事务的一致性要求，即事务将数据库从一种状态变为了一种不一致的状态。因此，事务是一致性的单位，如果事务中某个动作失败了，系统可以自动撤销事务——返回初始化的状态。

## I （ isolation ）隔离性

隔离性还有其他的称呼，如并发控制（ concurrency control ）、可串行化（ serializability ）、锁（ locking ）等。事务的隔离性要求每个读写事务的对象对其他事务的操作对象能相互分离，即该事务提交前对其他事务都不可见，通常这使用锁来实现。当前数据库系统中都提供了一种粒度锁（ granular lock ）的策略，允许事务仅锁住一个实体对象的子集，以此来提高事务之间的并发度。

## D （ durability ）持久性
事务一旦提交，其结果就是永久性的。即使发生宕机等故障，数据库也能将数据恢复。需要注意的是，只能从事务本身的角度来保证结果的持久性。例如，在事务提交后，所有的变化都是永久的。即使当数据库因为崩溃而需要恢复时，也能保证恢复后提交的数据都不会丢失。但若不是数据库本身发生故障，而是一些外部的原因，如硬件损坏、自然灾害等原因导致数据库发生问题，那么所有提交的数据可能都会丢失。因此持久性保证事务系统的高可靠性（ High Reliability ），而不是高可用性（ High Availability ）。对于高可用性的实现，事务本身并不能保证，需要一些其它的机制来共同配合来完成。

# 事务的实现
---

事务的隔离性由我们在[上一章](http://www.zhuxiaodong.net/2018/learning-mariadb-mysql-lock-part-9/)介绍的锁机制实现。而原子性、一致性、持久性通过数据库的 redo log 和 undo log 来完成。 redo log 称之为重做日志，用来保证事务的原子性和持久性。 undo log 用来保证事务的一致性。

redo 和 undo 的作用都可以视为是一种恢复操作， redo 恢复提交事务修改的页操作，而 undo 回滚行记录到某个特定版本。因此两者记录的内容不同， redo 通常是物理日志，记录的是页的物理修改操作。 undo 是逻辑日志，根据每行记录进行记录。

## redo

### 基本概念

重做日志用来实现事务的持久性，即事务 ACID 中的 D 。它由两部分组成：一是内存中的重做日志缓冲（ redo log buffer ），由于是在内存中，因此很容易丢失的；二是重做日志文件（ redo log file ），由于是磁盘上，因此是持久不容易丢失的。

InnoDB 是事务的存储引擎，通过 Force Log at Commit 机制实现事务的持久性，即当事务提交（ COMMIT ）时，必须先将该事务的所有日志写入到重做日志文件进行持久化，待事务的 COMMIT 操作完成才算完成。在 InnoDB 存储引擎中，重做日志由两部分组成，即 redo log 和 undo log 。 redo log 用来保证事务的持久性， undo log 用来帮助事务回滚及 MVCC 的功能。 redo log 基本上都是顺序写的，在数据库运行时不需要对 redo log 的文件进行读取操作。而 undo log 是需要进行随机读写的。

为了确保每次日志都写入重做日志文件，在每次将重做日志缓冲写入重做日志文件后， InnoDB 存储引擎都需要调用一次 fsync 操作。由于重做日志文件打开并没有使用 O_DIRECT 选项，因此重做日志缓冲先写入文件系统缓存。为了确保重做日志写入磁盘，必须进行一次 fsync 操作。由于 fsync 的效率取决于磁盘的性能，因此磁盘的性能决定了事务提交的性能，也就是数据库的性能。

关于 fsync 可以参考： http://blog.csdn.net/cywosp/article/details/8767327

InnoDB 存储引擎允许用户手工设置非持久性的情况发生，以此提高数据库的性能。即当事务提交时，日志不写入重做日志文件，而是等待一个时间周期后再执行 fsync 操作。由于并非强制在事务提交时进行一次 fsync 操作，显然这可以显著提高数据库的性能。但是当数据库发生宕机时，由于部分日志未刷新到磁盘，因此会丢失最后一段时间的事务。

参数 innodb_flush_log_at_trx_commit 用来控制重做日志刷新到磁盘的策略。该参数的默认值为 1。可以设置以下3个默认值：
* 0 ：表示事务提交时不进行写入重做日志操作，这个操作仅在 master thread 中完成，而在 master thread 中每1秒会进行一次重做日志文件的 fsync 操作。
* 1 ：表示事务提交时必须调用一次 fsync 操作。
* 2 ：表示事务提交时将重做日志写入重做日志文件，但仅写入文件系统的缓存中，不进行 fsync 操作。在这个设置下，当 Mysql 数据库发生宕机而操作系统不发生宕机时，并不会导致事务的丢失。而当操作系统宕机时，重启数据库后会丢失文件系统缓存中，还没有来得及同步到日志文件的那部分事务。

我们可以在某些不需要 ACID 特性的情况下，临时将 innodb_flush_log_at_trx_commit 参数设置为 0 或 2 ，从而大大提升执行的性能。

### binlog 与 redo log 的区别

从表面上看，二进制日志与重做日志非常类似，都记录了对数据库操作的日志。但是从本质上来看，两者有着非常大的不同。

重做日志是在 InnoDB 存储引擎层产生的，而二进制日志是在 Mysql 数据库的层面上产生的，这意味着对于任何存储引擎都会产生二进制日志。

两种日志记录的内容形式不同。 Mysql 数据库上层的二进制日志是一种逻辑日志，其记录的是对应的 SQL 语句。 InnoDB 存储引擎层面的重做日志是物理格式日志，其记录的是对于每个页的修改。

两种日志记录写入磁盘的时间点不同，二进制日志只在事务提交完成后进行一次写入。而 InnoDB 存储引擎的重做日志在事务进行中不断地被写入，这表现为日志并不是随事务提交的顺序进行写入的。

### log block

在 InnoDB 存储引擎中，重做日志都是以 512 字节进行存储的。这意味着重做日志缓存、重做日志文件都是以块（ block ）的方式进行保存的，称之为重做日志块（ redo log block ），每块的大小为 512 字节。

若一个页中产生的重做日志数量大于 512 字节，就需要分割为多个重做日志块进行存储。由于重做日志块的大小和磁盘扇区大小一样，都是 512 字节，因此重做日志的写入可以保证原子性，不需要 doublewrite 技术。关于扇区大小和块大小的关系，可以参考[这里](http://blog.csdn.net/my_bai/article/details/73331360)。

重做日志除了日志本身之外，还由 log block header 和 log block tailer 两部分组成。重做日志头一共占用了 12 字节，重做日志尾占用 8 字节。因此每个重做日志块实际可以存储的大小为 492 字节。

![redo-log-buffer](https://www.zhuxiaodong.net/static/images/redo-log-buffer.png)

### 重做日志的格式

不同的数据库操作会有对应的重做日志格式。此外，由于InnoDB存储引擎的存储管理是基于页的，故其重做日志格式也是基于页的。虽然有着不同的重做日志格式，但是它们有着通用的头部格式。

* redo_log_type ：重做日志的类型。
* space ： 表空间的ID。
* page_no ：页的偏移量。

### LSN （ Log Sequence Number ）

LSN 表示的是日志的序列号，在 InnoDB 存储引擎中，LSN 占用 8 字节，并且是单调递增的。其表示的含义为：

* 重做日志写入的总量
* checkpoint 的位置
* 页的版本

LSN 表示事务写入重做日志的字节的总量。例如当前重做日志的 LSN 为1 000，有一个事务 T1 写入了100字节的重做日志，那么 LSN 就变为了1100，若又有事务 T2 写入了 200 字节的重做日志，那么 LSN 就变为了1300。可见 LSN 记录的是重做日志的总量，其单位为字节。

LSN 不仅记录在重做日志中，还存在于每个页中。在每个页的头部，有一个值 FIL_PAGE_LSN ，记录了该页的 LSN 。在页中， LSN 表示该页最后刷新时 LSN 的大小。因为重做日志记录的是每个页的日志，因此页中的LSN用来判断页是否需要进行恢复操作。例如，页 P1 的 LSN 为 10000，而数据库启动时， InnoDB 检测到写入重做日志中的 LSN 为 13000，并且该事务已经提交，那么数据库需要进行恢复操作，将重做日志应用到 P1 页中。同样的，对于重做日志中 LSN 小于 P1 页的 LSN ，不需要进行重做，因为 P1 页中的 LSN 表示页已经被刷新到该位置。

我们可以通过 SHOW ENGINE INNODB STATUS 查看 LSN 的情况：

```
---
LOG
---
Log sequence number 11 3047174608
Log flushed up to   11 3047174608
Last checkpoint at  11 3047174608
0 pending log writes, 0 pending chkp writes
142 log i/o's done, 0.00 log i/o's/second
```

Log sequence number 表示当前的 LSN ,  Log flushed up to 表示刷新到重做日志文件的 LSN， Last checkpoint at 表示刷新到磁盘的 LSN 。

虽然在上面的例子中， Log sequence number 和 Log flushed up to 的值是相同的，但是在实际生产环境中，该值有可能是不同的。因为在一个事务中从日志缓冲刷新到重做日志文件并不只是在事务提交时发生，每秒都会有从日志缓冲刷新到重做日志文件的动作。

### 恢复

InnoDB 存储引擎在启动时不管上次数据库运行时是否正常关闭，都会尝试进行恢复操作。因为重做日志记录的是物理日志，因此恢复的速度比逻辑日志，如二进制日志，要快很多。与此同时， InnoDB 存储引擎自身也对恢复进行了一定程度的优化，如顺序读取及并行应用重做日志，这样可以进一步地提高数据库恢复的速度。

由于 checkpoint 表示已经刷新到磁盘页上的 LSN ，因此在恢复过程中仅需恢复checkpoint开始的日志部分。对于下图中的例子，当数据库在 checkpoint 的 LSN 为 10000 时发生宕机，恢复操作仅恢复 LSN 10000～13000范围内的日志。

![redo-log-recover](https://www.zhuxiaodong.net/static/images/redo-log-recover.png)

## undo

### 基本概念

重做日志记录了事务的行为，可以很好地通过其对页进行“重做”操作。但是事务有时还需要进行回滚操作，这时就需要 undo 。因此在对数据库进行修改时， InnoDB 存储引擎不但会产生 redo ，还会产生一定量的 undo 。这样如果用户执行的事务或语句由于某种原因失败了，又或者用户用一条 ROLLBACK 语句请求回滚，就可以利用这些 undo 信息将数据回滚到修改之前的样子。

undo 存放在数据库内部的一个特殊段（ segment ）中，这个段称为 undo 段（ undo segment ）。 undo 段位于共享表空间内。可以通过 py_innodb_page_info.py 工具来查看当前共享表空间中 undo 的数量。可以看到如下的输出中，共享表空间 ibdata1 内有 1763 个 undo 页。

```
./py_innodb_page_info.py /alidata1/mysql/9000/data/ibdata1
Total number of page: 4864:
Insert Buffer Bitmap: 1
System Page: 162
Transaction system Page: 3
Freshly Allocated Page: 2865
Undo Log Page: 1763
File Segment inode: 4
B-tree Node: 65
File Space Header: 1
```

用户通常对 undo 有这样的误解： undo 用于将数据库物理地恢复到执行语句或事务之前的样子——但事实并非如此。 undo 是逻辑日志，因此只是将数据库逻辑地恢复到原来的样子。所有修改都被逻辑地取消了，但是数据结构和页本身在回滚之后可能大不相同。这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如，一个事务在修改当前一个页中某几条记录，同时还有别的事务在对同一个页中另几条记录进行修改。因此，不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。

例如，用户执行了一个 INSERT 10W 条记录的事务，这个事务会导致分配一个新的段，即表空间会增大。在用户执行 ROLLBACK 时，会将插入的事务进行回滚，但是表空间的大小并不会因此而收缩。因此，当 InnoDB 存储引擎回滚时，它实际上做的是与先前相反的工作。对于每个 INSERT ， InnoDB 存储引擎会完成一个 DELETE ；对于每个 DELETE ， InnoDB 存储引擎会执行一个 INSERT ；对于每个 UPDATE， InnoDB 存储引擎会执行一个相反的 UPDATE ，将修改前的行放回去。

除了回滚操作， undo 的另一个作用是 MVCC ，即在 InnoDB 存储引擎中 MVCC 的实现是通过 undo 来完成。当用户读取一行记录时，若该记录已经被其他事务占用，当前事务可以通过 undo 读取之前的行版本信息，以此实现非锁定读取。

最后也是最为重要的一点是， undo log 会产生 redo log ，也就是 undo log 的产生会伴随着 redo log 的产生，这是因为 undo log 也需要持久性的保护。

### Undo 存储管理

InnoDB 存储引擎对 undo 的管理方式采用段的方式，有 rollback segment ，每个回滚段中记录了 1024 个 undo log segment ，而在每个 undo log segment 段中进行 undo 页的申请。共享表空间偏移量为 5 的页（0，5）记录了所有 rollback segment header 所在的页，这个页的类型为 FILE_PAGE_TYPE_SYS 。

从 InnoDB 1.2 版本开始，可以对 rollback segment 做进一步的设置：

* innodb_undo_directory ：设置 rollback segment 文件所在的路径。这意味着 rollback segment 可以存放在共享表空间以外的位置，即可以设置为独立表空间。该参数的默认值为“.”，表示当前 InnoDB 存储引擎的目录。
* innodb_undo_logs ：用来设置 rollback segment 的个数，默认值为 128 。在 InnoDB 1.2版本中，该参数用来替换之前版本的参数 innodb_rollback_segments 。
* innodb_undo_tablespaces ：用来设置构成 rollback segment 文件的数量，这样 rollback segment 可以较为平均地分布在多个文件中。设置该参数后，会在路径 innodb_undo_directory 看到 undo 为前缀的文件，该文件就代表 rollback segment 文件。

```
show variables like 'innodb_undo%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_undo_directory    | ./    |
| innodb_undo_log_truncate | OFF   |
| innodb_undo_logs         | 128   |
| innodb_undo_tablespaces  | 0     |
+--------------------------+-------+
```

需要特别注意的是，事务在 undo log segment 分配页并写入 undo log 的这个过程同样需要写入重做日志。当事务提交时， InnoDB 存储引擎会做如下的两件事情：
* 将 undo log 放入列表中，以供之后的 purge 操作
* 判断 undo log 所在的页是否可以重用，若可以分配给下个事务使用

事务提交后并不能马上删除 undo log 及 undo log 所在的页。这是因为可能还有其他事务需要通过 undo log 来得到行记录之前的版本。故事务提交时将 undo log 放入一个链表中，是否可以最终删除 undo log 及 undo log 所在页由 purge 线程来判断。

此外，若为每一个事务分配一个单独的 undo 页会非常浪费存储空间，特别是对于 OLTP 的应用类型。因为在事务提交时，可能并不能马上释放页。假设某应用的删除和更新操作的TPS（ transaction per second ）为1000，为每个事务分配一个 undo 页，那么一分钟就需要 1000*60 个页，大约需要的存储空间为 1GB 。若每秒的 purge 页的数量为20，这样的设计对磁盘空间有着相当高的要求。因此，在 InnoDB 存储引擎的设计中对 undo 页可以进行重用。具体来说，当事务提交时，首先将 undo log 放入链表中，然后判断 undo 页的使用空间是否小于3/4，若是则表示该 undo 页可以被重用，之后新的 undo log 记录在当前 undo log 的后面。由于存放 undo log 的列表是以记录进行组织的，而 undo 页可能存放着不同事务的 undo log ，因此 purge 操作需要涉及磁盘的离散读取操作，是一个比较缓慢的过程。

### purge

delete 和 update 操作可能并不会直接删除原有的数据。例如： `delete from t where a=1;` ，表 t 上列 a 有聚集索引，列 b 上有辅助索引。对于上述的 delete 操作，通过前面关于 undo log 的介绍已经知道仅是将主键列等于1的记录 delete flag 设置为1，记录并没有被删除，即记录还是存在于B+树中。其次，对辅助索引上 a 等于1， b 等于1的记录同样没有做任何处理，甚至没有产生 undo log 。而真正删除这行记录的操作其实被“延时”了，最终在 purge 操作中完成。

purge 用于最终完成 delete 和 update 操作。这样设计是因为 InnoDB 存储引擎支持 MVCC ，所以记录不能在事务提交时立即进行处理。这时其他事务可能正在引用这行，故 InnoDB 存储引擎需要保存记录之前的版本。是否可以删除该条记录通过 purge 来进行判断。若该行记录已不被任何其他事务引用，那么就可以进行真正的 delete 操作。可见， purge 操作是清理之前的 delete 和 update 操作，将上述操作“最终”完成。而实际执行的操作为 delete 操作，清理之前行记录的版本。

# 事务控制语句
---

## auto commit

在 Mysql 的默认设置下，事务都是自动提交的（ auto commit ）的，即执行的 SQL 语句后就会马上执行 COMMIT 操作。

更多请参考： https://dev.mysql.com/doc/refman/5.7/en/innodb-autocommit-commit-rollback.html

## Mysql 支持的事务语句包括

* START TRANSACTION | BEGIN ：显示开启一个事务。
* COMMIT ：要想使用这个语句的最简形式，只需发出 COMMIT 。也可以更详细一些，写为 COMMIT WORK ，不过这二者几乎是等价的。 COMMIT 会提交事务，并使得已对数据库做的所有修改成为永久性的。   
* ROLLBACK ：要想使用这个语句的最简形式，只需发出 ROLLBACK 。同样地，也可以写为 ROLLBACK WORK ，但是二者几乎是等价的。回滚会结束用户的事务，并撤销正在进行的所有未提交的修改。   
* SAVEPOINT identifier ：SAVEPOINT 允许在事务中创建一个保存点，一个事务中可以有多个 SAVEPOINT 。
* RELEASE SAVEPOINT identifier ：删除一个事务的保存点，当没有一个保存点执行这句语句时，会抛出一个异常。   
* ROLLBACK TO[SAVEPOINT]identifier ：这个语句与 SAVEPOINT 命令一起使用。可以把事务回滚到标记点，而不回滚在此标记点之前的任何工作。例如可以发出两条 UPDATE 语句，后面跟一个 SAVEPOINT ，然后又是两条 DELETE 语句。如果执行 DELETE 语句期间出现了某种异常情况，并且捕获到这个异常，同时发出了 ROLLBACK TO SAVEPOINT 命令，事务就会回滚到指定的 SAVEPOINT ，撤销 DELETE 完成的所有工作，而 UPDATE 语句完成的工作不受影响。   
* SET TRANSACTION：这个语句用来设置事务的隔离级别。 InnoDB 存储引擎提供的事务隔离级别有：READ UNCOMMITTED 、 READ COMMITTED 、 REPEATABLE READ 、 SERIALIZABLE 。

### 隐式提交的 SQL 语句

隐式提交意味着，在完成语句执行之后，会有一个隐式的 COMMIT 操作，**这意味着你无法通过 ROLLBACK 语句来回滚已经执行的这些操作**。

* DDL 语句： ALTER TABLE | VIEW | PROCEDURE | DATABASE ； CREATE TABLE | VIEW | PROCEDURE | DATABASE ； DROP TABLE | VIEW | PROCEDURE | DATABASE ； **TRUNCATE TABLE**

* 用户或权限相关的语句：CREATE USER ； DROP USER ； GRANT | RENAME USER ； REVOKE | SET PASSWORD

* 管理语句： ANALYZE TABLE ； CHECK TABL ； OPTIMIZE TABLE ； REPAIR TABLE

# 事务操作的统计
---

由于 InnoDB 存储引擎是支持事务的，因此 InnoDB 存储引擎的应用需要在考虑每秒请求数（ Question Per Second, QPS ）的同时，还应该关注每秒事务处理的能力（ Transaction Per Second, TPS ）。

计算 TPS 的公式为： `(com_commit + com_rollback) / time`

利用这种方式计算的前提是：所有事务都必须都是显式提交（即使用 START TRANSACTION / COMMIT / ROLLBACK）的 ，因为如果存在隐式地提交和回滚（ autocommit = 1 ），相关的数据就不会被统计到 com_commit 和 com_rollback 变量中。

下面的示例可以很好的说明这一点：

```
show global status like 'com_commit'\G
*************************** 1. row ***************************
Variable_name: Com_commit
        Value: 3124

insert into t (a, b, c) values (7, 4, 5);

# com_commit 的值仍然是 3124
show global status like 'com_commit'\G
*************************** 1. row ***************************
Variable_name: Com_commit
        Value: 3124
```

更多关于监控方面的内容，可以参考：

https://jin-yang.github.io/post/mysql-monitor.html
http://hongyitong.github.io/2017/01/11/Mysql%20%E7%9B%91%E6%8E%A7%E6%80%A7%E8%83%BD%E7%8A%B6%E6%80%81%20QPS%E3%80%81TPS/

# 事务的隔离级别
---

* READ UNCOMMITTED
* READ COMMITTED
* REPEATABLE READ
* SERIALIZABLE

InnoDB 存储引擎默认支持的隔离级别是 REPEATABLE READ ，但是与标准 SQL 不同的是，InnoDB 存储引擎在 REPEATABLE READ 事务隔离级别下，使用 Next-Key Lock 锁的算法，因此避免幻读的产生。这与其他数据库系统（如MicrosoftSQL Server数据库）是不同的。所以说， InnoDB 存储引擎在默认的 REPEATABLE READ 的事务隔离级别下已经能完全保证事务的隔离性要求，即达到 SQL 标准的 SERIALIZABLE 隔离级别。

隔离级别越低，事务请求的锁越少或保持锁的时间就越短。这也是为什么大多数数据库系统默认的事务隔离级别是 READ COMMITTED 。 InnoDB 存储引擎中选择 REPEATABLE READ 的事务隔离级别并不会有任何性能的损失。同样地，即使使用 READ COMMITTED 的隔离级别，用户也不会得到性能的大幅度提升。

在 InnoDB 存储引擎中，可以使用以下命令来设置当前会话或全局的事务隔离级别：

```
SET [GLOBAL | SESSION] TRANSACTION ISOLATION LEVEL
{ READ UNCOMMITTED
  | READ COMMITTED
  | REPEATABLE READ
  | SERIALIZABLE
}
```

如果想在 Mysql 数据库启动时就设置事务的默认隔离级别，就需要修改 Mysql 的配置文件，可以在 my.cnf 配置文件中添加如下的行：

```
[mysqld]
transaction-isolation = READ-COMMITTED
```

查看当前 session 的事务隔离级别：

```
select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
```

查看全局的事务隔离级别：

```
select @@global.tx_isolation;
+-----------------------+
| @@global.tx_isolation |
+-----------------------+
| REPEATABLE-READ       |
+-----------------------+
```

在 REPLICATION 的情况下，Mysql 5.1 之前的版本中，如果使用 READ COMMITTED 隔离级别， 并且二进制日志使用的是 STATEMENT 的方式， 可能会出现主从数据不一致问题的。

在 MySQL 5.1 版本之后，因为支持了 ROW 格式的二进制日志记录格式，可以避免该情况的发生，所以可以放心使用 READ COMMITTED 的事务隔离级别。但即使不使用 READ COMMITTED 的事务隔离级别，也应该考虑将二进制日志的格式更换成 ROW ，因为这个格式记录的是行的变更，而不是简单的SQL语句，所以可以避免一些不同步现象的产生，进一步保证数据的同步。 InnoDB 存储引擎的创始人 HeikkiTuuri 也在 http://bugs.mysql.com/bug.php?id=33210 这个帖子中建议使用 ROW 格式的二进制日志。

# 不好的事务编码习惯
---

## 在循环中提交

我们需要先准备好一些测试数据表：

```
CREATE TABLE t1(
  id int NOT NULL AUTO_INCREMENT
  ,name char(80) NULL
  ,CONSTRAINT PK_t1_id PRIMARY KEY (id)
)engine=InnoDB default charset utf8mb4;
```

然后分别创建以下下面3个存储过程，这3个存储过程采用不同的执行方式：

```
# load1 存储过程
CREATE PROCEDURE load1(count INT UNSIGNED)
BEGIN
DECLARE s INT UNSIGNED DEFAULT 1;
DECLARE c CHAR(80) DEFAULT REPEAT('a', 80);

WHILE s <= count DO
  INSERT INTO t1 SELECT NULL, c;
  COMMIT;
  SET s = s + 1;
END WHILE;
END;

# load2 存储过程，仅仅是在 load1 的基础去掉了显式 commit
CREATE PROCEDURE load2(count INT UNSIGNED)
BEGIN
DECLARE s INT UNSIGNED DEFAULT 1;
DECLARE c CHAR(80) DEFAULT REPEAT('a', 80);

WHILE s <= count DO
  INSERT INTO t1 SELECT NULL, c;
  SET s = s + 1;
END WHILE;
END;

# 显式开启事务，并在循环完成了之后再进行 commit
CREATE PROCEDURE load3(count INT UNSIGNED)
BEGIN
DECLARE s INT UNSIGNED DEFAULT 1;
DECLARE c CHAR(80) DEFAULT REPEAT('a', 80);

START TRANSACTION;

WHILE s <= count DO
  INSERT INTO t1 SELECT NULL, c;
  SET s = s + 1;
END WHILE;
COMMIT;
END;
```

下面来分别执行一下3个存储过程的执行时间：

```
call load1(10000);
Query OK, 0 rows affected (1 min 3.14 sec)

truncate table t1;

call load2(10000);
Query OK, 1 row affected (1 min 3.05 sec)

truncate table t1;

call load3(10000);
Query OK, 0 rows affected (0.56 sec)
```

可以看到 load1 和 load2 的执行时间花费了 1 分 3 秒左右，而 load3 的执行时间仅仅只有 0.56 秒。原因是由于 load1 和 load2 实际写了10000次重做日志文件，而对于 load3 来说，实际只写了 1 次。这里尤其需要注意的是 load2 存储过程，由于 autocommit = 1 ， 实际上循环当中每一次执行 SQL 之后，都会隐式的执行 commit transaction 的操作。

如下的示例中，我们将执行 load2 存储过程的方式调整为了， 再执行存储过程之前显式的开启了一个事务并提交，可以看到，性能也得到了非常大的提升。执行时间和 load3 不相上下，差不多也是 0.56 秒。

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> call load2(10000);
Query OK, 1 row affected (0.56 sec)

mysql> commit;
Query OK, 0 rows affected (0.02 sec)
```

总结：无论是在程序当中，还是存储过程当中，我们都不应该在一个较大的循环中反复进行提交操作，不论是显式的提交还是隐式的提交。

## 使用自动提交

自动提交并不是一个好的习惯，可能使一些开发人员产生错误的理解，如我们在前一小节中提到的循环提交问题。MySQL数据库默认设置使用自动提交（ autocommit = 1 ），可以使用如下语句来改变当前自动提交的方式：

```
set autocommit = 0;
```

除此之外，也可以使用 START TRANSACTION ，BEGIN 来显式地开启一个事务。在显试开启事务后， Mysql 会自动地执行 SET AUTOCOMMIT = 0 的命令，并在 COMMIT 或 ROLLBACK 结束一个事务后执行 SET AUTOCOMMIT = 1 。

对于不同语言或框架的 API ，自动提交的默认设置是不相同的。例如 JAVA JDBC 会设置 AUTOCOMMIT = 1，而 Python API 则会设置 AUTOCOMMIT = 0 。

## 使用自动回滚

很多开发人员会在存储过程进行错误处理时，编写如下的代码：

```
DECLARE EXIT HANDLER FOR SQLEXCEPTION BEGIN ROLLBACK;
```

这样做会存在一个问题，虽然运行存储过程的时候，当发生了任何异常时，事务能够正常回滚，但是程序调用该存储过程时，会无法知道到底存储过程是由于什么原因发生了异常，使排查错误变得困难。

解决这个问题通常有两种方式：

* 可以在去掉存储过程中的事务处理代码，改为在程序当中进行事务处理。通过绝大多数语言支持的 try catch 异常处理机制来完成。
* 利用 Mysql 5.5 版本之后添加的 RESIGNAL 语法来重新抛出异常，例如：

```
DROP TABLE IF EXISTS xx;
delimiter //
CREATE PROCEDURE p ()
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    SET @error_count = @error_count + 1;
    IF @a = 0 THEN RESIGNAL; END IF;
  END;
  DROP TABLE xx;
END//
delimiter ;
SET @error_count = 0;
SET @a = 0;
CALL p();

# output
DA 1. ERROR 1051 (42S02): Unknown table 'xx'
```

更多内容可以参考： https://dev.mysql.com/doc/refman/5.5/en/resignal.html
