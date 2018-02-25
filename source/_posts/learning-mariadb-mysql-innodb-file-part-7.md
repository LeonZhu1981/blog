title: learning mariadb & mysql innodb-file(part-7)
date: 2018-01-30 09:48:35
categories: programming
tags:
- mysql
- mariadb
- innodb
---

# 参数文件
---

mysql 在启动时，会先读取一个配置参数文件，用来寻找数据库的各种文件所在位置以及指定某些初始化参数，配置文件的所在位置可以通过如下的命令查询：

```
mysql --help | grep my.cnf
```

<!--more-->

## 什么是参数

参数其实就是一个 key/value ， 例如: innodb_buffer_pool_size 。可以通过如下的两种方式来参看当前系统当中设置的参数：

```
select * from information_schema.global_variables where variable_name like 'innodb_buffer%';
+-------------------------------------+----------------+
| VARIABLE_NAME                       | VARIABLE_VALUE |
+-------------------------------------+----------------+
| INNODB_BUFFER_POOL_FILENAME         | ib_buffer_pool |
| INNODB_BUFFER_POOL_DUMP_NOW         | OFF            |
| INNODB_BUFFER_POOL_CHUNK_SIZE       | 67108864       |
| INNODB_BUFFER_POOL_POPULATE         | OFF            |
| INNODB_BUFFER_POOL_DUMP_PCT         | 25             |
| INNODB_BUFFER_POOL_DUMP_AT_SHUTDOWN | ON             |
| INNODB_BUFFER_POOL_INSTANCES        | 1              |
| INNODB_BUFFER_POOL_LOAD_ABORT       | OFF            |
| INNODB_BUFFER_POOL_SIZE             | 67108864       |
| INNODB_BUFFER_POOL_LOAD_NOW         | OFF            |
| INNODB_BUFFER_POOL_LOAD_AT_STARTUP  | ON             |
+-------------------------------------+----------------+
```

```
show variables like 'innodb_buffer%';
+-------------------------------------+----------------+
| Variable_name                       | Value          |
+-------------------------------------+----------------+
| innodb_buffer_pool_chunk_size       | 67108864       |
| innodb_buffer_pool_dump_at_shutdown | ON             |
| innodb_buffer_pool_dump_now         | OFF            |
| innodb_buffer_pool_dump_pct         | 25             |
| innodb_buffer_pool_filename         | ib_buffer_pool |
| innodb_buffer_pool_instances        | 1              |
| innodb_buffer_pool_load_abort       | OFF            |
| innodb_buffer_pool_load_at_startup  | ON             |
| innodb_buffer_pool_load_now         | OFF            |
| innodb_buffer_pool_populate         | OFF            |
| innodb_buffer_pool_size             | 67108864       |
+-------------------------------------+----------------+
```

## 参数类型

* 动态参数 ：可以在实例运行期中进行更改。
* 静态参数 ：在整个实例的生命周期内都不得修改，参数是只读的。

可以通过 SET 命令对动态的参数值进行修改

```
SET
| [global | session] system_var_name=expr
| @@global | @@session @@system_var_name=expr
```

global 表示是参数修改基于当前实例的整个生命周期；session 表示是参数修改是基于当前会话的。某些参数的只能在当前 session 当中修改，例如 autocommit ；某些参数只能在当前实例的整个生命周期进行该修改，例如 binlog_cache_size ；而有一些参数是既可以针对当前会话，又可以针对当前实例的整个生命周期修改生效，如 read_buffer_size。

```
set read_buffer_size=524288;
select @@session.read_buffer_size;
+----------------------------+
| @@session.read_buffer_size |
+----------------------------+
|                     524288 |
+----------------------------+

select @@global.read_buffer_size;
+---------------------------+
| @@global.read_buffer_size |
+---------------------------+
|                    131072 |
+---------------------------+

# 如果需要设置整个实例生命周期的参数，可以使用 @@global.var_name 来修改
set @@global.read_buffer_size=524288;

select @@global.read_buffer_size;
+---------------------------+
| @@global.read_buffer_size |
+---------------------------+
|                    524288 |
+---------------------------+
```

需要注意的是，使用 `set @@global.var_name` 虽然调整了整个实例生命周期的参数值，但是 Mysql 重启了之后，其值会失效，需要修改 my.cnf 配置文件当中的参数值，才能够确保重启了之后也能生效。

## 日志文件

### 错误日志

错误日志文件对 Mysql 的启动、运行、关闭过程进行了记录。当我们在遇到问题的时候，应该首先排查错误日志文件当中的内容。可以通过如下命令来定位该文件的位置:

```
show variables like 'log_error';
+---------------+---------------------------------+
| Variable_name | Value                           |
+---------------+---------------------------------+
| log_error     | /data/mysql/3306/data/error.log |
+---------------+---------------------------------+
```

### 慢查询日志

慢查询日志可以帮助我们定位可能存在问题的 SQL 语句，在 Mysql 启动时设置一个阈值，将运行时间超过该值的所有 SQL 语句记录到慢查询日志文件中。

```
show variables like 'long_query_time';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 0.100000 |
+-----------------+----------+

show variables like '%slow_query%';
+---------------------+--------------------------------+
| Variable_name       | Value                          |
+---------------------+--------------------------------+
| slow_query_log      | ON                             |
| slow_query_log_file | /data/mysql/3306/data/slow.log |
+---------------------+--------------------------------+
```

long_query_time 用于设置这个阈值，单位为秒。 slow_query_log 指定是否开启慢查询。 slow_query_log_file 指定了慢查询日志的记录位置。

还有一个和慢查询相关的参数是 log_queries_not_using_indexes ，用于记录没有使用索引的 SQL 语句至慢查询日志。

```
show variables like 'log_queries_not_using_indexes';
+----------------------------------------+-------+
| Variable_name                          | Value |
+----------------------------------------+-------+
| log_queries_not_using_indexes          | OFF   |
+----------------------------------------+-------+
```

默认情况下，文件是慢查询日志的输出方式，不过我们可以将其调整为 TABLE ，这样就可以通过 mysql.slow_log 查询到慢查询日志了。

```
show variables like 'log_output';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+

set global log_output='TABLE';

CREATE TABLE `slow_log` (
 `start_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
 `user_host` mediumtext NOT NULL,
 `query_time` time(6) NOT NULL,
 `lock_time` time(6) NOT NULL,
 `rows_sent` int(11) NOT NULL,
 `rows_examined` int(11) NOT NULL,
 `db` varchar(512) NOT NULL,
 `last_insert_id` int(11) NOT NULL,
 `insert_id` int(11) NOT NULL,
 `server_id` int(10) unsigned NOT NULL,
 `sql_text` mediumblob NOT NULL,
 `thread_id` bigint(21) unsigned NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='Slow log'
```

可以看到 slow_log 的存储引擎为 CSV 。我们可以修改 slow_log 表的存储引擎为 MyISAM ，以提升查询效率。

```
# 需要先将 slow_query_log 关闭了之后，才能够修改 slow_log 表的存储引擎
set global slow_query_log=off;

alter table mysql.slow_log engine=MyISAM;

set global slow_query_log=on;
```

除了官方提供的 mysqldumpslow 工具之外，建议使用 percona toolkit 工具中的 `pt-query-digest` 进行 slow log 的分析。参考[这里](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)和[这里](http://blog.csdn.net/seteor/article/details/24017913)。

### 查询日志

查询日志记录了所有对 Mysql 数据库请求的信息，无论这些请求是否得到了正确的执行。

```
show variables like 'general_log%';
+------------------+-------------------------------------+
| Variable_name    | Value                               |
+------------------+-------------------------------------+
| general_log      | OFF                                 |
| general_log_file | /data/mysql/3306/data/localhost.log |
+------------------+-------------------------------------+
```

### 二进制日志 （ Binary log ）

二进制日志记录了对 Mysql 数据库执行更改的所有操作，但是不包括 SELECT 和 SHOW 这类操作，因为这些操作并不包括对数据本身的修改。

```
show master status;
+-----------------+----------+--------------+------------------+
| File            | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+-----------------+----------+--------------+------------------+
| mybinlog.000017 |      995 |              |                  |
+-----------------+----------+--------------+------------------+

# 注意，由于 binlog 日志文件可能会非常大，因此输出具体信息可能会花费很长的时间。

show binlog events in 'mybinlog.001957';

*************************** 7. row ***************************
   Log_name: mybinlog.000017
        Pos: 548
 Event_type: Query
  Server_id: 1
End_log_pos: 750
       Info: use `employees`; CREATE TABLE tb_aria_test (
    fd1 INT NOT NULL,
    fd2 VARCHAR(10) NOT NULL,
    PRIMARY KEY(fd1)
) ENGINE=Aria TRANSACTIONAL=1
*************************** 9. row ***************************
   Log_name: mybinlog.000017
        Pos: 792
 Event_type: Query
  Server_id: 1
End_log_pos: 995
       Info: use `employees`; CREATE TABLE tb_aria_test2 (
    fd1 INT NOT NULL,
    fd2 VARCHAR(10) NOT NULL,
    PRIMARY KEY(fd1)
) ENGINE=Aria ROW_FORMAT=page
```

二进制日志的作用：
* 恢复（ recovery ）：某些数据的恢复需要二进制日志，例如，在一个数据库全备文件恢复后，用户可以通过二进制日志进行 point-in-time 的恢复。

* 复制（ replication ）：通过 replication 实现 master-slave 架构。

* 审计（ audit ）：用户可以通过二进制日志中的信息来进行审计。

bin-log 文件的位置只能通过 log_bin 参数进行指定。如果不指定的话，默认的二进制文件名为主机名，后缀名为二进制日志的序列号，所在路径为 datadir。

```
log-bin = /data/mysql/3306/data/mybinlog

show variables like '%datadir%';
+---------------+------------------------+
| Variable_name | Value                  |
+---------------+------------------------+
| datadir       | /data/mysql/3306/data/ |
+---------------+------------------------+

show variables like 'log_bin%';
+---------------------------------+--------------------------------------+
| Variable_name                   | Value                                |
+---------------------------------+--------------------------------------+
| log_bin                         | ON                                   |
| log_bin_basename                | /data/mysql/3306/data/mybinlog       |
| log_bin_index                   | /data/mysql/3306/data/mybinlog.index |
| log_bin_trust_function_creators | OFF                                  |
| log_bin_use_v1_row_events       | OFF                                  |
+---------------------------------+--------------------------------------+
```

生成的 binlog 文件大致如下：

```
-rw-r-----. 1 mysql mysql 1091925644 Jan 30 14:08 mybinlog.001956
-rw-r-----. 1 mysql mysql  814324722 Jan 30 15:36 mybinlog.001957
-rw-r-----. 1 mysql mysql       3306 Jan 30 14:08 mybinlog.index
```

mybinlog.001956 和 mybinlog.001957 都是二进制文件，mybinlog.index 用来记录之前产生的二进制日志序号。

有如下的一些参数会影响到二进制日志记录的信息和行为：

#### max_binlog_size

max_binlog_size 指定了单个二进制日志文件的最大值，如果超过这个设定的值，就会新产生新的二进制日志文件，并将后缀加 1 ，新生成的序号会记录到 .index 文件中。 max_binlog_size 参数可以设置其大小，默认值为 1G 。

```
show variables like 'max_binlog_size';
+-----------------+------------+
| Variable_name   | Value      |
+-----------------+------------+
| max_binlog_size | 1073741824 |
+-----------------+------------+
```

#### binlog_cache_size

执行事务操作时，所有的 uncommitted 的二进制日志会被记录到一个 buffer 中，等该事务操作 committed 时，会直接从 buffer 写入到二进制日志文件中，该 buffer 的大小由 binlog_cache_size 决定，默认大小为 32K 。

需要注意的是，`binlog_cache_size` 是基于 session 的，也就意味着 Mysql 会为每一个连接线程开启对应大小的空间。因此，我们需要设置一个合理的值，不能设置的过大，也不能设置的过小。

可以通过查看 binlog_cache_use （记录了使用 buffer 写二进制日志的次数） 和 binlog_cache_disk_use （记录了使用临时文件写二进制日志的次数） 的状态来判断 binlog_cache_size 是否设置好了合理的值。

```
show variables like 'binlog_cache_size';
+-------------------+---------+
| Variable_name     | Value   |
+-------------------+---------+
| binlog_cache_size | 4194304 |
+-------------------+---------+

show global status like 'binlog_cache%';
+-----------------------+---------+
| Variable_name         | Value   |
+-----------------------+---------+
| Binlog_cache_disk_use | 38786   |
| Binlog_cache_use      | 6034372 |
+-----------------------+---------+
```

从上面的输出可以看到，在当前这台 Mysql 服务器上，将 binlog_cache_size 设置到 4MB 之后， disk_use / cache use = 38786 / (6034372 + 38786) = 0.6% ，因此可能暂时还没有继续增加该值的需要。

### sync_binlog

在默认的情况下，二进制日志并不是在每一次都同步到磁盘。因此，如果在数据库宕机时，可能会有一部分数据没有写入二进制日志文件中，这会给恢复和复制带来问题。

sync_binlog 用于设置每写 buffer 多少次了之后就同步到磁盘。如果将其值设置为1表示采用同步写磁盘的方式来写二进制日志，这时写操作不使用操作系统的缓冲来写二进制日志。如果使用 InnoDB 存储引擎进行 replication ，并且想得到最大的高可用性，建议将其值设置为 ON 。

即使将 sync_binlog 设为 1 ，某些场景下还是可能会出现问题。当使用 InnoDB 时，在一个事务发出 Commit 动作之前，由于 sync_binlog 为 1，因此会将二进制日志立即写入磁盘，如果此时已经写入了二进制日志，但是提交还没有发生，此时发生了宕机，那么在 Mysql 数据库下次启动时，由于 Commit 操作并没有发生，这个事务会被回滚掉。但是二进制日志中已经记录了该事务信息，无法进行回滚。这个问题可以通过将参数 innodb_support_xa 设置为 1 来解决。

```
show variables like 'innodb_support_xa';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| innodb_support_xa | ON    |
+-------------------+-------+
```

### binlog-do-db & binlog-ignore-db

参数 binlog-do-db 和 binlog-ignore-db 表示需要写入或忽略写入哪些库的日志。默认为空，表示需要同步所有库的日志到二进制日志。


### log_slave_updates

如果需要搭建复杂一点的结构，比如说，A->B->C，其中B是A的从服务器，同时B又是C的主服务器，那么B服务器除了需要打开 log-bin 之外，还需要设置 log_slave_updates 为 ON 。

```
show variables like 'log_slave_updates%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| log_slave_updates | ON    |
+-------------------+-------+
```

### binlog_format

binlog_format 影响了二进制日志记录的格式，它非常重要。在 Mysql 5.1 版本之前，没有这个参数，所有的二进制文件的格式都是基于 SQL 语句（ statement ）级别的，这会导致很多主从服务器上表上很多数据不一致的问题。

Mysql 5.1 版本开始引入了 binlog_format 参数，能够设置如下3个值：

* STATEMENT ：与之前的 Mysql 版本一样，二进制文件记录的是日志的逻辑 SQL 语句。
* ROW ：将每次更改的数据记录到日志中。
* MIXED ：默认采用 STATEMENT 格式进行记录，但是在一些特殊场景下会使用 ROW 格式。

```
show variables like 'binlog_format%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
```

通常的情况下，如果 binlog_format 设置为了 ROW ，可以将 Mysql 的事务隔离级别由 Repeatable read 设置为 READ-COMMITED ，以提高事务处理的并发性。

此外将 binlog_format 设置为了 ROW 之后，相比 STATEMENT 模式，会让生成的 binlog 的文件占用空间增加很多。

### Binlog 扩展阅读

以下的2篇 blog 对 binlog 的二进制编码方式有更深入的解释，值得一读。

http://blog.csdn.net/u013256816/article/details/53020335
https://www.jianshu.com/p/c16686b35807


## socket 文件

UNIX 系统下本地连接 Mysql 可以采用 UNIX 域套接字方式，这种方式需要一个 socket 文件。套接字文件可由参数 socket 控制。

```
show variables like 'socket';
+---------------+---------------------+
| Variable_name | Value               |
+---------------+---------------------+
| socket        | /dev/shm/mysql.sock |
+---------------+---------------------+
```

## pid 文件

当 Mysql 实例启动时，会将自己的进程 ID 写入一个文件中： pid 文件。

```
show variables like 'pid_file';
+---------------+---------------------------------+
| Variable_name | Value                           |
+---------------+---------------------------------+
| pid_file      | /data/mysql/3306/data/mysql.pid |
+---------------+---------------------------------+
```

## 表结构定义文件

因为 Mysql 存储引擎的体系结构， Mysql 的数据存储是根据表进行的，每一个表都会有与之对应的文件。但无论采用什么存储引擎， Mysql 都有一个以 frm 为后缀名的文件，这个文件记录了该表的表结构定义。

# InnoDB 存储引擎文件

之前介绍的文件都是和 Mysql 数据库本身的文件，和存储引擎无关。除了这些文件之外，每个表存储引擎还有其自己独有的文件。主要包括：重做日志文件、表空间文件。

## 表空间文件

InnoDB 采用将存储的数据按照表空间（ tablespace ）进行存放的设计。在默认配置下会有一个初始大小为 10MB ，名为 ibdata1 的文件。该文件就是默认的表空间文件，用户可以通过参数 innodb_data_file_path 对其进行设置，并且用户可以通过多个文件组成一个表空间，同时制定文件的属性，格式如下：

```
[mysqld]
innodb_data_file_path = /db/ibdata1:2000M;/dr2/db/ibdata2:2000M:autoextend

show variables like 'innodb_data_file_path';
+-----------------------+------------------------+
| Variable_name         | Value                  |
+-----------------------+------------------------+
| innodb_data_file_path | ibdata1:12M:autoextend |
+-----------------------+------------------------+
```

上述的信息中，将 /db/ibdata1 和 /dr2/db/ibdata2 两个文件组成表空间。若这两个文件位于不同的磁盘上，磁盘的负载可能被平均分摊，因此可以提高数据库的整体性能。此外，指定了两个文件的大小为 2000MB ，其中第二个文件如果用完了 2000MB，该文件可以自动扩展（ autoextend ）。

设置 innodb_data_file_path 参数后，所有基于 InnoDB 存储引擎的表的数据都会记录到该共享表空间中。若设置了参数 innodb_file_per_table ，则用户可以将每个基于 InnoDB 存储引擎的表产生一个独立表空间。独立表空间的命名规则为： 表名.ibd 。

```
show variables like 'innodb_file_per_table';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
```

```
# 在数据存储目录下，会按照数据库名来生成文件夹，下面的 hms 就是其中的一个数据库名。
ll /data/mysql/3306/data/hms

# 由于 innodb_file_per_table 设置为了 ON，每一个表的 ibd 文件都生成。
-rw-r-----. 1 mysql mysql  10485760 Jan 30 20:43 user_siterelation.ibd
-rw-r-----. 1 mysql mysql  14680064 Jan 30 17:17 user_viewhistory.ibd
-rw-r-----. 1 mysql mysql    131072 Jan 29 13:33 webmgmt_themeactivity.ibd
-rw-r-----. 1 mysql mysql    196608 Jan 29 13:33 webmgmt_themeactivityregistration.ibd
```

需要注意的是，这些单独的表空间文件仅存储该表的数据、索引和插入缓冲 BITMAP 等信息，其余信息还是存放在默认的表空间中。

![ibd-arch](https://www.zhuxiaodong.net/static/images/ibd-arch.png)

## 重做日志文件

在默认的情况下， InnoDB 存储引擎的数据目录下会有两个名为 ib_logfile0 和 ib_logfile1 的文件，称为重做日志文件（ redo log file ），它们记录了 InnoDB 存储引擎的事务日志。

当实例或介质失败（ media failure ），例如数据库由于所在主机断电，导致实例失败，InnoDB 存储引擎会使用重做日志恢复到掉电前的时刻，以此来保证数据的完整性。

每个 InnoDB 存储引擎至少有1个重做日志文件组（ group ），每个文件组下至少有 2 个重做日志文件。为了得到更高的可靠性，用户可以设置多个镜像日志组，将不同的文件组放在不同的磁盘上，以此提高重做日志的高可用性。在日志组中会采用依次循环写入的方式，即 log1 写满了之后切换至 log2 ， log2 写满了之后写 log3 ， log3 写满了之后再写 log1 的方式。

下面的一些重要参数，影响了重做日志文件的行为。

**innodb_log_file_size**

参数 innodb_log_file_size 指定每个重做日志文件的大小。

```
show variables like 'innodb_log_file_size';
+----------------------+------------+
| Variable_name        | Value      |
+----------------------+------------+
| innodb_log_file_size | 2147483648 |
+----------------------+------------+
```

**innodb_mirrored_log_groups**

指定了重做日志文件组中重做日志文件的数量，默认为 2 。

```
show variables like 'innodb_log_files_in_group';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| innodb_log_files_in_group | 2     |
+---------------------------+-------+
```

**innodb_log_group_home_dir**

指定了日志文件组所在路径，默认为： ./ ，表示在 Mysql 数据库的数据目录下。

```
show variables like 'innodb_log_group_home_dir';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| innodb_log_group_home_dir | ./    |
+---------------------------+-------+
```

重做日志文件的大小设置对于 InnoDB 存储引擎的性能有着非常大的影响。一方面重做日志不能设置的过大，因为会造成恢复的时间过长；另一方面重做日志不能设置得太小了，否则可能导致一个事务的日志需要多次切换重做日志文件。

此外，重做日志文件太小会导致频繁地发生 async check-point ，导致性能的抖动。例如，我们可能在错误日志中看到如下的警告信息：

```
InnoDB:ERROR:the age of the last check-point is 9433645，InnoDB:which exceeds the log group capacity9433498
```

这是因为重做日志有一个 capacity 变量，该值代表了最后的检查点不能超过这个阈值，如果超过则必须将缓冲池中的脏页列表（ flush list ）中的部分脏数据页写回磁盘，这会导致用户线程的阻塞。

###  InnoDB 重做日志 与 Binlog 的区别

Binlog 会记录所有的日志记录，并且包括了其它存储引擎（ MyISAM 、Heap ）的日志。而 InnoDB 存储引擎的重做日志只记录 InnoDB 自己的事务日志。

其次，两者记录的内容也不太一样， Binlog 记录的是一个事务具体操作内容，即逻辑日志。而 InnoDB 存储引擎的重做日志记录的是关于每个页（ Page ）的更改的物理情况。

此外，写入的时间也不同，二进制日志文件仅在事务提交前写入磁盘一次，不论此时事务有多大。而 InnoDB 存储引擎的重做日志会在事务进行过程中，不断将内容写入到重做日志中。

### InnoDB 重做日志的格式

由 4 个部分组成：
* redo_log_type ：占用1个字节，表示重做日志的类型
* space ： 表示表空间的 ID ，但采用压缩的方式，因此占用的空间可能小于 4 字节。
* page_no ：表示页的偏移量，因此采用压缩的方式。
* redo_log_body ：表示每个重做日志的数据部分，恢复时需要调用相应的函数进行解析。

写入重做日志文件的操作不是直接写，而是先写入一个重做日志缓冲中，然后按照一定的条件顺序地写入日志文件。在重做日志从缓冲往磁盘写入时，是按照 512 字节，也就是一个磁盘扇区的大小写入。**因为扇区是写入磁盘的最小单位，因此可以保证写入是必定成功的，不需要额外的 dobulewrite 机制保证。**

### InnoDB 重做日志的触发条件

第一种触发条件是，Master Thread 每秒会将重做日志缓冲写入磁盘的重做日志中，无论事务是否已经提交。

第二种触发条件是，由参数 innodb_flush_log_at_trx_commit 控制，表示在提交操邹时，处理重做日志的方式。

innodb_flush_log_at_trx_commit 的有效值为：0、1、2 。

* 0 ：表示提交事务时，并不将事务的重做日志写入磁盘上的日志文件，而是等待 Master Thread 每1秒的刷新。
* 1 ：表示在执行 commit 时，将重做日志缓冲同步写到磁盘，即执行 fsync 的系统调用。
* 2 ：表示将重做日志异步写到磁盘，即写到文件系统的缓存中。

```
show variables like 'innodb_flush_log_at_trx_commit';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 1     |
+--------------------------------+-------+
```

通过参数的说明分析可以得出，如果要确保绝对的数据安全，即 ACID 当中的持久性（ Duration ），就需要将 innodb_flush_log_at_trx_commit 参数设置成 1 ；而设置为 0 或者 2 都可能导致，数据库进行恢复时部分事务的丢失。如果设置为 2 ，安全性很稍微高一些，因为如果出现仅仅是 Mysql 服务器进程 down 掉，但是操作系统或者服务器并没有宕机的情况时，由于此时未写入磁盘的事务日志保存在文件系统的缓存中，进行数据库恢复时同样可以确保数据不丢失。
