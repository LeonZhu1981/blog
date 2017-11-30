title: learing mariadb & mysql-execution plan(part-1)
date: 2017-11-28 22:30:28
categories:
tags:
---

## 体系结构
---

![mariadb-architecture](http://static.zhuxiaodong.net/blog/static/images/mariadb-architecture.png)

<!--more-->

### 查询执行步骤

* 接收客户端传入的 SQL 语句，进行语法分析并解析成 SQL 解析树。这个步骤由 `sql parser` 进行处理。
* 对 SQL 解析树进行优化处理，并生成最终的执行计划（execution plan），该步骤由 `sql optimizer` 进行处理，具体又可以分为以下几个步骤：
	* 删除不必要的查询条件，将复合预算简化。
	* 如果有多表关联查询，会确定读取顺序。
	* **根据各个表的条件以及索引统计信息，确定要使用的索引**。
	* 将获取的数据写入临时表，做二次运算处理后返回。
* 根据第二个步骤的执行计划，从 `store engine` 中实际读取记录。


### `sql optimizer` 的类型

* 基于代价的优化器（Cost-Based Optimizer CBO）：处理查询时会创建多种可用方法，然后根据各种方法的资源消耗以及目标数据表的统计信息，计算各个执行计划的代价，最终选择代价最小的执行计划。

* 基于规则的优化器（Rule-Based Optimizer RBO）：根据优化器内置的若干规则制定相应的执行计划。

基于规则的优化器目前大部分 RDBMS 都不再使用，基于代价的优化器是目前的主流实现方案。


## 统计信息

对于基于代价的优化器来说，统计信息的准确性非常重要。一但统计信息不准确，就会导致 `sql optimizer` 选择错误的执行计划，使得原本可以在较短时间之内完成的查询，执行时间被大大加长。

### Mysql 5.6 的统计信息

MariaDB 10.0 包含了 Mysql 5.6 的所有关于统计信息的功能。从 Mysql 5.6 开始，使用 mysql 数据库下的 `innodb_index_stats` 和 `innodb_table_stats` 表存储了对应的索引信息。

```
mysql> show tables like '%_stats';
+---------------------------+
| Tables_in_mysql (%_stats) |
+---------------------------+
| innodb_index_stats        |
| innodb_table_stats        |
+---------------------------+
```

可以在 `CREATE TABLE` 的时候使用 `STATS_PERSISTENT` 选项，来设定该表永久保存统计信息。

```
/*!40101 SET NAMES utf8 */;
CREATE TABLE IF NOT EXISTS CP_Role (
	`roleID` int NOT NULL AUTO_INCREMENT COMMENT '主键; 使用uuid()生成'
	,`roleName` varchar(50) NOT NULL COMMENT '角色的名称'
	,`roleDescription` varchar(100) NULL COMMENT '角色的描述'
	,`inUser` varchar(50) NOT NULL COMMENT '创建人'
	,`inDate` datetime(3) NOT NULL COMMENT '创建日期'
	,`lastEditUser` varchar(50) NOT NULL COMMENT '更新人'
	,`lastEditDate` timestamp(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更改日期'
	,`isActive` boolean NOT NULL DEFAULT 1 COMMENT '1:Active 0:Deactive'
	,CONSTRAINT PK_CP_Role_roleID PRIMARY KEY (roleID)
)engine=InnoDB STATS_PERSISTENT=1 default charset utf8mb4;
```

* STATS_PERSISTENT=0 ：采用 Mysql 5.5 以前的方式管理表的统计信息，该方式不会将统计信息持久化至 mysql 数据库的 `innodb_index_stats` 和 `innodb_table_stats` 表中。

* STATS_PERSISTENT=1 ：将统计信息持久化至 mysql 数据库的 `innodb_index_stats` 和 `innodb_table_stats` 表中。

* STATS_PERSISTENT=DEFAULT ：该选项与不显示设置的效果相同，是否持久化统计信息由 `innodb_stats_persistent` 系统变量设置值决定。

`innodb_stats_persistent` 系统变量的值默认为 ON（1）。

```
show variables like '%innodb_stats_persistent';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_stats_persistent | ON    |
+-------------------------+-------+
```

有如下常见的情况下，相关表的统计信息会进行更新：
* 新打开数据表时
* 大量修改表的记录时（有 1/16 的数据执行 UPDATE/INSERT/DELETE）
* 执行 ANALYZE TABLE 命令时
* 执行 SHOW TABLE STATUS 或 SHOW INDEX FROM 命令时
* 使用 InnoDB 监视器时
* innodb_stats_on_metadata 系统变量被设置为 ON ，并执行 SHOW TABLE STATUS 命令时

统计信息的经常变化会导致 `sql optimizer` 的优化策略无法准确的使用执行计划。因此，统计信息持久化了之后，可以有效地防止统计信息被意外地修改。 `innodb_stats_auto_recalc` 系统变量设置为 OFF 也可以防止自动收集统计信息。它的默认值为 ON 。若想使用永久统计信息，可以将其设置为 OFF 。

```
show variables like '%innodb_stats_auto_recalc';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_stats_auto_recalc | ON    |
+--------------------------+-------+
```

在创建数据表时，也可以使用 `STATS_AUTO_RECALC` 选项，以表为单位进行调整，决定是否自动收集统计信息。

* STATS_AUTO_RECALC=1 ：以 Mysql 5.5 之前的方式自动收集统计信息。
* STATS_AUTO_RECALC=0 ：只在执行 ANALYZE TABLE 命令时收集数据表统计信息。
* STATS_AUTO_RECALC=DEFAULT ：与不显示指定的效果一致。是否收集统计信息由系统变量 `innodb_stats_auto_recalc` 决定。

Mysql 5.6 当中使用了两个系统变量来决定如何收集统计信息：

* `innodb_stats_transient_sample_pages` ：收集统计信息采取多少个页的样本数据来进行分析，并将分析结果用于统计信息。默认值为8。

```
show variables like '%innodb_stats_transient_sample_pages';
+-------------------------------------+-------+
| Variable_name                       | Value |
+-------------------------------------+-------+
| innodb_stats_transient_sample_pages | 8     |
+-------------------------------------+-------+
```

* `innodb_stats_persistent_sample_pages` ：执行 ANALYZE TABLE 命令时将采取多个页面进行分析，并将分析结果保存到永久统计信息数据表以供使用。默认值为20。

```
show variables like '%innodb_stats_persistent_sample_pages';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| innodb_stats_persistent_sample_pages | 20    |
+--------------------------------------+-------+
```

如果使用永久统计信息，可以在 MariaDB 服务器维护或者系统使用不多的时间段来进行收集。同时，通过将上述两个系统变量的值设置大一些，能够得到更加准确的统计信息，但也会增加收集统计信息的时间。

### MariaDB 10.0 的统计信息 @MariaDB

MariaDB 5.5 版本之前，由于依据不同存储引擎，以不同的形态管理统计信息，会存在的如下的问题：

* 各个存储引擎提供的统计信息内容不够充分。
* 不允许收集非索引列的统计信息。
* 不容易从MariaDB层面进行控制和扩展。

从 MariaDB 10.0 版本之后，提供了综合统计信息管理功能，该功能不会受到存储引擎的限制，可以管理非索引列并能够永久使用。此外，这些统计信息可以由管理员直接修改，或者单独备份。

综合统计信息由 mysql 数据库下的 `table_stats` 、 `column_stats` 、 `index_stats` 表进行存储，使用 MyISAM 存储引擎。

```
show tables like '%_stats';
+---------------------------+
| Tables_in_mysql (%_stats) |
+---------------------------+
| column_stats              |
| index_stats               |
| table_stats               |
+---------------------------+
```

```
CREATE TABLE `table_stats` (
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `cardinality` bigint(21) unsigned DEFAULT NULL COMMENT '数据表的记录数',
  PRIMARY KEY (`db_name`,`table_name`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Statistics on Tables'
```

```
CREATE TABLE `column_stats` (
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `column_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `min_value` varbinary(255) DEFAULT NULL,
  `max_value` varbinary(255) DEFAULT NULL,
  `nulls_ratio` decimal(12,4) DEFAULT NULL COMMENT 'NULL值的比率',
  `avg_length` decimal(12,4) DEFAULT NULL COMMENT '列值的平均字节数',
  `avg_frequency` decimal(12,4) DEFAULT NULL COMMENT '拥有重复值的平均记录数（1：无重复值）',
  `hist_size` tinyint(3) unsigned DEFAULT NULL,
  `hist_type` enum('SINGLE_PREC_HB','DOUBLE_PREC_HB') COLLATE utf8_bin DEFAULT NULL,
  `histogram` varbinary(255) DEFAULT NULL,
  PRIMARY KEY (`db_name`,`table_name`,`column_name`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Statistics on Columns'
```

```
CREATE TABLE `index_stats` (
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `index_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `prefix_arity` int(11) unsigned NOT NULL COMMENT '索引键区号',
  `avg_frequency` decimal(12,4) DEFAULT NULL COMMENT '拥有重复值的平均记录数（1：无重复值）',
  PRIMARY KEY (`db_name`,`table_name`,`index_name`,`prefix_arity`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Statistics on Indexes'
```

MariaDB 10.0 综合统计信息与 Mysql 5.6 的永久统计信息可以同时进行使用，并通过系统变量 use_stat_tables 设置决定使用哪种统计信息，其取值为：`never` 、 `complementary` 、 `preferably` ，默认值为 `never` 。

* use_stat_tables='never' ：采用 Mysql 5.6 的永久统计信息的方式，不会将统计信息存储到上述的3张表中。

* use_stat_tables='complementary' ：优先使用各存储引擎提供的统计信息，在存储引擎未提供信息或者提供的信息不足时，就会使用 MariaDB 的综合统计信息。

* use_stat_tables='preferably' ：优先使用 MariaDB 的综合统计信息，然后再使用存储引擎的统计信息。

将系统变量 `innodb_stats_auto_recalc` 设置为 ON 之后，数据表结构发生变化或者有大量数据发生变化时，InnoDB 存储引擎会自动收集统计信息。

```
# 设置 innodb 自动收集统计信息
set global innodb_stats_auto_recalc=1;

# 测试有大量数据写入某张表
# 自动刷新该表 innoDB 的统计信息
INSERT INTO tab_recalc SELECT * FROM employees;
Query OK, 300024 rows affected (1.46 sec)
Records: 300024  Duplicates: 0  Warnings: 0

# 可以发现 mariaDB 的综合统计信息并没有数据
# 因为 use_stat_tables='never' 默认值为 never
select * from mysql.table_stats where table_name = 'tab_recalc';
Empty set (0.00 sec)

# innodb 的统计信息有被更新
select * from mysql.innodb_table_stats where table_name = 'tab_recalc'\G
*************************** 1. row ***************************
           database_name: employees
              table_name: tab_recalc
             last_update: 2017-11-24 03:55:46
                  n_rows: 299866
    clustered_index_size: 929
sum_of_other_index_sizes: 0
```

执行 `ANALYZE TABLE` 命令能够收集 MariaDB 的综合统计信息

```
analyze table tab_recalc;
+----------------------+---------+----------+-----------------------------------------+
| Table                | Op      | Msg_type | Msg_text                                |
+----------------------+---------+----------+-----------------------------------------+
| employees.tab_recalc | analyze | status   | Engine-independent statistics collected |
| employees.tab_recalc | analyze | status   | OK                                      |
+----------------------+---------+----------+-----------------------------------------+

# 从 cardinality 收集的数据来看
# 其值比 innodb 收集的row count 要准确地多
select * from mysql.table_stats where table_name = 'tab_recalc';
+-----------+------------+-------------+
| db_name   | table_name | cardinality |
+-----------+------------+-------------+
| employees | tab_recalc |      300024 |
+-----------+------------+-------------+
```

> 注意：由于 `ANALYZE TABLE` 命令会进行全表扫描或全索引全扫描，因此一般在生产环境中采用 master/slave 的方式，在 slave 的机器上做 ANALYZE TABLE，再单独将 mysql 数据库中的数据同步给 master。

MariaDB 10.0 对 `ANALYZE TABLE` 进行了扩展，以支持不同粒度的操作：

```
# 对 tb1 数据表的col1、col2列以及idx1、idx2进行收集
ANALYZE TABLE tb1 PERSISTENT FOR COLUMNS (col1,col2) INDEXES (idx1,idx2)

# 对 tb1 数据表的col1、col2列进行收集
ANALYZE TABLE tb1 PERSISTENT FOR COLUMNS (col1,col2) INDEXES ()

# 对 tb1 数据表的idx1、idx2索引进行收集
ANALYZE TABLE tb1 PERSISTENT FOR COLUMNS () INDEXES (idx1,idx2)

# 对 tb1 数据表进行收集
ANALYZE TABLE tb1 PERSISTENT FOR COLUMNS () INDEXES ()
```

### MariaDB 10.0 的直方图（Histogram-Based）统计信息 @MariaDB

直方图（Histogram）是 RDBMS 中提供的一种基础的统计信息，最典型的用途是估计查询谓词的选择率，以便选择优化的查询执行计划。常见的直方图种类有：等宽直方图、等高直方图、V-优化的直方图，MaxDiff 直方图等等。

在 MariaDB 10.0 中，由索引形成的数据列以及未建索引的数据列都可以保存直方图的信息。表中的 min_value 、 max_value 、 null_radio 、 avg_length 、 avg_frequency 都由直方图进行收集，并存储在 msyql.column_stats 表中。

MariaDB 10.0 采用 Height-Balanced Histogram 算法来管理直方图。对比其它的 RDBMS 系统可以参考：https://www.slideshare.net/SergeyPetrunya/histograms-in-mariadb-mysql-and-postgresql

Height-Balanced Histogram 算法先对所有列的值进行排序，再按照相同的记录个数将其划为几个组，然后取各组的排序后最大的那个值保存到直方图。其中，分组个数由 histogram_size 系统变量来决定。分组的个数越多，直方图越准确，但这需要更多的存储空间与分析时间。

> histogram_size

> The histogram_size variable determines the size, in bytes, from 0 to 255, used for a histogram. This is effectively the number of bins for histogram_type=SINGLE_PREC_HB or number of bins/2 for histogram_type=DOUBLE_PREC_HB. **If it is set to 0 (the default), no histograms are created when running an ANALYZE TABLE.**

```
show variables like '%histogram%';
+----------------+----------------+
| Variable_name  | Value          |
+----------------+----------------+
| histogram_size | 0              |
| histogram_type | SINGLE_PREC_HB |
+----------------+----------------+
```

MariaDB 会为创建出的直方图值按照不同的组别各分配一个字节，然后存储到 VARBINRY(255) 类型的 histogram 数据列当中。其中 histogram 列中值的计算公式为：

`组最大值 / (列最大值 - 列最小值) * alpha`

alpha 系数的值越大，直方图的准确度越高，但需要更多的存储空间。该系数的值由系统变量 `histogram_type` 来决定。 

* SINGLE_PREC_HB ：alpha 的值为28
* DOUBLE_PREC_HB ：alpha 的值为216

由于 histogram 列的类型为 VARBINRY(255) ，因此 MariaDB 提供了 DECODE_HISTOGRAM 函数来进行解析：

```
set session histogram_size=30;
set session histogram_type='DOUBLE_PREC_HB';

ANALYZE TABLE salaries;

SELECT table_name
,min_value
,max_value
,hist_size
,hist_type
,DECODE_HISTOGRAM(hist_type,histogram) as histogram FROM mysql.column_stats 
WHERE table_name='salaries' 
AND column_name='salary'\G

*************************** 1. row ***************************
table_name: salaries
 min_value: 38623
 max_value: 158220
 hist_size: 30
 hist_type: DOUBLE_PREC_HB
 histogram: 0.02759,0.02521,0.02394,0.02263,0.02202,0.02196,0.02222,0.02274,0.02377,0.02547,0.02792,0.03194,0.03830,0.04883,0.07225,0.54322
1 row in set (0.00 sec)
```

从上述结果可以看出，实际直方图组个数只有16个，而并非 hist_size 设置的30个。因此 hist_size 的设置值不直接决定实际直方图的组数，它决定存储直方图空间的实际字节数。如果 hist_type 为 SINGLE_PREC_HB 时，hist_size 也是直方图的实际组数；但当 hist_type 为 DOUBLE_PREC_HB 时，直方图的实际组数为 (hist_size/2)。

通过 salaries 表的总记录数，可以计算出每一个分组的记录数：2844047/16=177752.93，然后我们可以通过下面的 sql 去计算直方图的准确度：

```
# min_value: 38623
# max_value: 158220 
# 第一个分组的比率: 0.02759
# 第一个分组的最大值: 38623 + (158220 - 38623) * 0.02759 = 41922.68123

SELECT count(*) FROM salaries WHERE salary <= 41922.68123;
+----------+
| count(*) |
+----------+
|   177733 |
+----------+
```

查询出的记录数：177733 与 177752.93 非常接近。

sql optimizer 在使用直方图信息时，可以使用系统变量 `optimizer_use_condition_selectivity` 来混合使用 MariaDB 引擎级别与各存储引擎级别的统计信息：

* optimizer_use_condition_selectivity=1 ：使用 MariaDB 5.5 中的预测方式。（默认值也为1）

* optimizer_use_condition_selectivity=2 ：只针对创建了索引的列使用判断选择度。

* optimizer_use_condition_selectivity=3：对所有的列使用条件判断选择度，但不使用直方图。

* optimizer_use_condition_selectivity=4：对所有的列使用条件判断选择度，同时使用直方图。

* optimizer_use_condition_selectivity=5：对所有的列使用直方图判断选择度，对不能进行索引范围扫描的列采用记录采样以判断选择度。

更多信息请参考：
https://mariadb.com/kb/en/library/histogram-based-statistics/
http://mysql.taobao.org/monthly/2016/10/09/

### 连接（JOIN）优化器选项

MariaDB 中，有两个优化器选项用于优化连接（JOIN）查询的执行计划。 MariaDB 有提供两种制定最优连接执行计划的算法：

* Exhaustive Search ：Mysql 5.0 以及以前版本使用的连接优化算法，它会针对 FROM 子句指定的所有数据表的组合计算执行计划的代价，然后从中选择最优的一个。如果数据表有10个，那么可能的连接组合为20!。因此数据表超过10个时，该算法的运行的效率非常低。

* Greedy Search (Heuristic Search)：从 Mysql 5.0 开始引入的连接优化算法。该算法的步骤包括：
	* 全部 N 个表中，通过 otimizer_search_depth 系统变量指定个数的表创建可能的连接组合。
	* 在步骤1中创建的连接组合中，选择代价最小的执行计划。
	* 将步骤2选定的执行计划的第一个数据表选作“部分执行计划”的第一个数据表。
	* 全部 N-1 个数据表中（不包含步骤3中选择的数据表），，通过 otimizer_search_depth 系统变量指定个数的表创建可能的连接组合。
	* 将步骤4中建的连接组合逐个带入步骤3中创建的“部分执行计划”，计算执行代价。
	* 从步骤5中选择一个代价最小的执行计划，从中选择第二个数据表作为步骤3中创建的“部分执行计划”的第二个数据表。
	* 重复步骤 4 ~ 6，直到没有剩余的数据表，并将数据表的连接顺序记录到“部分执行计划”。
	* 最后，“部分执行计划”被确定为数据表的连接顺序。

```
show variables like '%optimizer_search_depth%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| optimizer_search_depth | 62    |
+------------------------+-------+
```

由于 `optimizer_search_depth` 的默认值是62，即使连接的数据表较少，也会耗费相当长的时间。因此提供了用于优化性能的系统变量：

* optimizer_prune_level ：该变量决定了使用哪一种优化算法。如果指为 OFF ，则使用 Exhaustive Search 算法，否则使用 Greedy Search 算法。默认值为1。

* optimizer_search_depth ：该值仅用于 Greedy Search 算法，根据其值的不同，制定连接查询的执行计划的时间也有所不同。该值越大，所需要制定执行计划花费的时间越久，但执行计划的准确度越高。

### 示例数据的准备

#### my.cnf 配置

```
[client]
port    = 3306
socket    = /dev/shm/mysql.sock
default-character-set = utf8mb4

[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci
init-connect='SET NAMES utf8mb4'
default-time_zone = '+8:00'
user    = mysql
port    = 3306
datadir    = /data/mysql/3306/data
socket  = /dev/shm/mysql.sock
pid-file = /data/mysql/3306/data/mysql.pid
slow_query_log = 1
slow_query_log_file = /data/mysql/3306/data/slow.log
log-error = /data/mysql/3306/data/error.log
long_query_time = 0.1
log-bin = /data/mysql/3306/data/mybinlog

innodb_stats_auto_recalc=0
use_stat_tables='preferably'
innodb_stats_transient_sample_pages=8
innodb_stats_persistent_sample_pages=20
innodb_stats_persistent=1
innodb_stats_on_metadata=0
innodb_page_size=16K
innodb_file_per_table=1
innodb_file_format=Antelope
innodb_buffer_pool_size=64MB
```

#### 示例数据库的下载

https://github.com/wikibook/realmysql/tree/master/TestDatabase-Employee


需要执行: 

```
SOURCE employees.sql
SOURCE employees_schema_modification.sql
```

employees 数据库下包含的表为：

```
MariaDB [employees]> show tables;
+---------------------+
| Tables_in_employees |
+---------------------+
| departments         |
| dept_emp            |
| dept_manager        |
| employee_name       |
| employees           |
| salaries            |
| tb_dual             |
| titles              |
+---------------------+
```

依次对上述所有的表执行 ANALYZE TABLE 命令，以生成对应的统计信息。

#### optimizer_switch 系统变量

可以通过设置 optimizer_switch 系统变量来控制一系列和优化器有关的变量，这些变量会影响执行计划的不同。

```
show variables like '%optimizer_switch%'\G
*************************** 1. row ***************************
Variable_name: optimizer_switch
        Value: index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,index_merge_sort_intersection=off,engine_condition_pushdown=off,index_condition_pushdown=on,derived_merge=on,derived_with_keys=on,firstmatch=on,loosescan=on,materialization=on,in_to_exists=on,semijoin=on,partial_match_rowid_merge=on,partial_match_table_scan=on,subquery_cache=on,mrr=off,mrr_cost_based=off,mrr_sort_keys=off,outer_join_with_cache=on,semijoin_with_cache=on,join_cache_incremental=on,join_cache_hashed=on,join_cache_bka=on,optimize_join_buffer_size=off,table_elimination=on,extended_keys=on,exists_to_in=on,orderby_uses_equalities=on,condition_pushdown_for_derived=on
```

可以通过如下命令对 optimizer_switch 进行调整

```
SET GLOBAL optimizer_switch='index_merge=off';
```