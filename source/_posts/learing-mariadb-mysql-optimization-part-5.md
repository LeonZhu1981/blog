title: learing mariadb & mysql-optimization(part-5)
date: 2018-01-09 11:59:57
categories: programming
tags:
- mysql
- mariadb
- performance optimization
---

## 全表扫描
---

顾名思义，全表扫描就是在处理查询请求时不使用索引，直接遍历表中所有的数据，时间复杂度 O(n) ，MariaDB 优化器主要在下列条件下选择使用全表扫描：

* 数据表当中记录非常少，使用全表扫描比通过索引读取更快时。
* WHERE 或 ON 子句中无法使用索引时。
* 即使查询可以使用索引范围扫描，优化器判断符合条件的记录太多时。

<!--more-->

整个数据表通常比索引大得多，从头到尾读数据表需要相当多的读磁盘操作。对于 MariaDB 中的 MyISAM 引擎而言，在执行全表扫描时，无法通过变量对一次读取多少个 page 进行设置，因此会逐页进行读取。而对于 InnoDB 和 XtraDB ，连续读取特定数据表的数据页面时，后台线程就会自动开始预读（ Read ahead ）作业，即预先读取一定的数据页，后台线程可能一次读取 4 ~ 8 个页面，并且数量不断地增加，一次最大可以读取 64 个数据页，然后将其存入缓冲池。对于XtraDB，可以通过设置 innodb_read_ahead_threshold （参考 (这里)[https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_read_ahead_threshold]）来控制，在 OLAP 的场景下，可以将其值设置得更小一些，以使读取频率更高。

## Order BY 处理（ Using filesort ）
---

在处理排序时，有两种方式，分别是使用索引和使用 filesort 处理。

* 使用索引：
  * 优点：执行 INSERT/UPDATE/DELETE 查询时，由于索引已经排序，只需要一次读取即可，处理速度很快
  * 缺点：写操作时需要额外的添加或删除索引，会耗费一定的性能；索引也会占用一定的磁盘空间；索引个数越多，越需要更多的 InnoDB Buffer 或 MyISAM 的主键缓存空间。

* 使用 Filesort：
  * 优点：因为不用创建索引，因此没有上述索引的那些缺点；如果记录数比较少的情况下，直接内存当中排序会很快。
  * 缺点：如果处理的记录数太多，查询性能就会受到较大的影响。

在下面的特殊场景下，可能无法使用索引进行排序：
* 排序基准太多，无法依据每一个基准来全部创建索引。
* 要对 GROUP BY 的结果或 DISTINCT 等处理结果排序时。
* 要对临时表的结果 （比如 UNION 的结果集）重新排序时。
* 需要随机获取结果记录时。

正如我们在前面章节中学习到的，Extra 列中出现了 Using filesort 后，就可以判断出是否在内存中进行额外的排序处理操作。下面我们来介绍一些关于排序的原理的介绍。

### 排序缓冲（ sort buffer ）

执行排序操作时，MariaDB 会另外分配内存空间进行排序处理，这个内存空间被称为 “排序缓冲” （ sort buffer ）。只有排序需要时才开辟缓冲空间，其大小随着要排序的记录大小而变化，它的最大值由 innodb_sort_buffer_size 系统变量设置（ Mysql 5.7 的默认值为 1MB）。在排序查询完成了之后，用于排序的内存空间会立即返还给系统。

前面已经提到了，如果排序的记录数非常少，只用分配的排序缓冲就能完成排序，那对性能不会有什么影响。但是如果排序的记录数一但大于分配的排序缓冲空间的大小，此时 MariaDB 需要将排序记录划分为许多片段，并将其临时保存到磁盘。

具体解释来说，就是先取一批数据放入内存缓冲区中，再将排序的结果数据临时存储到磁盘上，再获取另外一批数据，重复上述步骤直至所有记录都完成了排序，最后再合并排好序的记录。上述过程中合并过程被称为“多路合并” （ Multi-merge ），多路合并的执行次数由 Sort-merge-passes 状态变量进行累计，可以通过 SHOW STATUS VARIABLES; 进行查看。

根据基准测试的结果，不是排序缓冲的设置越大，其性能就越好，一般设置在 256KB ~ 512KB之间时，能获得最优的性能。MariaDB 使用的内存分为全局内存与会话内存区域，用于排序缓冲的内存区域属于会话级别的内存区域。这意味着每一个 session 就会开辟一个 sort buffer，因此，如果我们毫无根据的设置较大的内存空间，可能就会造成服务器没有足够的内存空间，被内核 OOM-Killer 强制终止 MariaDB 进程。

### 排序算法

根据排序时是将所有记录放入排序缓冲，还是只将排序列放入排序缓冲，我们可以将排序算法分为两种：

* 一次扫描算法（ Single pass ）：
该算法将查询（ SELECT ）到的所有列（含排序列）放入排序缓冲，完成排序后直接输出结果。

```
explain select emp_no, first_name, last_name from employees order by first_name;
```

上述查询会读取 employees 表的排序列和非排序列全部读入排序缓冲中，进行排序处理，最后直接输出排序缓冲中的所有内容。

![single-pass-sort](http://static.zhuxiaodong.net/blog/static/images/single-pass-sort.png)

* 两遍扫描算法（ Two pass ）

排序时，只将排序列与主键值放入排序缓冲进行排序处理，然后再依据排序顺序使用主键读取数据表，最终获取要查询的数据列。

**Single pass VS Two pass**：

Single Pass 一次扫描算法虽然在性能上理论上会更高效一些，但是由于需要将所有的查询列放入到 sort buffer 中，因此会占用更多的存储空间。假设 sort_buffer 为 128 KB，使用两遍扫描算法可以对 7000 多条记录进行排序，而使用一次扫描算法大约只能对 3500 条记录进行排序。

MariaDB 5.x 版本中，一般使用使用 Single Pass 算法排序。但也有例外情况：
* 记录大小比 max_length_for_sort_data 的全局变量的值大时。
* 查询字段中，包含 BLOB 或 TEXT 类型的数据列时。（这个很好理解，因为 BLOB 或 TEXT 这种大字段会占用很多空间，sort_buffer 的空间有限）

如果排序的记录大小或条数较小时，使用一次性扫描算法速度会更快；反之，使用两遍扫描算法会更高效。

此外需要注意的是，使用 SELECT * 获取所有列，会让排序缓冲的效率降低（这也不难理解，如果 MariaDB 采用 Single Pass 进行排序，需要将所有的字段都放入到 sort_buffer 中，sort_buffer 的容量有限，更多的列就会导致 Multi-merge 的次数越多）。

### 排序处理方式

处理查询中的 ORDER BY 子句时，必定使用以下三种方式之一进行排序处理。

| 排序处理方法 | 执行计划的 Extra 列     | 性能 |
| :------------- | :------------- | :------------- |
| 使用索引排序       | 无       |  快  |
| 只对驱动表进行排序（包括无 JOIN 连接的场景） | Using filesort      |  一般  |
| 将连接结果保存到临时表后再进行排序 | Using temporary; Using filesort      |  慢  |


#### 使用索引排序

如果需要使用索引排序，必须满足如下几个条件：

* ORDER BY 子句中的数据列必须属于最先读取的数据表（即 JOIN 时的驱动表），并且必须由依据 ORDER BY 顺序生成的索引。
* WHERE 条件含有第一个读取的数据表的列，则该条件与 ORDER BY 应当可以使用相同的索引。
* 必须是 B-Tree 索引，Hash 索引或全文索引不能使用索引排序。
* 多表关联查询时，只能是嵌套循环（ Nested-loop ）方式的连接，才能够使用索引排序。

```
explain select * from employees e inner join salaries s on e.emp_no = s.emp_no where e.emp_no between 100002 and 100020 order by e.emp_no;

explain select * from employees e inner join salaries s on e.emp_no = s.emp_no where e.emp_no between 100002 and 100020;

+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+-------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref                | rows | Extra       |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+-------------+
|    1 | SIMPLE      | e     | range | PRIMARY       | PRIMARY | 4       | NULL               |   19 | Using where |
|    1 | SIMPLE      | s     | ref   | PRIMARY       | PRIMARY | 4       | employees.e.emp_no |    9 |             |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+-------------+
```

上面的两个 sql 语句，无论是否使用 order by e.emp_no 进行排序，都会得到相同的执行计划。原因在于，首先读取 employees 表的 emp_no 主键时，已经进行排序。

#### 仅对驱动表进行排序

一般情况下，执行连接操作前，首先对第一个数据表的记录进行排序，然后再进行连接，这样会比对整个连接结果进行排序更有效率。

```
explain select * from employees e inner join salaries s on e.emp_no = s.emp_no where e.emp_no between 100001 and 100010 order by e.last_name;
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+-----------------------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref                | rows | Extra                       |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+-----------------------------+
|    1 | SIMPLE      | e     | range | PRIMARY       | PRIMARY | 4       | NULL               |   10 | Using where; Using filesort |
|    1 | SIMPLE      | s     | ref   | PRIMARY       | PRIMARY | 4       | employees.e.emp_no |    9 |                             |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+-----------------------------+
```

上面的查询中，由于排序的字段 e.last_name 与驱动表 employees 的主键毫无关联，因此无法使用索引排序。执行计划当中的 `Using filesort` 也说明了这一点。这个查询的内部使用了 sort_buffer 来进行排序，具体过程如下：

* 使用 emp_no 索引 range 查询出了 100001 与 100010 条件的9条记录。
* 使用 last_name 列对检索结果进行了排序（ Filesort ）。
* 依次读取排好序的结果，与 Salaries 表进行连接，最终获得86条记录。

![sort_buffer](http://static.zhuxiaodong.net/blog/static/images/sort_buffer.png)

#### 使用临时表排序

下面的这个查询语句，排序就使用到了临时表，原因是由于排序的基准列不在驱动表中，而是在被驱动表中。也就是说，排序前一定要读取 salaries 数据表，所以只能对连接后的数据进行排序。

```
explain select * from employees e inner join salaries s on e.emp_no = s.emp_no where e.emp_no between 100002 and 100010 order by s.salary;
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+----------------------------------------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref                | rows | Extra                                        |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+----------------------------------------------+
|    1 | SIMPLE      | e     | range | PRIMARY       | PRIMARY | 4       | NULL               |    9 | Using where; Using temporary; Using filesort |
|    1 | SIMPLE      | s     | ref   | PRIMARY       | PRIMARY | 4       | employees.e.emp_no |    9 |                                              |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+----------------------------------------------+
```

#### 排序方式的性能比较

由于在真实的业务场景中，很多查询都会带有 ORDER BY 与 LIMIT 子句，不管你的 WHERE 条件索引优化的有多好，如果 ORDER BY 或 GROUP BY 使用不当，都会让查询变得缓慢。

我们先从两种查询处理方法：流处理与缓冲处理，来理解其中的原因。

##### 流处理方式

在流处理方式下，无论服务器端有多少条数据要处理，每检索到1条记录就逻辑发送给客户端。使用该方式时，客户端会马上得到符合查询要求的第一条记录。采用这种方式，能够保障 MariaDB 服务器的快速响应时间。

在流处理方式下，使用 LIMIT 等限制结果条数的条件可以帮助大大减少整个查询的执行时间。这是因为获取的记录数的减少，能够大大减少获取最后一条记录前的时间间隔。

根据不同的 MariaDB 客户端或驱动程序的不同，流处理方式也有所不同。例如 Java JDBC 库，MariaDB 服务器虽然按照流处理的方式，在获取到第一条记录的时候，便立即向客户端发送结果，但是 JDBC 会将记录存储到内存缓冲中，一直等待接收到最后一条记录时，才将缓冲中的所有结果返回给客户端应用程序。JDBC 采用非会话方式交换数据，能够将不必要的网络请求将至最低， 采用这种方式能够显著地提高吞吐量。

##### 缓冲处理方式

处理带有 ORDER BY 或 GROUP BY 子句的查询时，由于先要获取符合 WHERE 条件的所有记录，再进行排序或分组，然后依次发送处理结果，所以无法对查询的结果进行流处理。 这种方式被称之为缓冲处理，采用这种方式时，客户端什么也不能做，只能耐心等待，响应速度较慢。

在缓冲处理的方式下，由于所有记录都需要在 MariaDB 服务器进行集中排序处理，再排序处理完成了之后才能够进行 LIMIT 等额外操作，因此 LIMIT 语句限制返回条数，对性能的影响不会太大，当然能够减少一部分网络请求的大小。

```
select * from tb_test1 t1 inner join tb_test2 t2 on t1.col1 = t2.col1 order by t1.col2 limit 10;
```

假设 tb_test1 数据表的记录为100条，tb_test2 数据表的记录为1000条（ tb_test1 的1条记录对应 tb_test2 的10条记录 ），两个数据表的连接结果总共1000条。下面针对不同排序处理方式下的读取和排序的记录数进行比较：

* tb_test1 为驱动表

| 排序方式     | 要读取的记录数     | 访问次数     | 要排序的记录数     |
| :------------- | :------------- | :------------- | :------------- |
| 使用索引       | tb_test1: 1条<br/>tb_test2:10条       | 1次       | 0条       |
| 只对驱动表排序       | tb_test1: 100条<br/>tb_test2:10条       | 10次       | 100条       |
| 使用临时表排序       | tb_test1: 100条<br/>tb_test2:1000条       | 100次       | 1000条       |

* tb_test2 为驱动表

| 排序方式     | 要读取的记录数     | 访问次数     | 要排序的记录数     |
| :------------- | :------------- | :------------- | :------------- |
| 使用索引       | tb_test2: 10条<br/>tb_test1:10条       | 10次       | 0条       |
| 只对驱动表排序       | tb_test2: 1000条<br/>tb_test1:10条       | 10次       | 1000条       |
| 使用临时表排序       | tb_test2: 1000条<br/>tb_test1:100条       | 1000次       | 1000条       |

### ORDER BY .. LIMIT n 优化

```
select emp_no, first_name, last_name from employees where emp_no between 10001 and 10900 order by last_name limit 10;
```

上述查询中，需要排序的记录数为900条，但查询最终只返回10条记录。在这种排序记录很多而最终返回的记录较少时，Mysql 5.6 会在排序缓冲中创建优先级队列（ Priority Queue ），然后利用队列排序。

只要返回的记录的占用字节数小于 sort_buffer 的大小，就能够在

### 与排序相关的状态变量

进行重要处理时，MariaDB 服务器会将相关处理的执行次数存储到相关的状态变量中。我们可以通过如下的命令查看当前已排序记录数和排序缓冲间的合并处理次数。

```
select first_name, last_name from employees group by first_name, last_name;

show session status like 'sort%';
+---------------------------+--------+
| Variable_name             | Value  |
+---------------------------+--------+
| Sort_merge_passes         | 5      |
| Sort_priority_queue_sorts | 0      |
| Sort_range                | 0      |
| Sort_rows                 | 279408 |
| Sort_scan                 | 1      |
+---------------------------+--------+
```

* Sort_merge_passes ：表示多次合并的处理次数。
* Sort_range ：表示通过索引范围扫描检索的结果进行排序的次数。
* Sort_scan ：表示通过全表扫描检索的结果进行排序的次数。Sort_range 与 Sort_scan 均为排序处理次数的累积值。
* Sort_rows ：表示目前为止已排序的全部记录数。

## GROUP BY 处理

### 使用索引扫描处理 GROUP BY （紧凑索引扫描）

使用连接驱动表中的数据列进行分组时，若 GROUP BY 的参考列（分组依据的列）已经创建索引，同时进行分组处理，然后再对分组结果进行连接处理。即使采用索引处理 GROUP BY ，有时也需要临时表处理聚合函数等的分组值。当使用这种方式时，Extra 列不会另外显示任何信息。

### 使用松散索引扫描处理 GROUP BY

在松散索引扫描方式下，不需要连续扫描索引中的每一个元祖，扫描时仅仅考虑索引中的一部分。

```
explain select emp_no from salaries where from_date='1985-03-01' group by emp_no;
+------+-------------+----------+-------+---------------+---------+---------+------+--------+---------------------------------------+
| id   | select_type | table    | type  | possible_keys | key     | key_len | ref  | rows   | Extra                                 |
+------+-------------+----------+-------+---------------+---------+---------+------+--------+---------------------------------------+
|    1 | SIMPLE      | salaries | range | NULL          | PRIMARY | 7       | NULL | 316006 | Using where; Using index for group-by |
+------+-------------+----------+-------+---------------+---------+---------+------+--------+---------------------------------------+
```

上述查询中的 Extra 列显示为 `Using index for group-by` ，其处理步骤为：

* 依次扫描 emp_no + from_date 索引，找出 emp_no 的首个唯一值 “10001”；
* emp_no + from_date 索引中，从 emp_no 为 10001 记录只获取 from_date 值为 "1985-03-01" 的记录。该检索方法将步骤1找到的“10001”值与查询的 WHERE 子句中使用的 “from-date='1985-03-01'” 的条件进行合并。
* emp_no + from_date 索引中，获取 emp_no 的下一个主键。
* 重复执行上述步骤直到无更多结果返回。

需要注意的是，MariaDB 只在对单一数据表进行 GROUP BY 处理时才能使用松散索引扫描的方式，且前缀索引无法使用松散索引扫描方式。

假设 tb_test 表的索引为（ col1 + col2 + col3 ），下面的查询可以使用松散索引扫描：

```
select col1, col2 from tb_test group by col1, col2;
select distinct col1, col2 from tb_test;
select col1, min(col2) from tb_test group by col1;
select col1, col2 from tb_test where col3 = const group by col1, col2;
```

下面的查询无法使用松散索引扫描：

```
-- 除聚合函数（ min(), max() ）之外，其它都无法使用松散索引扫描。
select col1, sum(col2) from tb_test group by col1;

-- 因为 group by 中的数据列无法满足最左索引前缀的匹配规则
select col1, col2 from tb_test group by col2, col3;

-- 因为 select 数据列与 group by 中的不一致
select col1, col3 from tb_test group by col1, col2;
```

### 使用临时表处理 GROUP BY

无论 GROUP BY 的参考列属于驱动表还是被驱动表，只要无法使用索引，就会使用临时表处理 GROUP BY。

```
explain select e.last_name, avg(s.salary) from employees e inner join salaries s group by e.last_name;
+------+-------------+-------+-------+---------------+-----------+---------+------+---------+-------------------------------------------------+
| id   | select_type | table | type  | possible_keys | key       | key_len | ref  | rows    | Extra                                           |
+------+-------------+-------+-------+---------------+-----------+---------+------+---------+-------------------------------------------------+
|    1 | SIMPLE      | e     | ALL   | NULL          | NULL      | NULL    | NULL |  300024 | Using temporary; Using filesort                 |
|    1 | SIMPLE      | s     | index | NULL          | ix_salary | 4       | NULL | 2844047 | Using index; Using join buffer (flat, BNL join) |
+------+-------------+-------+-------+---------------+-----------+---------+------+---------+-------------------------------------------------+
```

上述查询 sql 的处理过程为：
* 使用全表扫描方式读取 employees 表；
* 使用步骤1中读取的 employees 表的 emp_no 值检索 salaries 表；
* 将步骤2中获取的连接结果记录保存到临时表。 此时临时表存储这原查询中 GROUP BY 子句的数据列与 SELECT 的数据列。临时表中使用 GROUP BY 子句中的数据列创建唯一键，即使用临时表处理 GROUP BY 时，所用的临时表总是拥有唯一键；
* 重复执行步骤1到步骤3，直到连接完成，之后依据临时表的唯一键顺序读取记录并返回给客户端。其中，如果 ORDER BY 子句的数据列与 GROUP BY 子句中的数据列一样，则不需要额外的排序处理；否则要经过 Filesort 过程，再次进行排序处理。

## DISTINCT 处理

### SELECT DISTINCT

DISTINCT 与 GROUP BY 的作用类似，不同的地方在于， DISTINCT 无法保证排序顺序。

### DISTINCT 用于集合函数内部

DISTINCT 关键字也可以用于 COUNT()、MIN()、MAX() 这类集合函数的内部，与 SELECT DISTINCT 不同的是，SELECT DISTINCT 是作用整个行，而集合内函数内的 DISTINCT 只作用于以函数参数形式传递的数据列，即让参数数据列的值唯一。

```
explain select count(distinct s.salary) from employees e inner join salaries s on e.emp_no = s.emp_no where e.emp_no between 100001 and 100100;
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+--------------------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref                | rows | Extra                    |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+--------------------------+
|    1 | SIMPLE      | e     | range | PRIMARY       | PRIMARY | 4       | NULL               |  100 | Using where; Using index |
|    1 | SIMPLE      | s     | ref   | PRIMARY       | PRIMARY | 4       | employees.e.emp_no |    9 |                          |
+------+-------------+-------+-------+---------------+---------+---------+--------------------+------+--------------------------+
```

## 临时表（ Using temporary ）

对从存储引擎接收到的记录行排序或分组时， MariaDB 引擎会使用内部临时表（ 与 CREATE TEMPORARY TABLE 语句创建的临时表进行区分，这里称之为内部临时表 ）。 通常的情况下，内部临时表先在内存中创建，然后随着表记录的增大而转移到磁盘上创建。临时表使用内存存储时会使用 MEMORY 存储引擎，保存到磁盘时会使用 Aria 存储引擎。

与 MariaDB 不用的是，Mysql 和 Percona Server 需要将内部临时表存储到磁盘时，会使用 MyISAM 存储引擎。因此，在 Mysql 当中如果有大量内部临时表，需要对 MyISAM 存储引擎进行优化。

### 需要使用临时表的查询

在下列的场景下，MariaDB 引擎处理时会创建内部临时表：
* 查询中 ORDER BY 与 GROUP BY 中的数据列不同。
* 查询中 ORDER BY 或 GROUP BY 中的数据列不在第一个连接的数据表中。
* 查询中同时有 DISTINCT 与 ORDER BY 时，或无法使用索引处理 DISTINCT 时。
* 查询使用 UNION 或 UNION DISTINCT 时，即 select_type 列为 UNION RESULT 时。
* 查询使用 UNION ALL 时。
* 执行计划中，select_type 列为 DERIVED 时。

根据执行计划中的 Extra 列是否显示 Using temporary 信息，可以判断处理查询时是否使用临时表。但是某些情况下，使用临时表时执行计划当中也不会显示 Using temporary ，例如上述的最后三种查询。

上述的1, 2, 3, 4创建的临时表会创建拥有唯一索引的内部临时表；而5, 6查询则不会创建唯一索引的临时表。前者由于拥有唯一索引，因此会高效很多。

### 在磁盘上创建临时表（ 使用 Aria 存储引擎 ）@MariaDB

通常的情况下，内部临时表通常在内存中创建的，但下列的情况下，临时表由磁盘上使用 Aria 存储引擎的数据表创建而成。

* 查询返回的列当中包含 BLOB 或 TEXT 类型的大容量的数据列时。
* 存储在临时表中的记录大小，或从 UNION、UNION ALL 中查询到列中存在长度大于 512 字节时。
* GROUP BY 或 DISTINCT 数据列中含有大于512字节时。
* 存储在临时表中的数据列的大小比 tmp_table_size 或 max_heap_table_size 系统设置值要大时。

上述1, 2, 3这三种情况，会直接使用 Aria 存储引擎在磁盘上创建内部临时表。对于第四种情况，最开始会使用 MEMORY 存储引擎在内存中创建内部临时表，当临时表的大小超过 tmp_table_size 或 max_heap_table_size 的值时，才会使用 Aria 存储引擎在磁盘上创建内部临时表。

### 临时表有关的状态变量

```
show session status like 'Created_tmp%tables';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 2     |
| Created_tmp_tables      | 6     |
+-------------------------+-------+

select first_name, last_name from employees group by first_name, last_name;

show session status like 'Created_tmp%tables';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 3     |
| Created_tmp_tables      | 8     |
+-------------------------+-------+
```

* Created_tmp_tables ：表示处理查询时内部临时表创建数量，这个值是一个累加值，包括了内存当中创建的临时表和磁盘上创建的临时表。
* Created_tmp_disk_tables ：同上，但是只包含磁盘上创建的临时表数量。

可以看到，再执行了一次查询语句之后， Created_tmp_disk_tables 由 2 变成了 3， Created_tmp_tables 由 6 变成了 8。因此可以得出结论的是，`select first_name, last_name from employees group by first_name, last_name;` 查询语句分别在内存和磁盘上创建了一次临时表。

### 带索引的内部临时表

```
explain select * from (select dept_name from departments group by dept_name) x inner join (select dept_name from departments group by dept_name) y on x.dept_name = y.dept_name;
+------+-------------+-------------+-------+---------------+-------------+---------+-------------+------+-------------+
| id   | select_type | table       | type  | possible_keys | key         | key_len | ref         | rows | Extra       |
+------+-------------+-------------+-------+---------------+-------------+---------+-------------+------+-------------+
|    1 | PRIMARY     | <derived2>  | ALL   | NULL          | NULL        | NULL    | NULL        |    9 |             |
|    1 | PRIMARY     | <derived3>  | ref   | key0          | key0        | 162     | x.dept_name |    2 |             |
|    3 | DERIVED     | departments | index | NULL          | ux_deptname | 162     | NULL        |    9 | Using index |
|    2 | DERIVED     | departments | index | NULL          | ux_deptname | 162     | NULL        |    9 | Using index |
+------+-------------+-------------+-------+---------------+-------------+---------+-------------+------+-------------+
```

上述执行计划中，key0 表示在 <derived3> (y) 表上创建了索引。 在 MariaDB 10.0 中，默认优化器选项设置为内部临时表自动创建索引，可以通过如下的命令进行确认是否开启。

```
set optimizer_switch='derived_with_keys=on';
```

### 内部临时表的注意事项

如果记录数不多，在内存上创建的内部临时表，不会带来太大的性能问题。但是在磁盘上创建的内部临时表（ Aria 存储引擎 ），可能会导致一些性能问题。

```
explain select * from employees group by last_name order by first_name;
+------+-------------+-----------+------+---------------+------+---------+------+--------+---------------------------------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra                           |
+------+-------------+-----------+------+---------------+------+---------+------+--------+---------------------------------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 300024 | Using temporary; Using filesort |
+------+-------------+-----------+------+---------------+------+---------+------+--------+---------------------------------+
```

上述的查询中， GROUP BY 与 ORDER BY 中的数据列是不同的。由于 last_name 数据列没有索引，因此在查询时需要使用临时表进行分组和排序。 其详细的处理步骤如下：

* 创建临时表（ MEMORY 存储引擎 ），用于存储 employees 表的所有列。
* 从 InnoDB 当中读取 employees 表的第一条记录。
* 检查临时表中是否存在相同的 last_name。
* 若无相同的 last_name ，则 INSERT 到临时表。
* 若有相同的 last_name ，则 UPDATE 到临时表或忽略。
* 临时表大小超过指定的大小时 ，则创建 Aria 存储引擎的临时表，并将内存当中的记录写到磁盘中。
* 重复 2 ~ 6 步骤，直到 employees 表中再无数据 （本次查询需要循环执行 30W 次）。
* 对内部临时表当中的结果进行排序处理，并最终返回给客户端。

从上述步骤中可以看出，如果在磁盘当中创建临时表，记录数较大的情况下，必然会导致性能方面的问题。因此，如果查询能够正确的使用索引，就能够避免创建临时表；在创建临时表无法避免的情况下，也需要想办法减少查询的记录数，让数据尽可能的存储在内存中；在内存中创建的临时表，也需要考虑尽可能返回减少的数据列，以避免超过 tmp_table_size 或 max_heap_table_size 系统设置值；最后，如果上述办法还不能解决问题，只能通过适当地增大： tmp_table_size 或 max_heap_table_size ，该系统变量参考[这里](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_heap_table_size)。

## 索引条件下推 （ Index Condition Pushdown ）

我们通过几个实验来说明一下索引条件下推 （ Index Condition Pushdown ）功能。

首先我们调整 optimizer_switch ，关闭 “索引条件下推” 功能：

```
set optimizer_switch='index_condition_pushdown=off';
show variables like 'optimizer_switch';
```

下面的查询中， first_name LIKE '%sal' 条件仅用作检测条件（过滤条件），因此不会传递到存储引擎层，从 Extra 列显示的 Using where 信息，表示由于部分查询条件没有传递到存储引擎层，因此存储引擎返回的数据需要在 MariaDB 层进行再次过滤，这种方式显然是非常低效的。尤其体现在这个查询中，满足 first_name LIKE '%sal' 的数据只有1条，last_name='Action' 的数据有10万条，这意味着返回给 MariaDB 层的数据有10万条，最终需要在这一层中查询出满足 first_name LIKE '%sal' 那一条数据。

```
alter table employees add index ix_lastname_firstname(last_name, first_name);
explain select * from employees where last_name='Action' AND first_name like '%sal';
+------+-------------+-----------+------+-----------------------+-----------------------+---------+-------+------+-------------+
| id   | select_type | table     | type | possible_keys         | key                   | key_len | ref   | rows | Extra       |
+------+-------------+-----------+------+-----------------------+-----------------------+---------+-------+------+-------------+
|    1 | SIMPLE      | employees | ref  | ix_lastname_firstname | ix_lastname_firstname | 66      | const |    1 | Using where |
+------+-------------+-----------+------+-----------------------+-----------------------+---------+-------+------+-------------+
```

当我们再次开启索引条件下推之后，执行计划就变成了 Using index condition 。

```
set optimizer_switch='index_condition_pushdown=on';
explain select * from employees where last_name='Action' AND first_name like '%sal';
+------+-------------+-----------+------+-----------------------+-----------------------+---------+-------+------+-----------------------+
| id   | select_type | table     | type | possible_keys         | key                   | key_len | ref   | rows | Extra                 |
+------+-------------+-----------+------+-----------------------+-----------------------+---------+-------+------+-----------------------+
|    1 | SIMPLE      | employees | ref  | ix_lastname_firstname | ix_lastname_firstname | 66      | const |    1 | Using index condition |
+------+-------------+-----------+------+-----------------------+-----------------------+---------+-------+------+-----------------------+
```

## 多范围读

传统的关系型数据库通过索引快速选择目标记录范围，而通过索引读取实际数据文件过程会发生随机 I/O ，并且是有多少条记录就发生多少次随机 I/O 。 为了减少磁盘随机 I/O ，MariaDB 引入了“多范围读”（ Multi range read， MRR ）。 MRR 算法是先读取合适数量索引集合，再按照 RowID 对该集合进行排序，最后根据排序后的 RowID 从数据文件中批量的读取数据。由于主键ID已经排好序了，因此在读取数据文件时，只需要顺序读取即可。

假设有这样一段 SQL ： `select first_name, emp_no from employees;`， employees 表的索引为：first_name， emp_no 为主键。“多范围读”的优化原理如下图所示：

![MRR](http://static.zhuxiaodong.net/blog/static/images/mrr.png)

首先通过索引读取 employees 表的 first_name 和 emp_no 主键，并复制到 MRR 缓冲中进行排序，然后根据排好序的主键顺序批量读取数据文件当中的记录。

需要注意的是，需要设置优化器选项，才能够开启 MRR 优化选项。

```
set optimizer_switch='mrr=on';
set optimizer_switch='mrr_sort_keys=on';
```

在使用“多范围读”时，执行计划会在 Extra 列显示 Rowid-ordered scan 、 Key-ordered scan 、 Key-ordered Rowid-ordered scan。

### 基于 RowId 排序（ Rowid-ordered scan ）

```
explain select * from employees where first_name>='A' and first_name<'B';
+------+-------------+-----------+-------+---------------+--------------+---------+------+-------+-------------------------------------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows  | Extra                                     |
+------+-------------+-----------+-------+---------------+--------------+---------+------+-------+-------------------------------------------+
|    1 | SIMPLE      | employees | range | ix_firstname  | ix_firstname | 58      | NULL | 41650 | Using index condition; Rowid-ordered scan |
+------+-------------+-----------+-------+---------------+--------------+---------+------+-------+-------------------------------------------+
```

上述查询语句中能够看到 Extra 列显示为 Rowid-ordered scan 。由于索引 first_name 查询条件可能需要扫描很多行（41650），为了避免过多的随机读，MRR 算法首先扫描了 ix_firstname 索引，再根据 employees 数据表的 RowId （即 emp_no 主键）顺序对结果进行排序，最后按照 RowId 顺序访问实际 employees 表数据文件。

如果我们修改一下上述 SQL ，将查询条件可能返回的记录数缩小（ WHERE first_name='Matt'，记录行数减少到233行 ），就会发现 MariaDB 不会启用 MRR 算法，原因是在数据量较小的情况下，无法发挥出 MRR 算法的优势，反而会降低性能。

```
explain select * from employees where first_name='Matt';
+------+-------------+-----------+------+---------------+--------------+---------+-------+------+-----------------------+
| id   | select_type | table     | type | possible_keys | key          | key_len | ref   | rows | Extra                 |
+------+-------------+-----------+------+---------------+--------------+---------+-------+------+-----------------------+
|    1 | SIMPLE      | employees | ref  | ix_firstname  | ix_firstname | 58      | const |  233 | Using index condition |
+------+-------------+-----------+------+---------------+--------------+---------+-------+------+-----------------------+
```

在 Mysql 5.6 版本中，只支持 Rowid-ordered scan 的 MRR 算法，不支持 Key-ordered scan 、 Key-ordered Rowid-ordered scan，并且在 Extra 列只会显示 `Using MRR` 。

### 基于 Key 排序 （ Key-ordered scan ） @MariaDB

基于 Key 排序时会先读取驱动表，再对连接列进行排序，然后根据排序读取被驱动表。被驱动表是使用 XtraDB 或 InnoDB 存储引擎的数据表且用于连接条件的数据列是被驱动表的主键是，会使用基于 Key 值的排序算法，此时执行计划会显示为： `Key-ordered scan` 。

```
set join_cache_level=8;
set optimizer_switch='mrr=on';
set optimizer_switch='mrr_sort_keys=on';
set optimizer_switch='join_cache_hashed=on';
set optimizer_switch='join_cache_bka=on';

explain select * from employees e inner join salaries s on e.emp_no=s.emp_no;
+------+-------------+-------+------+---------------+---------+---------+--------------------+--------+-------------------------------------------------------+
| id   | select_type | table | type | possible_keys | key     | key_len | ref                | rows   | Extra                                                 |
+------+-------------+-------+------+---------------+---------+---------+--------------------+--------+-------------------------------------------------------+
|    1 | SIMPLE      | e     | ALL  | PRIMARY       | NULL    | NULL    | NULL               | 300024 |                                                       |
|    1 | SIMPLE      | s     | ref  | PRIMARY       | PRIMARY | 4       | employees.e.emp_no |      9 | Using join buffer (flat, BKAH join); Key-ordered scan |
+------+-------------+-------+------+---------------+---------+---------+--------------------+--------+-------------------------------------------------------+
```

### 总结

* MRR 优化算法可用于 range 访问方法或 BKA 连接。
* 只读取少量记录时，使用 MRR 算法反而会降低处理性能。
* 执行 MRR 优化时，会采用 Rowid-ordered scan 、 Key-ordered scan 、 Key-ordered Rowid-ordered scan 之一。
* 只有开启优化器选项的对应设置，才能够启用 MRR 优化算法。

## 索引合并 （Index merged）

通常的情况下， MariaDB 只能使用一个索引，但是索引合并则可以使用多个索引，它出现在多个 WHERE 条件且每个条件只能使用不同索引，而估计满足各条件的记录数很多时。

```
explain select * from employees where first_name='Matt' and hire_date between '1995-01-01' and '2000-01-01';
+------+-------------+-----------+------+--------------------------+--------------+---------+-------+------+------------------------------------+
| id   | select_type | table     | type | possible_keys            | key          | key_len | ref   | rows | Extra                              |
+------+-------------+-----------+------+--------------------------+--------------+---------+-------+------+------------------------------------+
|    1 | SIMPLE      | employees | ref  | ix_firstname,ix_hiredate | ix_firstname | 58      | const |  233 | Using index condition; Using where |
+------+-------------+-----------+------+--------------------------+--------------+---------+-------+------+------------------------------------+
```

上述查询中， possible key 为 ix_firstname,ix_hiredate , 最终选择了 ix_firstname ，是因为 first_name='Matt' 条件的记录数会远远小于 hire_date between '1995-01-01' and '2000-01-01' 的查询条件。

如果我们修改了查询条件，将 first_name 的查询范围扩大，由于查询条件 first_name between 'A' and 'B' 大概有20000条记录，而 hire_date between '1995-01-01' and '2000-01-01' 也有30000多条记录，因此 MariaDB 采用了 index merge 算法，Extra 列当中显示了 `Using sort_intersect(ix_firstname,ix_hiredate)`

```
set optimizer_switch='index_merge_sort_intersection=on';

explain select * from employees where first_name between 'A' and 'B' and hire_date between '1995-01-01' and '2000-01-01';
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+-------------------------------------------------------------+
| id   | select_type | table     | type        | possible_keys            | key                      | key_len | ref  | rows | Extra                                                       |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+-------------------------------------------------------------+
|    1 | SIMPLE      | employees | index_merge | ix_firstname,ix_hiredate | ix_firstname,ix_hiredate | 58,3    | NULL | 9182 | Using sort_intersect(ix_firstname,ix_hiredate); Using where |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+-------------------------------------------------------------+
```

### Using union

```
explain select * from employees where first_name = 'Matt' or hire_date = '1987-03-31';
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+----------------------------------------------------+
| id   | select_type | table     | type        | possible_keys            | key                      | key_len | ref  | rows | Extra                                              |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+----------------------------------------------------+
|    1 | SIMPLE      | employees | index_merge | ix_firstname,ix_hiredate | ix_firstname,ix_hiredate | 58,3    | NULL |  344 | Using union(ix_firstname,ix_hiredate); Using where |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+----------------------------------------------------+
```

上述的查询中 WHERE 条件是由 OR 连接的。 Using union(ix_firstname,ix_hiredate) 表示 index_merge 使用 union 算法 ix_firstname 和 ix_hiredate 索引的结果集合并起来。

我们还可以发现 Using union 算法在内部进行来排除重复的处理，因为 first_name = 'Matt' 和 hire_date = '1987-03-31' 条件都会返回 emp_no = 13163 的记录， Using union 会将该记录直接删除，其使用的是优先队列的排序算法（ Priority Queue ）。

AND 和 OR 运算符不同的是，OR 运算符只要有一个条件无法使用索引，就有可能造成全表扫描。参考如下的 SQL :

```
explain select * from employees where first_name like '%Matt' or hire_date = '1987-03-31';
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | ALL  | ix_hiredate   | NULL | NULL    | NULL | 300024 | Using where |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
```

### Using sort union

```
explain select * from employees where first_name = 'Matt' or hire_date between '1987-03-01' and '1987-03-31';
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+---------------------------------------------------------+
| id   | select_type | table     | type        | possible_keys            | key                      | key_len | ref  | rows | Extra                                                   |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+---------------------------------------------------------+
|    1 | SIMPLE      | employees | index_merge | ix_firstname,ix_hiredate | ix_firstname,ix_hiredate | 58,3    | NULL | 3197 | Using sort_union(ix_firstname,ix_hiredate); Using where |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+---------------------------------------------------------+
```

上述的查询的 Extra 列显示为 Using sort_union(ix_firstname,ix_hiredate) ， 因为无法使用优先队列进行删除重复记录，只能分别从两个结果集中使用 emp_no 排序，然后再删除重复记录。

### Using intersect

多个 WHERE 条件并且使用 AND 运算符的情况下，就会使用 Using intersect 优化算法。

```
explain select * from employees where first_name = 'Matt' and hire_date = '1987-03-31';
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+--------------------------------------------------------+
| id   | select_type | table     | type        | possible_keys            | key                      | key_len | ref  | rows | Extra                                                  |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+--------------------------------------------------------+
|    1 | SIMPLE      | employees | index_merge | ix_firstname,ix_hiredate | ix_hiredate,ix_firstname | 3,58    | NULL |    1 | Using intersect(ix_hiredate,ix_firstname); Using where |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+--------------------------------------------------------+
```

### Using sort_intersect @MariaDB

与 Using intersect 不同的是，当 WHERE 条件当中出现了范围查找时，就会启用 Using sort_intersect 算法。

```
explain select * from employees where first_name like 'Matt%' and hire_date = '1987-03-31';
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+-------------------------------------------------------------+
| id   | select_type | table     | type        | possible_keys            | key                      | key_len | ref  | rows | Extra                                                       |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+-------------------------------------------------------------+
|    1 | SIMPLE      | employees | index_merge | ix_firstname,ix_hiredate | ix_hiredate,ix_firstname | 3,58    | NULL |    1 | Using sort_intersect(ix_hiredate,ix_firstname); Using where |
+------+-------------+-----------+-------------+--------------------------+--------------------------+---------+------+------+-------------------------------------------------------------+
```

## 表连接 （ JOIN ）

在 Mysql 5.5 与 MariaDB 5.2 之前的连接处理方式非常简单，仅仅只支持嵌套循环。在此之后，开始支持了多种连接算法。

### 连接类型

按照普通类型划分：

* INNER JOIN

* OUTER JOIN
  * LEFT OUTER JOIN
  * RIGHT OUTER JOIN
  * FULL OUTER JOIN

按照连接条件的显示方式划分：

* NATURAL JOIN

* CROSS JOIN
  * FULL JOIN
  * CARTESIAN JOIN

连接处理中，确定优先读取哪个数据表是相当重要的，由此引起的处理效率也大不一样。

#### JOIN （ INNER JOIN ）

内连接在 MariaDB 中一直使用嵌套循环的方式进行处理，类似于我们在编写普通程序时的双层 FOR 语句。（复杂度 O(n2) ）

```
for (record1 in table1) { // 外部循环（ OUTER ）
  for (record2 in table2) { // 内部循环（ INNER ）
    if (record1.join_column == record2.join_column) {
      // 执行找到记录的逻辑
      join_record_found(record1.*, record2.*);
    } else {
      // 执行未找到记录的逻辑
      join_record_not_found();
    }
  }
}
```

外部循环的表会比内部循环的表先读取， OUTER table 在连接中起主导作用，被称为驱动（ Driving ）表，INNER table 被称为被驱动表（ Driven ）表。嵌套循环中，若最终选择的记录由内部循环决定，则称之为内连接（ INNER JOIN ）。

#### OUTER JOIN

其执行的大概逻辑如下：

```
for (record1 in table1) { // 外部循环（ OUTER ）
  for (record2 in table2) { // 内部循环（ INNER ）
    if (record1.join_column == record2.join_column) {
      // 执行找到记录的逻辑
      join_record_found(record1.*, record2.*);
    } else {
      // 执行未找到记录的逻辑
      join_record_found(record1.*, null);
    }
  }
}
```

注意 else 当中的逻辑： `join_record_found(record1.*, null);` ，也就是说在 OUTER JOIN 中就算没有找到对应的记录，也会返回其中某个表的对应数据（具体根据是 LEFT OUTER JOIN 还是 RIGHT OUTER JOIN 来决定返回的是哪一个表的数据）。

#### 笛卡尔连接

笛卡尔连接（ cartesian join ）又被称为 FULL JOIN 或 CROSS JOIN。实际的场景使用得很少，可以参考[这里](https://coolshell.cn/articles/3463.html)。


#### NATURAL JOIN

NATURAL JOIN 表示在两个或多个表关联时，这些表中有多个字段名相同，就会直接使用这些字段来进行关联。实际业务当中不推荐使用，因为表结构发生变化之后难以维护，并且无法显式的知道有哪些字段被作为关联查询的条件。

```
-- 由于 employees 表与 salaries 表字段名相同的是 emp_no
-- 因此下面的查询也就等于: INNER JOIN ON e.emp_no = s.emp_no
select * from employees e natural join salaries s;
```

### 连接算法

* 简单嵌套循环（ Simple Nested Loop ）
* 块嵌套循环 （ Block Nested Loop ）
* 块嵌套循环散列 （ Block Nested Loop Hash ）
* 块索引 （ Block Index Join，Batched Key Access ）
* 块索引散列 （ Block Index Hash Join, Batched Key Access Hash ）

#### 连接缓存级别 （ join_cache_level ） @MariaDB

在 MariaDB 10.0 中提供了4种基于块的连接算法，每种算法又可以使用 Flat 方式 （将记录字段复制到连接缓冲）与增量（ Incremental ）方式（只将指针存储到连接缓冲中），因此，总共有下面8种算法：

(1) Block Nested Loop-Flat
(2) Block Nested Loop-Incremental
(3) Block Nested Loop Hash-Flat
(4) Block Nested Loop Hash-Incremental
(5) Batched Key Access-Flat
(6) Batched Key Access-Incremental
(7) Batched Key Access Hash-Flat
(8) Batched Key Access Hash-Incremental

有3个 optimizer_switch 选项 与 join_cache_level 系统变量共同决定了最终使用上述哪种算法：
* optimizer_switch 中的 join_cache_bka 设置为 OFF ，将会禁用上述的 5 ~ 8 算法。
* optimizer_switch 中的 join_cache_hashed 设置为 OFF ，将会禁用上述的 3 ~ 4 算法。
* optimizer_switch 中的 join_cache_incremental 设置为 OFF ，将会禁用上述的 2、4、6、8 这四种算法。
* join_cache_level 系统变量：其值的取值范围为 0 ~ 8，只有算法编号小于 join_cache_level 变量值时，其对应的连接算法才能启用。这意味着，join_cache_level = 0 时，只有 Nested Loop 基本连接算法能够使用；设置为 1 时，则可以使用 Block Nested Loop-Flat 算法，以此类推。当设置为 8 时，所有的算法才能使用。其默认值为：2，参考[这里](https://mariadb.com/kb/en/library/server-system-variables/#join_cache_level)。

此外，如果在处理 OUTER JOIN 时也想使用基于块的连接算法，需要将 `outer_join_with_cache` 优化器选项设置为 ON ，对于半连接（ SEMI-JOIN ），需要将 semijoin_with_cache 优化器选项设置为 ON 。

#### 设置连接缓冲 @MariaDB

不同的连接算法，某些不需要使用额外的内存空间，比如简单的嵌套循环连接，而某些算法则需要使用另外的内存缓冲空间以提升连接性能。

* `join_buffer_size` 系统变量： 设置最大连接缓冲的大小。参考[这里](https://mariadb.com/kb/en/library/server-system-variables/#join_buffer_size)
* `optimizer_join_buffer_size` 优化器的开关设置为 ON，才能够开启使用连接缓冲。

#### 简单嵌套循环 （ Simple Nested Loop ）

我们已经通过前面的伪代码例子简单介绍了 Simple Nested Loop ，实际上就是多层 for 循环，常常出现于使用合适的索引进行连接查询的场景，需要注意的是，Simple Nested Loop 不会使用到连接缓冲（ join_buffer_size ）。

```
explain select d.dept_name,e.first_name from departments d, employees e, dept_emp de where de.dept_no=d.dept_no and e.emp_no=de.emp_no;
+------+-------------+-------+--------+---------------------------+-------------+---------+---------------------+-------+-------------+
| id   | select_type | table | type   | possible_keys             | key         | key_len | ref                 | rows  | Extra       |
+------+-------------+-------+--------+--------å-------------------+-------------+---------+---------------------+-------+-------------+
|    1 | SIMPLE      | d     | index  | PRIMARY                   | ux_deptname | 162     | NULL                |     9 | Using index |
|    1 | SIMPLE      | de    | ref    | PRIMARY,ix_empno_fromdate | PRIMARY     | 16      | employees.d.dept_no | 36844 | Using index |
|    1 | SIMPLE      | e     | eq_ref | PRIMARY                   | PRIMARY     | 4       | employees.de.emp_no |     1 |             |
+------+-------------+-------+--------+---------------------------+-------------+---------+---------------------+-------+-------------+
```

从执行计划来看，最先读取 d 数据表（ department ），然后依次读取 de 数据表（ dept_emp ）与 e 数据表（ employees ）。在读取 de 和 e 数据表时，执行计划的 ref 列显示了使用哪些列用作比较条件。其执行的伪代码大概如下

```
for (record1 in departments) {
  for (record2 in dept_emp && record2.dept_no == record1.dept_no) {
    for (record3 in employees && record3.emp_no == record2.emp_no) {
      return {record1.dept_name, record3.first_name}
    }
  }
}
```

#### 块嵌套循环 （ Block Nested Loop ， BNL ）

与 Simple Nested Loop 不用的是， Block Nested Loop 使用了连接缓冲（ join_buffer_size ），此外，连接中以何种顺序连接驱动表与被驱动表也决定了是否启用 Block Nested Loop 。是否使用连接缓冲，体现到执行计划当中是使用：`Using Join buffer` 来进行表示。

连接时会根据驱动表中符合条件的记录数检索并处理被驱动表。也就是说，驱动表可以1次读完，而被驱动表要读多次。例如，驱动表中符合条件的记录有1000条，若被驱动表的连接条件无法使用索引，则要对被驱动表执行1000次全表扫描，查询的效率非常低效。

如果无论使用何种方式，都无法避免对被驱动表使用全表扫描或索引全扫描，则优化器会将驱动表中读取的记录缓存到内存，然后将被驱动表与内存缓存进行连接处理。此时用到的内存缓存被称为连接缓冲（ Join Buffer ），其大小由 join_buffer_size 系统变量进行设置，连接完成了之后会立即释放连接缓冲。

下面的查询中，dept_emp 数据表中满足 from_date > '2000-01-01' 的数据一共有 10616 条， employees 数据表满足 emp_no < 109004 的数据一共有 99003 条，并对齐进行笛卡尔连接。

```
select * from dept_emp de, employees e where de.from_date > '2000-01-01' and e.emp_no < 109004;
```

如果不使用连接缓冲的情况下，假设 dept_emp 为驱动表， employees 为被驱动表，其执行过程如下图所示：

![non-join-buffer](http://static.zhuxiaodong.net/blog/static/images/non-join-buffer.png)

对于 dept_emp 数据表中满足 from_date > '2000-01-01' 条件的每条记录，都要从 employees 数据表获取满足 emp_no < 109004 条件的 99003 条记录。对于 dept_emp 数据表的每条记录, 读取 employees 表虽然每次从被驱动表获取的结果是一样的，但是仍然需要执行 10616 次，这无疑是十分低效的做法。

但是如果使用了连接缓冲，情况就会有所不同。下面的 sql 的执行计划中， Extra 列显示为：`Using join buffer (flat, BNL join)`

```
explain select * from dept_emp de, employees e where de.from_date > '2000-01-01' and e.emp_no < 109004;
+------+-------------+-------+-------+---------------+-------------+---------+------+--------+-------------------------------------------------+
| id   | select_type | table | type  | possible_keys | key         | key_len | ref  | rows   | Extra
                           |
+------+-------------+-------+-------+---------------+-------------+---------+------+--------+-------------------------------------------------+
|    1 | SIMPLE      | de    | range | ix_fromdate   | ix_fromdate | 3       | NULL |  19268 | Using index condition; Rowid-ordered scan       |
|    1 | SIMPLE      | e     | range | PRIMARY       | PRIMARY     | 4       | NULL | 149711 | Using where; Using join buffer (flat, BNL join) |
+------+-------------+-------+-------+---------------+-------------+---------+------+--------+-------------------------------------------------+
```

使用连接缓冲大概会有4个步骤：
* 使用 dept_emp 数据表的 ix_fromdate 索引检索满足 from_date > '2000-01-01' 条件的记录。
* 从 dept_emp 数据表读取连接需要的其余数据列，存入连接缓冲。
* 使用 employees 数据表的主键，检索满足 emp_no < 109004 条件的记录。
* 将步骤3中检索的结果与步骤2中存入连接缓冲的记录连接后，将结果返回给用户。
* 如果首次读取的表结果太多而无法全部存入 join buffer ，则会反复执行上述 1 ~ 4 步。

![using-join-buffer](http://static.zhuxiaodong.net/blog/static/images/using-join-buffer.png)

MariaDB 5.3 开始修改了存储在连接缓冲中的记录格式，新格式主要有以下优点：

* 提升了处理可变长数据列与 NULL 值的效率：NULL 值不再使用连接缓冲空间，并且在处理类型 VARCHAR 的可变长度数据列时，不必再使用 0 填充到最大长度。

* 支持连接缓冲增量（ Incremental ）模式：连接3个以上的数据表时，会将重复的内容从连接缓冲中删除，只存储不重复的内容。假设 dept_emp、 employees、 departments 三个数据表全部使用连接缓冲进行连接。首先连接 dept_emp 与 employees 两个数据表，将结果存入连接缓冲。然后再将连接缓冲中的结果与 departments 连接，因此一共需要两个连接缓冲。第一个连接缓冲 Buffer1 用于存放 dept_emp 的记录，第二个连接缓冲 Buffer2 用于存放 dept_emp 与 employees 的部分连接结果。 在增量模式下 Buffer2 只需要存储 Buffer1 的第一个数据表记录与第二个数据表 employees 记录的指针，所以 Buffer2 的空间利用效率非常高。此时，将 Buffer2 称之为 增量连接缓冲（ Incremental join buffer ）。

#### 块嵌套循环散列 （ Block Nested Loop Hash ） @MariaDB

Hash 连接只用于相等比较条件中，比如 “tab1.fd1=tab2.fd2” 。 让我们来看一下如下的执行计划：

```
set optimizer_switch='join_cache_incremental=on';
set optimizer_switch='join_cache_hashed=on';
set optimizer_switch='mrr=on';
set optimizer_switch='mrr_sort_keys=on';
set join_cache_level=8;

explain select * from dept_emp de, departments d where d.dept_no=de.dept_no;
+------+-------------+-------+-------+---------------+-------------+---------+---------------------+-------+-------------------------------------------------------+
| id   | select_type | table | type  | possible_keys | key         | key_len | ref                 | rows  | Extra                                                 |
+------+-------------+-------+-------+---------------+-------------+---------+---------------------+-------+-------------------------------------------------------+
|    1 | SIMPLE      | d     | index | PRIMARY       | ux_deptname | 162     | NULL                |     9 | Using index                                           |
|    1 | SIMPLE      | de    | ref   | PRIMARY       | PRIMARY     | 16      | employees.d.dept_no | 36844 | Using join buffer (flat, BKAH join); Key-ordered scan |
+------+-------------+-------+-------+---------------+-------------+---------+---------------------+-------+-------------------------------------------------------+
```

散列连接大致可分为创建阶段（ Build phase ）与探测阶段（ Probe phase ）。创建阶段会使用记录较少的数据表，读取所选数据表的记录，计算连接数据列的 Hash 值，并创建的 Hash 表。探测阶段会扫描其余的数据表，同样创建连接列的 Hash 值，并在创建阶段生成的散列表中进行检索，得到最终连接结果。在散列连接中，由于使用与连接列值长度无关的连接列的散列值进行检索，所以它的处理效率会比嵌套循环连接更快的处理速度。

![hash-join](http://static.zhuxiaodong.net/blog/static/images/hash-join.png)

散列连接创建阶段生成的散列表存储到连接缓冲中（ join_buffer_size ）。若创建阶段的数据表太大，生成的散列表比连接缓冲大， MariaDB 服务器就会将创建阶段与探测极端切分为多个小的单位，并多次执行。因此，创建阶段使用的数据表记录很多，可以将连接缓冲（ join_buffer_size ）适当设置得大一些，有助于提高处理性能。

#### 块索引连接 （ Block Index Join - Batched Key Access，BKA ）

块索引连接也被称为 “批量主键访问连接” （ Batched Key Access，BKA ），内部使用 MRR 优化功能进行连接。

在“批量主键访问连接”中先读取驱动表，再将所需数据列与连接列存储到连接缓冲。连接缓冲填满时，就会将连接缓冲中的内容传给 MRR 引擎。 MRR 引擎先采用适当的方式对接收的缓冲内容进行排序，然后读取被驱动表并返回，这样批量主键访问进程即完成了连接处理。

#### 块索引散列连接 （ Block Index Hash Join， Batched Key Access Hash ） @MariaDB
块索引散列连接采用批量主键访问方式读取数据表，然后进行散列连接。其工作方式如下：
* 创建阶段： 采用 MRR 方式访问驱动表，生成散列表。
* 探测阶段： 读取被驱动表，检索散列表并返回结果。

## 子查询
---

### 半连接子查询优化

半连接子查询通常是指是指 IN() ， NOT IN () ， EXISTS ， NOT EXISTS 等方式的子查询，在 Mysql 5.5 之前的版本中，一般会将 IN 的方式手工修改成成 EXISTS 的方式来进行调优。

在 MariaDB 10.0 版本中提供了5种优化方式：
* Table pullout
* FirstMatch
* Semi-join Materialization
* LooseScan
* DuplicateWeedout

为了对这5种优化方式对子查询进行优化，需要遵守以下几项限制条件。

* 查询中使用 IN 或 = （subquery）形式的条件
* 子查询中不包含 UNION 的单一 SELECT 时
* 子查询不带有聚合函数（ SUM 或 COUNT 等 ）与 HAVING 子句
* 子查询的 WHERE 条件与外部查询的其它条件通过 AND 运算符进行连接时
* 子查询不是使用连接的 UPDATE 或 DELETE 语句时
* 不使用事先制定的执行计划时（使用 PreparedStatement 重新使用执行计划）
* 外部查询与子查询都使用实际存在的数据表时（对于 SELECT 1 虚拟表查询，无法使用半连接优化）
* 外部查询与子查询都不使用 STRAIGHT_JOIN 提示时

#### Table pullout

Table pullout 优化首先将用于半连接子查询的数据表抽取到外部查询，然后将查询改写成连接查询。

```
explain extended select * from employees e where e.emp_no IN (select de.emp_no from dept_emp de where de.dept_no='d009');
+------+-------------+-------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------------------------------------------------------+
| id   | select_type | table | type   | possible_keys             | key     | key_len | ref                 | rows  | filtered | Extra                                                 |
+------+-------------+-------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------------------------------------------------------+
|    1 | PRIMARY     | de    | ref    | PRIMARY,ix_empno_fromdate | PRIMARY | 16      | const               | 46012 |   100.00 | Using where; Using index                              |
|    1 | PRIMARY     | e     | eq_ref | PRIMARY                   | PRIMARY | 4       | employees.de.emp_no |     1 |   100.00 | Using join buffer (flat, BKAH join); Key-ordered scan |
+------+-------------+-------+--------+---------------------------+---------+---------+---------------------+-------+----------+-------------------------------------------------------+

show warnings;

select `employees`.`e`.`emp_no` AS `emp_no`,`employees`.`e`.`birth_date` AS `birth_date`,`employees`.`e`.`first_name` AS `first_name`,`employees`.`e`.`last_name` AS `last_name`,`employees`.`e`.`gender` AS `gender`,`employees`.`e`.`hire_date` AS `hire_date` from `employees`.`dept_emp` `de` join `employees`.`employees` `e` where `employees`.`e`.`emp_no` = `employees`.`de`.`emp_no` and `employees`.`de`.`dept_no` = 'd009'
```

可以看到当使用 explain extended 时查看详细的执行计划时， 上面的半连接查询已经被优化器优化成了 JOIN 的方式。

Table pullout 可以通过设置 optimizer_switch 来进行开启或关闭，即:

```
set optimizer_switch='semijoin=OFF';
```

需要注意的是： **我们无法针对某种优化算法单独设置是否开启**， 例如上述的语句会导致所有的半连接优化算法都被关闭。

#### FirstMatch

FirstMatch 是将 IN 形式优化成 EXISTS 形式。

```
explain select * from employees e where e.first_name = 'Matt' and e.emp_no in (select t.emp_no from titles t where t.from_date between '1995-01-01' and '1995-01-30');
+------+-------------+-------+------+----------------------+--------------+---------+--------------------+------+-----------------------------------------+
| id   | select_type | table | type | possible_keys        | key          | key_len | ref                | rows | Extra                                   |
+------+-------------+-------+------+----------------------+--------------+---------+--------------------+------+-----------------------------------------+
|    1 | PRIMARY     | e     | ref  | PRIMARY,ix_firstname | ix_firstname | 58      | const              |  233 | Using index condition                   |
|    1 | PRIMARY     | t     | ref  | PRIMARY              | PRIMARY      | 4       | employees.e.emp_no |    1 | Using where; Using index; FirstMatch(e) |
+------+-------------+-------+------+----------------------+--------------+---------+--------------------+------+-----------------------------------------+

select `employees`.`e`.`emp_no` AS `emp_no`,`employees`.`e`.`birth_date` AS `birth_date`,`employees`.`e`.`first_name` AS `first_name`,`employees`.`e`.`last_name` AS `last_name`,`employees`.`e`.`gender` AS `gender`,`employees`.`e`.`hire_date` AS `hire_date` from `employees`.`employees` `e` semi join (`employees`.`titles` `t`) where `employees`.`t`.`emp_no` = `employees`.`e`.`emp_no` and `employees`.`e`.`first_name` = 'Matt' and `employees`.`t`.`from_date` between '1995-01-01' and '1995-01-30'
```

上述的查询语句中，由执行计划 id 字段值都为 1 可得出并没有使用子查询的方式执行，而被优化成了 JOIN 的方式。Extra 列中显示为: `FirstMatch(e)` 。

optimizer_switch 中有2个选项会影响到:

```
set optimizer_switch='semijoin=ON';
set optimizer_switch='firstmatch=ON';
```

#### Semi-join Materialization

Semi-join Materialization 用于将半连接中使用的子查询全部具体化（ Materialization简言之就是创建内部临时表 ）。

我们编写的下面的 sql 相对于 firstmatch 的示例只是删除了 e.first_name = 'Matt' 的条件

```
explain select * from employees e where e.emp_no in (select t.emp_no from titles t where t.from_date between '1995-01-01' and '1995-01-30');
+------+--------------+-------------+--------+---------------+--------------+---------+------+--------+--------------------------+
| id   | select_type  | table       | type   | possible_keys | key          | key_len | ref  | rows   | Extra
             |
+------+--------------+-------------+--------+---------------+--------------+---------+------+--------+--------------------------+
|    1 | PRIMARY      | e           | ALL    | PRIMARY       | NULL         | NULL    | NULL | 300024 |
             |
|    1 | PRIMARY      | <subquery2> | eq_ref | distinct_key  | distinct_key | 4       | func |      1 |
             |
|    2 | MATERIALIZED | t           | index  | PRIMARY       | PRIMARY      | 209     | NULL | 442486 | Using where; Using index |
+------+--------------+-------------+--------+---------------+--------------+---------+------+--------+--------------------------+
```

其中 select_type 字段显示为了 MATERIALIZED 。这表示在执行过程中创建了临时表。首先执行了读取 titles 数据表的子查询，用执行结果创建临时表（ <subquery2> ）。然后将 employees 数据表与子查询具体化形式的临时表（ <subquery2> ）连接起来，最终形成结果返回给用户。

第二行的 key 列显示为 distinct_key 。 原因是 IN (subquery) 的查询中，子查询必须只返回唯一的值。因此，执行计划中为了保证 emp_no 列值唯一，优化器在具体化后的临时表中自动创建了唯一索引。

但是在上述的查询语句，还有进一步可以优化的可能性，我们在 titles 数据表上 from_date 创建索引了之后，再来看看执行计划。

```
ALTER TABLE titles ADD INDEX ix_fromdate(from_date);

explain select * from employees e where e.emp_no in (select t.emp_no from titles t where t.from_date between '1995-01-01' and '1995-01-30');
+------+--------------+-------------+--------+---------------------+-------------+---------+--------------------+------+--------------------------+
| id   | select_type  | table       | type   | possible_keys       | key         | key_len | ref                | rows | Extra                    |
+------+--------------+-------------+--------+---------------------+-------------+---------+--------------------+------+--------------------------+
|    1 | PRIMARY      | <subquery2> | ALL    | distinct_key        | NULL        | NULL    | NULL               | 2690 |                          |
|    1 | PRIMARY      | e           | eq_ref | PRIMARY             | PRIMARY     | 4       | employees.t.emp_no |    1 |                          |
|    2 | MATERIALIZED | t           | range  | PRIMARY,ix_fromdate | ix_fromdate | 3       | NULL               | 2690 | Using where; Using index |
+------+--------------+-------------+--------+---------------------+-------------+---------+--------------------+------+--------------------------+
```

可以看到 <subquery2> 在创建 from_date 的索引之后，type 列由 index 变为了 range 。

MATERIALIZED 优化包括 Materialization-Scan 和 Materialization-Lookup 两个策略。上述的查询示例在创建索引之前，具体化后的临时表在连接中用作被驱动表，并且使用 distinct_key 进行检索，这称之为 Materialization-Lookup 。创建索引之后，具体化后的临时表被用作驱动表，并且进行全表扫描，这称之为 Materialization-Scan 。

Semi-join Materialization 优化与其他子查询优化不同，即使子查询中有 GROUP BY 子句，也可以使用该优化策略。

```
explain select * from employees e where e.emp_no in (select t.emp_no from titles t where t.from_date between '1995-01-01' and '1995-01-30' group by t.title);
+------+--------------+-------------+--------+---------------------+-------------+---------+--------------------+------+--------------------------+
| id   | select_type  | table       | type   | possible_keys       | key         | key_len | ref                | rows | Extra                    |
+------+--------------+-------------+--------+---------------------+-------------+---------+--------------------+------+--------------------------+
|    1 | PRIMARY      | <subquery2> | ALL    | distinct_key        | NULL        | NULL    | NULL               | 2690 |                          |
|    1 | PRIMARY      | e           | eq_ref | PRIMARY             | PRIMARY     | 4       | employees.t.emp_no |
 1 |                          |
|    2 | MATERIALIZED | t           | range  | PRIMARY,ix_fromdate | ix_fromdate | 3       | NULL               | 2690 | Using where; Using index |
+------+--------------+-------------+--------+---------------------+-------------+---------+--------------------+------+--------------------------+
```

可以使用 Semi-join Materialization 优化的查询有如下限制条件与特征：
* IN(subquery) 中的子查询必须不是相关子查询。
* 即使使用 GROUP BY 或聚合函数，子查询也可以使用具体化。
* 具体化会创建内部临时表。

optimizer_switch 必须将 semijoin 选项和 Materialization 优化选项设置为 ON 才能够使用 Semi-join Materialization 。

#### LooseScan 优化

```
# 在 MariaDB 10.0 版本中，如果不关闭 Materialization 和 firstmatch，优化器就不会优先使用 LooseScan 的优化方式

set optimizer_switch='materialization=off';
set optimizer_switch='firstmatch=off';

explain select * from departments d where d.dept_no IN (select de.dept_no from dept_emp de);
+------+-------------+-------+--------+---------------+---------+---------+----------------------+--------+------------------------+
| id   | select_type | table | type   | possible_keys | key     | key_len | ref                  | rows   | Extra                  |
+------+-------------+-------+--------+---------------+---------+---------+----------------------+--------+------------------------+
|    1 | PRIMARY     | de    | index  | PRIMARY       | PRIMARY | 20      | NULL                 | 331603 | Using index; LooseScan |
|    1 | PRIMARY     | d     | eq_ref | PRIMARY       | PRIMARY | 16      | employees.de.dept_no |      1 |                        |
+------+-------------+-------+--------+---------------+---------+---------+----------------------+--------+------------------------+
```

LooseScan 优化有如下特点：
* LooseScan 优化先采用松散索引扫描读取子查询数据表，然后以外部表作为被驱动表进行连接。因此，子查询部分必须具备使用 Loose Index Scan 的条件，才能使用该优化。可以用于以下形式的查询：

```
SELECT .. FROM .. WHERE expr IN (SELECT keypart1 FROM tab WHERE ...)
SELECT .. FROM .. WHERE expr IN (SELECT keypart2 FROM tab WHERE keypart2='常量')
```

#### Duplicate Weedout

Duplicate Weedout 会将半连接子查询转化为一般的 INNER JOIN 查询执行，最后再删除重复记录。

```
set optimizer_switch='materialization=off';
set optimizer_switch='firstmatch=off';
set optimizer_switch='loosescan=off';

explain select * from employees e where e.emp_no in (select s.emp_no from salaries s where s.salary>150000);
+------+-------------+-------+--------+-------------------+-----------+---------+--------------------+------+-------------------------------------------+
| id   | select_type | table | type   | possible_keys     | key       | key_len | ref                | rows | Extra                                     |
+------+-------------+-------+--------+-------------------+-----------+---------+--------------------+------+-------------------------------------------+
|    1 | PRIMARY     | s     | range  | PRIMARY,ix_salary | ix_salary | 4       | NULL               |   36 | Using where; Using index; Start temporary |
|    1 | PRIMARY     | e     | eq_ref | PRIMARY           | PRIMARY   | 4       | employees.s.emp_no |    1 | End temporary                             |
+------+-------------+-------+--------+-------------------+-----------+---------+--------------------+------+-------------------------------------------+
```

使用 Duplicate Weedout 优化算法处理查询时，会像上面那样将原查询转换为 INNER JOIN + GROUP BY 子句的形式。处理过程如下：
* 扫描 salaries 数据表的 ix_salary 索引，检索 salary 大于 1500000 的员工，与 employees 表进行连接。
* 将结果存储至临时表中。
* 从保存在临时表的结果删除 emp_no 重复的记录。
* 删除重复记录后，返回剩余记录。

我们可以看到 Extra 列显示了 Start temporary 和 End temporary ，这意味着优化器使用了 Duplicate Weedout 优化过程。

#### 非半连接的子查询优化

非半连接子查询的典型示例包括：

```
-- 使用 IN(subquery) 与 OR 运算符连接不同条件
SELECT ...
FROM ...
WHERE expr1 [NOT] IN (SELECT ...) OR expr2;

-- NOT IN(subquery) 形式
SELECT ...
FROM ...
WHERE expr1 NOT IN (SELECT ...);

-- SELECT 语句含有子查询
SELECT ..., (SELECT ...)
FROM ... WHERE ...

-- HAVING 语句含有子查询
SELECT ...
FROM ... WHERE ...
HAVING (SELECT ...);

-- 子查询中使用 UNION 时
SELECT ...
FROM ...
WHERE expr IN (SELECT ... UNION SELECT ...);
```

这些场景下，只能使用两种优化方式： Materialization 和 In-to-EXISTS 优化。

#### 子查询缓存

如果要根据外部查询执行结果中的记录条数反复执行相关子查询时，就会使用子查询缓存优化算法。

```
explain extended select * from employees e where e.emp_no in (select de.emp_no from dept_emp de where de.dept_no='d009') or e.first_name='Matt';
+------+--------------------+-------+--------+---------------------------------------+---------+---------+------------+--------+----------+--------------------------+
| id   | select_type        | table | type   | possible_keys                         | key     | key_len | ref        | rows   | filtered | Extra                    |
+------+--------------------+-------+--------+---------------------------------------+---------+---------+------------+--------+----------+--------------------------+
|    1 | PRIMARY            | e     | ALL    | ix_firstname                          | NULL    | NULL    | NULL       | 300024 |   100.00 | Using where              |
|    2 | DEPENDENT SUBQUERY | de    | eq_ref | PRIMARY,ix_fromdate,ix_empno_fromdate | PRIMARY | 20      | const,func |      1 |   100.00 | Using where; Using index |
+------+--------------------+-------+--------+---------------------------------------+---------+---------+------------+--------+----------+--------------------------+

show warnings;

select `employees`.`e`.`emp_no` AS `emp_no`,`employees`.`e`.`birth_date` AS `birth_date`,`employees`.`e`.`first_name` AS `first_name`,`employees`.`e`.`last_name` AS `last_name`,`employees`.`e`.`gender` AS `gender`,`employees`.`e`.`hire_date` AS `hire_date` from `employees`.`employees` `e` where <expr_cache><`employees`.`e`.`emp_no`>(<in_optimizer>(`employees`.`e`.`emp_no`,<exists>(select `employees`.`de`.`emp_no` from `employees`.`dept_emp` `de` where `employees`.`de`.`dept_no` = 'd009' and <cache>(`employees`.`e`.`emp_no`) = `employees`.`de`.`emp_no`))) or `employees`.`e`.`first_name` = 'Matt'
```

可以在执行 `show warnings` 之后看到， IN(subquery) 转换为了 EXISTS(subquery) 的形式，此外 <expr_cache> 表示优化器将子查询的执行结果放入另外的内存缓存中，要执行相同的子查询时，不用再次执行子查询，直接返回缓存中的内容即可。

子查询缓存与参数值、子查询的结果一起保存到内部临时表中，并将保存参数的数据列创建唯一索引。最初会在内存当中创建临时表，但如果比 tmp_table_size 或 max_heap_table_size 设定的值大时，就需要计算缓存命中率，以决定如何维护缓存。

* HitRatio（缓存命中率）比较低时 ( HitRatio < 0.2 )：自动禁用子查询缓存
* HitRatio（缓存命中率）合理时 ( 0. 2<= HitRatio < 0.7 )：将内部临时表中缓存至今的子查询结果全部删除，重新将子查询结果缓存到内部临时表。
* HitRatio（缓存命中率）高时 ( HitRatio >= 0.7 )：将保存在子查询缓存的内部临时表从内存转移到磁盘，继续缓存子查询结果。
