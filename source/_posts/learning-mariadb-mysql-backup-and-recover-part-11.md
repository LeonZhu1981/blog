title: learning mariadb & mysql backup and recover(part-11)
date: 2018-02-08 22:12:43
categories: programming
tags:
- mysql
- mariadb
- backup
- recover
---

# 概述
---

通常我们可以将备份的方法分为以下三种：

* Hot Backup （热备）：是指对数据库运行中直接备份，对正在运行的数据库操作没有任何影响。这种方式在 Mysql 官方手册中称为 Online Backup （在线备份）。
* Cold Backup （冷备）：是指备份操作在数据库停止的情况下进行的备份，一般只需要复制相关的数据库物理文件即可。这种方式在 Mysql 官方手册中称为 Offline Backup （离线备份）。
* Warm Backup （温备）：同样是指在数据库运行中进行的，但是对当前数据库操作有所影响，比如加入一个全局读锁以保证备份数据的一致性。

<!--more-->

按照备份后的文件内容，又可以分为：
* 逻辑备份 ：是指备份出的文件内容是可读的，一般是文本文件。内容一般是指一条条 SQL 语句，或者是表内实际数据组成。比如使用 mysqldump 和 SELECT * INTO OUTFILE 的方法。这种方式的好处是可以直接观察到导出文件的内容，一般适用于数据库升级、迁移等工作，但是缺点是恢复所需要的时间往往比较长。
* 裸文件备份 ：是指复制数据库的物理文件，既可以是在数据库运行中的复制（比如使用 ibbackup 、 xtrabackup 这类工具），也可以是在数据库停止运行时直接的数据文件复制。它的优势是恢复时间往往较逻辑备份的时间短很多。

按照备份数据库的内容来分，备份还可以分为：
* 完全备份：是指对数据库进行一个完整的备份。
* 增量备份：是指在上次完全备份的基础上，对于更改的数据进行备份。
* 日志备份：是指对 Mysql 数据库二进制日志的备份，通过对一个完全备份进行二进制日志的重做（ replay ）来完成数据库的 point-in-time 的恢复工作。 Mysql 数据库复制（ replication ）的原理是异步实时地将二进制日志重做传送并应用到从（ slave/standby ）数据库。

xtrabakcup 是能够很好的支持增量备份的工具。其原理是记录当前每页最后的检查点的 LSN ，如果大于之前全备时的 LSN ，则备份改业，否则不用备份，这大大加快了备份的速度和恢复时间。

此外还需要理解数据库备份的一致性，这种备份要求在备份的时候数据在这一时间点上是一致的。举例来说，在一个网络游戏中有一个玩家购买了道具，这个事务的过程是：先扣除相应的金钱，然后向其装备表中插入道具，确保扣费和得到道具是互相一致的。否则，在恢复时，可能出现金钱被扣除了而装备丢失的问题。

对于 InnoDB 存储引擎来说，因为其支持 MVCC 功能，因此实现一致的备份比较简单。用户可以先开启一个事务，然后导出一组相关的表，最后提交。当然用户的事务隔离级别必须设置为 REPEATABLE READ ，这样的做法就可以给出一个完美的一致性备份。然而这个方法的前提是需要用户正确地设计应用程序。对于上述的购买道具的过程，不可以分为两个事务来完成，如一个完成扣费，一个完成道具的购买。若备份这时发生在这两者之间，则由于逻辑设计的问题，导致备份出的数据依然不是一致的。

对于 mysqldump 备份工具来说，可以通过添加--single-transaction选项获得 InnoDB 存储引擎的一致性备份。

# 冷备份
---

对于 InnoDB 存储引擎的冷备非常简单，只需要单独备份 Mysql 数据库的 frm 文件，共享表空间文件，独立表空间文件（ .ibd ），重做日志文件这四种类型的文件。此外，对于 my.cnf 配置文件也建议定时进行备份。

在冷备份的过程中，需要注意以下几点：

* 不要遗漏原本需要备份的物理文件，例如：共享表空间和重做日志文件，少了这些文件可能数据库都无法启动。
* 为了提高可靠性，需要将物理文件备份至专门的远程备份服务器上。

冷备份的优点：

* 备份简单，只要复制相关文件即可。   
* 备份文件易于在不同操作系统，不同MySQL版本上进行恢复。   
* 恢复相当简单，只需要把文件恢复到指定位置即可。   
* 恢复速度快，不需要执行任何SQL语句，也不需要重建索引。

冷备份的缺点：

* InnoDB 存储引擎冷备的文件通常比逻辑文件大很多，因为表空间中存放着很多其他的数据，如undo段，插入缓冲等信息。   
* 冷备也不总是可以轻易地跨平台。操作系统、MySQL的版本、文件大小写敏感和浮点数格式都会成为问题。

# 逻辑备份
---

## mysqldump

mysqldump 通常用来完成转存（ dump ）数据库的备份以及不同数据库之间的移植，如从 Mysql 低版本升级到到高版本，或者是从 Mysql 数据库移植到 Oracle ， SQL Server 等。

mysqldump 的语法如下：

```
mysqldump [arguments] > file_name
```

如果想要备份所有的数据库，可以使用 `-all-databases` 选项：

```
mysqldump --all-databases > dump.sql
```

如果想要备份指定的数据库，可以使用 `--databases` 选项：

```
mysqldump --databases db1 db2 db3 > dump.sql
```

以下是一个完整的 bash 脚本示例：

```
#!/bin/bash
data=$(date +%F%H)
/usr/local/mysql/bin/mysqldump -h host -u xxx -p"xxx" --single-transaction dbname > /data/backup/data/dbname$data.sql
cd /data/backup/data/
Tar=`ls -l|awk -F " " '{print $9}'|grep "sql"|grep -v "tar.gz"|grep -v "mysql.sh"`
for s in $Tar;do tar -zcvf $s.tar.gz $s; done
find /data/backup/data/ -name "*.sql"| xargs rm -f
```

mysqldump 的功能非常丰富，下面列出一些比较重要的参数：

* --single-transaction ：在备份开始前，先执行 START TRANSACTION 命令，以此来获得备份的一致性，当前该参数只对 InnoDB 存储引擎有效。当启用该参数并进行备份时，确保没有其他任何的DDL语句执行，因为一致性读并不能隔离DDL操作。

* --lock-tables(-l) ：在备份中，依次锁住每个架构下的所有表。一般用于 MyISAM 存储引擎，当备份时只能对数据库进行读取操作，不过备份依然可以保证一致性。对于 InnoDB 存储引擎，不需要使用该参数，用 --single-transaction 即可。并且 --lock-tables 和 --single-transaction 是互斥的，不能同时使用。如果用户的 MySQL 数据库中，既有 MyISAM 存储引擎的表，又有 InnoDB 存储引擎的表，那么这时用户的选择只有 --lock-tables 了。此外，正如前面所说的那样， --lock-tables 选项是依次对每个架构中的表上锁的，因此只能保证每个架构下表备份的一致性，而不能保证所有架构下表的一致性。

* --master-data[=value] ：通过该参数产生的备份转存文件主要用来建立一个 replication 。当 value 的值为1时，转存文件中记录 CHANGE MASTER 语句。当 value 的值为2时， CHANGE MASTER 语句被写出 SQL 注释。

* --events(-E) ：备份事件调度器。
* --routines(-R) ：备份存储过程和用户自定义函数。
* --triggers ： 备份触发器。
* --hex-blob ：将 BINARY 、 VARBINARY 、 BLOG 和 BIT 列类型备份为 16 进制的格式。 mysqldump 导出的文件一般是文本文件，但是如果导出的数据中有上述这些类型，在文本文件模式下可能有些字符不可见，若添加 --hex-blob 选项，结果会以十六进制的方式显示。

此外， mysqldump 还可以实现以 `SELECT...INTO OUTFILE` 的方式来导出一张表或者多张表，并且实现导出数据的一致性。使用 `--where=‘where_condition’` 来导出指定条件的数据。

## SELECT...INTO OUTFILE

SELECT...INTO 语句也是一种逻辑备份的方法，更准确地说是导出一张表中的数据。

```
select * into outfile '/root/a.txt' from test_table;
```

需要注意的，当前文件所在路径的权限必须是 mysql:mysql ，否则会报出没有权限的错误。

如果不想使用默认的 TAB 进行列的分隔，可以使用 FIELDS TERMINATED BY 'string' 选项：

```
select * into outfile '/root/a.txt' fields terminated by ',' from test_table;
```

在 windows 平台下，由于换行符是 `\r\n` ，因此在导出时可能需要指定 LINE TERMINATED BY 选项：

```
select * into outfile '/root/a.txt' fields terminated by ',' line terminated by '\r\n' from test_table;
```

## 逻辑备份的恢复

mysqldump 的恢复操作比较简单，因为备份的文件就是导出的 SQL 语句，一般只需要执行这个文件就可以了，可以通过以下的方法：

```
mysql -uroot -p <test_backup.sql
```

也可以通过 source 命令来执行导出的逻辑备份文件：

```
source /home/mysql/test_backup.sql;
```

## LOAD DATA INFILE

如果通过 `mysqldump --tab` 或者 `select into outfile` 导出的数据需要恢复，可以通过 `LOAD DATA INFILE` 来进行导入。

```
load data infile '/home/mysql/a.txt' into table a;
```

## mysqlimport

mysqlimport 是 Mysql 数据库提供的一个命令行程序，从本质上来说，其实就是 LOAD DATA INFILE 的命令接口。

```
mysqlimport --use-threads=2 test /home/mysql/t.txt /home/mysql/s.txt
```

# 二进制日志备份与恢复
---

我们可以通过二进制日志完成 point-in-time 的恢复工作。 Mysql 数据库的 replication 同样需要二进制日志。我们首先需要在配置文件中进行设置：

```
[mysqld]
log-bin=mysql-bin
```

我们还需要启用一些其它的参数来保证最为安全和正确地记录二进制日志：

```
[mysqld]
log-bin=mysql-bin
sync_binlog=1
innodb_support_xa=1
```

在备份二进制日志文件前，可以通过 FLUSH LOGS 命令来生成一个新的二进制日志文件，然后备份之前的二进制日志。

恢复二进制日志比较简单：

```
mysqlbinlog binlog.0000001 | mysql -uroot -p test
```

可以通过 mysqlbinlog 命令导出一个文件，然后再通过 source 命令导入一个文件：

```
mysqlbinlog binlog.0000001 > /tmp/1.sql
mysqlbinlog binlog.0000002 >> /tmp/1.sql
mysql -uroot -p -e "source /tmp/1.sql"
```

我们还可以通过 --start-position 和 --stop-position 选项来从指定二进制日志的某个偏移量来进行恢复：

```
mysqlbinlog --start-position=107856 binlog.0000001 | mysql -uroot -p test
```

还可以通过 --start-datetime 和 --stop-datetime 选项来从某个时间点进行恢复。

# 热备
---

## ibbackup

ibbackup 是 Mysql Enterprise 版本提供的备份工具，可以同时备份 MyISAM 存储引擎和 InnoDB 存储引擎的表。

对于InnoDB存储引擎表其备份工作原理如下：
* 记录备份开始时， InnoDB 存储引擎重做日志文件检查点的 LSN 。
* 复制共享表空间文件以及独立表空间文件。
* 记录复制完表空间文件后， InnoDB 存储引擎重做日志文件检查点的 LSN 。
* 复制在备份时产生的重做日志。

其优点是：
* 在线备份，不阻塞任何的SQL语句。
* 备份性能好，备份的实质是复制数据库文件和重做日志文件。
* 支持压缩备份，通过选项，可以支持不同级别的压缩。
* 跨平台支持，ibbackup 可以运行在 Linux 、 Windows 以及主流的 UNIX 系统平台上。

但是由于其是收费软件，我们更加倾向于选择 Percona 公司编写的开源软件 XtraBackup 。

## XtraBackup

XtraBackup 备份工具是由 Percona 公司开发的开源热备工具。下载地址为：
https://www.percona.com/software/mysql-database/percona-xtrabackup

其基本的使用方式为：

```
./xtrabackup --backup
```

### XtraBackup 实现增量备份

MySQL 数据库本身提供的工具并不支持真正的增量备份，更准确地说，二进制日志的恢复应该是 point-in-time 的恢复而不是增量备份。而 XtraBackup 工具支持对于 InnoDB 存储引擎的增量备份，其工作原理如下：

* 首选完成一个全备，并记录下此时检查点的 LSN 。
* 在进行增量备份时，比较表空间中每个页的 LSN 是否大于上次备份时的 LSN ，如果是，则备份该页，同时记录当前检查点的 LSN 。
