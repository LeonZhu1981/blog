title: 一次分表分库实践
date: 2017-06-08 14:51:09
categories: programming
tags:
- mysql
- sharding-db
toc: true
---
# 分库分表设计方案:

## 总体设计方案:
1. 分库策略我们选择了“二叉树分库分表”，所谓“二叉树分库分表”指的是：我们在进行数据库扩容时，都是以2的倍数进行扩容。比如：1台扩容到2台，2台扩容到4台，4台扩容到8台，8台扩容至16台，以此类推。这种分库方式的好处是，我们在进行扩容时，只需DBA进行表级的数据同步，而不需要自己写脚本进行行级数据同步。
2. 分片键使用ProjectID。按照项目的粒度进行分库，初期采用8个分库。DB编号统一从0开始。
3. 每个分库下按照设备分类进行分表。每一个设备分类在每个分库下会拆分成8个表。表编号统一从0开始。

整体结构大致如下:
> device_type_0, device_type_1表示不同的设备类型对应的分表.
> 例如: environment_0表示环境监测设备分表0.

```
+-----------------------+     +-----------------------+     +-----------------------+
|                       |     |                       |     |                       |
|     environment_0     |     |     environment_0     |     |     environment_0     |
|                       |     |                       |     |                       |
|     environment_1     |     |     environment_1     |     |     environment_1     |
|                       |     |                       |     |                       |
|     environment_2     |     |     environment_2     |     |     environment_2     |
|                       |     |                       |     |                       |
|        ......         |     |        ......         |     |        ......         |
|                       |     |                       |     |                       |
|     environment_7     |     |     environment_7     |     |     environment_7     |
|                       |     |                       |     |                       |
|     device_type_0     |     |     device_type_0     |     |     device_type_0     |
|                       |     |                       |     |                       |
|     device_type_1     |     |     device_type_1     |     |     device_type_1     |
|                       |     |                       |     |                       |
|        ......               |        ......               |        ......
|                       +     |                       +     |                       +
|     device_type_7     |     |     device_type_7     |     |     device_type_7     |
|                       |     |                       |     |                       |
+-----------------------+     +-----------------------+     +-----------------------+
            DB0                           DB1                           DB2

+-----------------------+     +-----------------------+     +-----------------------+
|                       |     |                       |     |                       |
|     environment_0     |     |     environment_0     |     |     environment_0     |
|                       |     |                       |     |                       |
|     environment_1     |     |     environment_1     |     |     environment_1     |
|                       |     |                       |     |                       |
|     environment_2     |     |     environment_2     |     |     environment_2     |
|                       |     |                       |     |                       |
|        ......         |     |        ......         |     |        ......         |
|                       |     |                       |     |                       |
|     environment_7     |     |     environment_7     |     |     environment_7     |
|                       |     |                       |     |                       |
|     device_type_0     |     |     device_type_0     |     |     device_type_0     |
|                       |     |                       |     |                       |
|     device_type_1     |     |     device_type_1     |     |     device_type_1     |
|                       |     |                       |     |                       |
|        ......               |        ......               |        ......
|                       +     |                       +     |                       +
|     device_type_7     |     |     device_type_7     |     |     device_type_7     |
|                       |     |                       |     |                       |
+-----------------------+     +-----------------------+     +-----------------------+
            DB3                           DB4                           DB5

            +-----------------------+              +-----------------------+
            |                       |              |                       |
            |     environment_0     |              |     environment_0     |
            |                       |              |                       |
            |     environment_1     |              |     environment_1     |
            |                       |              |                       |
            |     environment_2     |              |     environment_2     |
            |                       |              |                       |
            |        ......         |              |        ......         |
            |                       |              |                       |
            |     environment_7     |              |     environment_7     |
            |                       |              |                       |
            |     device_type_0     |              |     device_type_0     |
            |                       |              |                       |
            |     device_type_1     |              |     device_type_1     |
            |                       |              |                       |
            |        ......                        |        ......
            |                       +              |                       +
            |     device_type_7     |              |     device_type_7     |
            |                       |              |                       |
            +-----------------------+              +-----------------------+
                        DB6                                    DB7

```

<!--more-->
## 分表分库了之后, 各个类型的设备ID（分布式主键ID）的生成方案:
常见的分布式主键ID生成方案可以参考[这里](http://www.jianshu.com/p/a0a3aa888a49)，我们将采用twitter的Snowflake算法. 该算法理论上能够保证每秒单台机器生成400W左右.

**目前的设计，最大支持扩容64个分库，每种类型的分表最大支持16张分表。**

```
+---------+-----------+---------+------------+-------+--------------+
|  1 bit  |   6 bit   |  4 bit  |   41 bit   | 3 bit |    9 bit     |
|   vid   |   db_id   |   tid   |  timestamp |  sid  |    seq_no    |
+---------+-----------+---------+------------+-------+--------------+

```

> * 1 bit vid(version id): 1位表示版本号。如果今后算法发生变化了之后，用于做版本兼容处理。
> * 6 bit db_id(分库信息): 6位表示分库信息。1 << 6 **最大支持64个分库**。 
> * 4 bit t_id(table id，分表信息): 4位表示分表信息。1 << 4 **最大支持16个分表**。
> * 41 bit timestamp(时间戳信息): 41位时间戳, 与通用Snowflak算法保持一致。
> * 3 bit sid(server id): 3 bit服务器编号。1 << 3 **最大支持扩展8台机器，该机器是指部署生成分布式ID的Java服务器**。
> * 9 bit seq_no(Seqence Number): 9 bit自增序列号。1 << 9 **1ms之内最大支持自增512个**

**具体生成规则举例**:

假设我们初期上线时，只有8台数据库服务器，每一种类型的设备数据在每个分库下又分为了8张表。

如果直接对projectID % 8的话，以后在进行扩容的时候（例如8台增加至16台），分库路由将会无法正确地工作。为了解决此问题，我们需要对分库精度进行冗余，也就是说一开始就使用最大支持的分库数量来进行取模。

* 分库信息：(projectID % 64)
* 实际分库信息：分库信息 % 8 = (projectID % 64) % 8
* 分表信息：(projectID % 8)

以projectID为9527为列，分库信息为：9527 % 64 = 55，实际分库信息为：(9527 % 64) % 8 = 7，分表信息为：9527 % 8 = 7

```
                         +---------------+
                         |    projectID  |
                +--------+      9527     +------+
                |        +---------------+      |
                |                               |
                |                               |
                |                               |
         +------v------+                  +-----v-----+
         |             |                  |           |
         |   mod 64    |                  |   mod 8   |
         |             |                  |           |
         +------+------+                  +-----+-----+
                |                               |
                |                               |
                |                               |
                |                               |
         +------v------+                  +-----v-----+
         |   result    |   +--------------+   result  |
         |     55      |   |              |     8     |
         +------+------+   |              +-----------+
                |          |
                |          |
                |          |
+---------+-----v-----+----v----+------------+-------+--------------+
|  1 bit  |   6 bit   |  4 bit  |   41 bit   | 3 bit |    9 bit     |
|   vid   |   db_id   |   tid   |  timestamp |  sid  |    seq no    |
+---------+-----+-----+----+----+------------+-------+--------------+
                |          |
                |          |
                |          |
          +-----v-----+    |               +---------+
          |           |    +---------------> table_7 |
          |  55 mod 8 |                    +---------+
          |           |
          +-----+-----+
                |
                |
          +-----v-----+
          |   result  |
          |     7     |
          +-----+-----+
                |
                |
            +---v---+
            |  DB7  |
            +-------+

```

**需要注意的是: 生成的分布式ID中6 bit分库信息存储的是取模64的值，而非实际的分库信息（即取模8的值）。**

### 分布式ID算法Java实现:

sharding-jdbc当中已经默认包含了3种分布式ID的实现算法, 参考[这里](https://github.com/dangdangdotcom/sharding-jdbc/tree/master/sharding-jdbc-id-generator-parent/sharding-jdbc-self-id-generator/src/main/java/com/dangdang/ddframe/rdb/sharding/id/generator/self) 和 [这里](http://www.jianshu.com/p/80e68ae9e3a4)。我们需要做的是，在[CommonSelfIdGenerator](https://github.com/dangdangdotcom/sharding-jdbc/blob/master/sharding-jdbc-id-generator-parent/sharding-jdbc-self-id-generator/src/main/java/com/dangdang/ddframe/rdb/sharding/id/generator/self/CommonSelfIdGenerator.java)的基础上，实现我们自己的算法。