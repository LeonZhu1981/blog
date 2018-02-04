title: learning mariadb & mysql-basic knowledge
date: 2017-11-28 21:55:32
categories: programming
tags:
- mysql
- mariadb
---

## 概述
---

虽然 MariaDB 和 Mysql 在大部分场景和功能上没有太多区别，但是仍然有一些细微的差别。我们将会在描述时，用以下的方式说明哪些特性是 MariaDB 特有的特性：

```
标题 @MariaDB
```

<!--more-->

## status 命令
---

使用 `status` 命令或者 `\s` 命令，能够查看当前连接数据库、 MariaDB 版本信息、字符集、以及一些汇总信息。

```
MariaDB [employees]> \s
--------------
mysql  Ver 15.1 Distrib 10.2.10-MariaDB, for Linux (x86_64) using readline 5.1

Connection id:          10
Current database:       employees
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server:                 MariaDB
Server version:         10.2.10-MariaDB-log MariaDB Server
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    utf8mb4
Conn.  characterset:    utf8mb4
UNIX socket:            /dev/shm/mysql.sock
Uptime:                 7 hours 39 min 51 sec

Threads: 8  Questions: 437  Slow queries: 302  Opens: 41  Flush tables: 1  Open tables: 23  Queries per second avg:
0.015
--------------
```

## mysql 客户端程序的命令选项
---

以下列出几个之前不太熟悉，但非常有用的命令行参数选项：

* --auto-rehash ：在 mysql 客户端程序内输入表或列名时，能够使用 TAB 键自动补全。
* --batch ：以批处理模式（非交换模式）运行 mysql 客户端程序。
* --execute 、 -e ： mysql 客户端程序在连接 MariaDB 服务器时，同时执行参数给出的语句，用于非交互模式。
* --skip-column-names 、 -N ：在 mysql 客户端中部显示查询结果的列名。
* --safe-updates 、 -U ： 以安全模式运行 mysql 客户端。在该模式下， SELECT、UPDATE、DELETE 查询不能使用索引并使用全表扫描时，查询会自动停止。同时，也能够有效的避免不带 WHERE 条件的 UPDATE ， DELETE 误操作。

下面来重点解释一下这几个重要的参数：

### --safe-updates

开启 --safe-update 模式, 并启用 --auto-rehash 方便使用 TAB 自动补全:

```
mysql -u root -p --safe-updates --auto-rehash
```

我们创建一个测试表来进行测试 --safe-updates 的行为:

```
-- create test table
create table test_table (uid INT, uname VARCHAR(20), PRIMARY KEY(uid));

-- create test data
INSERT INTO test_table (uid, uname) VALUES (1, 'leon');

MariaDB [employees]> DELETE FROM test_table;
ERROR 1175 (HY000): You are using safe update mode and you tried to update a table without a WHERE that uses a KEY column

MariaDB [employees]> UPDATE test_table set uname = 'jimmy' where uname = 'leon';
ERROR 1175 (HY000): You are using safe update mode and you tried to update a table without a WHERE that uses a KEY column

-- create index for uname field;
CREATE INDEX ix_uname ON test_table(uname);

MariaDB [employees]> UPDATE test_table set uname = 'jimmy' where uname = 'leon';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

第三行的 DELETE 语句由于没有带上 WHERE 条件，因此提示了错误的信息。
第四行的 UPDATE 语句由于 uname 字段上没有索引，因此也提示了错误信息。 当在为 uname 字段建立索引了之后，可以看到第六行当中的 sql 语句就能够被正确地执行。

不过 --safe-update 模式对于 `TRUNCATE TABLE` 语句无法起到验证的作用。

```
MariaDB [employees]> TRUNCATE TABLE test_table;
Query OK, 0 rows affected (0.02 sec)
```

### --execute

该参数在非交互模式下会非常有用，例如编写一个 bash 脚本，或者直接导出数据等场景。

```
mysql -u root -p -D employees -e"select * from test_table;"

Enter password:
+-----+-------+
| uid | uname |
+-----+-------+
|   1 | leon  |
+-----+-------+
```

不过上述方式还需要输入一次密码，不是非常方便。可以直接使用 `-p{yourpassword}` 的方式来直接运行需要执行的 sql 脚本。（注意：这种方式会有安全隐患，因为密码暴露在了命令当中）

```
mysql -u root -p{yourpassword} -D employees -e"select * from test_table;"
```

### --batch & --execute & --skip-column-names

```
mysql -u root -p{yourpassword} -D employees --batch -e"select * from test_table;"

uid     uname
1       leon
```

可以看到，添加了 `--batch` 参数之后，查询结果会以无线表格形态输出。这能够非常方便地在 bash 脚本当中去进行解析。同时，如果我们添加了 `--skip-column-names` 参数，能够在查询结果当中不输出列名。

```
mysql -u root -p{yourpassword} -D employees --batch --skip-column-names -e"select * from test_table;"

1       leon
```

## MariaDB 用户账户识别与权限
---

### 用户识别

与其它的关系型数据库不太一样的是， MariaDB 的用户账号不仅包含用户名，还包含了用户的连接地址信息（主机名、域名、 IP 地址）。

以下是一些具体的示例：

```
# id 这个账号只能在 mariaDB 的服务器上进行登录
`id`@`127.0.0.1`

# id 这个账号只能在 IP 地址是 192.168.0.10 服务器上进行登录
`id`@`192.168.0.10`

# id 这个账号能够在所有的机器上进行登录
`id`@`%`
```

如果存在多个重复的账号，例如：

```
# 假设这个账号的密码是: 123
`id`@`192.168.0.10`

# 假设这个账号的密码是: abc
`id`@`%`
```

MariaDB 会优先匹配授权范围最小的那个，也就是 `'id'@'192.168.0.10'`；因此如果用户在 192.168.0.10 这台机器上使用 abc 这个密码登录，将会被服务器拒绝访问。

### 权限

#### 术语
* Privileges ：权限
* Role ：权限组。可以包含多个权限。

#### 授权 SQL 语句的构成

```
GRANT {privilege_list} ON {object} TO {account}
```

* {object} ：指定的数据库对象。可以分为：服务器权限、数据库级权限、表级权限、存储程序级权限。
    * `*.*` ：指定为整个 MariaDB 服务器。此时 {privilege_list} 只能用全局权限。
    * `db1.` ：指定为某个特定的数据库。此时 {privilege_list} 可以出现除了全局权限之外的数据库级、表级、存储过程级。
    * `db1.table1` ：指定为某个特定的数据库下的某一个表。此时 {privilege_list} 只能使用表级权限。
    * `db1.stored_program1` ：指定为某个特定的数据库下的某一个存储过程或自定义函数。此时 {privilege_list} 只能使用存储过程级权限。

* {privilege_list} ：具体的权限列表
* 全局权限：

| 权限命令名称 | 说明 |
| --- | --- |
| CREATE USER | 创建一个新账号的权限 |
| LOAD FILE | 使用 LOAD DATA INFILE 或 LOAD_FILE() 函数访问磁盘数据的权限 |
| GRANT OPTION | 对其他用户授权 |
| PROCESS | SHOW PROCESSLIST 命令的权限 |
| RELOAD | FLUSH 命令的权限 |
| REPLICATION CLIENT | SHOW MASTER STATUS 或 SHOW SLAVE STATUS 的权限 |
| REPLICATION SLAVE | Slave MariaDB 连接 Master MariaDB 服务器时的权限|
| SHOW DATABASES | 显示数据库列表 |
| SHUTDOWN | 关闭 MariaDB 服务器的权限 |
| SUPER | 该权限能够突破某些限制直接进行操作；例如在 read_only 的服务器上修改数据 |

* 数据库级权限：

| 权限命令名称 | 说明 |
| --- | --- |
| CREATE | 创建数据库的权限 |
| CREATE ROUTINE | 创建存储过程或者自定义函数的权限 |
| CREATE TEMPORARY TABLES | 创建临时表的权限 |
| DROP | 删除数据库的权限 |
| EVENT | 创建、删除事件的权限 |
| GRANT OPTION | 可以将自己拥有的权限赋予其它用户 |
| LOCK TABLES | 锁定表的权限 |

* 表级权限：

| 权限命令名称 | 说明 |
| --- | --- |
| ALTER | 修改表的权限 |
| CREATE | 创建表的权限 |
| CREATE VIEW | 创建视图的权限 |
| DELETE | 删除表记录的权限 |
| DROP | 删除表的权限 |
| GRANT OPTION | 可以将自己拥有的权限赋予其它用户 |
| INDEX | 可以使用 CREATE INDEX 来创建索引 |
| INSERT | INSERT 数据的权限 |
| SELECT | 查询数据的权限 |
| SHOW VIEW | 查看视图结构的权限 |
| UPDATE | 修改表记录的权限 |

* 存储程序级权限：

| 权限命令名称 | 说明 |
| --- | --- |
| ALTER ROUTINE | 修改存储程序（存储过程和自定义函数）的权限 |
| EXECUTE | 执行存储程序（存储过程和自定义函数）的权限 |
| GRANT OPTION | 可以将自己拥有的权限赋予其它用户 |

#### 授权

下面是几个常见的授权的示例：

```
# 对特定账号授权
GRANT {privilege_list} ON {object} TO 'user'@'host';

# 如果用户不存在，则先创建用户
GRANT {privilege_list} ON {object} TO 'user'@'host' IDENTIFIED BY 'password';

# WITH GRANT OPTION 比较特殊，用于“将自己拥有的权限赋予其它用户”
GRANT {privilege_list} ON {object} TO 'user'@'host' IDENTIFIED BY 'password' WITH GRANT OPTION;

# 授予所有的权限
GRANT ALL PRIVILEGE ON *.* TO 'user'@'host';

GRANT SELECT,INSERT,UPDATE,DELETE ON *.* TO 'user'@'host';
GRANT SELECT,INSERT,UPDATE,DELETE ON employees.* TO 'user'@'host';
GRANT SELECT,INSERT,UPDATE,DELETE ON employees.department TO 'user'@'host';
```

其中 `{privilege_list}` 中有多个权限时，使用逗号隔开。

#### 权限组 @MariaDB

* 创建权限组：
拥有 CREATE USER 权限的用户能够使用 CREATE ROLE 命令创建权限组。权限组的信息会存储在 mysql.user 表中，使用字段 is_role 列进行存储。

```
CREATE ROLE dba;
CREATE ROLE developer WITH ADMIN {CURRENT_USER | CURRENT_ROLE | user | role};

GRANT ALL PRIVILEGES ON *.* TO dba;
GRANT SELECT,INSERT,UPDATE,DELETE PRIVILEGES ON 'db'.* TO developer;
```

* 授予权限组：

GRANT 命令将特定的权限组授予特定用户，数据存储在 mysql.roles_mapping 表中。

```
GRANT developer TO 'user1'@'%';
```

## MariaDB 数据库基本操作
---

### 默认用户

安装好 MariaDB 之后，会默认创建的账号包括：

* 'root'@'127.0.0.1'
* 'root'@'::1'
* ''@'localhost'
* 'root'@'localhost'

如果是 *nix 操作系统时，如果是在 DB 服务器本机上指定的是 localhost ，使用的是 Unix Domain Socket 文件方式进行连接，而不是 TCP/IP 。

初始化安装好了 MariaDB 之后，请记得使用 mysql_secure_installation 脚本来进行基本的服务器安全设置。

### 默认数据库

```
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

> NOTE: 实际默认数据库中还应该包含 test 数据库，出于安全方面的考虑，运行 mysql_secure_installation 脚本能够会提示删除 test 数据库。

* mysql ：数据库存储了用户认证信息，存储程序与事件信息，时区， Replication 等数据。
* information_schema ：该数据库中的数据表不会在物理磁盘上实际的存储数据文件。在 MariaDB 启动时，会将所有的数据库、数据表、数据列、存储程序的 metadata 读入内存，并通过 information_schema 数据库去访问这些信息。
* performance_schema ：以数字的形式记录了处理查询请求发生的各种统计数据，包括：事件、锁、锁等待。需要说明的是，该数据库的表结构会在物理磁盘上实际存储，但是表中的实际数据只在内存中加载。

### 创建和删除数据库

需要注意的是删除数据库的时候，如果数据库中有大量的数据表和记录，删除数据库时需要耗费很长的时间，因此建议最好在系统使用很少的时间点进行操作。

```
CREATE DATABASE IF NOT EXISTS dbName
CHARACTER SET = 'utf8mb4'
COLLATE = 'utf8mb4_general_ci';

DROP DATABASE dbName;
```

### 创建数据表

```
USE hms;

/*!40101 SET NAMES utf8 */;
CREATE TABLE IF NOT EXISTS CP_AuthUser_OperationLog (
	`id` int NOT NULL AUTO_INCREMENT
	,`authUserID` int NOT NULL
	,`opertaionType` tinyint NOT NULL COMMENT '1--登录2--注销3--切换公司'
	,`currentSelectedCompanyCode` int NULL COMMENT '如果用户切换公司时, 该字段会记录用户切换的CompanyCode'
	,`inDate` timestamp(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3)
	,CONSTRAINT PK_CP_AuthUser_OperationLog_id PRIMARY KEY (id)
)engine=InnoDB default charset utf8mb4;

CREATE INDEX IF NOT EXISTS IX_CP_AuthUser_OperationLog_authUserID ON CP_AuthUser_OperationLog(authUserID);
```

* `/*!40101 SET NAMES utf8 */;` 用于在注释（ COMMENT ）当中支持中文描述信息。
* `CREATE INDEX` 时， Mysql 不支持 `IF NOT EXISTS`， MariaDB 能够支持。
* `engine=InnoDB default charset utf8mb4;` 显示指定了存储引擎和字符集。

#### 查看表结构

```
DESC table_name;

SHOW CREATE TABLE table_name;
```

其中 `SHOW CREATE TABLE` 能够查看到的信息更加全面，包括了索引信息。

```
CREATE TABLE `employees` (
  `emp_no` int(11) NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) NOT NULL,
  `last_name` varchar(16) NOT NULL,
  `gender` enum('M','F') NOT NULL,
  `hire_date` date NOT NULL,
  PRIMARY KEY (`emp_no`),
  KEY `ix_firstname` (`first_name`),
  KEY `ix_hiredate` (`hire_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

### 修改数据表

#### 离线修改 Schema

在 MariaDB 5.5 与 Mysql 5.5 之前的版本并不支持在线修改 Schema 的功能，而是采取先将数据复制到临时表，然后删除原表，并将临时表修改为原表名的方式进行。假设我们需要修改 test_table 表，相关的修改步骤如下：

* lock test_table 数据表
* 创建对应的临时表
* 从 test_table 中将对应所有记录复制到步骤2的临时表中
* 删除 test_table 数据表，将临时表重命名为 test_table
* unlock test_table

由于修改数据表的结构时，会使其它连接无法修改该表的数据，因此对应一个大型的互联网应用来说，需要考虑如下的问题：

* 修改大表记录的表结构时，最好放在业务量较少的情况下进行，例如在凌晨执行。
* 考虑使用多 master 的模式进行更新。
* 将多个修改语句合并成一个 ALTER TABLE 语句执行。例如：

```
ALTER TABLE test_table
ADD created DATETIME NOT NULL
,ADD INDEX IX_created(created);
```

更多相关信息可以参考：http://www.oschina.net/translate/avoiding-mysql-alter-table-downtime

事实上，在 MariaDB 5.5版本之前，并非所有对 Schema 的修改都会进行记录的复制，例如如下的类型就不会做记录的复制：

* 修改列名
* 修改数值类型的长度，比如从 INT(2) 调整为 INT(3)
* 修改表的注释信息
* 向 ENUM 类型项目列表最后添加新项目
* 修改数据表名

#### 在线修改 Schema @MariaDB

`ALTER ONLINE TABLE` 中的 `ONLINE` 关键字能够判断出该表结构更改是否需要复制数据。

下面我们将使用 MariaDB 的 10.2.10 版本来进行测试，测试的案例是笔者在日常工作当中经常会遇到的。

* 向已有的表当中添加字段：不需要复制数据。

```
ALTER ONLINE TABLE test_table
ADD modified DATETIME NULL
,ADD uold TINYINT NOT NULL;
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

* 修改字段类型（将 varchar 类型修改为 char 类型）：**需要复制数据**

```
ALTER ONLINE TABLE test_table MODIFY uname char(20) NULL;
ERROR 1846 (0A000): LOCK=NONE is not supported. Reason: Cannot change column type INPLACE. Try LOCK=SHARED
```

* 修改字段名：不需要复制数据

注意：修改字段名在生产环境一定谨慎执行，因为可能有很多历史程序还在使用老的字段名。

```
ALTER ONLINE TABLE test_table CHANGE uold old TINYINT NOT NULL;
Query OK, 0 rows affected (0.01 sec)
```

* 修改字段长度（ varchar(20) --> varchar(50) ）：不需要复制数据

```
ALTER ONLINE TABLE test_table MODIFY uname VARCHAR(50) NULL;
Query OK, 0 rows affected (0.01 sec)
```

* 修改字段（从 NULL --> NOT NULL ）：不需要复制数据

```
ALTER ONLINE TABLE test_table MODIFY uname VARCHAR(50) NOT NULL;
Query OK, 0 rows affected (0.01 sec)
```

* 增加索引：不需要复制数据

```
ALTER ONLINE TABLE test_table ADD INDEX ix_modified(modified);
Query OK, 0 rows affected (0.01 sec)
```

* 删除索引：不需要复制数据

```
ALTER ONLINE TABLE test_table DROP INDEX ix_modified;
Query OK, 0 rows affected (0.01 sec)
```

* 删除字段：不需要复制数据

注意：修改字段名在生产环境一定谨慎执行，因为可能有很多历史程序还在使用。

```
ALTER ONLINE TABLE test_table DROP COLUMN old;
Query OK, 0 rows affected (0.01 sec)
```

#### MariaDB 和 Mysql 都支持的在线修改 Schema 功能

Mysql 5.6 及以上的版本，`ALTER TABLE` 支持新的子句命令，包括：

* LOCK={ DEFAULT | NONE | SHARED | EXCLUSIVE }

| 选项 | 说明 |
| --- | --- |
| NONE | 允许修改表结构的执行读和写操作 |
| SHARED | 共享锁，只允许执行读操作 |
| EXCLUSIVE | 排它锁，读写操作都不允许 |
| DEFAULT | 默认选项，相当于不明确使用 LOCK 子句 |

* ALGORITHM={ DEFAULT | INPLACE | COPY }

| 选项 | 说明 |
| --- | --- |
| INPLACE | 不需要复制数据至临时表，而是在原位置（inplace）直接进行修改。 |
| COPY | 复制数据至临时表 |
| DEFAULT | 默认选项，相当于不明确使用 ALOGRITHM 子句 |

需要重点说明的是，INPLACE 算法应用于在线修改 Schema 时，Mysql 服务器会按照如下的操作步骤进行：

1.) 检查数据表所使用的存储引擎是否支持 INPLACE 的方式修改。
2.) 为 INPLACE 方式修改 Schema 做准备工作，准备关于新建索引信息，并为在线修改 Schema 期间的修改数据作准备。
3.) 修改数据表 Schema 以及记录新 DML ：实际修改 Schema，在此期间，其它连接的 DML 操作不会阻塞。在线修改 Schema 期间，其他线程中用户发出的 DML 命令会记录到另外的日志中；
4.) 执行日志记录：在线修改 Schema 期间，其他用户发出的 DML 命令会保存到另外的日志。
5.) 以 INPLACE 方式完成（ COMMIT ） Schema 修改。

上述步骤中的2和4中，会启用 EXCLUSIVE Lock，其他连接的 DML 将会短暂等待。实际的修改操作在步骤3中进行，耗时较多，但其他连接的 DML 操作不会被阻塞。

**在线修改日志（ Online alter log ）**：

采用 INPLACE 算法在线修改数据表期间，如果此时系统有新的 DML 语句需要执行，会将这些 SQL 语句放入 `在线修改日志` 的**内存**空间中。 Schema 的修改完成了之后，才会将 `在线修改日志` 的内存空间中的数据迁移到修改后的表中。

`在线修改日志` 存储在内存中，而非磁盘中，它的大小由 `innodb_online_alter_log_max_size` 系统变量确定，它的默认大小为 128 MB。参考: [这里](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_online_alter_log_max_size)

需要注意的是，如果在修改一个非常大的表的 Schema 时，而在修改期间恰好又是业务的高峰时间，很有可能会出现有大量的 DML 语句到来，导致超过了 128 MB的默认大小，此时服务器可能会产生如下的错误：

```
ERROR 1799 (HY000) at line 1: Creating index 'PRIMARY' required more than 'innodb_online_alter_log_max_size' bytes of modification log. Please try again.
```

这会导致可能之前已经执行了几个小时的 Schema 修改操作前功尽弃，因此建议有上述场景时，将 `innodb_online_alter_log_max_size` 设置为较大的值。

```
set global innodb_online_alter_log_max_size=402653184;
```

**使用了 INPLACE 算法，并不意味着该过程中不会对数据进行复制**：

例如，对数据表进行添加或者删除数据列等操作，也会产生数据复制的过程。因此在 Mysql 5.6 的用户手册中，将 INPLACE 算法进行了再一次的精准定义：

* REOGRANIZE ：采用 INPLACE 算法进行处理并复制数据表记录。
* REBUILD ：不能采用 INPLACE 算法进行处理但需要复制数据表记录。

**在线修改 Schema （Online DDL）的各种处理方式**：

下表列出了一些**重要的**修改 Schema 的时候，对是否采用 INPLACE 算法，是否会阻塞 DML 以及 SELECT 查询语句进行了说明：

| Type  | INPLACE  | 是否阻塞 DML | 是否阻塞 SELECT  | 说明 |
| --- | --- | --- | --- | --- |
| 添加全文索引  | O | X | X | 创建第一个 FULLTEXT INDEX 时需要复制记录 |
| 添加数据列 | O | O | O | **添加 AUTO_INCREMENT 数据列时，会禁止直行其他链接的 DML 。需要复制数据表记录，会导致系统负荷高** |
| 删除数据列 | O | O | O | 需要复制数据表记录，会导致**系统负荷高** |
| 修改数据列类型 | X | X | O | **禁止使用 INPLACE 算法， 禁止执行其他连接的 DML** |
| 添加主键 | O | O | O | **需要复制数据表记录，系统负荷高。此外，在添加主键并将 NULLABLE 修改为 NOT NULL 时，不可使用 INPLACE 算法，也无法执行其他连接的 DML** |
| 删除主键同时添加主键 | O | O | O | **需要复制数据表记录，系统负荷高** |
| 仅删除主键 | X | X | O | |
| 修改或定义数据列字符集 | X | X | O | 需要重建数据表 |

**lock_wait_timeout**：

启用 LOCK=NONE 选项在线 Schema 修改时，修改的初始与最终阶段需要对表的元数据进行锁定。如果在该过程中无法锁定数据表的元数据，将会导致在线修改 Schema 超时而失败。

`lock_wait_timeout` 全局和 session 级别的变量决定了此超时时间。默认的时间是86400秒。

```
show global variables like 'lock_wait_timeout';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| lock_wait_timeout | 86400 |
+-------------------+-------+
```

#### pt-online-schema-change

ref url:
http://seanlook.com/2016/05/27/mysql-pt-online-schema-change/
http://seanlook.com/2016/05/24/mysql-online-ddl-concept/
http://www.fromdual.ch/online-ddl_vs_pt-online-schema-change
https://help.aliyun.com/knowledge_detail/41734.html


#### 删除数据表

```
DROP TABLE IF EXISTS test_table;
```

使用 `DROP TABLE` 命令时，需要注意以下两点：

* 从缓冲池中删除相关的数据表页
* 从物理磁盘中删除数据文件

由于整个删除操作会相当消耗资源，尤其 EXT3 文件系统中删除大表文件时。因此，建议在删除较大表时，在凌晨执行。


#### REPLACE VS ON DUPLICATE KEY UPDATE

```
REPLACE test_table SET uid = 1, uname = 'leonzhu';

INSERT INTO test_table (uid, uname, created) VALUES (1, 'leon', '2017-11-28') ON DUPLICATE KEY UPDATE uname = 'leon', created = '2017-11-28';
```

REPLACE 和 INSERT INTO ON DUPLICATE KEY UPDATE 都能够实现在插入数据的时候，主键或者唯一索引重复时，执行 UPDATE 操作，然而它们是有一些区别的：

REPLACE 语句会将记录做删除操作，然后再执行 INSERT 操作。而 INSERT INTO ON DUPLICATE KEY UPDATE 只会执行 UPDATE 操作。因此 REPLACE 操作的资源消耗更高，我们应该尽量使用后者。
