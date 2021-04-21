---
layout: post
title: MySQL-Explain详解
categories: MySQL
description: 介绍什么是Explain，以及如何使用Explain
keywords: MySQL, Explain
---

本文介绍什么是**Explain**以及如何使用**Explain**分析**SQL**语句的瓶颈。([**Explain**的官方文档地址](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html))

## 什么是Explain

### 简介

​	Explain可以与`SELECT`, `DELETE`, `INSERT`, `REPLACE`以及 `UPDATE` 语句一起使用，用于提供有关MySQL如何执行语句的信息，这一过程中并没有执行SQL语句。常被用于分析SQL语句的瓶颈所在。

​	对于`SELECT`语句来说，Explain为SELECT语句中使用的每一张表返回一行信息，并以顺序(MySQL在处理时，读取表的顺序)列出。Explain之后可以使用SHOW WARNNINGS 显示更多的扩展信息。

### Explain的输出列及其含义

​	列名显示在表的第一列中；第二列提供使用FORMAT = JSON时在输出中显示的等效属性名称；第三列显示该列的含义(仅翻译部分字段)。

`Tips`本文抽取以下字段的部分进行简单介绍，详情请见官方文档。

|            **Column**             |   JSON Name   |                    Meaning                     |
| :-------------------------------: | :-----------: | :--------------------------------------------: |
|          :star:[id](#id)          |   select_id   |            The `SELECT` identifier             |
| :star:[select_type](#select_type) |     None      |               The `SELECT` type                |
|          [table](#table)          |  table_name   |          The table for the output row          |
|            partitions             |  partitions   |            The matching partitions             |
|        :star:[type](#type)        |  access_type  |                 The join type                  |
|        :star:possible_keys        | possible_keys |               **可能**选择的索引               |
|             :star:key             |      key      |              **实际上**使用的索引              |
|  :star:[key_length](#key_length)  |  key_length   |          The length of the chosen key          |
|            [ref](#ref)            |      ref      |       The columns compared to the index        |
|               rows                |     rows      |               **估计**要检查的行               |
|             filtered              |   filtered    | Percentage of rows filtered by table condition |
|          [Extra](#Extra)          |     None      |             Additional information             |

## Explain的具体介绍

​	对**Explain**的介绍，结合具体SQL来学习更为高效([转送门](#结合SQL语句理解Explain))

### 输出列详细介绍

#### id

​	`SELECT`的标识符，**id**越大执行的优先级越高，当**id**相同时，执行顺序从上往下

#### select_type

- SIMPLE：简单的SELECT（不使用UNION或子查询）
- PRIMARY：最外层的SELECT
- SUBQUERY：子查询中的第一个SELECT
- DERIVED：衍生表，通常出现在From子句的后面

#### table

​	表的名字，或者以下内容

- union`M`,`N`: The row refers to the union of the rows with `id` values of *`M`* and *`N`*.
- derived`N`: The row refers to the derived table result for the row with an `id` value of *`N`*. A derived table may result, for example, from a subquery in the `FROM` clause.
- subquery`N`: The row refers to the result of a materialized subquery for the row with an `id` value of *`N`*. 

#### type

​	该字段描述了表是如何链接的。以下按 **效率的高低** 排序，**system**效率最高

- **system**：该表本身只有一行(=system table) ，这是**const**的一个特例

- **const**：该表中至多只有一行匹配行，因为只有一行，优化器可以将这一行的值视为常量

- **eq_ref**：对于previous tables中的行的每种组合，从此表中读取**一行**。它出现在以下两种情况下：

  - 索引的所有列都被使用
    - 并且该索引是主键索引
    - 该索引是UNIQUE NOT NULL 索引

- **ref**：对于previous tables中的行的每种组合，将从该表中读取具有匹配索引值的**所有行**。它出现在以下情况中：

  - 连接使用的索引的**最左前缀**
  - 所使用的键不是**主键**或者**UNIQUE**索引

  `Tips`：和**eq_ref**相比，它对于previous tables的每种组合，查出的结果可能超过一行

- **ref_or_null**：这种连接类型类似于ref，但是MySQL会额外搜索包含NULL值的行。

- **range**：使用索引去检索指定范围内的数据，可能在以下场景中出现

  - 使用`=`，`<>`，`>`，`>=`，`<`，`<=`，`IS NULL`，`<=>`，`BETWEEN`，`LIKE` 或 `IN` 运算符将**键列**与**常量**进行比较

- **index**：`index` 和 `all`，都是全表扫描，但是`index`扫描的是索引树。它会在以下场景中出现

  - 索引是查询的**覆盖索引**，并且可用于满足表中所需的所有数据
  - 使用对索引的读取来执行全表扫描，以按索引顺序查找数据行。 `Uses index` 未出现在 `Extra` 列中

  `Tips`：当查询仅使用属于单个索引一部分的列时，可能出现该类型。

- **ALL**：全表扫描(扫聚簇索引的所有根节点)

`Tips`: 个人理解——`ALL`和`index`的区别在于`ALL`使用**主键索引树**进行全表扫描，而`index`则使用**二级索引树**进行全表扫描

#### key_length

​	key_len列指示MySQL决定使用的键值的长度。 通过key_len的值能够确定MySQL实际使用了索引的哪些部分。计算方法如下：

- **数值类型**
  - **tinyint**：1字节
  - **smallint**：2字节
  - **int**：4字节
  - **bigint**：8字节

- **字符串**
  - **char(n)**：n字节的长度
  - **varchar(n)**：如果是utf-8，长度为 3n+2 字节，加的2个字节用来存储字符串长度

- **时间类型**
  - **date**：3字节
  - **timestamp**：4字节
  - **datetime**：8字节
- **如果字段允许为空**，需要1字节记录NULL值

#### ref

​	ref列显示将哪些列或常量与键列中命名的索引进行比较，以从表中选择行。

#### Extra

​	此列包含有关MySQL如何解析查询的其他信息。

- Using index：仅使用索引树中的信息从表中检索列信息，而不必进行其他查找以读取实际行。(覆盖索引)
- Using where：使用`where`来处理结果
- Using temporary
- Using filesort

## ⭐结合SQL语句理解Explain

### 实验所需表

> 表**actor**

- 表结构

<img src="/MySQL/t_actor.png" style="zoom:80%;" />

- 插入数据

```sql
INSERT INTO `actor` (`id`, `name`, `update_time`) VALUES (1,'a','2017-12-22 15:27:18'), (2,'b','2017-12-22 15:27:18'), (3,'c','2017-12-22 15:27:18');
```

> 表**film**

- 表结构

<img src="/images/posts/MySQL/t_film.png" style="zoom:80%;" />

- 插入数据

```sql
INSERT INTO `film` (`id`, `name`) VALUES (3,'film0'),(1,'film1'),(2,'film2');
```

> 表**film_actor**

- 表结构

<img src="/images/posts/MySQL/t_film_actor.png" style="zoom:80%;" />

- 插入数据

```sql
INSERT INTO `film_actor` (`id`, `film_id`, `actor_id`) VALUES (1,1,1),(2,1,2),(3,2,1);
```

### 结合SQL分析

> **案例一**

```sql
-- 关闭mysql5.7新特性对衍生表的合并优化
SET SESSION optimizer_switch='derived_merge=off';
-- 还原默认配置
SET SESSION optimizer_switch='derived_merge=on';
-- 分析语句
EXPLAIN SELECT (SELECT 1 FROM actor WHERE id = 1) FROM (SELECT * FROM film WHERE id = 1) der;
```

![](/images/posts/MySQL/derived_merge=off.png)

**分析**

- **select_type**列
  - `FROM` 后面的表为衍生表(**DERIVED**)
  - `SELECT` 后面的表为子查询表(**SUBQUERY**)
  - `PRIMARY` 是最外层的 `SELECT`

- **id**列
  - **DERIVED**表**id**值最高，**SUBQUERY**的**id**值次之，执行顺序为 3 2 1



> **案例二**

```sql
-- 关闭mysql5.7新特性对衍生表的合并优化
SET SESSION optimizer_switch='derived_merge=off';
-- 还原默认配置
SET SESSION optimizer_switch='derived_merge=on';
-- 分析语句
EXPLAIN EXTENDED SELECT * FROM (SELECT * FROM film WHERE id = 1) tmp;
```

![](/images/posts/MySQL/system_const.png)

**分析**

- **type**列
  - **id**为 2 的查询：表中**至多**只有**一行**匹配行(id为主键)，优化器可以将这一行的值视为常量，所以`type` = **const**
  - **id**为 1 的查询：该表(tmp)本身只有一行，是**const**的特例，所以 `type` = **system**



> **案例三**

```sql
EXPLAIN SELECT * FROM film_actor LEFT JOIN film ON film_actor.film_id = film.id;
```

![](/images/posts/MySQL/eq_ref.png)

**分析**

​	这个查询属于简单的内连接 联表查询，由于没有子查询，所以 **id** 值相等，执行顺序从上到下

- **type**列
  - **film_actor**作为驱动表，全表扫描不可避免(使用了 ***** 没有**覆盖索引**可以使用)，因此 `type` = **ALL**
  - **film**表中，对于表**film_actor**的每种情况(每一行)，只需要在**film**表中读取一行(<font color="red">film_id为主键索引</font>)，因此`type` = **eq_ref**



> **案例四**

```sql
explain select * from film where name = 'film1';
```

![](/images/posts/MySQL/ref_Using index.png)

**分析**

​	需要注意，**film**表中仅仅只有 `name` 和 `id` 两个字段，`id` 为主键，因此这里相当于**覆盖索引**

- **type**列
  - 这里查询 `name` 字段，虽然使用了索引，但是不满足**UNIQUE**，因此还是有可能查询出多条结果，因此 `type` = **ref**
- **Extra**列
  - **Using index**，由之前所述，这里走了覆盖索引，因此可以直接从索引树上检索出满足查询条件的信息，不需要回表操作



> **案例五**

```sql
EXPLAIN SELECT film_id FROM film LEFT JOIN film_actor ON film.id = film_actor.film_id;
```

![](/images/posts/MySQL/ref_index.png)

**分析**

​	需要注意，和案例四一样 **film**表的二级索引**idx_name**的索引树上，已经存储了所有信息

- **key**列
  - 对于表 **film** MySQL 选择了 **idx_name** 因为二级索引通常比主键索引要小(这里大小差不多)
  - 对于表 **film_actor** MySQL 走了 **idx_film_actor_id** 索引

- **type**列
  - 对于表 **film**，作为驱动表，全表扫描不可避免，MySQL选择了一般而言较小的二级索引树，所以`type` = **index**
  - 对于表 **film_actor**，从**key_len**来看，索引仅命中了 **id** 字段(即只走了索引的部分前缀)，因此不能保证结果只有一行，因此`type` = **ref**



> **案例六**

```sql
EXPLAIN SELECT * FROM actor WHERE id > 1;
```

![](/images/posts/MySQL/range_Using where.png)

**分析**

- **type**列
  - 这里使用了`>`，所以`type` = **range**



> **案例七**

```sql
EXPLAIN SELECT * FROM film_actor WHERE film_id > 1;
```

![](/images/posts/MySQL/Using index condition.png)

**分析**

- **Extra**列
  - 这里只使用了索引 **idx_film_actor_id** 的部分前缀，因此`Extra` = **Using index condition**



> **案例八**

```sql
EXPLAIN SELECT * FROM film ORDER BY NAME;
EXPLAIN SELECT * FROM actor ORDER BY NAME;
```

![](/images/posts/MySQL/Using filesort.png)

**分析**

- **Extra**列
  - 由于表 **film** 的 `name` 字段有索引可走，故`Extra` = **Using index**
  - 由于表 **actor** 的 `name` 字段没有索引可走，只能先找出结果，再对结果进行排序，故`Extra` = **Using filesort**



## 最后总结

​	在实际应用中的情况十分复杂，**MySQL**会根据具体情况采取不同的优化，即便**表结构**和**SQL语句**都相同的情况下，也可能由于表数据量等因素的影响，使得结果不同。因此，本文所介绍的内容，仅用于初步了解、入门。