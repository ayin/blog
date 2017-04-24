---
categories:
  - Technology
tags:
  - 数据库
---

参考[Mysql Explain 详解](http://www.cnitblog.com/aliyiyi08/archive/2008/09/09/48878.html)简单介绍了下如何用explain查看SQL可能执行的情况

# 语法
explain < SQL >

# 详细
```
+----+-------------+---------------+-------+----------------+---------+---------+-----+------+-------------+
| id | select_type | table         | type  | possible_keys  | key     | key_len | ref | rows | Extra       |
+----+-------------+---------------+-------+----------------+---------+---------+-----+------+-------------+
```

* id: 多个select嵌套时，SQL执行的顺利的标识,SQL从大到小的执行
* select_type: 就是select的类型
  * SIMPLE: 简单SELECT(不使用UNION或子查询等) 
  * PRIMARY：包含子查询时最外层的select
  * UNION：UNION中的第二个或后面的SELECT语句
  * DEPENDENT UNION：UNION中的第二个或后面的SELECT语句，取决于外面的查询
  * UNION RESULT：UNION的结果
  * SUBQUERY：子查询中的第一个SELECT
  * DEPENDENT SUBQUERY：子查询中的第一个SELECT，取决于外面的查询
  * DERIVED：派生表的SELECT(FROM子句的子查询)
* table:显示这一行的数据是关于哪张表的.有时不是真实的表名字,看到的是derivedx
* type:显示了连接使用了哪种类别,有无使用索引,从最好到最差的连接类型为const、eq_reg、ref、range、index和ALL
  * system:这是const联接类型的一个特例。表仅有一行满足条件
  * const:表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。const表很快，因为它们只读取一次
  * eq_ref
  * ref
  * ref_or_null
  * index_merge
  * unique_subquery
  * index_subquery
  * range
  * index
  * ALL
* possible_keys
* key: key列显示MySQL实际决定使用的键（索引）。如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。
* key_len:key_len列显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL。使用的索引的长度。在不损失精确性的情况下，长度越短越好 
* ref:ref列显示使用哪个列或常数与key一起从表中选择行。
* rows:rows列显示MySQL认为它执行查询时必须检查的行数。
* Extra:该列包含MySQL解决查询的详细信息
  * Distinct :一旦MYSQL找到了与行相联合匹配的行，就不再搜索了 
  * Not exists: MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了 
  * Range checked for each: Record（index map:#,没有找到理想的索引，因此对于从前面表中来的每一个行组合，MYSQL检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一 
  * Using filesort:看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行 
  * Using index:列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候 
  * Using temporary: 看到这个的时候，查询需要优化了。这里，MYSQL需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上 
  * Using where:使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。如果不想返回表中的全部行，并且连接类型ALL或index，这就会发生，或者是查询有问题

# 参考
1. [Mysql Explain 详解](http://www.cnitblog.com/aliyiyi08/archive/2008/09/09/48878.html)