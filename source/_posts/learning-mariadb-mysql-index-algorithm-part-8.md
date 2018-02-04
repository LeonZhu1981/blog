title: learning mariadb & mysql index and algorithm(part-8)
date: 2018-01-31 14:03:19
categories: programming
tags:
- mysql
- mariadb
- index
- algorithm
---

# InnoDB 存储引擎索引概述

InnoDB 存储引擎支持以下几种常见的索引：

* B+树索引
* 全文索引
* 哈希索引

InnoDB 所支持的哈希索引是自适应的，会根据表的使用情况自动生成哈希索引，并且无法人为干预。

B+树索引就是传统意义上的索引，构造类似于二叉树，根据 Key Value 快速找到数据。需要注意的是，B+树的B不是代表二叉（ binary ），而是表示平衡（ balance ），因为 B+树是从最早的平衡二叉树演化而来的。需要注意的是：**B+树索引并不能找到一个给定键值的具体行，而只是能查找到数据行所在的页，通过把页读入到内存之后，再在内存上进行查找到具体的行**。

<!--more-->

# 数据结构和算法

## 二分查找 （ binary search ）

关于 binary search 算法，可以参考[这里](http://www.zhuxiaodong.net/2017/algorithems-binary-search/) 。

InnoDB 存储引擎中，二分查找主要运用在：当数据从磁盘上的页读取了到内存之后，通过 Page Directory （页目录）进行二分查找，能够快速地找到对应的记录（ O(log2n) 时间复杂度 ）。

Page Directory（页目录）中存放了记录的相对位置（注意，这里存放的是页相对位置，而不是偏移量），有些时候这些记录指针称为Slots（槽）或目录槽（Directory Slots）。与其他数据库系统不同的是，在 InnoDB 中并不是每个记录拥有一个槽，InnoDB 存储引擎的槽是一个稀疏目录（sparse directory），即一个槽中可能包含多个记录。

## 二叉查找树和平衡二叉树

在了解B+树之前，我们需要先了解一下二叉查找树。B+树是通过二叉查找树，再由平衡二叉树，B树演化而来。

![btree-01](http://static.zhuxiaodong.net/blog/static/images/btree-01.png)

上述图中表示一个二叉查找树，图中的每一个数字代表每个节点的键值，在二叉查找树中，左子树的键值总是小于根的键值，右子树的键值总是大于根的键值。因此我们可以通过中序遍历得到键值的排序输出：2、3、5、6、7、8。

对于这颗树的查找，例如查找键值为5的记录，其查找过程为：先找到最顶端的根节点，其键值是6，6大于5，因此查找6的左子树，找到3；而5大于3，再找其右子树；一共查找了3次。让我们对比一下顺序查找的方式，同样条件下，其查找次数为：3次。同样的，对于 5 和 8 两个数字的查找，二叉查找树的查找次数都是为3次，而顺序查找则为：3次（查找键值5） 和 6次（查找键值8）。

二叉查找树的平均查找次数为：（3 + 3 + 2 + 2 + 1）/ 6 = 2.3 次，而顺序查找的平均值为：（1 + 2 + 3 + 4 + 5 + 6）/ 6 = 3.3 次。

当然，并不是每一个二叉查找树的查询效率都会很高，我们来看下面一个例子：

![btree-02](http://static.zhuxiaodong.net/blog/static/images/btree-02.png)

通过二叉查找树的查找算法可以看出，这颗树的查询效率会很低，因为数据集中在了树的右侧。平均查找次数为 (1 + 2 + 3 + 4 + 5 + 5) / 6 = 3.16 次。

因此，如果想最大性能地构造一颗二叉查找树，需要这颗二叉树是平衡的，这就是平衡二叉树 （ AVL 树 ）。
平衡二叉树是在二叉查找树的基础上，**必须满足任何节点的两个子树的高度最大差为1。** 刚才我们举例的图1就是一颗平衡二叉树，而图2则不是。

平衡二叉树在写入时，由于需要维持其平衡性，因此维护的代价较大，需要1次或多次左旋和右旋来得到插入或更新后树的平衡性。让我们来举例说明这个问题：

![btree-03](http://static.zhuxiaodong.net/blog/static/images/btree-03.png)

上图中，当需要插入一个新的键值为9的节点时，需要将7这个节点左旋来保证平衡。

再来看一个更复杂的例子：

![btree-04](http://static.zhuxiaodong.net/blog/static/images/btree-04.png)

## B+树

B+树由B树和索引顺序访问方法（ ISAM ， Index sequence access method 【 MyISAM 最初就是参考 ISAM 数据结构来开发的 】）演化而来的。B+树是为磁盘或其它直接存取辅助设备设计的一种平衡查找树，所有记录节点都是按键值的大小顺序存放在同一层的叶子节点上，由各叶子节点指针进行连接。

下面这个B+树，其高度为2，每页可以存放4条记录，扇出（ fan out ）为5。所有记录都在叶子节点上，并且是顺序存放的，如果用户从最左边开始循序遍历，可以得到所有键值的顺序排序。

![b+tree-01](http://static.zhuxiaodong.net/blog/static/images/b+tree-01.png)

### B+树的插入操作

B+树的插入必须保证插入后的叶子节点中的记录依然是排好序的，其中有三种情况需要考虑：

| Leaf Page 满     | Index Page 满     | 操作   |
| :-------------: | :-------------: | :------------- |
| no       | no       | 直接将记录插入到叶子节点 |
| yes       | no       | 1、拆分 Leaf Page<br/>2、将中间的节点放入到 Index Page中<br/> 3、小于中间节点的记录放左边<br/>4、大于或等于中间节点的记录放右边<br/>|
| yes       | yes       | 1、拆分 Leaf Page<br/>2、小于中间节点的记录放左边<br/> 3、大于或等于中间节点的记录放右边<br/>4、拆分 index Page<br/>5、小于中间节点的记录放左边<br/>6、大于中间节点的记录放右边<br/>7、大于中间节点的记录放右边 |

这里用一个例子来分析B+树的插入。例如，对于下图中的这颗B+树，若用户插入 28 这个键值，发现当前的 Leaf Page 和 Index Page 都没有写满，直接插入即可，之后得到的结果为：

![b+tree-02](http://static.zhuxiaodong.net/blog/static/images/b+tree-02.png)

接着再插入 70 这个键值，这时原先的 Leaf Page 已经满了，但是 Index Page 还没有满，因此插入 Leaf Page 后的情况为 50、55、60、65、70，并根据中间的值60来拆分叶子节点。

![b+tree-03](http://static.zhuxiaodong.net/blog/static/images/b+tree-03.png)

最后插入键值95，即 Leaf Page 和 Index Page 都满了，这时需要做一次 Leaf Page 的拆分和一次 Index Page 的拆分。

![b+tree-04](http://static.zhuxiaodong.net/blog/static/images/b+tree-04.png)

可以通过上述过程中分析得出，无论如何变化，B+ 树始终会保持平衡。但是为了保持平衡，对于新插入的键值需要做大量的拆分页（ split ）操作，由于B+ 树的主要运用磁盘，因此页拆分会产生大量的磁盘操作，所以应该尽量减少页的拆分。B+树提供了类似于AVL树的旋转（ Rotation ）功能。

Rotation 发生在 Leaf Page 已满，但是其的左右兄弟节点没有满的情况下。这时 B+树并不会急于去做拆分页的操作，而是将记录移动到所在页的兄弟节点上。在通常的情况下，左兄弟会首先检查用来做旋转操作。以上面的情况为例，若插入键值70，由于左边的兄弟 Leaf Page 并没有满，这样做一次左旋转操作之后，得到如下的图示效果。可以看到采用 Rotation 操作减少了一次页的拆分，同时这颗B+树的高度依然还是2。

![b+tree-05](http://static.zhuxiaodong.net/blog/static/images/b+tree-05.png)

### B+树的删除操作

B+树使用填充因子（ fill factor ）来控制树的删除变化，50%是填充因子可以设置的最小值。B+树的删除操作同样必须要保证删除后叶子节点的记录是排序的。删除操作会有如下3种情况：

| 叶子节点小于填充因子 | 中间节点小于填充因子     | 操作 |
| :------------- | :------------- | :----------- |
| No       | No       | 直接将记录从叶子节点删除，如果该节点还是 Index Page 的节点，用该几点的右节点代替 |
| Yes       | No       | 合并叶子节点和它的兄弟节点，同时更新 Index Page |
| Yes       | Yes       | 1) 合并叶子节点和它的兄弟节点<br/>2) 更新 Index Page<br/>3) 合并 Index Page 和它的兄弟节点 |

下图是我们在 b+tree-04.png 图示的基础上，删除键值为70的这条记录，它符合上表讨论的第一种情况，删除了之后得到如下图示：

![b+tree-06](http://static.zhuxiaodong.net/blog/static/images/b+tree-06.png)

接着删除键值为25的记录，它也符合上表讨论的第一种情况，只不过由于25在 Index Page 中，因此还需要用它的右节点 28 来做一次更新。

![b+tree-07](http://static.zhuxiaodong.net/blog/static/images/b+tree-07.png)

最后看删除键值为60的情况。由于 Leaf Page 删除了60之后， Fill Factor 小于 50%，这是需要做合并操作，此外，Index Page 中 也存在 60 的记录，也需要做合并操作。

![b+tree-08](http://static.zhuxiaodong.net/blog/static/images/b+tree-08.png)

## B+树索引

B+树索引的本质就是B+树在数据库的实现，但是B+树索引在数据库中有一个特点就是高扇出性，因此在数据库中，树的高度一般都是 2 ~ 4 层，这意味着查找某一键值的行记录时，最多只需要2到4次 IO ，一般的机械硬盘每秒至少能够做 100 次 IO， 2 ~ 4 次的 IO 意味着查询时间只需要 0.02 ~ 0.04 秒。

数据库中的B+树索引可以分为：聚集索引（ clustered index ）和辅助索引（ secondary index ），其内部都是B+树实现的，叶子节点存放着所有的数据。其中聚集索引的叶子节点存放了一整行的信息。

### 聚集索引 （ clustered index ）

聚集索引是按照每张表的主键构造出一颗B+树，同时叶子节点存放了整张表的行记录数据，叶子节点也被称为数据页。聚集索引的这个特性决定了索引组织表中数据页是索引的一部分。每一个数据页都通过一个双向链表来进行连接。

由于实际的数据页只能按照一颗B+树进行排序，因此每张表只能拥有一个聚集索引。

我们接下来创建一张表来做一下测试，这里以人为的方式让每个页中只能存放两个行的记录（插入列 b 的长度为7000）。

```
/*!40101 SET NAMES utf8 */;
CREATE TABLE IF NOT EXISTS table_test (
	`a` int NOT NULL
	,`b` varchar(8000)
	,`c` int NOT NULL
	,CONSTRAINT PK_table_test_a PRIMARY KEY (a)
)engine=InnoDB;

CREATE INDEX IX_b ON table_test(c);

INSERT INTO table_test SELECT 1,REPEAT('a',7000),-1;
INSERT INTO table_test SELECT 2,REPEAT('a',7000),-2;
INSERT INTO table_test SELECT 3,REPEAT('a',7000),-3;
INSERT INTO table_test SELECT 4,REPEAT('a',7000),-4;
```

然后使用 py_innodb_page_info 工具来进行分析表空间。

> 该工具可以从 https://github.com/qingdengyue/david-mysql-tools/ 进行下载
> git clone https://github.com/qingdengyue/david-mysql-tools.git
> cd py_innodb_page_type
> chmod 755 *

```
./py_innodb_page_info.py -v /alidata1/mysql/9000/data/hms2/table_test.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0001>
page offset 00000004, page type <B-tree Node>, page level <0000>
page offset 00000005, page type <B-tree Node>, page level <0000>
page offset 00000006, page type <B-tree Node>, page level <0000>
page offset 00000007, page type <B-tree Node>, page level <0000>
page offset 00000000, page type <Freshly Allocated Page>
Total number of page: 9:
Freshly Allocated Page: 1
Insert Buffer Bitmap: 1
File Space Header: 1
B-tree Node: 5
File Segment inode: 1
```

其中 page level <0000> 表示数据页，而我们这里需要分析的是 index page ：page level <0001> 的页，当前聚集索引的高度为 2 （因为 page level 只有0000 和 0001 两个层级），因此 page level <0001> 页是B+树的根。

使用 hexdump 工具来分析 table_test.ibd 二进制文件，可以得到（直接从0000c000 开始分析的原因是: page level <0001> 的 page offset 00000003，也就是第三个数据页，按照每页 16k 来计算， 16 * 1024 * 3 = 49152 byte 转换为16进制就是为: 0xc000 ）:

```
hexdump -Cv  /alidata1/mysql/9000/data/hms2/table_test.ibd > table_test.txt
vim table_test.txt

## 从 0000c000 位置开始分析
0000c000  79 28 0f 14 00 00 00 03  ff ff ff ff ff ff ff ff  |y(..............|
0000c010  00 00 00 01 14 26 98 51  45 bf 00 00 00 00 00 00  |.....&.QE.......|
0000c020  00 00 00 00 04 29 00 02  00 a2 80 05 00 00 00 00  |.....)..........|
0000c030  00 9a 00 02 00 02 00 03  00 00 00 00 00 00 00 00  |................|
0000c040  00 01 00 00 00 00 00 00  09 56 00 00 04 29 00 00  |.........V...)..|
0000c050  00 02 00 f2 00 00 04 29  00 00 00 02 00 32 01 00  |.......).....2..|
0000c060  02 00 1b 69 6e 66 69 6d  75 6d 00 04 00 0b 00 00  |...infimum......|
0000c070  73 75 70 72 65 6d 75 6d  00 10 00 11 00 0e 80 00  |supremum........|
0000c080  00 01 00 00 00 05 00 00  00 19 00 0e 80 00 00 02  |................|
0000c090  00 00 00 06 00 00 00 21  ff d6 80 00 00 04 00 00  |.......!........|
0000c0a0  00 07 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0000c0b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0000c0c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0000c0d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0000c0e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
0000c0f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
... 中间部分省略
0000fff0  00 00 00 00 00 70 00 63  79 28 0f 14 14 26 98 51  |.....p.cy(...&.Q|
```

从最后一行 0000fff0（页尾的 Page Directory 来分析此页） 的 0063 可以知道该页的开始行位置为: 0xc063 。 `0000c060  02 00 1b 69 6e 66 69 6d  75 6d 00 04 00 0b 00 00  |...infimum......|` 这行是从 0xc060 开始的，往前数3个字节，即从 `69 6e 66 69 6d 75 6d 00` 表示 infinum 为行记录, 之前的5个字节 `01 00 02 00 1b` 就是 Recorder Header 。 其中 Recorder Header 中的最后2个字节 `00 1b` 来推出下一条记录的位置，即 c063+1b = c07e ，c07e 位置开始之后的4个字节为: `80 00 00 01`，这就是我们写入的 table_test 表中主键为1的键值。`80 00 00 01` 后面的 `00 00 00 05` 表示数据页的页号。同样的方式我们可以找到 `80 00 00 02` 和 `80 00 00 04` 这两个键值以及它们指向的数据页。

通过上述的对非数据页节点的分析，可以发现数据页上存放的是完整的每行的记录，而在非数据页的索引页中，存放的仅仅是键值以及指向数据页的偏移量，而不是完整的行记录。其大致构造为：

![cluster-index-structure](http://static.zhuxiaodong.net/blog/static/images/cluster-index-structure.png)

聚集索引的存储并不是物理连续而是逻辑连续的，这其中有两点：一是页通过双向链表连接，页按照主键的顺序排序；二是每个页中的记录也是通过双向链表进行维护的，物理存储上可以同样不按照主键存储。

### 辅助索引（二级索引 非聚集索引 secondary index）

辅助索引与聚集索引最大的区别是，辅助索引的叶子节点并不包括行记录的全部数据。叶子节点除了包含键值以外，每个叶子节点中的索引行中还包含一个书签（ bookmark ）。该书签用来告诉 InnoDB 存储引擎哪里可以找到与索引相对应的行数据，InnoDB 存储引擎的辅助索引的书签就是相应行数据的聚集索引键。每张表可以有多个辅助索引。当通过辅助索引来查询数据时， InnoDB 存储引擎会遍历辅助索引并通过叶子节点的指针获得指向主键索引的主键，然后再通过主键索引来找到一个完整的行记录。

例如在一颗高度为3的辅助索引树中查找数据，就需要对这颗辅助索引树遍历3次找到指定主键，如果聚集索引树的高度同样为3，那么还需要对聚集索引树进行3次查找，最终才能找到完整的行数据所在的页，一共需要6次逻辑 IO 。

接下来我们会通过阅读表空间文件来分析一下 InnoDB 存储引擎中对非聚集索引的实际存储，使用之前的 `table_test` 表来做示例，主要对 c 列进行非聚集索引的分析。

```
UPDATE table_test SET c = 0-a;

show index from table_test;
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table      | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| table_test |          0 | PRIMARY  |            1 | a           | A         |           4 |     NULL | NULL   |      | BTREE      |         |               |
| table_test |          1 | IX_b     |            1 | c           | A         |           4 |     NULL | NULL   |      | BTREE      |         |               |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+

select a,c from table_test;
+---+----+
| a | c  |
+---+----+
| 4 | -4 |
| 3 | -3 |
| 2 | -2 |
| 1 | -1 |
+---+----+
```

然后使用 py_innodb_page_info 工具来分析表空间：

```
./py_innodb_page_info.py -v /alidata1/mysql/9000/data/hms2/table_test.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0001>
page offset 00000004, page type <B-tree Node>, page level <0000>
page offset 00000005, page type <B-tree Node>, page level <0000>
page offset 00000006, page type <B-tree Node>, page level <0000>
page offset 00000007, page type <B-tree Node>, page level <0000>
page offset 00000000, page type <Freshly Allocated Page>
Total number of page: 9:
Freshly Allocated Page: 1
Insert Buffer Bitmap: 1
File Space Header: 1
B-tree Node: 5
File Segment inode: 1
```

其中 `page offset 00000004, page type <B-tree Node>, page level <0000>` 就是非聚集索引所在的页，通过 16 * 1024 * 4 = 65536 = 0x1000 ，再使用 hexdump 计算查看 0x1000 地址。

```
hexdump -Cv  /alidata1/mysql/9000/data/hms2/table_test.ibd > table_test.txt

00010000  78 a5 01 67 00 00 00 04  ff ff ff ff ff ff ff ff  |x..g............|
00010010  00 00 00 01 14 26 98 77  45 bf 00 00 00 00 00 00  |.....&.wE.......|
00010020  00 00 00 00 04 29 00 02  00 ac 80 06 00 00 00 00  |.....)..........|
00010030  00 a4 00 01 00 03 00 04  00 00 00 00 00 03 f4 f3  |................|
00010040  00 00 00 00 00 00 00 00  09 57 00 00 04 29 00 00  |.........W...)..|
00010050  00 02 02 72 00 00 04 29  00 00 00 02 01 b2 01 00  |...r...)........|
00010060  02 00 41 69 6e 66 69 6d  75 6d 00 05 00 0b 00 00  |..Ainfimum......|
00010070  73 75 70 72 65 6d 75 6d  00 00 10 ff f3 7f ff ff  |supremum........|
00010080  ff 80 00 00 01 00 00 18  ff f3 7f ff ff fe 80 00  |................|
00010090  00 02 00 00 20 ff f3 7f  ff ff fd 80 00 00 03 00  |.... ...........|
000100a0  00 28 ff f3 7f ff ff fc  80 00 00 04 00 00 00 00  |.(..............|
......
00013ff0  00 00 00 00 00 70 00 63  78 a5 01 67 14 26 98 77  |.....p.cx..g.&.w|
```

下图显示了 table_test 表中的 IX_b 辅助索引与聚集索引之间的关系。可以看到辅助索引中的叶子节点包含了列 c 的值和主键的值。（例如：-1以 7f ff ff ff的方式进行存储，并存储了指向聚集索引主键的指针： 80 00 00 01 ）。

![secondary-index-structure](http://static.zhuxiaodong.net/blog/static/images/secondary-index-structure.png)

## B+树索引的管理

创建或删除索引可以通过 ALTER TABLE 或是 CREATE/DROP INDEX 语法来管理。

我们可以对整个列的数据进行索引，也可以只索引一个列的开头部分的数据，例如对 table_test 表的 b 字段的前100个字段进行索引。

```
ALTER TABLE table_test ADD KEY ix_b(b(100));
```

查看索引的信息可以通过 SHOW INDEX FROM TABLE 命令：

```
ALTER TABLE table_test ADD KEY ix_a_c(a, c);

show index from table_test;
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table      | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| table_test |          0 | PRIMARY  |            1 | a           | A         |           4 |     NULL | NULL   |      | BTREE      |         |               |
| table_test |          1 | ix_a_c   |            1 | a           | A         |           4 |     NULL | NULL   |      | BTREE      |         |               |
| table_test |          1 | ix_a_c   |            2 | c           | A         |           4 |     NULL | NULL   |      | BTREE      |         |               |
| table_test |          1 | ix_b     |            1 | b           | A         |           1 |      100 | NULL   | YES  | BTREE      |         |               |
| table_test |          1 | ix_c     |            1 | c           | A         |           4 |     NULL | NULL   |      | BTREE      |         |               |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
```

SHOW INDEX FROM TABLE 的几个关键字段需要解释一下：

* Seq_in_index ：标识了联合索引中该字段在索引中的位置。例如第三行中 c 字段 ix_a_c 索引中的位置为第2个。
* Collation ： 列以什么方式存储在索引中。可以是 A 或是 NULL。B+树索引总是为 A 。 如果使用了 Heap 存储引擎，并且建立了 Hash 索引，这里就会显示为 NULL 。
* Cardinality ： 表示索引中唯一值的数量的**估计值**。该字段是索引建立是否合理的重要判断依据，如果接近于1，说明重复的值非常多。
* Sub_part ： 是否是字段的部分值被索引。例如 ix_b 这个索引，显示的 Sub_part 为100。
* Packed ： 表示如何被压缩，如果没有被压缩，则为 NULL。
* NULL ： 表示索引的列中是否含有 NULL 值。
* Index_type ： 索引的类型。

需要注意的是 Cardinality 作为 SQL 优化器判断是否使用该索引的重要依据，在某些场景下可能出现该值不准备的问题。此时就需要通过 ANALYZE TABLE 命令来更新 Cardinality 的值，由于 ANALYZE TABLE 命令比较耗费性能，需要尽量放在系统的低峰时期运行。

## Cardinality 值

### 什么是 Cardinality

并不是在所有的查询条件中出现的列都需要添加索引，我们需要根据该字段在表中是否唯一值比例比较高来进行判断。

下面的 estateName 字段的唯一性就比较好，接近 100% ，适合创建辅助索引。当 estateName 这个字段没有创建辅助索引时，可以看到执行计划中的 type 列为 ALL ，即全表扫描。

```
select count(distinct estateName) / count(estateName) as cardinality from estate_basicinfo;
+-------------+
| cardinality |
+-------------+
|      0.9238 |
+-------------+

explain select estateName from estate_basicinfo where estatename = '中建梅溪湖中心';
+----+-------------+------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table            | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | estate_basicinfo | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9402 |    10.00 | Using where |
+----+-------------+------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

接下来我们为该字段创建索引， Cardinality 的值为9402， 并且其执行计划的 type 就变成了 ref ，其查询性能得到了较大的提升。

```
CREATE INDEX ix_estateName ON estate_basicinfo(estateName);

*************************** 5. row ***************************
        Table: estate_basicinfo
   Non_unique: 1
     Key_name: ix_estateName
 Seq_in_index: 1
  Column_name: estateName
    Collation: A
  Cardinality: 9402
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:

explain select estateName from estate_basicinfo where estatename = '中建梅溪湖中心';
+----+-------------+------------------+------------+------+---------------+---------------+---------+-------+------+----------+-------------+
| id | select_type | table            | partitions | type | possible_keys | key           | key_len | ref   | rows | filtered | Extra       |
+----+-------------+------------------+------------+------+---------------+---------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | estate_basicinfo | NULL       | ref  | ix_estateName | ix_estateName | 802     | const |    1 |   100.00 | Using index |
+----+-------------+------------------+------------+------+---------------+---------------+---------+-------+------+----------+-------------+
```

### InnoDB 存储引擎对 Cardinality 的统计

在实际的场景中，索引的更新操作可能是非常频繁的。如果每次在发生数据更新操作时，就对其 Cardinality 值进行更新，那么会给数据库带来较大的负担。同时，如果一张表的数据量很大，统计一次 Cardinality 的值可能会耗费较长的时间。因此，数据库会采用采样（ sample ）的方式来更新。

在 InnoDB 存储引擎中， Cardinality 统计信息的更新发生在：Insert 和 Update 。其更新策略为：
* 表中 1/16 的数据已发生过变化。
* stat_modified_counter > 2 000 000 000 ： stat_modified_counter 计数器记录了 UPDATE 行变化的次数。

InnoDB 存储引擎对 Cardinality 的更新主要是使用采样的方式，其主要过程为：
* 取得 B+树索引中叶子节点的数量，记为 A 。
* 随机取得 B+树索引中的8个叶子节点。统计每个页不同记录的个数，即为 P1 ， P2 ， ... ， P8 。
* 根据采样信息给出 Cardinality 的预估值： Cardinality = (P1 + P2 + ... + P8) * A / 8 。

由于是采样了8个叶子节点的值，预估计算得到的，因此 Cardinality 不是一个精准的值。另外，由于是随机采样，因此 Cardinality 的值可能在每次查询时得到的值不一致。

与 Cardinality 相关重要的参数设置包括：

| 参数 | 说明     |
| :------------- | :------------- |
| innodb_stats_persistent       | 是否将 ANALYZE TABLE 计算得到的 Cardinality 值存放到磁盘上。默认值：ON       |
| innodb_stats_on_metadata       | 通过 SHOW TABLE STATUS 、 SHOW INDEX 以及访问 INFORMATION_SCHEMA 数据库下的表 TABLE 和 STATISTICS 时， 是否需要重新计算索引的 Cardinality 值。默认值：OFF      |
| innodb_stats_persistent_sample_pages       | 如果 innodb_stats_persistent 设置为 ON ，该参数表示 ANALYZE TABLE 更新 Cardinality 值时每次采样页的数量。默认值：20 。      |
| innodb_stats_transient_sample_pages       | 每次采样页的数量。默认值：8 。      |

## B+树索引的使用

### 联合索引

联合索引是指对表上的多个列进行索引。让我们来看下面这个例子，t 表上创建了一个联合索引 ix_a_b 。

```
CREATE TABLE t (
    a INT,
    b INT,
    c INT,
    PRIMARY KEY (a),
    KEY ix_b_c (b,c)
)ENGINE=INNODB;

INSERT INTO t (a, b, c) VALUES (1, 1, 1);
INSERT INTO t (a, b, c) VALUES (2, 1, 2);
INSERT INTO t (a, b, c) VALUES (3, 2, 1);
INSERT INTO t (a, b, c) VALUES (4, 2, 4);
INSERT INTO t (a, b, c) VALUES (5, 3, 1);
INSERT INTO t (a, b, c) VALUES (6, 3, 2);
```

联合索引其实也是一颗 B+树，不同的是联合索引的键值的数量不是1，而是大于等于2。

![union-index](http://static.zhuxiaodong.net/blog/static/images/union-index.png)

上述的图示中，表示 t 表中存储的 ix_a_b 索引的结构，其中键值都是排序的，通过叶子节点可以逻辑上顺序地读出所有数据，即 (1, 1) 、(1, 2) 、 (2, 1) 、 (2, 4) 、 (3, 1) 、 (3, 2) 。按照 (a, b) 的顺序进行存放。

根据这个存放顺序，我们可以得出以下的结论：

```
### 能够正确地使用 ix_b_c 索引
explain SELECT * FROM t WHERE b = 1 AND c = 2;
+----+-------------+-------+------------+------+---------------+--------+---------+-------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key    | key_len | ref         | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+--------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | t     | NULL       | ref  | ix_b_c        | ix_b_c | 10      | const,const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+--------+---------+-------------+------+----------+-------------+

### 能够正确地使用 ix_b_c 索引
explain SELECT * FROM t WHERE b = 1;
+----+-------------+-------+------------+------+---------------+--------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key    | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+--------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | t     | NULL       | ref  | ix_b_c        | ix_b_c | 5       | const |    2 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+--------+---------+-------+------+----------+-------------+

### 虽然使用到了 ix_b_c 索引， 但是只能使用索引扫描的方式。
explain SELECT * FROM t WHERE c = 1;
+----+-------------+-------+------------+-------+---------------+--------+---------+------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys | key    | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | t     | NULL       | index | NULL          | ix_b_c | 10      | NULL |    6 |    16.67 | Using where; Using index |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+------+----------+--------------------------+
```

我们重点分析一下第三段 SQL ，可以发现对 c 字段单独查询的时候，只能索引全扫描的方式。原因是由于对叶子节点的 c 列的值 1、2、1、4、1、2 并没有进行排序，因此只能采用索引全扫描的方式。

联合索引的第二个好处是已经对第二个键值进行了排序处理。在下面这个示例中，我们将模拟电商当中需要查询某个用户最近三次购买记录的场景：

```
CREATE TABLE buy_log(
    userID int unsigned not NULL
    ,buyDate DATE
)ENGINE=INNODB;

INSERT INTO buy_log (userID, buyDate) VALUES (1, '2009-01-01');
INSERT INTO buy_log (userID, buyDate) VALUES (2, '2009-01-01');
INSERT INTO buy_log (userID, buyDate) VALUES (3, '2009-01-01');
INSERT INTO buy_log (userID, buyDate) VALUES (1, '2009-02-01');
INSERT INTO buy_log (userID, buyDate) VALUES (3, '2009-02-01');
INSERT INTO buy_log (userID, buyDate) VALUES (1, '2009-03-01');
INSERT INTO buy_log (userID, buyDate) VALUES (1, '2009-04-01');

ALTER TABLE buy_log ADD KEY ix_userID(userID);
ALTER TABLE buy_log ADD KEY ix_userID_buyDate(userID, buyDate);
```

如果对上述的表查询 userID = 1 的最近3条购买记录，使用了索引 ix_userID_buyDate ，因为 buyDate 字段在此联合索引中已经排好序了，因此无需额外的一次排序操作。

```
explain select * from buy_log where userID = 1 order by buyDate DESC limit 3;
+----+-------------+---------+------------+------+-----------------------------+-------------------+---------+-------+------+----------+--------------------------+
| id | select_type | table   | partitions | type | possible_keys               | key               | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+---------+------------+------+-----------------------------+-------------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | buy_log | NULL       | ref  | ix_userID,ix_userID_buyDate | ix_userID_buyDate | 4       | const |    4 |   100.00 | Using where; Using index |
+----+-------------+---------+------------+------+-----------------------------+-------------------+---------+-------+------+----------+--------------------------+
```

假设我们让查询强制（ 使用 force index 语法 ）使用 ix_userID 索引，会发现 extra 列显示为了 filesort ，即在内存当中进行了一次额外的排序操作。

```
explain select * from buy_log force index(ix_userID) where userID = 1 order by buyDate DESC limit 3;
+----+-------------+---------+------------+------+---------------+-----------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table   | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+---------+------------+------+---------------+-----------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | buy_log | NULL       | ref  | ix_userID     | ix_userID | 4       | const |    4 |   100.00 | Using index condition; Using filesort |
+----+-------------+---------+------------+------+---------------+-----------+---------+-------+------+----------+---------------------------------------+
```

再来看一个例子，假设有一张表中有 (a,b,c) 联合索引，下列查询可以不使用索引全扫描就能获取到结果:

```
select * from table where a = xxx order by b;
select * from table where a = xxx and b = xxx order by c;
```

而对于不满足左值匹配原则的查询语句来说， 例如如下的 SQL 查询， 其还需要额外的 filesort 排序操作，因为索引 (a, c) 并未排序：

```
select * from table where a = xxx order by c;
```

### 覆盖索引 （ covering index ）

从 Mysql 5.0 以上的版本，即 InnoDB Plugin 版本开始， InnoDB 存储引擎开始支持覆盖索引。所谓覆盖索引，是指能够从辅助索引中就可以查询到对应的记录，而不需要额外的再到聚集索引中去查询。其好处是辅助索引不包含整行的所有记录，它的大小远小于聚集索引，因此可以减少大量的 IO 操作。

对于 InnoDB 存储引擎的辅助索引而言，由于其包含了主键信息，因此其叶子节点存放的数据为 (PK1， PK2，...，key1，key2，...) 。如下的 SQL 查询语句都使用一次辅助索引来完成查询：

```
select key2 from table where key1 = xxx;
select pk2, key2 from table where key1 = xxx;
select pk1, key1, key2 from table where key1 = xxx;
select pk1, pk2, key2 from table where key1 = xxx;
```

覆盖索引还可以优化某一些统计类问题。例如如下的查询语句：

```
explain select count(*) from buy_log;
+----+-------------+---------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | buy_log | NULL       | index | NULL          | ix_userID | 4       | NULL |    7 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
```

上述统计 buy_log 表记录数的查询语句中，可以看到 InnoDB 存储引擎并没有选择聚集索引来进行统计，而是使用了 ix_userID 这个辅助索引来统计，原因就是上面我们提到的辅助索引的大小远小于聚集索引，可以减少 IO 操作。使用覆盖索引的一个重要标志是执行计划中， Extra 列显示为 `Using index` 。

### 优化器选择不使用覆盖索引的情况

在某些情况下，当分析 SQL 的执行计划时，会发现 SQL 优化器并没有选择辅助索引去查找数据，而是通过扫描聚集索引，也就是索引全表扫描来查询数据。

出现这种情况得大部分原因是，SQL 语句中 SELECT 列包含了辅助索引之外的数据列，因此除了通过辅助索引查询到指定的数据之后，还需要一次指针的访问来查找整行的数据。

我们还是通过一个示例来说明，创建如下的表：

```
CREATE TABLE buy_log(
    id int NOT NULL AUTO_INCREMENT
    ,userID int unsigned not NULL
    ,buyDate DATE
    ,productID int
    ,PRIMARY KEY (id)
)ENGINE=INNODB;

INSERT INTO buy_log (userID, buyDate, productID) VALUES (1, '2009-01-01', 2);
INSERT INTO buy_log (userID, buyDate, productID) VALUES (2, '2009-01-01', 1);
INSERT INTO buy_log (userID, buyDate, productID) VALUES (3, '2009-01-01', 5);
INSERT INTO buy_log (userID, buyDate, productID) VALUES (1, '2009-02-01', 6);
INSERT INTO buy_log (userID, buyDate, productID) VALUES (3, '2009-02-01', 7);
INSERT INTO buy_log (userID, buyDate, productID) VALUES (1, '2009-03-01', 8);
INSERT INTO buy_log (userID, buyDate, productID) VALUES (1, '2009-04-01', 9);

ALTER TABLE buy_log ADD KEY ix_userID(userID);
ALTER TABLE buy_log ADD KEY ix_userID_buyDate(userID, buyDate);
```

由于 SELECT * 所有列，因此无法使用 ix_userID 索引来进行查询。虽然 userID 是顺序存放的，但是再一次进行指针查找时的数据是无需的，变成了随机 IO 读，此时查询优化器选择了全表扫描 （ ALL ）。

```
explain select * from buy_log where userID >= 1 AND userID < 2;
+----+-------------+---------+------------+------+-----------------------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys               | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+-----------------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | buy_log | NULL       | ALL  | ix_userID,ix_userID_buyDate | NULL | NULL    | NULL |    7 |    57.14 | Using where |
+----+-------------+---------+------------+------+-----------------------------+------+---------+------+------+----------+-------------+
```

如果我们改变一下 SELECT 列，只返回建立辅助索引的列以及主键，此时又能使用到覆盖索引了：

```
explain select userID,id from buy_log  where userID >= 1 AND userID < 2;
+----+-------------+---------+------------+-------+-----------------------------+-----------+---------+------+------+----------+--------------------------+
| id | select_type | table   | partitions | type  | possible_keys               | key       | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+---------+------------+-------+-----------------------------+-----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | buy_log | NULL       | range | ix_userID,ix_userID_buyDate | ix_userID | 4       | NULL |    4 |   100.00 | Using where; Using index |
+----+-------------+---------+------------+-------+-----------------------------+-----------+---------+------+------+----------+--------------------------+
```

通过上述分析我们可以得出，很多公司 SQL 编写规范中，要求开发人员不能编写 `SELECT *` 的查询语句，除了有尽量减少网络传输的数据量的考虑之外，对于是否能够正确的使用索引，尤其是正确的使用覆盖索引，是能够起到至关重要的作用的。

## 哈希算法

哈希算法是一种较为常见的算法，时间复杂度为 O(1) 。

### 哈希表

哈希表（ Hash Table ）也被称为散列表，是由直接寻址表改进而来的。

我们首先来研究一下直接寻址表。当关键字的全域 U 比较小时，直接寻址是一种简单有效的技术。假设某应用要用到一个动态集合，其中每个元素都有一个取自全域 U = {0, 1, ..., m-1} 的关键字，并且假设没有2个元素具有相同的关键字。用一个数组（即直接寻址表） T[0...m-1] 表示动态集合，其中每个位置（或称为槽或桶）对应全域 U 中的一个关键字。下图说明了该方法，槽 k 指向集合中一个关键字为 k 的元素。如果该集合中没有关键字为 k 的元素，则 T[k] = NULL 。

![directly-address-table](http://static.zhuxiaodong.net/blog/static/images/directly-address-table.png)

直接寻址表存在一个明显的问题，如果全域 U 很大，在一台典型计算机的可用容量的限制下，要存储大小为 U 的一张表 T 就有点不实际。如果实际要存储的关键字集合 K 相对于 U 来说很小，那么分配给 T 的大部分空间都要浪费掉。

哈希表能够很好的解决这个问题。在哈希的方式下，该元素处于 h(k) 中，即利用哈希函数 h ， 根据关键
字 k 计算出槽的位置。函数 h 将关键字域 U 映射到哈希表 T[0...m-1]的槽位上。

![hash-table](http://static.zhuxiaodong.net/blog/static/images/hash-table.png)

假设两个关键字被哈希函数映射到了同一个槽中，被称之为发生了碰撞（ collision ）。在数据库中一般采用链接（ chaining ）的方式来解决碰撞问题。在链接法中，把散列到同一个槽中的所有元素都放在一个链表中。如下图所示，槽 j 中有一个指针，它指向由所有散列到 j 的元素构成的链表的头；如果不存在这样元素，则 j 中为 NULL。

![hash-table-chaining](http://static.zhuxiaodong.net/blog/static/images/hash-table-chaining.png)

最后要考虑的是哈希函数。哈希函数h必须可以很好地进行散列。最好的情况是能避免碰撞的发生。即使不能避免，也应该使碰撞在最小程度下产生。一般来说，都将关键字转换成自然数，然后通过除法散列、乘法散列或全域散列来实现。数据库中一般采用除法散列的方法。

在哈希函数的除法散列法中，通过取k除以m的余数，将关键字k映射到m个槽的某一个去，即哈希函数为：h(k) = k mod m

### InnoDB存储引擎中的哈希算法

对于缓冲池页的哈希表来说，在缓冲池中的Page页都有一个chain指针，它指向相同哈希函数值的页。而对于除法散列，m的取值为略大于2倍的缓冲池页数量的质数。例如：当前参数 innodb_buffer_pool_size 的大小为10M，则共有640个16KB的页。对于缓冲池页内存的哈希表来说，需要分配640×2=1280个槽，但是由于1280不是质数，需要取比1280略大的一个质数，应该是1399，所以在启动时会分配1399个槽的哈希表，用来哈希查询所在缓冲池中的页。

InnoDB 存储引擎将查找的页转换成自然数的算法的是：表空间都有一个 space_id ，用户所要查询的应该是某个表空间的某个连续16KB的页，即偏移量 offset 。InnoDB 存储引擎将 space_id 左移20位，然后加上这个 space_id 和 offset ，即关键字 K = space_id << 20 + space_id + offset，然后通过除法散列到各个槽中去。

### 自适应哈希索引

自适应哈希索引采用之前讨论的哈希表的方式实现。不同的是，这仅是数据库自身创建并使用的，DBA本身并不能对其进行干预。自适应哈希索引经哈希函数映射到一个哈希表中，因此对于字典类型的查找非常快速，如 `SELECT * FROM TABLE WHERE index_col='xxx'`。通过命令 `SHOW ENGINE INNODB STATUS` 可以看到当前自适应哈希索引的使用状况。

```
-------INSERT BUFFER AND ADAPTIVE HASH INDEX---------
Ibuf: size 2249, free list len 3346, seg size 5596,
374650 inserts, 51897 merged recs, 14300 merges
Hash table size 4980499, node heap has 1246 buffer(s)1
640.60 hash searches/s, 3709.46 non-hash searches/s
```

现在可以看到自适应哈希索引的使用信息了，包括自适应哈希索引的大小、使用情况、每秒使用自适应哈希索引搜索的情况。需要注意的是，哈希索引只能用来搜索等值的查询，如：SELECT * FROM table WHERE index_col='xxx'，而对于其他查找类型，如范围查找，是不能使用哈希索引的。因此，这里出现了 non-hash searches/s 的情况。通过 hash searches：non-hash searches 可以大概了解使用哈希索引后的效率。

由于自适应哈希索引是由 InnoDB 存储引擎自己控制的，因此这里的这些信息只供参考。不过可以通过参数 innodb_adaptive_hash_index 来禁用或启动此特性，默认为开启。
