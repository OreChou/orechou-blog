---
title: 【MySQL】Explain
date: 2021-08-06 17:00:00
tags: MySQL
---

# MySQL Explain
Explain 是 MySQL 自带的查询优化器。

使用 Explain + SQL 可查询出执行的相关信息，主要包含以下 10 个属性：
id, select_type, table, type, possible_key, key, key_len, ref, row, filtered, Extra

数据库性能瓶颈，主要关注 CPU 和 IO。

## id
反映的是表的读取顺序，或者查询中 SELECT 的执行顺序。

小表永远驱动大表，三种情况：
（1）id 相同，执行顺序是由上至下的
（2）id 不同，如果是子查询，id 序号会递增，id 值越大优先级越高，越先被执行
（3）id 存在相同的，也存在不同的，所有组中，id 越大越先执行，如果 id 相同的，从上往下顺序执行

## select_type
反映的是 MySQL 理解的查询类型，有几下几种：
+ SIMPLE：简单的 SELECT 查询，查询中不包含子查询或 UNION
+ PRIMARY：查询中若包含任何复杂的字部分，最外层查询标记为 PRIMARY
+ SUBQUERY：SELECT 或 WHERE 列表中的子查询
+ DERIVED：在 FROM 列表中包含的子查询，MySQL 会递归执行这些子查询，把结果放在临时表里
+ UNION：若第二个 SELECT 出现在 UNION 后，则被标记为 UNION，若 UNION 包含在 FROM 字句的子查询中，外层 SELECT 将被标记为 DERIVED
+ UNION RESULT：UNION 后的结果集

## table
反映的是数据从哪张表中读取出来。

例如 `<derived2>` 表示从 id 为 2 的临时表读取。

## type
type 是访问类型排序，反映的是 SQL 的优化状态，有如下几种：
+ system：从单表只查出一行记录（等于系统表），这是 const 类型的特例，一般不会出现
+ const：查询条件用到了常量，通过索引一次就找到，常在使用 primary key 或 unique 索引中出现
+ eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
+ ref：非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它可能会找到多个符合条件的行，与eq_ref的差别是eq_ref只匹配了一条记录
+ range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般是在where语句中出现了 between、<、>、in 等的查询
+ index：full Index scan，index 和 all 的区别为 index 类型只遍历索引树。这通常比 all 快，因为索引文件通常比数据文件小
+ all：全表扫描，如果查询数据量很大时，全表扫描效率是很低的

在 SQL 优化中至少做到 range 级别，最好能达到 ref 级别

## possible_key & key & key_len
possible_key 反映的是 MySQL 推测可能用到的索引，不一定被查询实际使用到。
key 反映的是实际使用到的索引，若为 null 则是因为没有建索引或者索引失效。
key_len 反映索引中使用的字节数，可计算计算查询中使用的索引的长度，越短越好。其显示的值为索引字段的最大可能长度，而非实际使用长度。

## ref
ref 反映的是哪些列或者常量被用于查找索引列上的值。

## rows
rows 反映的根据表的统计信息和索引选用的情况，大致估算出来到找到所有记录所需要读取的行数。

## filtered
使用 explain extended 时会出现这个列，5.7 之后的版本默认就有这个字段。这个字段表示存储引擎返回的数据在 server 层过滤后，剩下多少满足查询的记录数量的比例，不是具体记录数。

## Extra
Extra 反映的不适合在其他列显示，但是也很重要的信息，主要有以下几种：
+ Using filesort：MySQL 中无法利用索引完成的排序，这时会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。
+ Using temporary：使用了临时表保存中间结果，MySQL 在对查询结果排序时使用临时表。常见于排序 order by 和分组查询 group by。
+ Using index：MySQL 相应的 select 操作中使用了覆盖索引，避免了访问表的数据行，效率高。
+ Using where：MySQL 使用了 where 过滤。
+ Using join buffer：MySQL 使用了连接缓存。
+ Impossible where：where 子句的值为 false。
+ Distinct：优化 distinct 操作，在找到第一匹配的元组后即停止找同样值的动作。