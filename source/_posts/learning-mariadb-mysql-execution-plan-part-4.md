title: learing mariadb & mysql-execution plan(part-4)
date: 2018-01-04 13:55:44
categories: programming
tags:
- mysql
- mariadb
- execution plan
---

## EXPLAIN EXTENDED （ Filtered 列 ）

根据之前文章介绍的执行计划当中的 `Using where` 概念，由于在某些条件下，部分查询条件无法从 MariaDB 引擎`条件下推`至存储引擎，导致存储引擎会将多余的不符合查询条件的数据返回给 MariaDB 引擎。在 MariaDB 5.1.12 版本之前，MariaDB 引擎在存储引擎的从数据文件返回的记录中，最终到底过滤了多少条数据，是无法获取到这个信息的。

<!--more-->

从 MariaDB 5.1.12 之后的版本中，执行计划当中加入了 Filtered 列。我们通过 `EXPLAIN EXTENDED` 语句来获取到这个数据。

```
explain extended select * from employees where emp_no between 10001 and 10100 and gender = 'F';
+------+-------------+-----------+-------+---------------+---------+---------+------+------+----------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+----------+-------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL |  100 |   100.00 | Using where |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+----------+-------------+
```

filtered 列表示的是经过 MariaDB 引擎过滤后最终剩余的记录数 占 总记录数（存储引擎读取数据文件的记录数）的百分比。上述的 sql 中，filtered 列 100 表示存储引擎返回的记录数就是最终的查询结果。

filtered 的值最接近 100 ，则查询的效率越高效。

## EXPLAIN EXTENDED （附加优化器信息）

MariaDB 引擎制定查询的执行计划时，会分析查询语句并创建分析树。EXPLAIN 命令后使用 EXTENDED 选项能够重组分析树（ parse tree ），然后以查询语句顺序显示。

```
explain extended select e.first_name, (select count(*) from dept_emp de, dept_manager dm where dm.dept_no=de.dept_no) AS cnt from employees e where e.emp_no=10001;

+------+-------------+-------+-------+---------------+---------+---------+----------------------+-------+----------+-------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref                  | rows  | filtered | Extra       |
+------+-------------+-------+-------+---------------+---------+---------+----------------------+-------+----------+-------------+
|    1 | PRIMARY     | e     | const | PRIMARY       | PRIMARY | 4       | const                |     1 |   100.00 |             |
|    2 | SUBQUERY    | dm    | index | PRIMARY       | PRIMARY | 20      | NULL                 |    24 |   100.00 | Using index |
|    2 | SUBQUERY    | de    | ref   | PRIMARY       | PRIMARY | 16      | employees.dm.dept_no | 36844 |   100.00 | Using index |
+------+-------------+-------+-------+---------------+---------+---------+----------------------+-------+----------+-------------+
```

可以看到使用 extended 选项了之后，并没有输出其它额外的信息。我们还需要紧接着执行一次 `SHOW WARNINGS` 命令。

```
SHOW WARNINGS;

+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message

                 |
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1003 | select 'Georgi' AS `first_name`,(select count(0) from `employees`.`dept_emp` `de` join `employees`.`dept_manager` `dm` where `employees`.`de`.`dept_no` = `employees`.`dm`.`dept_no`) AS `cnt` from `employees`.`employees` `e` where 1 |
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

根据上述 MariaDB 引擎输出的分析树信息，我们能够了解 SQL 优化器是如何转换处理我们编写的 SQL 语句的。

## 优化器提示信息

优化器提示功能用于使用固定的格式告知优化器应该如何去制定执行计划，以解决某些时候创建的执行计划并不如意的情况。

### 优化提示的使用方法

```
select * from employees use index (primary) where emp_no=10001;
select * from employees /*! use index (primary) */ where emp_no=10001;
```

上述两种方式的作用是一致的，都是直接显式使用主键索引执行查询。第一个 SQL 并未使用注释标记的方式， 直接将提示作为 SQL 语句的一部分；第二个 SQL 语句将提示以注释的形式来表示。

让我们再来看两个例子：

```
CRETAE /*!32302 TEMPORARY */ TABLE temp_emp_stat (hire_year int not null, emp_count int, primary key (hire_year));

CREATE TEMPORARY table temp_emp_stat (hire_year int not null, emp_count int, primary key (hire_year));
```

第一个 SQL 语句中 “32302” 表示 MariaDB 版本号为: 3.23.02，"!32302" 表示在版本 3.32.02 以上时，会创建为临时表。即与第二个 SQL 执行同样的功能。如果低于 3.23.02 版本， 则只会创建普通数据表。

MariaDB 的版本号由三部分组成：第一个句点前面的是主版本号，第二个句点前面的是次版本号，最后一个数字是修订版本号。 比如 5.5.8 版本，使用提示信息需要写为： `/*!50508 TEMPORARY */` 。 使用这种版本提示信息的方式，能够帮助我们处理很多兼容性版本问题。

### STRAIGHT_JOIN

使用 STRAIGHT_JOIN 能够帮助我们在多个关联查询语句时，固定连接的顺序。 下面的这个查询有3个连接，但是无法判断出哪个是驱动表，哪个是被驱动表。优化器会根据每时每刻各数据表的统计信息与查询条件按最优顺序连接数据表。

```
explain select * from employees e, dept_emp de, departments d where e.emp_no=de.emp_no and d.dept_no=de.dept_no;
+------+-------------+-------+--------+---------------------------+-------------+---------+---------------------+-------+-------------+
| id   | select_type | table | type   | possible_keys             | key         | key_len | ref                 | rows  | Extra       |
+------+-------------+-------+--------+---------------------------+-------------+---------+---------------------+-------+-------------+
|    1 | SIMPLE      | d     | index  | PRIMARY                   | ux_deptname | 162     | NULL                |     9 | Using index |
|    1 | SIMPLE      | de    | ref    | PRIMARY,ix_empno_fromdate | PRIMARY     | 16      | employees.d.dept_no | 36844 |             |
|    1 | SIMPLE      | e     | eq_ref | PRIMARY                   | PRIMARY     | 4       | employees.de.emp_no |     1 |             |
+------+-------------+-------+--------+---------------------------+-------------+---------+---------------------+-------+-------------+
```

从执行计划可以看出 departments 表是驱动表，然后读取 dept_emp ，最后读取 employees 表。一般来说，用于连接的列的索引是否可用将决定连接顺序，然后再选择 WHERE 条件过滤之后，记录数最小的数据表作为驱动表。

如果我们需要更改连接的顺序，可用使用 STRAIGHT_JOIN 。

```
explain select STRAIGHT_JOIN e.first_name,e.last_name,d.dept_name from employees e, dept_emp de, departments d where e.emp_no=de.emp_no and d.dept_no=de.dept_no;

explain select /*! STRAIGHT_JOIN */ e.first_name,e.last_name,d.dept_name from employees e, dept_emp de, departments d where e.emp_no=de.emp_no and d.dept_no=de.dept_no;

+------+-------------+-------+--------+---------------------------+-------------------+---------+----------------------+--------+-------------+
| id   | select_type | table | type   | possible_keys             | key               | key_len | ref                  | rows   | Extra       |
+------+-------------+-------+--------+---------------------------+-------------------+---------+----------------------+--------+-------------+
|    1 | SIMPLE      | e     | ALL    | PRIMARY                   | NULL              | NULL    | NULL                 | 300024 |             |
|    1 | SIMPLE      | de    | ref    | PRIMARY,ix_empno_fromdate | ix_empno_fromdate | 4       | employees.e.emp_no   |      1 | Using index |
|    1 | SIMPLE      | d     | eq_ref | PRIMARY                   | PRIMARY           | 16      | employees.de.dept_no |      1 |             |
+------+-------------+-------+--------+---------------------------+-------------------+---------+----------------------+--------+-------------+
```

当显示使用 STRAIGHT_JOIN 调整连接顺序时，一定要评估根据 rows 列 看估算的返回行数，是否比调整之前返回的行数要小，否则调优只会让性能变得更差。如下的一些场景，比较适合强制使用 STRAIGHT_JOIN 调整连接顺序：

* 临时表 （内嵌视图或派生表）与普通表连接：一般选择临时表作为驱动表。
* 临时表间的连接：由于临时表或子查询生成的派生表大部分都不带索引，建议使用较小的表作为驱动表。
* 普通表间的连接：两个表的连接列都有索引或者都无所因时，选择记录数较少的数据表为驱动表更好。除此之外，建议选择连接列无索引的数据表为驱动表。

### USE INDEX / FORCE INDEX / IGNORE INDEX

某一些场景下数据表存在多个索引，MariaDB 优化器可能会选择不合适的索引，此时，可以使用索引提示强制优化器使用指定的索引。

* USE INDEX ：它只是建议 MariaDB 优化器使用特定表的索引。优化器可能会采用，也有可能不会采用。
* FORCE INDEX ：与 USE INDEX 类似，但影响范围会更大一些。
* IGNORE INDEX ：它使优化器不能使用指定索引。
* USE INDEX FOR JOIN ：注意，它不仅仅用于表连接查询，还可以用于普通的查询中。
* USE INDEX FOR ORDER BY ：明确指出用于 ORDER BY。
* USE INDEX FOR GROUP BY ：明确指出用于 GROUP BY。

### SQL_CACHE / SQL_NO_CACHE

MariaDB 的 SELECT 查询获取的结果会暂时存储到查询缓存中。使用 SQL_CACHE / SQL_NO_CACHE 提示以决定是否要将查询结果存储到查询缓存。

常见的使用场景是，使用 SQL_NO_CACHE 提示来比较不同查询语句的执行效率，因为忽略掉查询缓存对性能的影响，测试结果会更加精确一些。

## 总结：分析执行计划的注意事项

### select_type 列中需要注意的地方

#### DERIVED

DERIVED 是 FROM 子句中的子查询行程的临时表，临时表即可以存储到内存中，也可以存储到磁盘中，当数据很大并且将临时表存储到磁盘时，就会导致性能下降比较明显。

#### UNCACHEABLE SUBQUERY

当出现 UNCACHEABLE SUBQUERY 时，说明子查询无法被缓存，导致出现这种情况的原因可能是使用了用户变量或函数，因此需要考虑将用户变量或函数去掉或者找到对应的替代方案。

#### DEPENDENT SUBQUERY

DEPENDENT SUBQUERY 说明某个子查询依赖外部查询的结果，无法脱离开外部查询的结果而单独执行，因此就有可能导致整个查询都变慢。需要检查子查询是否有必要获取外部查询的值，若可能，需要去除子查询对外部查询的依赖。

### type 列中需要注意的地方

ALL ，index ：显而易见，这两种 type 值应该尽量的避免出现。

### Key 列中需要注意的地方

当查询无法使用索引时，执行计划的 Key 列将不显示任何内容。最好添加或修改 WHERE 条件，以使查询可以使用索引。

### Rows 列中需要注意的项目

Rows 列中显示的行数比查询实际获取的记录数多时，最好检查查询是否可以正常使用索引。需要注意的是，查询中带有 LIMIT 语句时，不应该在此种类型的情况考虑当中。

### Extra 列中需要注意的项目

#### 需要检查查询是否正确地表达条件

* Full scan on NULL key
* Impossible HAVING
* Impossible WHERE
* Impossible WHERE noticed after reading const tables
* No matching min/max row
* No matching row in const table
* Unique row not found

如果上述值出现在了 Extra 列当中，我们首先需要检查编写的 SQL 查询语句的业务逻辑是否正确，是否存在 Bug 。

#### 查询的执行计划不佳时

* Range checked for each record(index map: N)
* Using filesort
* Using join buffer
* Using temporary
* Using where

出现了上述值时，可以考虑尽可能的优化，尤其是执行计划的 Rows 列值比实际查询结果大很多时。

#### 查询的执行计划很好时

* Distinct
* Using index
* Using index for group-by

得到上述结果时，说明查询已经获得了优化。
