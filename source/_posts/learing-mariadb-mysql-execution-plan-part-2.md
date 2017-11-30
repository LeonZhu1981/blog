title: learing mariadb & mysql-execution plan(part-2)
date: 2017-11-30 11:08:22
categories: programming
tags:
- mysql
- mariadb
- execution plan
---

## 执行计划分析
---

只需要在对应 SELECT 查询语句前面加上 EXPLAIN ，就可以显示相关的执行计划。

<!--more-->

```
EXPLAIN SELECT e.emp_no
,e.first_name
,s.from_date
,s.salary
FROM employees e
INNER JOIN salaries s
ON e.emp_no=s.emp_no
LIMIT 1;
```

```
+------+-------------+-------+-------+---------------+--------------+---------+--------------------+--------+-------------+
| id   | select_type | table | type  | possible_keys | key          | key_len | ref                | rows   | Extra       |
+------+-------------+-------+-------+---------------+--------------+---------+--------------------+--------+-------------+
|    1 | SIMPLE      | e     | index | PRIMARY       | ix_firstname | 58      | NULL               | 300024 | Using index |
|    1 | SIMPLE      | s     | ref   | PRIMARY       | PRIMARY      | 4       | employees.e.emp_no |      9 |             |
+------+-------------+-------+-------+---------------+--------------+---------+--------------------+--------+-------------+
```

### id 列

用于以 SELECT 语句为单位（“单位 SELECT 查询”）的查询的唯一标识。

多个表的 JOIN 查询，执行计划中的 id 值是相同的（参考上面 JOIN 查询的执行计划， id 的值都为1）。

而下面的查询语句由3个“单位 SELECT 查询”组成，因此执行计划返回了不同的 id 值。

```
EXPLAIN SELECT
((SELECT COUNT(*) FROM employees) + (SELECT COUNT(*) FROM departments)) AS total_count;
```

```
+------+-------------+-------------+-------+---------------+-------------+---------+------+--------+----------------+
| id   | select_type | table       | type  | possible_keys | key         | key_len | ref  | rows   | Extra          |
+------+-------------+-------------+-------+---------------+-------------+---------+------+--------+----------------+
|    1 | PRIMARY     | NULL        | NULL  | NULL          | NULL        | NULL    | NULL |   NULL | No tables used |
|    3 | SUBQUERY    | departments | index | NULL          | PRIMARY     | 16      | NULL |      9 | Using index    |
|    2 | SUBQUERY    | employees   | index | NULL          | ix_hiredate | 3       | NULL | 300024 | Using index    |
+------+-------------+-------------+-------+---------------+-------------+---------+------+--------+----------------+
```

### select_type 列

用于标识“单位 SELECT 查询”的查询类型。具体的类型包括：

#### SIMPLE

不需要 UNION 操作或不包含子查询的简单 SELECT 查询时， select_type 为 SIMPLE。无论多复杂的查询语句， select_type 为 SIMPLE 的 “单位 SELECT 查询” 一定只有一个，通常为最外层的 SELECT 查询。

#### PRIMARY

与 SIMPLE 相对应，需要 UNION 操作或包含子查询的 SELECT 查询，位于最外层的单位查询 select_type 为 PRIMARY，并且有且只有一个。

#### UNION

由 UNION 操作联合而成的“单位 SELECT 查询”，除了第一个外（第一个为 DERIVED，本质是一个临时表，用于存储联合查询后的结果），第二个以后的所有单位 SELECT 查询的 select_type 都为 UNION。

```
EXPLAIN
SELECT * FROM (
	(SELECT emp_no FROM employees e1 LIMIT 10)
	UNION ALL
	(SELECT emp_no FROM employees e2 LIMIT 10)
	UNION ALL
	(SELECT emp_no FROM employees e3 LIMIT 10)
) tb;
```

```
+------+-------------+------------+-------+---------------+-------------+---------+------+--------+-------------+
| id   | select_type | table      | type  | possible_keys | key         | key_len | ref  | rows   | Extra       |
+------+-------------+------------+-------+---------------+-------------+---------+------+--------+-------------+
|    1 | PRIMARY     | <derived2> | ALL   | NULL          | NULL        | NULL    | NULL |     30 |             |
|    2 | DERIVED     | e1         | index | NULL          | ix_hiredate | 3       | NULL | 300024 | Using index |
|    3 | UNION       | e2         | index | NULL          | ix_hiredate | 3       | NULL | 300024 | Using index |
|    4 | UNION       | e3         | index | NULL          | ix_hiredate | 3       | NULL | 300024 | Using index |
+------+-------------+------------+-------+---------------+-------------+---------+------+--------+-------------+
```

#### DEPENDENT UNION

与 UNION select_type 类似，DENPENDENT UNION 出现在 UNION ALL 的集合查询中，DEPENDENT 表示 UNION 或 UNION ALL 联合而成的单位查询受到外部影响。

```
EXPLAIN 
SELECT *
FROM employees e1
WHERE e1.emp_no IN (
	SELECT e2.emp_no FROM employees e2 WHERE e2.first_name='Matt'
	UNION
	SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt'
);
```

```
+------+--------------------+------------+--------+----------------------+---------+---------+------+--------+-------------+
| id   | select_type        | table      | type   | possible_keys        | key     | key_len | ref  | rows   | Extra       |
+------+--------------------+------------+--------+----------------------+---------+---------+------+--------+-------------+
|    1 | PRIMARY            | e1         | ALL    | NULL                 | NULL    | NULL    | NULL | 300024 | Using where |
|    2 | DEPENDENT SUBQUERY | e2         | eq_ref | PRIMARY,ix_firstname | PRIMARY | 4       | func |      1 | Using where |
|    3 | DEPENDENT UNION    | e3         | eq_ref | PRIMARY              | PRIMARY | 4       | func |      1 | Using where |
| NULL | UNION RESULT       | <union2,3> | ALL    | NULL                 | NULL    | NULL    | NULL |   NULL |             |
+------+--------------------+------------+--------+----------------------+---------+---------+------+--------+-------------+
```

> 一个“单位 SELECT 查询”中还包含其他单位 SELECT 时，我们将内部包含的 SELECT 称为子查询。一个查询中含有子查询时，子查询通常会比外部查询优先执行，这种处理方式的速度一般会非常快。但对于 select_type 包含 DEPENDENT 关键字的子查询，由于它要依赖外部查询，因此查询的效率会比前者低。

#### UNION RESULT

UNION RESULT 为包含 UNION 结果的数据表。MariaDB 中， UNION ALL 或 UNION 会最终的结果创建临时表。因此在执行计划中，该临时表所在行的 select_type 为 UNION RESULT。由于 UNION RESULT 在实际查询中不是单位查询，所以没有单独的 id 值。

相关的执行计划输出可以参考 DEPENDENT UNION 示例中的执行计划。

#### SUBQUERY

这里的 SUBQUERY 特指除了 FROM 子句以外使用的子查询。

```
EXPLAIN
SELECT e.first_name
,(SELECT COUNT(*) 
FROM dept_emp de, dept_manager dm
WHERE dm.dept_no=de.dept_no
) AS cnt
FROM employees e
WHERE e.emp_no = 10001;
```

```
+------+-------------+-------+-------+---------------+---------+---------+----------------------+-------+-------------+
| id   | select_type | table | type  | possible_keys | key     | key_len | ref                  | rows  | Extra       |
+------+-------------+-------+-------+---------------+---------+---------+----------------------+-------+-------------+
|    1 | PRIMARY     | e     | const | PRIMARY       | PRIMARY | 4       | const                |     1 |             |
|    2 | SUBQUERY    | dm    | index | PRIMARY       | PRIMARY | 20      | NULL                 |    24 | Using index |
|    2 | SUBQUERY    | de    | ref   | PRIMARY       | PRIMARY | 16      | employees.dm.dept_no | 36844 | Using index |
+------+-------------+-------+-------+---------------+---------+---------+----------------------+-------+-------------+
```

FROM 子句当中子查询指的是:

```
SELECT fd1
FROM (
	SELECT fd2
	FROM table
) AS tb
```

这种情况下的执行计划的 select_type 为 DERIVED ，即派生表的意思。

子查询出现在不同的位置，有不同的称谓。

* 嵌套子查询（ Nested Query ）：用于 SELECT 列的子查询。
* 子查询（ Sub Query ）：一般用于 WHERE 子句的查询被称为子查询。
* 派生表（ Derived ）：FROM 子句中的子查询。

此外，根据子查询返回值的特点也可以进行区分。
* 标量子查询（ Scalar Query ）：只返回单个值（单行单列）的子查询。
* 单行子查询（ Row Sub Query ）：与列数无关，只返回单行的子查询。

#### DEPENDENT SUBQUERY

若子查询需要使用外部 SELECT 查询中定义的列，则称此子查询为 DEPENDENT SUBQUERY。

```
EXPLAIN
SELECT e.first_name
,(SELECT COUNT(*)
  FROM dept_emp de, dept_manager dm
  WHERE dm.dept_no = de.dept_no
  AND de.emp_no = e.emp_no) AS cnt
FROM employees e
WHERE e.first_name = 'Matt';
)
```

```
+------+--------------------+-------+------+---------------------------+-------------------+---------+----------------------+------+--------------------------+
| id   | select_type        | table | type | possible_keys             | key               | key_len | ref                  | rows | Extra                    |
+------+--------------------+-------+------+---------------------------+-------------------+---------+----------------------+------+--------------------------+
|    1 | PRIMARY            | e     | ref  | ix_firstname              | ix_firstname      | 58      | const                |  233 | Using where; Using index |
|    2 | DEPENDENT SUBQUERY | de    | ref  | PRIMARY,ix_empno_fromdate | ix_empno_fromdate | 4       | employees.e.emp_no   |    1 | Using index              |
|    2 | DEPENDENT SUBQUERY | dm    | ref  | PRIMARY                   | PRIMARY           | 16      | employees.de.dept_no |    2 | Using index              |
+------+--------------------+-------+------+---------------------------+-------------------+---------+----------------------+------+--------------------------+
```

需要注意的是，与 DEPENDENT UNION 类似， DEPENDENT SUBQUERY 也会先执行外部查询，在执行内部查询，所以它会比一般的子查询（不带 DEPENDENT 关键字）的处理速度要慢。

#### DERIVED

DERIVED 会在内存或者磁盘上创建临时数据表，也被称之为派生表。

```
EXPLAIN
SELECT *
FROM (SELECT de.emp_no
	FROM dept_emp de
	GROUP BY de.emp_no) tb, employees e
WHERE e.emp_no = tb.emp_no;
```

```
+------+-------------+------------+--------+---------------+-------------------+---------+-----------+--------+-------------+
| id   | select_type | table      | type   | possible_keys | key               | key_len | ref       | rows   | Extra       |
+------+-------------+------------+--------+---------------+-------------------+---------+-----------+--------+-------------+
|    1 | PRIMARY     | <derived2> | ALL    | NULL          | NULL              | NULL    | NULL      | 331603 |             |
|    1 | PRIMARY     | e          | eq_ref | PRIMARY       | PRIMARY           | 4       | tb.emp_no |      1 |             |
|    2 | DERIVED     | de         | index  | NULL          | ix_empno_fromdate | 7       | NULL      | 331603 | Using index |
+------+-------------+------------+--------+---------------+-------------------+---------+-----------+--------+-------------+
```

由于 DERVIED 会创建临时表，因此性能一般不会太好，我们需要在实际的开发过程中尽量使用 JOIN 来替代掉子查询。

#### UNCACHEABLE SUBQUERY

顾名思义，该 select_type 是指无法使用缓存结果的子查询。有如下的几种情况无法使用到缓存：

* 子查询当中包含了用户变量
* 子查询含有 NOT-DETERMINISTIC 属性的自定义函数时（关于 NOT-DETERMINISTIC 请参考[这里](https://mariadb.com/kb/en/library/create-function/#not-deterministic)）
* 子查询使用每次调用有不同结果值的函数时（比如 UUID() 或 RAND()）

```
EXPLAIN
SELECT *
FROM employees e
WHERE e.emp_no = (SELECT @status 
	FROM dept_emp de WHERE de.dept_no = 'd005'
);
```

```
+------+----------------------+-------+-------+---------------+---------+---------+-------+--------+--------------------------+
| id   | select_type          | table | type  | possible_keys | key     | key_len | ref   | rows   | Extra                    |
+------+----------------------+-------+-------+---------------+---------+---------+-------+--------+--------------------------+
|    1 | PRIMARY              | e     | const | PRIMARY       | PRIMARY | 4       | const |      1 | Using where              |
|    2 | UNCACHEABLE SUBQUERY | de    | ref   | PRIMARY       | PRIMARY | 16      | const | 165571 | Using where; Using index |
+------+----------------------+-------+-------+---------------+---------+---------+-------+--------+--------------------------+
```

#### UNCACHEABLE UNION

同上，只是换成了 UNCACHEABLE 和 UNION 的组合。

#### MATERIALIZED

MATERIALIZED 是从 MariaDB 5.3 与 Mysql 5.6 开始引入的 select_type，主要用于优化 FROM 子句或 IN(subquery) 的子查询。与 DERIVED 类似的是，MATERIALIZED 会生成临时表。

```
EXPLAIN
SELECT *
FROM employees e
WHERE e.emp_no IN (SELECT emp_no
		FROM salaries
		WHERE salary BETWEEN 100 AND 1000);
```

```
+------+--------------+-------------+--------+-------------------+-----------+---------+---------------------------+------+--------------------------+
| id   | select_type  | table       | type   | possible_keys     | key       | key_len | ref                       | rows | Extra                    |
+------+--------------+-------------+--------+-------------------+-----------+---------+---------------------------+------+--------------------------+
|    1 | PRIMARY      | <subquery2> | ALL    | distinct_key      | NULL      | NULL    | NULL                      |    1 |                          |
|    1 | PRIMARY      | e           | eq_ref | PRIMARY           | PRIMARY   | 4       | employees.salaries.emp_no |    1 |                          |
|    2 | MATERIALIZED | salaries    | range  | PRIMARY,ix_salary | ix_salary | 4       | NULL                      |    1 | Using where; Using index |
+------+--------------+-------------+--------+-------------------+-----------+---------+---------------------------+------+--------------------------+
```

可以看到，子查询部分（id = 2的记录）首先得到了处理。


#### INSERT @MariaDB

从 MariaDB 10.0 开始，可以查看 INSERT、UPDATE、DELETE 语句的执行计划（注意：Mysql 5.7 不支持）。

**执行简单的 INSERT 语句**

```
EXPLAIN
INSERT INTO employees VALUES 
(1, '2014-01-01', 'Matt', 'Lee', 'M', '2014-01-02');
```

```
+------+-------------+-----------+------+---------------+------+---------+------+------+-------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows | Extra |
+------+-------------+-----------+------+---------------+------+---------+------+------+-------+
|    1 | INSERT      | employees | ALL  | NULL          | NULL | NULL    | NULL | NULL | NULL  |
+------+-------------+-----------+------+---------------+------+---------+------+------+-------+
```

**执行 INSERT INTO ... SELECT ... 复杂语句**

这种情况下的 select_type 不再是 INSERT ，而是 SIMPLE 。

```
EXPLAIN
INSERT INTO employees 
SELECT 
e.* 
FROM employees e
INNER JOIN dept_emp de
ON e.emp_no = de.emp_no
WHERE e.hire_date > '2014-01-01';
```

```
+------+-------------+-------+-------+---------------------+-------------------+---------+--------------------+------+----------------------------------------+
| id   | select_type | table | type  | possible_keys       | key               | key_len | ref                | rows | Extra                                  |
+------+-------------+-------+-------+---------------------+-------------------+---------+--------------------+------+----------------------------------------+
|    1 | SIMPLE      | e     | range | PRIMARY,ix_hiredate | ix_hiredate       | 3       | NULL               |    1 | Using index condition; Using temporary |
|    1 | SIMPLE      | de    | ref   | ix_empno_fromdate   | ix_empno_fromdate | 4       | employees.e.emp_no |    1 | Using index                            |
+------+-------------+-------+-------+---------------------+-------------------+---------+--------------------+------+----------------------------------------+
```

**普通的 UPDATE 或 DELETE 语句**

```
EXPLAIN 
UPDATE employees SET gender = 'F'
WHERE first_name = 'Matt';
```

```
+------+-------------+-----------+-------+---------------+--------------+---------+------+------+-------------+
| id   | select_type | table     | type  | possible_keys | key          | key_len | ref  | rows | Extra       |
+------+-------------+-----------+-------+---------------+--------------+---------+------+------+-------------+
|    1 | SIMPLE      | employees | range | ix_firstname  | ix_firstname | 58      | NULL |  233 | Using where |
+------+-------------+-----------+-------+---------------+--------------+---------+------+------+-------------+
```

### table 列

table 列标识了执行计划中使用的具体的表。如果查询语句中有别名，则会使用别名。

如果查询语句中没有包含具体的表，table 列会显示为 NULL。例如：

```
EXPLAIN SELECT NOW();

+------+-------------+-------+------+---------------+------+---------+------+------+----------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra          |
+------+-------------+-------+------+---------------+------+---------+------+------+----------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | No tables used |
+------+-------------+-------+------+---------------+------+---------+------+------+----------------+
```

此外，如果 table 列中出现了以 <> 尖括号括起来的情况（例如：<derived N> 或 <union M, N> ），这表示使用了临时表。在 <> 出现了数字，表示的是单位 SELECT 查询的 id 列。

### id/select_type/table 列的总结

id/select_type/table 决定了 MariaDB 按照什么样的顺序来执行查询。

```
EXPLAIN 
SELECT *
FROM (SELECT emp_no
FROM dept_emp GROUP BY emp_no) as department_employee
INNER JOIN employees e
ON e.emp_no = department_employee.emp_no;

+------+-------------+------------+--------+---------------+-------------------+---------+----------------------------+--------+-------------+
| id   | select_type | table      | type   | possible_keys | key               | key_len | ref                        | rows   | Extra       |
+------+-------------+------------+--------+---------------+-------------------+---------+----------------------------+--------+-------------+
|    1 | PRIMARY     | <derived2> | ALL    | NULL          | NULL              | NULL    | NULL                       | 331603 |             |
|    1 | PRIMARY     | e          | eq_ref | PRIMARY       | PRIMARY           | 4       | department_employee.emp_no |      1 |             |
|    2 | DERIVED     | dept_emp   | index  | NULL          | ix_empno_fromdate | 7       | NULL                       | 331603 | Using index |
+------+-------------+------------+--------+---------------+-------------------+---------+----------------------------+--------+-------------+
```

从上面的执行计划可以看出：

* 第一行 table 列为 <derived2>，它表示 id 为 2 的查询先执行，查询结果会保存到派生临时表中。
* 从第三行 id 为 2 的查询来看， select_type 列值为 DERIVED，它读取了 dept_emp 表的数据并创建了派生表。
* 第三行分析完成了之后，再开始执行第一行。
* 由于第一行和第二行有相同的 id 值，因此需要将 <derived2> 和 e 表 JOIN 起来。需要注意的是，<derived2> 表显示在 e 表之上，因此 <derived2> 是驱动表，e 是被驱动表。进一步来说，就是先读取 <derived2> 表，再和 e 表进行 JOIN 操作。



