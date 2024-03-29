---
title: MySQL · 最佳实践 · 分区表基本类型(转载)
date: 2018-11-02
categories:
  - SQL
tags:
    - mysql
---
## MySQL分区表概述
随着`MySQL`越来越流行，`Mysql`里面的保存的数据也越来越大。
在日常的工作中，我们经常遇到一张表里面保存了上亿甚至过十亿的记录。
这些表里面保存了大量的历史记录。 对于这些历史数据的清理是一个非常头疼事情，由于所有的数据都一个普通的表里。
所以只能是启用一个或多个带`where`条件的`delete`语句去删除（一般`where`条件是时间）。 
这对数据库的造成了很大压力。即使我们把这些删除了，但底层的数据文件并没有变小。
面对这类问题，最有效的方法就是在使用分区表。最常见的分区方法就是按照时间进行分区。 
分区一个最大的优点就是可以非常高效的进行历史数据的清理。

## 分区类型
目前`MySQL`支持范围分区（RANGE），列表分区（LIST），哈希分区（HASH）以及`KEY`分区四种。下面我们逐一介绍每种分区：

### RANGE分区
基于属于一个给定连续区间的列值，把多行分配给分区。最常见的是基于时间字段. 基于分区的列最好是整型，如果日期型的可以使用函数转换为整型。本例中使用`to_days`函数
```sql
CREATE TABLE my_range_datetime(
    id INT,
    hiredate DATETIME
) 
PARTITION BY RANGE (TO_DAYS(hiredate) ) (
    PARTITION p1 VALUES LESS THAN ( TO_DAYS('20171202') ),
    PARTITION p2 VALUES LESS THAN ( TO_DAYS('20171203') ),
    PARTITION p3 VALUES LESS THAN ( TO_DAYS('20171204') ),
    PARTITION p4 VALUES LESS THAN ( TO_DAYS('20171205') ),
    PARTITION p5 VALUES LESS THAN ( TO_DAYS('20171206') ),
    PARTITION p6 VALUES LESS THAN ( TO_DAYS('20171207') ),
    PARTITION p7 VALUES LESS THAN ( TO_DAYS('20171208') ),
    PARTITION p8 VALUES LESS THAN ( TO_DAYS('20171209') ),
    PARTITION p9 VALUES LESS THAN ( TO_DAYS('20171210') ),
    PARTITION p10 VALUES LESS THAN ( TO_DAYS('20171211') )，
    PARTITION p11 VALUES LESS THAN (MAXVALUE) 
);
```
`p11`是一个默认分区，所有大于`20171211`的记录都会在这个分区。`MAXVALUE`是一个无穷大的值。`p11`是一个可选分区。如果在定义表的没有指定的这个分区，当我们插入大于`20171211`的数据的时候，会收到一个错误。

我们在执行查询的时候，必须带上分区字段。这样可以使用分区剪裁功能

```shell
mysql> insert into my_range_datetime select * from test;                                                                    
Query OK, 1000000 rows affected (8.15 sec)
Records: 1000000  Duplicates: 0  Warnings: 0

mysql> explain partitions select * from my_range_datetime where hiredate >= '20171207124503' and hiredate<='20171210111230'; 
+----+-------------+-------------------+--------------+------+---------------+------+---------+------+--------+-------------+
| id | select_type | table             | partitions   | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+-------------------+--------------+------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | my_range_datetime | p7,p8,p9,p10 | ALL  | NULL          | NULL | NULL    | NULL | 400061 | Using where |
+----+-------------+-------------------+--------------+------+---------------+------+---------+------+--------+-------------+
1 row in set (0.03 sec)
```

注意执行计划中的`partitions`的内容，只查询了`p7`，`p8`，`p9`，`p10`三个分区，由此来看，使用`to_days`函数确实可以实现分区裁剪。

上面是基于`datetime`的，如果是`timestamp`类型，我们遇到上面问题呢？

事实上，`MySQL`提供了一种基于`UNIX_TIMESTAMP`函数的`RANGE`分区方案，而且，只能使用`UNIX_TIMESTAMP`函数，如果使用其它函数，譬如`to_days`，
会报如下错误：`“ERROR 1486 (HY000): Constant, random or timezone-dependent expressions in (sub)partitioning function are not allowed”`。

而且官方文档中也提到`“Any other expressions involving TIMESTAMP values are not permitted. (See Bug #42849.)”`。

下面来测试一下基于`UNIX_TIMESTAMP`函数的`RANGE`分区方案，看其能否实现分区裁剪。

针对`TIMESTAMP`的分区方案

创表语句如下：

```sql
CREATE TABLE my_range_timestamp (
    id INT,
    hiredate TIMESTAMP
)
PARTITION BY RANGE ( UNIX_TIMESTAMP(hiredate) ) (
    PARTITION p1 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-02 00:00:00') ),
    PARTITION p2 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-03 00:00:00') ),
    PARTITION p3 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-04 00:00:00') ),
    PARTITION p4 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-05 00:00:00') ),
    PARTITION p5 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-06 00:00:00') ),
    PARTITION p6 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-07 00:00:00') ),
    PARTITION p7 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-08 00:00:00') ),
    PARTITION p8 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-09 00:00:00') ),
    PARTITION p9 VALUES LESS THAN ( UNIX_TIMESTAMP('2017-12-10 00:00:00') ),
    PARTITION p10 VALUES LESS THAN (UNIX_TIMESTAMP('2017-12-11 00:00:00') )
);
```

插入数据并查看上述查询的执行计划

```shell
mysql> insert into my_range_timestamp select * from test;
Query OK, 1000000 rows affected (13.25 sec)
Records: 1000000  Duplicates: 0  Warnings: 0

mysql> explain partitions select * from my_range_timestamp where hiredate >= '20171207124503' and hiredate<='20171210111230';
+----+-------------+-------------------+--------------+------+---------------+------+---------+------+--------+-------------+
| id | select_type | table             | partitions   | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+-------------------+--------------+------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | my_range_timestamp | p7,p8,p9,p10 | ALL  | NULL          | NULL | NULL    | NULL | 400448 | Using where |
+----+-------------+-------------------+--------------+------+---------------+------+---------+------+--------+-------------+
1 row in set (0.00 sec)
```

同样也能实现分区裁剪。

在**5.7**版本之前，对于`DATA`和`DATETIME`类型的列，如果要实现分区裁剪，只能使用`YEAR()` 和`TO_DAYS()`函数，在**5.7**版本中，又新增了`TO_SECONDS()`函数。

### LIST 分区
`LIST`分区和`RANGE`分区类似，区别在于`LIST`是枚举值列表的集合，`RANGE`是连续的区间值的集合。二者在语法方面非常的相似。同样建议`LIST`分区列是非`null`列，
否则插入`null`值如果枚举列表里面不存在`null`值会插入失败，这点和其它的分区不一样，`RANGE`分区会将其作为最小分区值存储，`HASH\KEY`分为会将其转换成**0**存储，
主要`LIST`分区只支持整形，非整形字段需要通过函数转换成整形.

```sql
create table t_list( 
　　a int(11), 
　　b int(11) 
　　)(partition by list (b) 
　　partition p0 values in (1,3,5,7,9), 
　　partition p1 values in (2,4,6,8,0) 
　　);
```

### Hash 分区
我们在实际工作中经常遇到像会员表的这种表。并没有明显可以分区的特征字段。但表数据有非常庞大。
为了把这类的数据进行分区打散`mysql` 提供了`hash`分区。基于给定的分区个数，将数据分配到不同的分区，
`HASH`分区只能针对整数进行`HASH`，对于非整形的字段只能通过表达式将其转换成整数。表达式可以是`mysql`中任意有效的函数或者表达式，
对于非整形的`HASH`往表插入数据的过程中会多一步表达式的计算操作，所以不建议使用复杂的表达式这样会影响性能。

`Hash`分区表的基本语句如下：

```sql
CREATE TABLE my_member (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    created DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY HASH(id)
PARTITIONS 4;
```

注意：
1.`HASH`分区可以不用指定`PARTITIONS`子句，如上文中的`PARTITIONS` 4，则默认分区数为1。
2.不允许只写`PARTITIONS`，而不指定分区数。
3.同`RANGE`分区和`LIST`分区一样，`PARTITION BY HASH (expr)`子句中的`expr`返回的必须是整数值。
4.`HASH`分区的底层实现其实是基于MOD函数。譬如，对于下表

`CREATE TABLE t1 (col1 INT, col2 CHAR(5), col3 DATE) PARTITION BY HASH( YEAR(col3) ) PARTITIONS 4;`
如果你要插入一个`col3`为`“2017-09-15”`的记录，则分区的选择是根据以下值决定的：

`MOD(YEAR(‘2017-09-01’),4) = MOD(2017,4) = 1`

### LINEAR HASH分区
`LINEAR HASH`分区是`HASH`分区的一种特殊类型，与`HASH`分区是基于`MOD`函数不同的是，它基于的是另外一种算法。

格式如下：

```sql
CREATE TABLE my_members (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY LINEAR HASH( id )
PARTITIONS 4;
```

说明： 它的优点是在数据量大的场景，譬如`TB`级，增加、删除、合并和拆分分区会更快，缺点是，相对于`HASH`分区，它数据分布不均匀的概率更大。

### KEY分区
`KEY`分区其实跟`HASH`分区差不多，不同点如下：

1.`KEY`分区允许多列，而`HASH`分区只允许一列。
2.如果在有主键或者唯一键的情况下，`key`中分区列可不指定，默认为主键或者唯一键，如果没有，则必须显性指定列。
3.`KEY`分区对象必须为列，而不能是基于列的表达式。
4.`KEY`分区和`HASH`分区的算法不一样，`PARTITION BY HASH (expr)，MOD`取值的对象是`expr`返回的值，而`PARTITION BY KEY (column_list)`，基于的是列的`MD5`值。

格式如下：

```sql
CREATE TABLE k1 (
    id INT NOT NULL PRIMARY KEY,    
    name VARCHAR(20)
)
PARTITION BY KEY()
PARTITIONS 2;
```

在没有主键或者唯一键的情况下，格式如下：

```sql
CREATE TABLE tm1 (
    s1 CHAR(32)
)
PARTITION BY KEY(s1)
PARTITIONS 10;
```

## 总结：
1.`MySQL`分区中如果存在主键或唯一键，则分区列必须包含在其中。
2.对于原生的`RANGE`分区，`LIST`分区，`HASH`分区，分区对象返回的只能是整数值。
3.分区字段不能为`NULL`，要不然怎么确定分区范围呢，所以尽量`NOT NULL`

## 相关链接
- [数据库内核月报 － 2017 / 11](http://mysql.taobao.org/monthly/2017/11/09/)