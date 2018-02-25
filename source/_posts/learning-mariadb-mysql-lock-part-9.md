title: learning mariadb & mysql lock(part-9)
date: 2018-02-04 21:32:26
categories: programming
tags:
- mysql
- mariadb
- innodb
- lock
---

# 什么是锁
---

锁是数据库系统区别于文件系统的一个关键特性。锁机制用于管理对共享资源的并发访问。 InnoDB 存储引擎会在行级别上对表数据进行上锁，也会在数据库内部其他多个地方使用锁，从而允许对多种不同资源提供并发访问。因此数据库使用锁是为了支持对共享资源进行并发访问，并提供数据库的完整性和一致性。

<!--more-->

InnoDB 存储引擎中锁的实现和 Oracle 数据库非常类似，提供一致性的非锁定读，行级锁支持。行级锁没有相关额外的开销，并可以同时得到并发性和一致性。

# lock 与 latch

latch 一般称为闩锁（轻量级的锁），因为其要求锁定的时间必须非常短。若持续的时间长，则应用的性能会非常差。在 InnoDB 存储引擎中，latch又可以分为 mutex （互斥量）和 rwlock （读写锁）。其目的是用来保证并发线程操作临界资源的正确性，并且通常没有死锁检测的机制。

lock 的对象是事务，用来锁定的是数据库中的对象，如表、页、行。并且一般 lock 的对象仅在事务 commit 或 rollback 后进行释放（不同事务隔离级别释放的时间可能不同）。此外， lock ，正如在大多数数据库中一样，是有死锁机制的。

下表显示了lock与latch的不同：

|      | lock     | latch |
| :------------- | :------------- | :------------- |
|   对象     |  事务      | 线程 |
|   保护     |  数据库对象（表，页，行）      | 内存数据结构 |
|   持续时间     |  整个事务过程      | 临界资源 |
|   模式     |  行锁、表锁、意向锁      | 读写锁、互斥量 |
|   死锁     |  通过 waits-for graph 、 time out 等机制进行检测      | 无死锁检测机制、仅通过应用程序加锁的顺序保证无死锁的情况发生 |
|   存在于     |  lock manager 的哈希表中      | 每个数据结构的对象中 |

对于 InnoDB 存储引擎中的 latch ，可以通过命令 `SHOW ENGINE INNODB MUTEX` 来进行查看：

```
SHOW ENGINE INNODB MUTEX;
+--------+-----------------------------+----------+
| Type   | Name                        | Status   |
+--------+-----------------------------+----------+
| InnoDB | rwlock: log0log.cc:753      | waits=6  |
| InnoDB | sum rwlock: buf0buf.cc:1468 | waits=40 |
+--------+-----------------------------+----------+
```

# InnoDB 存储引擎中的锁

## 锁的类型

InnoDB 存储引擎实现了如下两种标准的行级锁：
* 共享锁 （ S Lock ），允许事务读一行数据。
* 排他锁 （ X Lock ），允许事务删除或更新一行数据。

如果一个事务 T1 已经获得了行 r 的共享锁，那么另外的事务 T2 可以立即获得行 r 的共享锁，因为读取并没有改变行 r 的数据，称这种情况为锁兼容（ Lock Compatible ）。但若有其他的事务 T3 想获得行 r 的排他锁，则其必须等待事务 T1 、 T2 释放行 r 上的共享锁——这种情况称为锁不兼容。

|      | X     |  S     |
| :------------- | :------------- | :------------- |
| X       | 不兼容       | 不兼容 |
| S       | 不兼容       | 兼容 |

从上表中可以发现 X 锁与任何锁都不兼容，而 S 锁仅仅和 S 锁兼容。 S 和 X 锁都是行锁，兼容是指对同一记录（ row ）锁的兼容性情况。

此外， InnoDB 存储引擎支持多粒度（ granular ）锁定，这种锁定允许事务在航迹上的锁和表级上的锁同时存在。为了支持在不同粒度上进行加锁操作， InnoDB 存储引擎支持一种额外的锁方式，称之为意向锁（ Intention Lock ），是将锁定的对象分为多个层次，能够对事务在更细粒度上进行加锁。

![lock-object](https://www.zhuxiaodong.net/static/images/lock-object.png)

若将上锁的对象看成一棵树，那么对最下层的对象上锁，也就是对最细粒度的对象进行上锁，那么首先需要对粗粒度的对象上锁。如果需要对页上的记录 r 进行上 X 锁，那么分别需要对数据库 A 、表、页上意向锁 IX ，最后对记录 r 上 X 锁。若其中任何一个部分导致等待，那么该操作需要等待粗粒度锁的完成。举例来说，在对记录 r 加 X 锁之前，已经有事务对表1进行了 S 表锁，那么表1上已存在 S 锁，之后事务需要对记录 r 在表1上加上 IX ，由于不兼容，所以该事务需要等待表锁操作的完成。

InnoDB 存储引擎支持意向锁设计比较简单，其意向锁即为表级别的锁。设计目的主要为了在一个事务中揭示下一行将被请求的锁类型，支持两种意向锁：

* 意向共享锁（ IS Lock ），事务想要获得一张表中某几行的共享锁。
* 意向排他锁（ IX Lock ），事务想要获得一张表中某几行的排他锁。

由于 InnoDB 存储引擎支持的是行级别的锁，因此意向锁其实不会阻塞除全表扫描以外的任何请求。表级意向锁与行级锁的兼容性如下表：

|      | IS | IX | S | X |
| :------------- | :------------- |
| IS      | 兼容 | 兼容 | 兼容 | 不兼容 |
| IX      | 兼容 | 兼容 | 不兼容 | 不兼容 |
| S      | 兼容 | 不兼容 | 兼容 | 不兼容 |
| X      | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

我们可以通过 `SHOW ENGINE INNODB STATUS` 命令来查看当前锁请求。从 InnoDB 1.0 版本开始，可以在 INFORMATION_SCHEMA 数据库下的表 INNODB_TRX 、 INNODB_LOCKS 、 INNODB_LOCK_WAITS 下查看当前事务并分析锁问题。

## 锁的实例分析

### 查询 InnoDB 锁的整体情况

```
show status like 'innodb_row_lock%';
+-------------------------------+---------+
| Variable_name                 | Value   |
+-------------------------------+---------+
| Innodb_row_lock_current_waits | 0       |
| Innodb_row_lock_time          | 2760208 |
| Innodb_row_lock_time_avg      | 175     |
| Innodb_row_lock_time_max      | 1341    |
| Innodb_row_lock_waits         | 15693   |
+-------------------------------+---------+
```

如果 Innodb_row_lock_waits 和 Innodb_row_lock_time_avg 比较大，则表明锁竞争比较严重。

### 查看当前运行的事务

```
show full processlist;
+---------+--------+---------------------+------+---------+------+----------+-----------------------+
| Id      | User   | Host                | db   | Command | Time | State    | Info                  |
+---------+--------+---------------------+------+---------+------+----------+-----------------------+
| 4097023 | hmsdbo | 10.10.115.171:42466 | hms  | Sleep   |  203 |          | NULL                  |
| 4097024 | hmsdbo | 10.10.115.171:42467 | hms  | Sleep   |   82 |          | NULL                  |
| 4097025 | hmsdbo | 10.10.115.171:42468 | hms  | Sleep   |   62 |          | NULL                  |
| 4097026 | hmsdbo | 10.10.115.171:42469 | hms  | Sleep   |  203 |          | NULL                  |
| 4097031 | hmsdbo | 10.10.115.171:42494 | hms  | Sleep   |  159 |          | NULL                  |
| 4097050 | hmsdbo | 10.10.115.171:42657 | hms  | Sleep   |  109 |          | NULL                  |
| 4097099 | hmsdbo | localhost           | NULL | Query   |    0 | starting | show full processlist |
+---------+--------+---------------------+------+---------+------+----------+-----------------------+
```

### 模拟 InnoDB 死锁

先创建一张表

```
CREATE TABLE test_user(id INT PRIMARY KEY, name VARCHAR(20)) DEFAULT CHARSET utf8 ENGINE = INNODB;

INSERT INTO test_user(id, name) VALUES(1, 'A');
INSERT INTO test_user(id, name) VALUES(2, 'B');
INSERT INTO test_user(id, name) VALUES(3, 'C');
```

首先新开一个 session ，启动**事务1**：

```
start transaction;
update test_user set name='AA' where id = 1;
```

查看当前事务的情况：

```
SELECT * FROM information_schema.innodb_trx \G
*************************** 1. row ***************************
                    trx_id: 47893097
                 trx_state: RUNNING
               trx_started: 2018-02-06 10:37:03
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 4098475
                 trx_query: SELECT * FROM information_schema.innodb_trx
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0


## 当前并没有产生 lock。
SELECT * FROM information_schema.innodb_locks\G
Empty set, 1 warning (0.00 sec)

SELECT * FROM information_schema.innodb_lock_waits\G
Empty set, 1 warning (0.00 sec)
```

先设置锁超时时间设置成较大的值：

```
set global innodb_lock_wait_timeout=600;

SHOW VARIABLES LIKE '%innodb_lock_wait%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_lock_wait_timeout | 600   |
+--------------------------+-------+
```

此时，启动另外一个 session ，更新 test_user 表的记录时，由于之前的事务没有提交，会导致 id = 1 这条记录被锁住，因此会在 50 秒之后，显示锁超时的异常。我们使用 innodb_trx 观察当前事务的状态时，能够发现 trx_state 为 LOCK WAIT 。

```
UPDATE test_user SET name='AAA' WHERE id = 1;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

SELECT * FROM information_schema.innodb_trx \G
*************************** 1. row ***************************
                    trx_id: 47895922
                 trx_state: LOCK WAIT
               trx_started: 2018-02-06 11:01:38
     trx_requested_lock_id: 47895922:724:3:9
          trx_wait_started: 2018-02-06 11:01:38
                trx_weight: 2
       trx_mysql_thread_id: 4098907
                 trx_query: update test_user set name='AA' where id = 1
       trx_operation_state: starting index read
         trx_tables_in_use: 1
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 47895632
                 trx_state: RUNNING
               trx_started: 2018-02-06 11:01:33
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 4098836
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
```

如果更新其它的记录，会看到可以立即执行成功并返回结果。这证明 InnoDB 存储引擎开启事务时，增加的**行锁** 。

```
UPDATE test_user SET name='AAA' WHERE id = 2;
Query OK, 1 row affected (0.01 sec)
```

```
show full processlist;
+---------+--------+---------------------+------+---------+------+----------+---------------------------------------------+
| Id      | User   | Host                | db   | Command | Time | State    | Info                                        |
+---------+--------+---------------------+------+---------+------+----------+---------------------------------------------+
| 4097023 | hmsdbo | 10.10.115.171:42466 | hms  | Sleep   |   56 |          | NULL                                        |
| 4097025 | hmsdbo | 10.10.115.171:42468 | hms  | Sleep   |  197 |          | NULL                                        |
| 4097026 | hmsdbo | 10.10.115.171:42469 | hms  | Sleep   |  196 |          | NULL                                        |
| 4097031 | hmsdbo | 10.10.115.171:42494 | hms  | Sleep   |   31 |          | NULL                                        |
| 4097050 | hmsdbo | 10.10.115.171:42657 | hms  | Sleep   |   29 |          | NULL                                        |
| 4097268 | hmsdbo | 10.10.115.171:44984 | hms  | Sleep   |  197 |          | NULL                                        |
| 4097269 | hmsdbo | 10.10.115.171:44985 | hms  | Sleep   |   99 |          | NULL                                        |
| 4098836 | hmsdbo | localhost           | hms2 | Sleep   |  104 |          | NULL                                        |
| 4098907 | hmsdbo | localhost           | hms2 | Query   |   99 | updating | update test_user set name='AA' where id = 1 |
| 4098973 | hmsdbo | localhost           | hms2 | Query   |    0 | starting | show full processlist                       |
+---------+--------+---------------------+------+---------+------+----------+---------------------------------------------+

select * from information_schema.innodb_locks;
+------------------+-------------+-----------+-----------+--------------------+------------+------------+-----------+----------+-----------+
| lock_id          | lock_trx_id | lock_mode | lock_type | lock_table         | lock_index | lock_space | lock_page | lock_rec | lock_data |
+------------------+-------------+-----------+-----------+--------------------+------------+------------+-----------+----------+-----------+
| 47895922:724:3:9 | 47895922    | X         | RECORD    | `hms2`.`test_user` | PRIMARY    |        724 |         3 |        9 | 1         |
| 47895632:724:3:9 | 47895632    | X         | RECORD    | `hms2`.`test_user` | PRIMARY    |        724 |         3 |        9 | 1         |
+------------------+-------------+-----------+-----------+--------------------+------------+------------+-----------+----------+-----------+

select * from information_schema.innodb_lock_waits;
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 47895922          | 47895922:724:3:9  | 47895632        | 47895632:724:3:9 |
+-------------------+-------------------+-----------------+------------------+
```

我们可以通过 `show full processlist` 的 Id 字段为： 4098907 到 innodb_trx 的 trx_mysql_thread_id 关联查询到开启的对应事务的ID: trx_id : 47895922。

然后再通过关联 innodb_locks 表的 lock_trx_id 字段关联查询到对应的锁。我们还可以通过 innodb_lock_waits 表的 requesting_trx_id 字段关联查询到锁等待的情况，比较关键是 blocking_trx_id 字段，可以查询到当前锁是由于哪一个事务造成的。


### 一致性非锁定读

一致性非锁定读（ consistent nonlocking read ）是指 InnoDB 存储引擎通过多版本控制（ MMVC ）的方式来读取当前执行时间中，数据库行的数据。如果读取的行正在执行 DELETE 或 UPDATE 操作，这时读取操作不会因此去等待行上锁的释放，因此不会造成读阻塞。此时， InnoDB 存储引擎会读取行的一个快照数据。

采用一致性非锁定读时，不需要等待访问的行上 X 锁的释放。快照数据是指行的之前版本的数据，该实现是通过 undo 日志来完成的。由于 undo 日志用来在事务中回滚数据，因此快照数据本身是没有额外的开销的。需要注意的是，并不是所有的事务隔离级别下，读取时都会采用非锁定一致性读，对于快照数据的定义也各不相同。

快照数据其实就是当前行数据之前的历史版本，每行记录可能有多个版本。一个行记录可能有多个快照数据，一般称为行多版本技术。由此带来的并发控制，称为多版本并发控制 （ Multi Version Concurrency Control ， MVCC ）。

在事务隔离级别为 READ COMMITTED 和 REPEATABLE READ 下， InnoDB 存储引擎会使用非锁定的一致性读。

但是这两个事务隔离级别对于快照数据的定义并不相同。在 READ COMMITTED 下，非一致性锁定读总是读取的是被锁定行的最新一份快照数据。而在 REPEATABLE READ 下，非一致性锁定读总是读取事务最开始时的行数据版本。

要理解上述两个事务隔离级别对于一致性非锁定读下的区别，我们来看一个实际的实例：

首先创建一个测试数据表

```
CREATE TABLE test_user(id INT PRIMARY KEY, name VARCHAR(20), old INT) DEFAULT CHARSET utf8 ENGINE = INNODB;

INSERT INTO test_user(id, name, old) VALUES(1, 'A', 1);
INSERT INTO test_user(id, name, old) VALUES(2, 'B', 2);
INSERT INTO test_user(id, name, old) VALUES(3, 'C', 3);
```

打开一个 Session A ，执行下面的 SQL， 并故意让事务不 commit ：

```
# Session A
start transaction;

select * from test_user where id = 1;
```

再打开另外一个 Session B ，执行下面的 SQL ：

```
# Session B
start transaction;

update test_user set id = 4 where id = 1;
```

在会话 B 中将表 test_user 中 id = 1 的记录修改为 id = 3 ， 但是同样也让事务不 commit，这相当于给 id = 1 的那行记录添加了一个 X 锁。

此时回到 Session A ， 接着执行如下的 SQL ， 会发现无论是哪一种事务隔离级别，显示的数据都是一样的。即 id 还是等于1。

```
# Session A
select * from test_user where id = 1;
+----+------+------+
| id | name | old  |
+----+------+------+
|  1 | A    |    1 |
+----+------+------+
```

然后回到 Session B，将事务提交。

```
# Session B
commit;
```

此时，在 Session A 中，如果事务的隔离级别是 REPEATABLE READ ，会发现此时是能够读取到 id = 1 的记录的。这是由于该隔离级别下，总是读取事务开始时的行记录。

```
# Session A
SELECT @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+

select * from test_user where id = 1;
+----+------+------+
| id | name | old  |
+----+------+------+
|  1 | A    |    1 |
+----+------+------+
```

但是事务的隔离级别是 READ COMMITTED ，此时就无法读取到 id = 1 的记录。因为在该隔离级别下，它总是读取行的最新版本，如果行被锁定了，则读取改行版本的最新一个快照。可以看到 READ COMMITTED 事务隔离级别中，违反了 ACID 中 I ，即隔离性的原则。

```
set global transaction isolation level read committed;

# Session A
select @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| READ-COMMITTED |
+----------------+

select * from test_user where id = 1;
Empty set (0.00 sec)
```

### 一致性锁定读

在某一些情况下，用户需要显式地对数据库读取操作进行加锁以保证数据逻辑的一致性。而这要求数据库支持加锁语句，即使对于 SELECT 的读操作也是一样的。

InnoDB 存储引擎对于 SELECT 语句支持两种一致性的锁定读（ locking read ）：
* SELECT ... FOR UPDATE ：对读取行记录加一个 X 锁，其他事务不能对已锁定的行加上任何锁。
* SELECT ... LOCK IN SHARE MODE ： 对读取行记录加一个 S 锁，其他事务可以向被锁定的行加 S 锁，但是如果增加的是 X 锁，则会被阻塞。

对于一致性非锁定读，即使读取的行已被执行了 SELECT...FOR UPDATE ，也是可以进行读取的，这和之前讨论的情况一样。此外， SELECT...FOR UPDATE ， SELECT...LOCK INSHARE MODE 必须在一个事务中，当事务提交了，锁也就释放了。因此在使用上述两句SELECT锁定语句时，务必加上 BEGIN， START TRANSACTION 或者 SET AUTOCOMMIT=0 。

更多内容参考[这里](http://blog.csdn.net/cug_jiang126com/article/details/50544728)。

### 自增长与锁

auto increment 在数据库中是非常常见的一种属性，很多表的主键都会通过设置自增长列来实现。在 InnoDB 存储引擎的内存结构中，对每个含有自增长值的表都有一个自增长计数器。当对含有自增长计数器的表进行插入操作时，这个计数器会被初始化，执行如下的语句来得到计数器的值：

```
SELECT MAX(auto_inc_col) FROM test_table2 FOR UPDATE;
```

插入操作会依据这个自增长的计数器值加 1 赋予自增长列。这个实现方式称为 AUTO_INC locking 。 这种锁采用的是一种特殊的表锁机制，为了提高插入的性能，锁不是在一个事务完成后才释放的，而是在完成对自增长值插入的 SQL 语句后立即释放的。

从 MySQL 5.1.22 版本开始， InnoDB 存储引擎中提供了一种轻量级互斥量的自增长实现机制，这种机制大大提高了自增长值插入的性能。并且从该版本开始， InnoDB 存储引擎提供了一个参数 innodb_autoinc_lock_mode 来控制自增长的模式，该参数的默认值为 1 。

我们先来看一下插入的类型：

| 插入类型 | 说明     |
| :------------- | :------------- |
| insert-like       | insert-like 指所有类型的插入语句，包括： INSERT 、 REPLACE 、 INSERT...SELECT 、 REPLACE...SELECT 、 LOAD DATA 等       |
| simple inserts       | simple inserts 指能在插入前就确定插入行数的语句。这些语句包括 INSERT 、 REPLACE 等，但不包括 INSERT ... ON DUPLICATE KEYS UPDATE 这类 SQL 语句       |
| bulk inserts       | bulk inserts 指在插入前不能确定具体的插入行数，如 INSERT...SELECT ， REPLACE...SELECT ， LOAD DATA        |
| mixed-mode inserts       | mixed-mode inserts 指在插入中有一部分的值是自增长的，有一部分是确定的，例如： INSERT INTO t (c1, c2) VALUES (1, 'a'),(NULL, 'b') 或者 INSERT ... ON DUPLICATE KEY UPDATE        |

innodb_autoinc_lock_mode 的类型包括：

| mode     | 说明     |
| :------------- | :------------- |
| 0       | MySQL 5.1.22 之前的实现方式，一般不会在实际的环境中使用        |
| 1       | 默认值。对于 simple inserts ，该值会有互斥量（ mutex ）去对内存中的计数器进行累加操作。对于 bulk inserts ，还是会使用传统表锁的 AUTO-INC Locking方式，如果不考虑回滚操作，对于自增值列的增长还是连续的。       |
| 2       | 对于所有的 "insert-like" 自增长值的产生都是通过互斥量，而不是 AUTO-INC Locking 的方式。这是几种模式中性能最高的方式。但是也带来了一定的问题，因为并发插入的存在，，在每次插入时，自增长的值可能不是连续的。另外对于 Statement-Base Replication 会出现问题，必须要使用 row-base Replication 的方式       |

关于自增长列不连续的问题，更多可以参考: [这里](http://www.cnblogs.com/zhoujinyi/p/3433823.html)


# 锁的算法
---

## 行锁的 3 种算法

* Record Lock ： 行锁，单个行记录上的锁。
* Gap Lock ： 间隙锁，锁定一个范围，但是不包含记录本身。
* Next-Key Lock ： 相当于前两者的组合，锁定一个范围并锁定记录本身。

Record Lock 总是会去锁住索引记录，如果 InnoDB 存储引擎表在建立的时候没有设置任何一个索引，那么这时 InnoDB 存储引擎会使用隐式的主键来进行锁定。

Next-Key Lock 是结合了 Gap Lock 和 Record Lock 的一种锁定算法，在 Next-Key Lock 算法下， InnoDB 对于行的查询都是采用这种锁定算法。例如一个索引有10，11，13和20这四个值，那么该索引可能被Next-Key Locking的区间为：
(- ∞, 10]
(10, 11]
(11, 13]
(13, 20]
(20, + ∞)

采用 Next-Key Lock 的锁定技术称为 Next-Key Locking 。其设计的目的是为了解决 Phantom Problem（幻象读）。而利用这种锁定技术，锁定的不是单个值，而是一个范围，是谓词锁（ predict lock ）的一种改进。除了 Next-Key Locking ，还有 Previous-Key Locking 技术。同样上述的索引10、11、13和20，若采用 Previous-Key Locking 技术，那么可锁定的区间为：
(- ∞, 10)
[10, 11)
[11, 13)
[13, 20)
[20, + ∞)

假设事务 T1 已经通过 Next-Key Locking 锁定了如下范围：(10, 11]、(11, 13] ，当插入新的记录12时，则锁定的范围会变成：(10, 11]、(11, 12]、(12, 13] 。然而，当查询的**索引含有唯一属性**时，InnoDB 存储引擎会对 Next-Key Lock 进行优化，将其降级为 Record Lock ，即仅锁住索引本身，而不是范围。

让我们来看下面的例子：

```
CREATE TABLE t (a int PRIMARY KEY);
INSERT INTO t SELECT 1;
INSERT INTO t SELECT 2;
INSERT INTO t SELECT 5;
```

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | |
| 2       |  select * from t where a=5 for update;      | |
| 3       |        | start transaction; |
| 4       |        | insert into t select 4; |
| 5       |        | commit; # 执行成功，不需要等待 |
| 6       |  commit;      | none |

在上面的示例中，表 t 共有 1 、 2 、 5 三个值。在会话 A 中首先对 a = 5 添加 X 锁。由于 a 是主键并且唯一，因此锁定的仅仅是 5 这个值，而不是（2，5）这个范围，这样会在会话 B 中插入值 4 但不会发生阻塞，可以立即插入并返回。这说明锁定算法由 Next-Key Lock 降级为了 Record Lock ，提升了应用的并发性。

如果是非唯一的辅助索引，会是另外的一种情况，让我们来看下面的例子：

```
CREATE TABLE z (a INT, b INT, PRIMARY KEY(a), KEY(b));
INSERT INTO z SELECT 1,1;
INSERT INTO z SELECT 3,1;
INSERT INTO z SELECT 5,3;
INSERT INTO z SELECT 7,6;
INSERT INTO z SELECT 10,8;
```

z 表中包括了主键 a 的聚集索引，以及 b 列的辅助索引。

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | |
| 2       |  select * from z where b=3 for update;     | |
| 3       |  select * from z where a=5 for update;     | |
| 4       |        | start transaction; |
| 5       |        | select * from z where a=5 LOCK IN SHARE MODE; #会被阻塞 |
| 6       |        | insert into z select 4,2; # 会被阻塞 |
| 7       |  | insert into z select 6,5; # 会被阻塞 |

对于序号 3 的 SQL ，对主键 a 列增加了 Record Lock；而对于序号 2 的 SQL ，对 b 列辅助索引增加了 Next-Key Lock ，锁定范围是（1，3），除此之外，对 b 列辅助索引下一个键值加上了 gap lock ，即还有一个辅助索引范围为（3，6）的锁。

* 序号5的被阻塞的原因是：在 session A 中执行的序号 3 的 SQL 语句已经对 a = 5 的记录增加了 X 锁。
* 序号6的被阻塞的原因是：写入的字段 b = 2，其范围在 b 列辅助索引 Next-Key Lock 的锁定范围（1，3）当中。
* 序号7的被阻塞的原因是：写入的字段 b = 5，虽然不在范围在 b 列辅助索引 Next-Key Lock 的锁定范围（1，3）当中，但是却在另外一个 gap lock （3，6）的范围中。

而如下的 SQL 语句，因为没有在锁定范围之内，因此都不会被阻塞：

```
insert into z select 8,6;
insert into z select 2,0;
insert into z select 6,7;
```

可以看到，gap lock 的作用是为了阻止多个事务将记录插入到同一范围中，因为这会导致 Phantom Read 。比如在上面的例子中，如果没有 Gap lock 锁定范围（3，6），那么用户就可以插入辅助索引 b 列为 3 的记录，这会导致 session A 中的用户再次查询时会返回不同的记录，导致幻读情况的发生。

我们可以通过如下的两种方式显式地关闭 gap lock ：

```
* 将事务的隔离级别由 REPEATABLE READ 降低为 READ COMMITTED
* 将参数 innodb_locks_unsafe_for_binlog 设置为 1
```

在上述的配置下，除了外键约束和唯一性检查依然需要的 Gap Lock ，其余情况仅使用 Record Lock 进行锁定。但需要牢记的是，上述设置破坏了事务的隔离性，并且对于 replication ，可能会导致主从数据的不一致。此外，从性能上来看， READ COMMITTED 也不会优于默认的事务隔离级别 READ REPEATABLE。

## 解决 Phantom Problem

在默认的事务隔离级别下，即 REPEATABLE READ 下， InnoDB 存储引擎采用 Next-Key Locking 机制来避免 Phantom Problem（幻像问题）。这点可能不同于与其他的数据库，如 Oracle 数据库，因为其可能需要在 SERIALIZABLE 的事务隔离级别下才能解决 Phantom Problem 。

Phantom Problem 是指在同一事务下，连续执行两次同样的 SQL 语句可能导致不同的结果，第二次的 SQL 语句可能会返回之前不存在的行。

我们还是使用之前创建的那张表 t 来作为示例，该表当中的记录为 1 、 2 、 5 。

```
# 首先需要将事务的隔离级别修改为 READ COMMITTED
set global transaction isolation level read committed;
```

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | |
| 2       |  select * from t where a>2 for update;  --输出 5    | |
| 3       |        | start transaction; |
| 4       |        | insert into t select 4; |
| 5       |        | commit;  |
| 6       |  select * from t where a>2 for update; --输出 4, 5     | none |

当 session A 执行 `select * from t where a>2 for update;` 时，注意这时事务 T1 并没有进行提交操作，上述应该返回5这个结果。若与此同时，另一个事务T2插入了 4 这个值，并且数据库允许该操作，那么事务 T1 再次执行上述SQL语句会得到结果 4 和 5 。这与第一次得到的结果不同，违反了事务的隔离性，即当前事务能够看到其他事务的结果。

InnoDB 存储引擎采用 Next-Key Locking 算法来避免 Phantom Problem 。对于上述的 SQL 语句 `select * from t where a>2 for update;` ，其锁住的不是 5 这单个值，而是对 （2，+无穷大）整个范围都加了 X 锁。因此对于这个范围的插入都是不被允许的，从而避免 Phantom Problem 。

InnoDB 存储引擎默认的事务隔离级别是 REPEATABLE READ，在该隔离级别下，其采用 Next-Key Locking 的方式来加锁。而在事务隔离级别 READ COMMITTED 下，其仅采用 Record Lock 。

我们可以通过 InnoDB 存储引擎的 Next-Key Locking 机制在应用程序上实现唯一性的检查，参考如下的示例：

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | |
| 2       |  select * from z where b = 4 lock in share model;    | |
| 3       |        | start transaction; |
| 4       |        | select * from z where b = 4 lock in share model; |
| 5       |  insert into z select 4, 4;  -- 阻塞     |   |
| 6       |       | insert into z select 4, 4; -- 产生死锁异常 |
| 7       |  -- 事务 B 产生死锁异常之后，事务 A 的 insert 执行成功     | none |

# 锁问题
---

## 脏读

在理解脏读（ Dirty Read ）之前，需要理解脏数据的概念。但是脏数据和之前所介绍的脏页完全是两种不同的概念。脏页指的是在缓冲池中已经被修改的页，但是还没有刷新到磁盘中，即数据库实例内存中的页和磁盘中的页的数据是不一致的，当然在刷新到磁盘之前，日志都已经被写入到了重做日志文件中。而所谓脏数据是指事务对缓冲池中行记录的修改，并且还没有被提交（ commit ）。

对于脏页的读取，是非常正常的。脏页是因为数据库实例内存和磁盘的异步造成的，这并不影响数据的一致性（或者说两者最终会达到一致性，即当脏页都刷回到磁盘）。并且因为脏页的刷新是异步的，不影响数据库的可用性，因此可以带来性能的提高。

脏数据却截然不同，脏数据是指未提交的数据，如果读到了脏数据，即一个事务可以读到另外一个事务中未提交的数据，则显然违反了数据库的隔离性。脏读指的就是在不同的事务下，当前事务可以读到另外事务未提交的数据，简单来说就是可以读到脏数据。

让我们来看一个实际的例子，首先需要将事务隔离级别修改为： READ UNCOMMITTED ：

```
set global transaction isolation level read uncommitted;
```

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | |
| 2       |  select * from t; --输出 1    | |
| 3       |        | start transaction; |
| 4       |  | insert into t select 2; |
| 5       |  select * from t; --输出 1，2     | none  |

可以看到在 session B 的事务并没有提交的情况下，在 session A 中读取到了 session B 中的插入的 2 那条记录，即产生了脏读，也违反了事务的隔离性。

只有在事务的隔离级别为 READ UNCOMMITTED 的时候，才有可能发生脏读。

## 不可重复读

在这个事务还没有结束时，另外一个事务也访问该同一数据集合，并做了一些 DML 操作。因此，在第一个事务中的两次读数据之间，由于第二个事务的修改，那么第一个事务两次督导的数据可能是不一样的。这样就发生了在一个事务内两次督导的数据是不一样的情况，这种情况被称为不可重复读。

不可重复读和脏读的区别是：脏读是读到未提交的数据，而不可重复读读到的却是已经提交的数据，但是其违反了数据库事务一致性的要求。

让我们来看下面的示例，首先需要将事务的隔离级别设置为 READ COMMITTED

```
set global transaction isolation level read committed;
```

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | start transaction; |
| 2       |  select * from t; --输出 1    | |
| 3       |  | insert into t select 2; |
| 4       |  select * from t; --输出 1 |  |
| 5       |  | commit; |
| 6       |  select * from t; --输出 1，2     | none  |

序号2执行时， session A 读取到的记录是 1 ，此时在 session B 中插入了值 2，在 session B 没有 commit 之前，也就是序号4执行时，session A 读取到的数据还是1，但是当 session B commit 之后，序号6 session A 读取到的数据就是 1 ，2 两条记录了。

一般来说，不可重复读的问题是可以接受的，因为其读到的是已经提交的数据，本身并不会带来很大的问题。因此，很多数据库厂商（如 Oracle、 Microsoft SQL Server ）将其数据库事务的默认隔离级别设置为READ COMMITTED，在这种隔离级别下允许不可重复读的现象。

在 InnoDB 存储引擎中，通过使用 Next-Key Lock 算法来避免不可重复读的问题。在MySQL官方文档中将不可重复读的问题定义为 Phantom Problem ，即幻像问题。在 Next-Key Lock 算法下，对于索引的扫描，不仅是锁住扫描到的索引，而且还锁住这些索引覆盖的范围（ gap ）。因此在这个范围内的插入都是不允许的。这样就避免了另外的事务在这个范围内插入数据导致的不可重复读的问题。因此， InnoDB 存储引擎的默认事务隔离级别是 READ REPEATABLE ，采用 Next-Key Lock 算法，避免了不可重复读的现象。

## 丢失更新

丢失更新是另外一个锁导致出现的问题，其本质是指一个事务的更新操作会被另外一个事务的更新操作所覆盖，从而导致数据的不一致。例如：

* 事务 T1 将行记录 r 更新为 v1 ，但是事务 T1 并未提交。
* 在几乎同一个时间点，事务 T2 将行记录 r 更新为 v2 ，事务 T2 也没有提交。
* 事务 T1 提交。
* 事务 T2 提交。

但是，在当前数据库的任何隔离级别下，都不会导致数据库理论意义上的丢失更新问题。这是因为，即使是 READ UNCOMMITTED 的事务隔离级别，对于行的 DML 操作，需要对行或其他粗粒度级别的对象加锁。因此在上述步骤2）中，事务 T2 并不能对行记录r进行更新操作，其会被阻塞，直到事务 T1 提交。

虽然数据库能阻止丢失更新问题的产生，但是在生产应用中还有另一个逻辑意义的丢失更新问题，而导致该问题的并不是因为数据库本身的问题。实际上，在所有多用户计算机系统环境下都有可能产生这个问题。简单地说来，出现下面的情况时，就会发生丢失更新：

* 事务 T1 查询一行数据，放入本地内存，并显示给一个终端用户 User1 。
* 事务 T2 也查询该行数据，并将取得的数据显示给终端用户 User2 。
* User1 修改这行记录，更新数据库并提交。
* User2 修改这行记录，更新数据库并提交。

上述这个过程中用户 User1 的修改更新操作“丢失”了。

要避免这个问题，我们可以使用“乐观锁”的方式来解决。

## 阻塞

因为不同锁之间的兼容性关系，在有些时刻一个事务中的锁需要等待另一个事务中的锁释放它所占用的资源，这就是阻塞。阻塞并不是一件坏事，其是为了确保事务可以并发且正常地运行。

在 InnoDB 存储引擎中，参数 innodb_lock_wait_timeout 用来控制等待的时间（默认是50秒）， innodb_rollback_on_timeout 用来设定是否在等待超时时对进行中的事务进行回滚操作（默认是OFF，代表不回滚）。

当发生超时， Mysql 数据库会抛出一个 1205 的错误，

```
ERROR 1205 (HY000): Lock wait timeout exceeded ; try restarting transaction
```

# 死锁
---

## 死锁的概念

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象。若无外力作用，事务都将无法推进下去。解决死锁问题最简单的方式是不要有等待，将任何的等待都转化为回滚，并且事务重新开始。毫无疑问，这的确可以避免死锁问题的产生。然而在线上环境中，这可能导致并发性能的下降，甚至任何一个事务都不能进行。而这所带来的问题远比死锁问题更为严重，因为这很难被发现并且浪费资源。

解决死锁问题最简单的一种方法是超时，即当两个事务互相等待时，当一个等待时间超过设置的某一阈值时，其中一个事务进行回滚，另一个等待的事务就能继续进行。在 InnoDB 存储引擎中，参数 innodb_lock_wait_timeout 用来设置超时的时间。

超时机制虽然简单，但是其仅通过超时后对事务进行回滚的方式来处理，或者说其是根据 FIFO 的顺序选择回滚对象。但若超时的事务所占权重比较大，如事务操作更新了很多行，占用了较多的 undo log ，这时采用 FIFO 的方式，就显得不合适了，因为回滚这个事务的时间相对另一个事务所占用的时间可能会很多。

InnoDB 存储引擎采用了一种更为主动的死锁检测方式：等待图（ wait-for graph ），它要求数据库保存以下两种信息：

* 锁的信息链表
* 事务等待链表

通过这两个链表可以构造出一张图，而在这个图中若存在任何一个回路，就表示存在死锁，资源间相互等待。在 wait-for graph 中，事务为图中的节点。而在图中，事务 T1 指向 T2 边的定义为：
* 事务 T1 等待事务 T2 所占用的资源。
* 事务 T1 最终等待 T2 所占用的资源，也就是事务之间在等待相同的资源，而事务 T1 发生在事务 T2 的后面。

事务和锁的状态图如下所示：

![t-lock-state-graph](https://www.zhuxiaodong.net/static/images/t-lock-state-graph.png)

在 Transaction Wait Lists 中可以看到共有4个事务 t1 、 t2 、 t3 、 t4 ，故在 wait-for graph 中应有4个节点。而事务 t2 对 row1 占用 x 锁，事务 t1 对 row2 占用 s 锁。事务 t1 需要等待事务 t2 中 row1 的资源，因此在 wait-for graph 中有条边从节点 t1 指向节点 t2 。事务 t2 需要等待事务 t1 、 t4 所占用的 row2 对象，故而存在节点 t2 到节点 t1、t4 的边。同样，存在节点 t3 到节点 t1、t2、t4 的边，因此最终的 wait-for graph 如下图所示：

![wait-for-graph](https://www.zhuxiaodong.net/static/images/wait-for-graph.png)

可以发现存在回路（t1，t2），因此存在死锁。

系统发生死锁的概率可以总结为：
* 系统中事务的数量越多，发生死锁的概率越大。
* 每个事务操作的数据行数量越多，发生死锁的概率越大。
* 操作的数据总集合数量越小，发生死多的概率越小。

## 死锁的示例

如果程序是串行的，那么不可能发生死锁。死锁只存在于并发的情况，而数据库本身就是一个并发运行的程序，因此可能会发生死锁。如下的操作演示了死锁的一种经典的情况，即 A 等待 B ， B 在等待 A，这种死锁问题被称为 AB-BA 死锁。

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | start transaction; |
| 2       |  select * from t where a=1 for update; --输出 1    | |
| 3       |  | select * from t where a=2 for update; --输出 2 |
| 4       |  select * from t where a=2 for update; --阻塞、等待 |  |
| 5       |  | select * from t where a=1 for update; --ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |
| 6       | -- 执行成功  | none |

可以看到当序号5 session B 发生了死锁异常之后， 序号6的 session A 的执行立即就返回执行成功了。这是由于 InnoDB 存储引擎检测到死锁之后，立即 rollback 了 session B 的异常。

再来看另一个死锁的常见示例，即当前事务持有了待插入记录的下一个记录的 X 锁，但是在等待队列中存在一个 S 锁的请求。

```
CREATE TABLE t (a int PRIMARY KEY);
INSERT INTO t SELECT 1;
INSERT INTO t SELECT 2;
INSERT INTO t SELECT 4;
INSERT INTO t SELECT 5;
```

| 序号 | session A     | session B |
| :------------- | :------------- | :------------- |
| 1       |  start transaction;      | start transaction; |
| 2       |  select * from t where a=4 for update; --输出 1    | |
| 3       |  | select * from t where a<=4 lock in share mode; --等待 |
| 4       |  insert into t value(3); --ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |  |
| 5       |  | --事务获得锁，正常运行  |

可以看到，session A 中已经对记录4持有了 X 锁，但是 session A 中插入记录3时会导致死锁发生。这个问题的产生是由于会话B中请求记录4的 S 锁而发生等待，但之前请求的锁对于主键值记录1、2都已经成功，若在序号5执行时能插入记录，那么 session B 在获得记录4持有的 S 锁后，还需要向后获得记录3的记录，这样就显得有点不合理。因此 InnoDB 存储引擎在这里主动选择了死锁，而回滚的是 undo log 记录大的事务，这与 AB-BA 死锁的处理方式又有所不同。
