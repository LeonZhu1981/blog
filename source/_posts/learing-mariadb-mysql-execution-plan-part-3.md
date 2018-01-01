title: learing mariadb & mysql-execution plan(part-3)
date: 2017-11-30 17:11:32
categories: programming
tags:
- mysql
- mariadb
- execution plan
---

## type 列
---

type 列显示了以哪种方式读取数据表（ **Access Type** ）的记录。作为执行计划当中判断索引是否高效使用的依据，它的重要程度不言而喻。

type 列一共包含了如下的 12 个值：（**注意：访问的效率是由最快至最慢，即 system 最快，其次是 const，最后是 ALL**）

* system
* const
* eq_ref
* ref
* fulltext
* ref_or_null
* unique_subquery
* index_subquery
* range
* index_merge
* index
* ALL

除了 ALL 之外，其它所有的访问方式都会使用索引。 此外，除了 index_merge 外，其它的访问方式只能使用一个索引。

<!--more-->

### system

访问仅有1条记录的数据表或者没有记录的空表时，type 会为 system 。**该访问方法只适用于 MyISAM 或 MEMORY 数据表**。

参考如下的实验:

`tb_dual` 表是 MyISAM 的存储引擎时：

```
select count(*) from tb_dual;
+----------+
| count(*) |
+----------+
|        1 |
+----------+

EXPLAIN SELECT * FROM tb_dual;
+------+-------------+---------+--------+---------------+------+---------+------+------+-------+
| id   | select_type | table   | type   | possible_keys | key  | key_len | ref  | rows | Extra |
+------+-------------+---------+--------+---------------+------+---------+------+------+-------+
|    1 | SIMPLE      | tb_dual | system | NULL          | NULL | NULL    | NULL |    1 |       |
+------+-------------+---------+--------+---------------+------+---------+------+------+-------+
```

将 `tb_dual` 修改为 InnoDB 存储引擎时：

```
ALTER TABLE tb_dual ENGINE = InnoDB;

EXPLAIN SELECT * FROM tb_dual;
+------+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
| id   | select_type | table   | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+------+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
|    1 | SIMPLE      | tb_dual | index | NULL          | PRIMARY | 1       | NULL |    1 | Using index |
+------+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
```

### const

在不受数据表记录条数限制的前提下，查询中含有使用主键或唯一键列的 WHERE 条件子句时，通常采用 const 方式处理这种查询，查询一定回返回1行记录。 该查询方式在其它 RDB 当中也被称之为 唯一索引扫描（ unique index scan ）。

```
explain select * from employees where emp_no=10001;
+------+-------------+-----------+-------+---------------+---------+---------+-------+------+-------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref   | rows | Extra |
+------+-------------+-----------+-------+---------------+---------+---------+-------+------+-------+
|    1 | SIMPLE      | employees | const | PRIMARY       | PRIMARY | 4       | const |    1 |       |
+------+-------------+-----------+-------+---------------+---------+---------+-------+------+-------+
```

当一个表中，主键由多个列组成，如果只使用索引的部分作为列条件时，这时是无法使用 const 类型的访问方法。

例如 dept_emp 表中的主键是由 dept_no 和 emp_no 两个字段构成的，当只使用 dept_no 作为查询条件时，访问方式将变为 ref 而非 const 。

```
show index from dept_emp;
+----------+------------+-------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table    | Non_unique | Key_name          | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+----------+------------+-------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| dept_emp |          0 | PRIMARY           |            1 | dept_no     | A         |          16 |     NULL | NULL   |      | BTREE      |         |               |
| dept_emp |          0 | PRIMARY           |            2 | emp_no      | A         |      331143 |     NULL | NULL   |      | BTREE      |         |               |
| dept_emp |          1 | ix_fromdate       |            1 | from_date   | A         |       11418 |     NULL | NULL   |      | BTREE      |         |               |
| dept_emp |          1 | ix_empno_fromdate |            1 | emp_no      | A         |      331143 |     NULL | NULL   |      | BTREE      |         |               |
| dept_emp |          1 | ix_empno_fromdate |            2 | from_date   | A         |      331143 |     NULL | NULL   |      | BTREE      |         |               |
+----------+------------+-------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
```

```
explain select * from dept_emp where dept_no='d005';
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
| id   | select_type | table    | type | possible_keys | key     | key_len | ref   | rows   | Extra       |
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
|    1 | SIMPLE      | dept_emp | ref  | PRIMARY       | PRIMARY | 16      | const | 165571 | Using where |
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
```

但是如果将主键当中所有的列都作为 WHERE 条件去查询时，则会使用 const 访问方式。

```
explain select * from dept_emp where dept_no='d005' AND emp_no=10001;
+------+-------------+----------+-------+---------------------------+---------+---------+-------------+------+-------+
| id   | select_type | table    | type  | possible_keys             | key     | key_len | ref         | rows | Extra |
+------+-------------+----------+-------+---------------------------+---------+---------+-------------+------+-------+
|    1 | SIMPLE      | dept_emp | const | PRIMARY,ix_empno_fromdate | PRIMARY | 20      | const,const |    1 |       |
+------+-------------+----------+-------+---------------------------+---------+---------+-------------+------+-------+
```

### eq_ref

eq_ref 只会出现在多个表连接( JOIN )查询的情况下。出现 eq_ref 的条件比较苛刻：

* 在连接查询中先读取的数据表的主键或唯一索引列作为检索条件，从第二个以后读取的数据表的 type 列为 eq_ref。
* 用唯一索引查询第二个以后的数据表时，唯一索引必须为 NOT NULL 。
* 如果主键或者唯一索引由多个数据列组成时，只有索引的全部列作为比较条件时， eq_ref 才可用。

```
explain select * from dept_emp de inner join employees e where e.emp_no = de.emp_no and de.dept_no='d005';
+------+-------------+-------+--------+---------------------------+---------+---------+---------------------+--------+-------------+
| id   | select_type | table | type   | possible_keys             | key     | key_len | ref                 | rows   | Extra       |
+------+-------------+-------+--------+---------------------------+---------+---------+---------------------+--------+-------------+
|    1 | SIMPLE      | de    | ref    | PRIMARY,ix_empno_fromdate | PRIMARY | 16      | const               | 165571 | Using where |
|    1 | SIMPLE      | e     | eq_ref | PRIMARY                   | PRIMARY | 4       | employees.de.emp_no |      1 |             |
+------+-------------+-------+--------+---------------------------+---------+---------+---------------------+--------+-------------+
```

### ref

ref 类型与连接的顺序无关，也没有主键或者唯一键的约束条件。使用相等（=）条件查询时， ref 访问不会受到索引类型的影响。 ref 类型由于只对相等条件进行比较，本质上是一种非常快的访问方法。但是由于 ref 类型无法保证一定有1条记录返回，因此它并没有 eq_ref 快。

```
explain select * from dept_emp where dept_no='d005';
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
| id   | select_type | table    | type | possible_keys | key     | key_len | ref   | rows   | Extra       |
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
|    1 | SIMPLE      | dept_emp | ref  | PRIMARY       | PRIMARY | 16      | const | 165571 | Using where |
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
```

从上面的例子可以看到，由于 dept_emp 表的主键是（ dept_no+emp_no ），而在查询条件中只用到其中一个字段： dept_no ，因此 type 列为 ref 而非 const 。需要注意的是，执行计划中的 ref 列显示的值为 const ，注意不要与 type 列的值混淆，它是指 where 条件中的常量值（'d005'）。

### const eq_ref ref type 的比较总结

* const ： 不受连接顺序的影响，对主键或者唯一键的**所有列**使用相等条件进行查询（有且返回1条记录）。注意：多个列组成的主键或者唯一索引，必须要用到所有列。

* eq_ref ： 使用连接中第一次读取的数据表列值，将第二张数据表用作主键或者唯一键进行相等条件查询（第二张表有且返回1条记录）。

* ref ：不受连接顺序的影响，使用相等条件查询（允许不保证返回1条记录）。

上述三种 type 的共同点是，在 WHERE 条件中使用的比较运算符必须是 “=” 相等比较运算符。此外，上述三种方法的性能都非常高，不用太多的调优考虑。

### fulltext

由于在实际的工作当中，基本上不会使用关系型数据库作为全文检索，而是使用专业的搜索引擎（ solr ，elastic search ），因此该类型不做过多讨论。

### ref_or_null

该访问方法与 ref 类型，只是增加了对 NULL 值的比较。此种查询方式在实际的工作当中使用得并不多见。

```
explain select * from titles where to_date='1985-03-01' or to_date is null;
+------+-------------+--------+-------------+---------------+-----------+---------+-------+------+--------------------------+
| id   | select_type | table  | type        | possible_keys | key       | key_len | ref   | rows | Extra                    |
+------+-------------+--------+-------------+---------------+-----------+---------+-------+------+--------------------------+
|    1 | SIMPLE      | titles | ref_or_null | ix_todate     | ix_todate | 4       | const |    2 | Using where; Using index |
+------+-------------+--------+-------------+---------------+-----------+---------+-------+------+--------------------------+
```

### unique_subquery

unique_subquery 类型经常出现于 WHERE 条件中 IN 形式的查询中，子查询只返回不重复的唯一值时。

```
explain select * from departments where dept_no IN (select dept_no from dept_emp where emp_no=10001);
+------+-------------+-------------+--------+---------------------------+-------------------+---------+----------------------------+------+-------------+
| id   | select_type | table       | type   | possible_keys             | key               | key_len | ref                        | rows | Extra       |
+------+-------------+-------------+--------+---------------------------+-------------------+---------+----------------------------+------+-------------+
|    1 | PRIMARY     | dept_emp    | ref    | PRIMARY,ix_empno_fromdate | ix_empno_fromdate | 4       | const                      |    1 | Using index |
|    1 | PRIMARY     | departments | eq_ref | PRIMARY                   | PRIMARY           | 16      | employees.dept_emp.dept_no |    1 |             |
+------+-------------+-------------+--------+---------------------------+-------------------+---------+----------------------------+------+-------------+
```

可以看到，上述执行计划中，并没有出现 unique_subquery ，而是出现了 eq_ref 。这是由于在 MariaDB 10.0 的版本中，对 IN 查询进行了优化，优化器使用了 JOIN 连接查询的方式来执行上述的 SQL 。

### index_subquery

前面提到的 unique_subquery 与 index_subquery 的区别是，unique_subquery 的 IN 查询子句中，不会出现重复的值。而往往在实际的业务场景中， IN 查询子句中是很容易出现重复值的。index_subquery 会先使用索引删除重复的值，再用其结果来进行查询。

```
explain select * from departments where dept_no in (select dept_no from dept_emp where dept_no between 'd001' and 'd003');
+------+-------------+-------------+-------+---------------+---------+---------+-------------------------------+-------+--------------------------------------+
| id   | select_type | table       | type  | possible_keys | key     | key_len | ref                           | rows  | Extra                                |
+------+-------------+-------------+-------+---------------+---------+---------+-------------------------------+-------+--------------------------------------+
|    1 | PRIMARY     | departments | range | PRIMARY       | PRIMARY | 16      | NULL                          |     3 | Using where                          |
|    1 | PRIMARY     | dept_emp    | ref   | PRIMARY       | PRIMARY | 16      | employees.departments.dept_no | 36844 | Using index; FirstMatch(departments) |
+------+-------------+-------------+-------+---------------+---------+---------+-------------------------------+-------+--------------------------------------+
```

可以看到，MariaDB 10.0 的优化器同样对上述 SQL 做出了优化，使用 JOIN 的方式来进行查询。

### range

range 即索引范围扫描，在一定的范围内进行检索索引。 主要出现在使用 <、>、IS NULL、BETWEEN、IN、LIKE 等运算符中。 range 同时也属于非常快速的查询类型。

```
EXPLAIN select * from employees where emp_no between 10002 and 10004;
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL |    3 | Using where |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
```

### index_merge

index_merge 先使用两个以上的索引分别获取搜索结果，然后再将其合并。它有如下几个特点：

* 由于需要读取多个索引，访问效率通常比 range 要慢。
* AND 与 OR 连接成的复杂查询中，很多时候不能正常优化。
* 使用全文索引的查询中，不使用 index_merge 。
* 需要对两个以上的结果集进行合并，删除重复等额外的操作。

基于上述原因，index_merge 的访问效率较差。

```
explain select * from employees where (emp_no between 10001 and 11000) or first_name='smith';
+------+-------------+-----------+-------------+----------------------+----------------------+---------+------+------+------------------------------------------------+
| id   | select_type | table     | type        | possible_keys        | key                  | key_len | ref  | rows | Extra                                          |
+------+-------------+-----------+-------------+----------------------+----------------------+---------+------+------+------------------------------------------------+
|    1 | SIMPLE      | employees | index_merge | PRIMARY,ix_firstname | PRIMARY,ix_firstname | 4,58    | NULL | 1001 | Using union(PRIMARY,ix_firstname); Using where |
+------+-------------+-----------+-------------+----------------------+----------------------+---------+------+------+------------------------------------------------+
```

如果将上述 sql 修改为 union 的方式，执行计划会好很多：

```
explain select * from employees where (emp_no between 10001 and 11000) union select * from employees where first_name='smith';
+------+--------------+------------+-------+---------------+--------------+---------+-------+------+-----------------------+
| id   | select_type  | table      | type  | possible_keys | key          | key_len | ref   | rows | Extra                 |
+------+--------------+------------+-------+---------------+--------------+---------+-------+------+-----------------------+
|    1 | PRIMARY      | employees  | range | PRIMARY       | PRIMARY      | 4       | NULL  | 1000 | Using where           |
|    2 | UNION        | employees  | ref   | ix_firstname  | ix_firstname | 58      | const |    1 | Using index condition |
| NULL | UNION RESULT | <union1,2> | ALL   | NULL          | NULL         | NULL    | NULL  | NULL |                       |
+------+--------------+------------+-------+---------------+--------------+---------+-------+------+-----------------------+
```

通过 show profile 验证，第二种查询也比第一种查询快27%。

```
show profiles;
+----------+------------+-----------------------------------------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                                                 |
+----------+------------+-----------------------------------------------------------------------------------------------------------------------+
|        1 | 0.00258577 | select * from employees where (emp_no between 10001 and 11000) or first_name='smith'                                  |
|        2 | 0.00190956 | select * from employees where (emp_no between 10001 and 11000) union select * from employees where first_name='smith' |
+----------+------------+-----------------------------------------------------------------------------------------------------------------------+
```

### index

index 其实是索引**全扫描**，复杂度相当于 O(N)，因此它是一种较为低效的方式（千万不要被其名字所误导了）。

与全表扫描（ ALL ）相比，index 也需要从头到尾比较同等的记录数。但是由于索引文件通常比数据文件要小，同时，查询内容使用已经排好顺序的索引，因此它比全表扫描要高效。

下面列出的条件中，如果满足条件1 + 条件2 或者满足条件1 + 条件3时，就会使用 index 访问方式。

1) 无法使用 range、const 或 ref 访问方式使用索引。
2) 仅用 index 中的列即可处理的查询（即不需要读取数据文件，只需要读取索引文件）。
3) 可以使用索引进行 order by 或 group 的情况（即不需要在数据文件当中进行排序）。

```
explain select * from departments order by dept_name desc limit 10;
+------+-------------+-------------+-------+---------------+-------------+---------+------+------+-------------+
| id   | select_type | table       | type  | possible_keys | key         | key_len | ref  | rows | Extra       |
+------+-------------+-------------+-------+---------------+-------------+---------+------+------+-------------+
|    1 | SIMPLE      | departments | index | NULL          | ux_deptname | 162     | NULL |    9 | Using index |
+------+-------------+-------------+-------+---------------+-------------+---------+------+------+-------------+
```

上面的这个查询中不包含任何 WHERE 条件，所以无法使用 range、const、或 ref 的访问方式。但是由于 ORDER BY 的列上有索引 `ux_deptname` ，因此使用了 index 的访问方式。

### ALL

ALL 就是全表扫描，该方式从头到尾从数据表当中读取数据，是一种非常低效的方式。 InnoDB 提供了一次读取多页的功能，被称为“预读”（Read Ahead），MariaDB 中若连续发生读取相邻页面，则后台读取线程会从磁盘一次读取64个页面，这比每次读取一个页面的方式要快得多。这对于全表扫描或索引全扫描这类需要大量磁盘 I/O 的作业时，它非常有用。

## possible_keys 列
---

通常在处理查询时， MariaDB 优化器会先考虑多种处理方法，然后从中选择一个代价最低的执行计划来进行查询，为了创建最优的执行计划，possible_keys 当中显示出了候选的多个索引列表，提供给优化器去选择。

## key 列
---

与 possible_keys 列中的索引是显示的候选索引不同的是， Key 列中显示的索引则是最终选用的执行计划当中所使用的索引。因此，必须关注 Key 列中显示的索引是否是所期望的索引。

除了 index_merge 之外，每一个数据表只能使用1个索引。 index_merge 可以使用两个以及以上的索引，在 Key 列中将显示为多个索引，以逗号进行分隔显示。

如果执行计划的 type 为 ALL 时，即完全不是用索引时，key 列将显示为 NULL 。

## key_len 列
---

key_len 列值会告诉我们用于处理查询的多列索引是由几个数据列构成的，表明索引的各个记录用到了几个字节。

```
explain select * from dept_emp where dept_no='d005';
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
| id   | select_type | table    | type | possible_keys | key     | key_len | ref   | rows   | Extra       |
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+
|    1 | SIMPLE      | dept_emp | ref  | PRIMARY       | PRIMARY | 16      | const | 165571 | Using where |
+------+-------------+----------+------+---------------+---------+---------+-------+--------+-------------+

show create table dept_emp;

| dept_emp | CREATE TABLE `dept_emp` (
  `emp_no` int(11) NOT NULL,
  `dept_no` char(4) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`dept_no`,`emp_no`),
  KEY `ix_fromdate` (`from_date`),
  KEY `ix_empno_fromdate` (`emp_no`,`from_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

可以看到 key_len 列值为16。 dept_no 的数据类型为 CHAR(4) ， dept_emp 的字符集为 utf8mb4 ，MariaDB 为 utf8mb4 分配内存空间时，是按照固定的4个字节计算，不受字符的影响，4 * 4 = 16 字节。

修改上述的 SQL ， 增加了 emp_no 的查询条件后，key_len 的值发生了变化，变为了 20 个字节。这是因为 emp_no 是 int 类型，占用了4个字节。

```
explain select * from dept_emp where dept_no='d005' and emp_no=10001;
+------+-------------+----------+-------+---------------------------+---------+---------+-------------+------+-------+
| id   | select_type | table    | type  | possible_keys             | key     | key_len | ref         | rows | Extra |
+------+-------------+----------+-------+---------------------------+---------+---------+-------------+------+-------+
|    1 | SIMPLE      | dept_emp | const | PRIMARY,ix_empno_fromdate | PRIMARY | 20      | const,const |    1 |       |
+------+-------------+----------+-------+---------------------------+---------+---------+-------------+------+-------+
```

## ref 列
---

ref 列显示了使用哪些值作为查询的参考条件；若指定了常数值，则显示为 const ；若为其它表的列值，则显示为表名或者列名。

注意，如果 ref 列当中出现了类似于 func 等比较特殊值的时候，说明 MariaDB 对查询条件当中做了一系列隐式转换，这通常导致性能的下降。让我们来看一下下面的一个示例：

```
explain select * from employees e, dept_emp de where e.emp_no = (de.emp_no-1);
+------+-------------+-------+--------+---------------+---------+---------+------+--------+-------------+
| id   | select_type | table | type   | possible_keys | key     | key_len | ref  | rows   | Extra       |
+------+-------------+-------+--------+---------------+---------+---------+------+--------+-------------+
|    1 | SIMPLE      | de    | ALL    | NULL          | NULL    | NULL    | NULL | 331603 |             |
|    1 | SIMPLE      | e     | eq_ref | PRIMARY       | PRIMARY | 4       | func |      1 | Using where |
+------+-------------+-------+--------+---------------+---------+---------+------+--------+-------------+
```

可以看到由于我们对比较值做了运算（de.emp_no - 1），因此 ref 列变为了 `func` 。除此之外，关联查询中如果列的类型不匹配，例如一个字段是 CHAR ，另外一个字段是 VARCHAR ，也会出现 ref 列出现 func ，需要特别注意。

## rows 列
---

rows 列用于优化器判断执行计划的效率而所预测的记录数。由于是根据存储引擎推算的记录数，因此并不一定准确。此外，该记录数并不是返回的实际记录行数，而是指处理查询时需要从磁盘读取与价差的多少条记录（扫描行数）。

让我们来看一个例子，下面的 sql 是查询了日期大于 1985-01-01 的所有记录，由于在 dept_emp 表中大于该日期的记录就是整个表的记录（331603）条，因此虽然在 from_date 字段上有创建 ix_fromdate 的索引，但是执行计划最终选择了全表扫描，而非 range 。原因就是因为 rows 列输出的值 331603 与 整个表的记录数一致，优化器认为 ALL 的效率会高于 range。

```
explain select * from dept_emp where from_date >= '1985-01-01';
+------+-------------+----------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table    | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+----------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | dept_emp | ALL  | ix_fromdate   | NULL | NULL    | NULL | 331603 | Using where |
+------+-------------+----------+------+---------------+------+---------+------+--------+-------------+
```

假设我们缩小日期的筛选范围，让扫描的记录数更少一些，此时，rows 列出现的扫描行数缩小为292，就会发现执行计划变成了 range 。

```
explain select * from dept_emp where from_date >= '2002-07-01';
+------+-------------+----------+-------+---------------+-------------+---------+------+------+-----------------------+
| id   | select_type | table    | type  | possible_keys | key         | key_len | ref  | rows | Extra                 |
+------+-------------+----------+-------+---------------+-------------+---------+------+------+-----------------------+
|    1 | SIMPLE      | dept_emp | range | ix_fromdate   | ix_fromdate | 3       | NULL |  292 | Using index condition |
+------+-------------+----------+-------+---------------+-------------+---------+------+------+-----------------------+
```

由此我们可以看出，不一定在任何场景下， ALL 全表扫描的查询效率都会很低。

## Extra 列
---

Extra 列当中有很多关于性能方面的重要信息，一般会同时显示 2 ~ 3 个信息。并且这些额外信息的性能与显示的顺序无关。

### const row not found

当 type 列的值为 const ，但是表中没有一条符合条件的记录时，Extra 列就会显示出 `const rows not found` 。建议在出现该值，先创建一些符合查询条件的数据，再来观察执行计划。

### Distinct

下面的 sql 会在 Extra 列显示出 `Distinct`

```
explain select distinct d.dept_no from departments d, dept_emp de where de.dept_no=d.dept_no;
+------+-------------+-------+-------+---------------+---------+---------+---------------------+-------+------------------------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref                 | rows  | Extra                        |
+------+-------------+-------+-------+---------------+---------+---------+---------------------+-------+------------------------------+
|    1 | SIMPLE      | d     | index | PRIMARY       | PRIMARY | 16      | NULL                |     9 | Using index; Using temporary |
|    1 | SIMPLE      | de    | ref   | PRIMARY       | PRIMARY | 16      | employees.d.dept_no | 36844 | Using index; Distinct        |
+------+-------------+-------+-------+---------------+---------+---------+---------------------+-------+------------------------------+
```

### Full scan on NULL key

`Full scan on NULL key` 经常会出现在带有类似 col1 IN (SELECT col2 FROM ...) 条件的查询语句中。如果 col1 的值为 NULL ，则条件最终变为 NULL IN (SELECT col2 FROM ...) 。 在此过程中，如果 col1 的值为 NULL ，则会使用全表扫描，并且 Extra 列的值为 `Full scan on NULL key` 。 因此**这种方式很容易导致性能问题**，解决方案是让 col1 的值不要为 NULL ，即增加 col1 IS NOT NULL 的查询条件。

```
explain select d.dept_no, NULL IN (select id.dept_name from departments id) from departments d;
+------+-------------+-------+-------+---------------+-------------+---------+-------+------+-------------------------------------------------+
| id   | select_type | table | type  | possible_keys | key         | key_len | ref   | rows | Extra                                           |
+------+-------------+-------+-------+---------------+-------------+---------+-------+------+-------------------------------------------------+
|    1 | PRIMARY     | d     | index | NULL          | PRIMARY     | 16      | NULL  |    9 | Using index                                     |
|    2 | SUBQUERY    | id    | ref   | ux_deptname   | ux_deptname | 162     | const |    1 | Using where; Using index; Full scan on NULL key |
+------+-------------+-------+-------+---------------+-------------+---------+-------+------+-------------------------------------------------+
```

### impossible HAVING

`impossible HAVING` 会出现在不存在满足查询 HAVING 子句条件的记录时

```
explain select e.emp_no, count(*) as cnt from employees e where e.emp_no=10001 group by e.emp_no having e.emp_no is null;
+------+-------------+-------+------+---------------+------+---------+------+------+-------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra             |
+------+-------------+-------+------+---------------+------+---------+------+------+-------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Impossible HAVING |
+------+-------------+-------+------+---------------+------+---------+------+------+-------------------+
```

上述查询语句中，由于 emp_no 是主键，是不可用为 NULL 值的，因此 e.emp_no is null 就导致了 Extra 列出现 `impossible HAVING` 。在绝大多数情况下，Extra 列出现 `impossible HAVING` 的原因是由于我们 sql 语句的编写不正确导致的，需要仔细排查和修复当中的 bug 。

### impossible WHERE

与 `impossible HAVING` 类似，`impossible WHERE` 会出现在 WHERE 条件比较结果总是为 false 时。

```
explain select * from employees where emp_no is null;
+------+-------------+-------+------+---------------+------+---------+------+------+------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra            |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Impossible WHERE |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------+
```

### impossible WHERE noticed after reading const table

`impossible WHERE` 不会实际去读取数据，而是根据数据表结构判断查询条件为 “不可能” 。 `impossible WHERE noticed after reading const table` 则是通过实际执行了查询之后，来判断出查询条件为 “不可能”。

```
explain select * from employees where emp_no = 0;
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                                               |
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Impossible WHERE noticed after reading const tables |
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
```

### No matching min/max row

`No matching min/max row` 与 `impossible WHERE` 类似，只是查询当中额外包含了 MIN() 或 MAX() 这样的聚合函数，并且无任何符合查询条件的记录返回。

```
explain select min(dept_no), max(dept_no) from dept_emp where dept_no = '';
+------+-------------+-------+------+---------------+------+---------+------+------+-------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                   |
+------+-------------+-------+------+---------------+------+---------+------+------+-------------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | No matching min/max row |
+------+-------------+-------+------+---------------+------+---------+------+------+-------------------------+
```

### No matching row in const table

如果使用 const 方式访问连接中的数据表，并且不存在满足查询条件的记录，则 Extra 列的值为 `No matching row in const table` 。

```
explain select * from dept_emp de, (select emp_no from employees where emp_no=0) tb1 where tb1.emp_no=de.emp_no and de.dept_no='d005';
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                                               |
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Impossible WHERE noticed after reading const tables |
+------+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
```

### No tables used

顾名思义，`No tables used` 表示不带 FROM 子句的查询或者 FROM DUAL （行列均为1的虚拟常数表）。

```
explain select 1;
explain select 1 from dual;
+------+-------------+-------+------+---------------+------+---------+------+------+----------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra          |
+------+-------------+-------+------+---------------+------+---------+------+------+----------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | No tables used |
+------+-------------+-------+------+---------------+------+---------+------+------+----------------+
```

### Not exists

在日常的开发当中，我们会经常使用 NOT IN 或者 NOT EXISTS 运算符去查询在 A 表中存在但是在 B 表中不存在的值。同时 NOT IN 或 NOT EXISTS 可以使用 LEFT OUTER JOIN 来替换以获得更好的处理性能。

```
explain select * from dept_emp de left join departments d on de.dept_no = d.dept_no where d.dept_no is null;
+------+-------------+-------+--------+---------------+---------+---------+----------------------+--------+-------------------------+
| id   | select_type | table | type   | possible_keys | key     | key_len | ref                  | rows   | Extra                   |
+------+-------------+-------+--------+---------------+---------+---------+----------------------+--------+-------------------------+
|    1 | SIMPLE      | de    | ALL    | NULL          | NULL    | NULL    | NULL                 | 331603 |                         |
|    1 | SIMPLE      | d     | eq_ref | PRIMARY       | PRIMARY | 16      | employees.de.dept_no |      1 | Using where; Not exists |
+------+-------------+-------+--------+---------------+---------+---------+----------------------+--------+-------------------------+
```

### Range checked for each record(index map:N)

```
explain select * from employees e1, employees e2 where e2.emp_no >= e1.emp_no;
+------+-------------+-------+------+---------------+------+---------+------+--------+------------------------------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows   | Extra                                          |
+------+-------------+-------+------+---------------+------+---------+------+--------+------------------------------------------------+
|    1 | SIMPLE      | e1    | ALL  | PRIMARY       | NULL | NULL    | NULL | 300024 |                                                |
|    1 | SIMPLE      | e2    | ALL  | PRIMARY       | NULL | NULL    | NULL | 300024 | Range checked for each record (index map: 0x1) |
+------+-------------+-------+------+---------------+------+---------+------+--------+------------------------------------------------+
```

执行查询时如果需要连接两个表，而连接的条件是两个变量（不是固定常数值）时，即用于 sql 优化器计算查询代价的基准值一直在发生变化，优化器无法判断出使用哪种方式来读取 e2 表更有效率。例如，假设员工编号为 1 ~ 100000000 ，那么会从头到尾扫描e1表，同时从e2表中查找出满足 e2.emp_no >= e1.emp_no 条件的记录。在该过程中，e1.emp_no=1时，就需要依次读取 e2 表中的所有记录；e1.emp_no=100000000 时，只读取 e2 数据表中的1条记录。

因此，上述场景下 sql 优化器所采用的优化方式是一种动态的方式，即 e1 数据表的 emp_no 很小时，使用全表扫描访问 e2 数据表；而 e1 数据表的 emp_no 很大时，就使用索引范围扫描来访问 e2 数据表。“对每条记录检查索引范围扫描”，此时的 Extra 列显示为 `Range checked for each record` 。

Extra 列当中的 `index map:0x1` 是候选索引编号，用于判断是否使用候选索引。 `index map` 采用16进制表示，解析首先需要将其转换为二进制数。0x1 转换为二进制数为1， 1表示可能使用的索引列为 PRIMARY KEY (emp_no) 。在上述查询处理每一条记录时，都需要确定是否使用 e2 表的第一个索引还是直接使用全表扫描的方式。

```
| employees | CREATE TABLE `employees` (
  `emp_no` int(11) NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) NOT NULL,
  `last_name` varchar(16) NOT NULL,
  `gender` enum('M','F') NOT NULL,
  `hire_date` date NOT NULL,
  PRIMARY KEY (`emp_no`),
  KEY `ix_firstname` (`first_name`),
  KEY `ix_hiredate` (`hire_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 |
```

`index map` 其实本质上是以位图（Bit map）的方式来存储索引列的序号。例如我们有如下的一个数据表，表结构为:

```
CREATE TABLE `tb_member` (
  `mem_id` int(11) NOT NULL,
  `mem_name` varchar(100) NOT NULL,
  `mem_nickname` varchar(100) NOT NULL,
  `mem_region` tinyint NULL,
  `mem_gender` tinyint NULL,
  `mem_phone` varchar(25) NULL,
  PRIMARY KEY (`mem_id`),
  KEY `ix_nick_name` (mem_nickname,mem_name),
  KEY `ix_nick_region` (mem_nickname,mem_region),
  KEY `ix_nick_gender` (mem_nickname,mem_gender),
  KEY `ix_nick_phone` (mem_nickname,mem_phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

如果在执行计划中出现了 `index map:0x19` ， 将其转换为二进制数为：11001，以二进制位图的方式表示如下：

| 位数     | 第五位     | 第四位 |  第三位     | 第二位 | 第一位 |
| :------------- | :------------- |:------------- |:------------- |:------------- |
| 位图值       | 1 | 1 | 0 | 0 | 1 |
| 索引列       | ix_nick_phone | ix_nick_gender | ix_nick_region | ix_nick_name | PRIMARY KEY |

因此，使用位图值为1的位所对应的索引，来作为候选索引，其选择的索引为:

* PRIMARY KEY
* ix_nick_gender
* ix_nick_phone

> 若查询的执行计划中 Extra 中有大量的 `Range checked for each record` ，可以通过观察 SHOW GLOBAL STATUS 中的 Select_range_check 值显示得较大。

```
SHOW GLOBAL STATUS LIKE '%Select_%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| Select_full_join       | 0     |
| Select_full_range_join | 0     |
| Select_range           | 4     |
| Select_range_check     | 0     |
| Select_scan            | 18    |
+------------------------+-------+
```

### Scanned N databases

MariaDB 提供 INFORMATION_SCHEMA 数据库击中存放着数据库的元数据信息（数据表、数据列、索引等 Schema 信息）。由于在查询的过程中会频繁访问该数据库的数据，从 MariaDB 5.1 版本开始大大提升了其访问性能，此时 Extra 列会显示为 `Scanned N databases` ，N 表示读取几个数据库的信息，其取值为：

* 0：只请求某个特定表的信息，并不读取整个数据库的元数据信息。
* 1：请求特定数据库内所有 Schema 信息，读取相关数据库的所有 Schema 信息。
* ALL：读取 MariaDB 服务器内所有 Schema 信息。

只有从 INFORMATION_SCHEMA 数据表中读取数据时，才会显示 `Scanned N databases` ，因此该类型的输出在普通的应用程序中很难出现。

```
explain select table_name from information_schema.tables where table_schema='employees' AND table_name='employees';
+------+-------------+--------+------+---------------+-------------------------+---------+------+------+---------------------------------------------------+
| id   | select_type | table  | type | possible_keys | key                     | key_len | ref  | rows | Extra                                             |
+------+-------------+--------+------+---------------+-------------------------+---------+------+------+---------------------------------------------------+
|    1 | SIMPLE      | tables | ALL  | NULL          | TABLE_SCHEMA,TABLE_NAME | NULL    | NULL | NULL | Using where; Skip_open_table; Scanned 0 databases |
+------+-------------+--------+------+---------------+-------------------------+---------+------+------+---------------------------------------------------+
```

### Select tables optimizer away

查询的 SELECT 语句中只使用 MIN() 或 MAX() ，或者用 GROUP BY 访问 MIN() 或 MAX() 时，若无法使用合适的索引，就会按升序或降序只读取1个索引，通过此种优化后，Extra 列就会显示出 `Select tables optimizer away` 。

此外，对于 MyISAM 存储引擎的表，SELECT 语句中只有 `COUNT(*)` 时，并且不包含 GROUP BY 语句，也会使用这种形式来做优化。原因是由于 MyISAM 单独管理了表的记录条数，即使不读取索引或数据也能够快速访问到最大、最小记录。但是如果有 WHERE 条件查询时，则无法使用这种优化。

```
explain select max(emp_no), min(emp_no) from employees;
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                        |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Select tables optimized away |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+

explain select max(from_date), min(from_date) from salaries where emp_no=10001;
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                        |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Select tables optimized away |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
```

上面的第一个查询中，由于 employees 表中的 emp_no 是主键索引，因此可以使用到 `Select tables optimizer away` 优化。

第二个查询中，会使用到 `PRIMARY KEY (emp_no,from_date)` 索引，在检索结果中按照升序或降序最后只访问到一条记录，所以可以使用上述优化。

### Skip_open_table、Open_frm_only、Open_trigger_only、Open_full_table

与 `Scanned N databases` 类似，只有从 INFORMATION_SCHEMA 数据表中读取数据时，才会显示下列的信息：
* Skip_open_table ：不必读取保存数据表元信息的文件
* Open_frm_only ：仅打开保存数据表元信息的文件（ `*.FRM` ）并进行读取
* Open_trigger_only ：仅打开保存触发器表元信息的文件（ `*.TRG` ）并进行读取
* Open_full_table ：由于无法优化，必须将数据表元信息文件（ `*.FRM` ）、数据文件（ `*.MYD` ）、索引文件（ `*.MYI` ）全部都读取。

需要注意的是，`*.FRM` 和 `*.MYI` 只针对 MyISAM 存储引擎才有效，不适用于 InnoDB 。

### unique row not found

两个表各自的唯一列（包括主键）执行连接查询时，若外连表中不存在一样的记录，则 Extra 列会显示 `unique row not found` 。

```
CREATE TABLE tb_test1 (fdpk INT, PRIMARY KEY(fdpk));
CREATE TABLE tb_test2 (fdpk INT, PRIMARY KEY(fdpk));

INSERT INTO tb_test1 VALUES (1),(2);
INSERT INTO tb_test2 VALUES (1);
```

```
explain select t1.fdpk from tb_test1 t1 left join tb_test2 t2 on t2.fdpk=t1.fdpk where t1.fdpk=2;
+------+-------------+-------+-------+---------------+---------+---------+-------+------+-------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref   | rows | Extra       |
+------+-------------+-------+-------+---------------+---------+---------+-------+------+-------------+
|    1 | SIMPLE      | t1    | const | PRIMARY       | PRIMARY | 4       | const |    1 | Using index |
+------+-------------+-------+-------+---------------+---------+---------+-------+------+-------------+
```

NOTE：上述查询在 MariaDB 10.0.8 当中并未出现 `unique row not found` 。

### Using filesort

当 ORDER BY 排序**无法使用索引时**，Extra 列会显示为 `Using filesort` 。这意味着 MariaDB 会将返回的记录集合复制到用于排序的内存缓冲区（ Sort Buffer ）中，然后使用**快速排序算法**进行排序。

可以通过调整 sort_buffer_size 来设置其大小。但是有设置之前有很多需要注意的地方，参见下文当中提到的 join_buffer_size 。

```
explain select * from employees order by last_name desc;
+------+-------------+-----------+------+---------------+------+---------+------+--------+----------------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra          |
+------+-------------+-----------+------+---------------+------+---------+------+--------+----------------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 300024 | Using filesort |
+------+-------------+-----------+------+---------------+------+---------+------+--------+----------------+
```

上面的查询中，由于 last_name 列没有索引，所以无法使用索引处理排序操作。出现 `Using filesort` 时，会给系统带来很大的额外开销，应尽量对其进行优化。

### Using index

若处理查询时完全不必读取数据，仅仅从索引上就能够获取到结果时，Extra 列会显示为 `Using index` （又被称之为覆盖索引）。使用索引处理查询的过程中，性能损耗最大的部分就是再检索完索引文件之后，使用查询到的索引值指针再去数据文件当中去读取其它列的数据。在最坏的情况下，通过索引检索到的每条记录都要读取一次磁盘。

```
explain select first_name, birth_date from employees force index (ix_firstname) where first_name between 'Babette' and 'Gad';
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-----------------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows   | Extra                 |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-----------------------+
|    1 | SIMPLE      | employees | range | ix_firstname  | ix_firstname | 58      | NULL | 105830 | Using index condition |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-----------------------+
```

注意：上述查询中使用了 force index 来强制使用索引 ix_firstname 。

![cover_index_btree](http://static.zhuxiaodong.net/blog/static/images/cover_index_btree.png)

上图反映了整个查询的过程，首先通过 ix_firstname 在索引叶节点获取到 first_name 列的记录，但是由于 birth_date 上没有建立索引，因此优化器必须通过索引记录上记录的数据文件的指针，二次查询出 birth_date 列的值。记录数越多，二次查询数据文件的成本就越大。因此，最好是一次性就能够从索引文件当中获取出需要的列。

不得不提到的是，InnoDB 表的所有辅助索引中都会主键的值，用来作为指向数据文件的指针。比如上面的图中，记录地址实际上就是 employees 表的主键 emp_no。这里引申出另一个话题，InnoDB 存储引擎中主键尽量为存储空间比较小，且连续的数据类型，例如 int auto_increment 。

这也是为什么很多公司的数据库查询规范中，提到的**尽量只查询必要的列，切忌不要使用 SELECT 所有列 ** 的原因。

假如我们去掉了返回 birth_date 列，执行计划当中的 Extra 列就会显示为 Using index ，表示使用了覆盖索引（ covering index ）。

```
explain select first_name from employees force index (ix_firstname) where first_name between 'Babette' and 'Gad';
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows   | Extra                    |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+--------------------------+
|    1 | SIMPLE      | employees | range | ix_firstname  | ix_firstname | 58      | NULL | 105830 | Using where; Using index |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+--------------------------+
```

同时，由于 InnoDB 的辅助索引中存储了主键值的特性，当我们增加返回 emp_no 一列时，实际上也不需要再从数据文件当中进行二次查询。

```
explain select emp_no, first_name from employees force index (ix_firstname) where first_name between 'Babette' and 'Gad';
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows   | Extra                    |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+--------------------------+
|    1 | SIMPLE      | employees | range | ix_firstname  | ix_firstname | 58      | NULL | 105830 | Using where; Using index |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+--------------------------+
```

一般来说，使用覆盖索引能够极大地提高查询性能，但是我们却无法为所有返回的列都建立索引，来满足所有业务场景下的查询。因为毕竟建立索引会占用额外的磁盘存储空间和更多的内存资源，并且会降低 Insert Update 操作的性能，因此我们需要平衡两者。

### Using index for group-by

处理 GROUP BY 时，需要先使用分组基准列进行排序，然后将排序结果进行分组，这会耗费较多的性能。但是如果使用 B-Tree 索引处理 GROUP BY ，就能够按照已经排好顺序的方式依次读取列，这样就仅仅只需要进行分组操作，因此能够大大地提升访问效率。这种使用索引处理 GROUP BY 时，Extra 列就会显示 `Using index for group-by` 。

#### 使用紧凑索引扫描（索引扫描）处理 GROUP BY

带有 GROUP BY 子句的查询中含有 AVG() 、 SUM() 、 `COUNT(*）` 等函数时，由于这些函数都需要进行额外的扫描或汇总操作，无法使用 “松散索引扫描” 的方式，因此在执行计划中不会显示出 `Using index for group-by`。

```
explain select first_name, count(*) as counter from employees group by first_name;
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | index | NULL          | ix_firstname | 58      | NULL | 300024 | Using index |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-------------+
```

#### 使用松散索引扫描（ Loose index scan ）处理 GROUP BY

使用 GROUP BY 排序时，如果 GROUP BY 的是单列索引，并且只返回该 GROUP BY 的字段，则可以使用 `Loose index scan` 。

```
explain select first_name from employees group by first_name;
+------+-------------+-----------+-------+---------------+--------------+---------+------+------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows | Extra                    |
+------+-------------+-----------+-------+---------------+--------------+---------+------+------+--------------------------+
|    1 | SIMPLE      | employees | range | NULL          | ix_firstname | 58      | NULL | 1277 | Using index for group-by |
+------+-------------+-----------+-------+---------------+--------------+---------+------+------+--------------------------+
```

如果 GROUP BY 的是多列索引，并且 SELECT 返回的列中只需要读取第一条或者最后一条记录（ MIN(), MAX() ），也可以使用 `Loose index scan` 。例如如下的查询中，salaries 表的索引为：emp_no + from_date ，对于每一个 emp_no 分组，通过索引读取第一个 from_date 值与最后一个 from_date 值即可，因此可以使用 `Loose index scan` 。

```
explain select emp_no, MIN(from_date) as first_changed, MAX(from_date) AS last_changed from salaries group by emp_no;
+------+-------------+----------+-------+---------------+---------+---------+------+--------+--------------------------+
| id   | select_type | table    | type  | possible_keys | key     | key_len | ref  | rows   | Extra                    |
+------+-------------+----------+-------+---------------+---------+---------+------+--------+--------------------------+
|    1 | SIMPLE      | salaries | range | NULL          | PRIMARY | 4       | NULL | 316006 | Using index for group-by |
+------+-------------+----------+-------+---------------+---------+---------+------+--------+--------------------------+
```

对于是否使用 `Loose index scan` 的方式的查询，WHERE 子句中使用的索引也会造成影响。下面我来分别分析一下：

* 无 WHERE 条件子句时：
这种情况下，只需要 GROUP BY 子句的数据列与由 SELECT 获取的数据列具备使用 `Loose index scan` 的条件即可。对于不具备上述条件的查询，则使用 `紧凑索引扫描` 或单独的排序过程进行处理（单独排序的过程通常会在 Extra 列当中出现 Using filesort）。

* 带有 WHERE 条件子句但无法使用索引扫描时：
WHERE 条件子句无法使用索引，但是 GROUP BY 条件可以使用索引，这种情况下只能使用 `紧凑索引扫描` 。

```
explain select first_name from employees where birth_date > '1994-01-01' group by first_name;
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | index | NULL          | ix_firstname | 58      | NULL | 300024 | Using where |
+------+-------------+-----------+-------+---------------+--------------+---------+------+--------+-------------+
```

* 带有 WHERE 条件子句并且可以使用索引扫描时：
由于 MariaDB 执行一个单位查询时，除了 type 为 index_merge 的情况下可以使用多个索引之外，其它的访问方式只能使用一个索引。因此，当 WHERE 条件子句使用索引时，GROUP BY 必须使用相同的索引，这样才能够满足使用 `Loose index scan` 的条件。当 WHERE 条件子句使用的索引与 GROUP BY 使用的索引不同时，优化器通常会使用 WHERE 条件子句所使用的索引。

```
explain select emp_no from salaries where emp_no between 10001 and 200000 group by emp_no;
+------+-------------+----------+-------+---------------+---------+---------+------+--------+---------------------------------------+
| id   | select_type | table    | type  | possible_keys | key     | key_len | ref  | rows   | Extra                                 |
+------+-------------+----------+-------+---------------+---------+---------+------+--------+---------------------------------------+
|    1 | SIMPLE      | salaries | range | PRIMARY       | PRIMARY | 4       | NULL | 157691 | Using where; Using index for group-by |
+------+-------------+----------+-------+---------------+---------+---------+------+--------+---------------------------------------+
```

当然，在某些场景下，由于 WHERE 条件子句查询到记录数非常少，即使不使用 `Loose index scan` ，也能够获得非常高的效率。只有对大量记录进行 GROUP BY 处理时，使用 `Loose index scan` 才能够获得明显的性能提升。优化器会根据实际的情况作出判断。

比如下面的例子中，我们只是修改了 WHERE 子句的查询条件，让查询的数据尽量的少，Extra 列中就没有再出现 `Using index for group-by` 了。

```
explain select emp_no from salaries where emp_no between 10001 and 10099 group by emp_no;
+------+-------------+----------+-------+---------------+---------+---------+------+------+--------------------------+
| id   | select_type | table    | type  | possible_keys | key     | key_len | ref  | rows | Extra                    |
+------+-------------+----------+-------+---------------+---------+---------+------+------+--------------------------+
|    1 | SIMPLE      | salaries | range | PRIMARY       | PRIMARY | 4       | NULL |  984 | Using where; Using index |
+------+-------------+----------+-------+---------------+---------+---------+------+------+--------------------------+
```

### Using join buffer(Block Nested Loop) 、 Using join buffer(Batched Key Access)

在进行多个表 JOIN 操作时，若被驱动的连接列没有索引，则会根据从驱动表（连接时首先要读取的表）中读取的记录条数，每次都对被驱动表（连接中后读取的数据表）进行全表扫描或索引全扫描。 MariaDB 为了解决被驱动表检索低效的问题，会将从驱动表中读取的记录保存到内存空间中，该空间被称为：连接缓冲区（ join buffer ）, 此时 Extra 列就会显示出 `Using join buffer` 。

从上述的原理中，我们可以看出如果在 Extra 列中显示出 Using join buffer ，说明 JOIN 的字段列并没有正确的使用到索引，需要考虑进行优化。

通过 join_buffer_size 系统变量可以设置用于连接的最大缓冲区大小。与 sort_buffer_size 一样，由于该变量的设置的值是针对于每一个 connection 的，因此在调优的过程中要谨慎，较大的值设置会导致内存分配效率降低。具体可以参考：
* https://www.percona.com/blog/2010/07/05/how-is-join_buffer_size-allocated/
* https://serverfault.com/questions/399518/join-buffer-size-4-m-is-not-advised
* https://stackoverflow.com/questions/17928366/what-is-the-recommended-max-value-for-join-buffer-size-in-mysql
* https://haydenjames.io/my-cnf-tuning-avoid-this-common-pitfall/

正如上面第四篇文章当中介绍的一样：永远不要在 my.cnf 或者 GLOBAL 级别 当中调整与每一个 connection 分配的 buffer 值，如果有需要的话，只能在 SESSION Level 做设置。

> **Not all buffers in my.cnf are global settings**
Buffers such as **join_buffer_size**, **sort_buffer_size**, **read_buffer_size** and **read_rnd_buffer_size** are allocated per connection. Therefore a setting of read_buffer_size=1M and max_connections=150 is asking MySQL to allocate – from startup – 1MB per connection x 150 connections. For more than a decade the default remains at 128K. Increasing the default is not only a waste of server memory, but often does not help performance. In nearly all cases its best to use the defaults by removing or commenting out these four buffer config lines. For a more gradual approach, reduce them in half to free up wasted RAM, keep reducing them towards default values over time. I’ve actually seen improved throughput by reducing these buffers. But there’s really no performance gains from increasing these buffers except in cases of very high traffic and/or other special circumstances. Avoid arbitrarily increasing these!

下面的查询中使用了无条件的笛卡尔连接（ Cartesian join ），这种情况下就会使用连接缓冲。

```
explain select * from dept_emp de, employees e where de.from_date>'2005-01-01' and e.emp_no<10904;
+------+-------------+-------+-------+---------------+-------------+---------+------+------+-------------------------------------------------+
| id   | select_type | table | type  | possible_keys | key         | key_len | ref  | rows | Extra                                           |
+------+-------------+-------+-------+---------------+-------------+---------+------+------+-------------------------------------------------+
|    1 | SIMPLE      | de    | range | ix_fromdate   | ix_fromdate | 3       | NULL |    1 | Using index condition                           |
|    1 | SIMPLE      | e     | range | PRIMARY       | PRIMARY     | 4       | NULL |  903 | Using where; Using join buffer (flat, BNL join) |
+------+-------------+-------+-------+---------------+-------------+---------+------+------+-------------------------------------------------+
```

其中 BNL join 表示 "Block Nested Loop Join"，除此之外，常见的还有 “Batched Key Access”，后续的章节当中会详细的进行讲解。
