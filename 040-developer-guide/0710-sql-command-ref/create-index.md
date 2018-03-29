# CREATE INDEX

# 创建索引

定义一个新的索引

## 概要

```
CREATE [UNIQUE] INDEX name ON table
       [USING btree|bitmap|gist]
       ( {column | (expression)} [opclass] [, ...] )
       [ WITH ( FILLFACTOR = value ) ]
       [TABLESPACE tablespace]
       [WHERE predicate]
```

## 描述

CREATE INDEX 构造指定表上的索引。索引主要用于增强数据库性能（尽管使用不当会导致性能降低）。

索引的关键字段被指定为列名称，或者替换为括号中的表达式。如果索引方法支持多列索引，则可以指定多个字段。

索引字段可以是从表行的一个或者多个列的值计算的表达式。 该功能可以基于基本数据的一些变换来获取对数据的快速访问。 列如，在 upper\(col\) 上计算的索引将允许子句 WHERE upper\(col\) = 'JIM' 来使用索引。

HashData 数据库提供索引方法 B-tree，位图，和 GiST。 用户可以自定义自己的索引方法，但这是相当复杂的。

当存在 WHERE 子句时，将创建部分索引，部分索引是仅包含表的一部分条目的索引， 通常是与表的其余部分相比对索引更有用的地方。例如，如果用户的表包含已结算订单和未结算订单，其中未列出的订单占总表中的一小部分，但最常选择，则可以通过在该部分上创建索引来提高性能。

WHERE 子句中使用的表达式可能仅指基础表的列，但它可以使用所有列，而不仅仅是索引的列。在 WHERE 中也禁止子查询和聚合表达式。 相同的限制适用于表达式的索引字段。

索引定义中使用的所有函数和操作符必须是不可改变的。他们的结果必须仅仅取决于他们的参数，没有任何外界的影响（如另一个表的内容或参数值）。此限制可以确保索引的行为被很好的定义。要在索引表达式或 WHERE 子句中使用用户定义的函数， 请记住在创建函数时标记函数为 IMMUTABLE。

## 参数

UNIQUE

当创建索引并添加每个时间数据时，检查表中的重复值。重复条目将产生错误。唯一索引仅适用于 B 树索引。在 HashData 数据库中，只有索引关键字与 HashData 分发密钥（或者它的超集）相同时才允许使用唯一索引。在分区表上，唯一索引仅在单个分区中支持，而不是跨越所有分区。

name

要创建索引的名称。索引始终与父表在统一模式中创建。

table

要索引表的名称（可选方案限定）

btree \| bitmap \| gist

要使用的索引方法，该有的选择是 b 树，位图，gist。 默认方法是 b 树。

column

表中要创建索引列的名称，仅 B-tree, 位图, GiST 索引支持多列索引。

expression

基于表的一个或多个列的表达式。 表达式通常用括号括起来，如语法所示。但是如果表达式有函数调用的形式，则可以省略括号。

opclass

操作符类的名称。操作符类指明了该列的索引要使用的操作符。例如，四字节整数的 B 树索引将使用 int4\_ops 类（该操作包含了四字节整数的比较函数）。 实际上，列的数据类型的默认的操作符类通常是足够的。 操作符类的要点是对于某些数据类型，可能会有多个有意义的排序。例如，复杂数据类型可以按绝对值或实数排序。我们可以通过为数据类型定义两个操作符类，然后在进行索引的时候选择适当的操作符类来实现。

FILLFACTOR

索引的 fillfactor 是一个百分比，用于确定索引方法将索引页填充什么程度。 对于 B 树来说，在初始化索引编译期间和扩展右侧索引（最大键值）时将页面填充该百分比程度，如果页面随后变得充满，它将会被分隔，导致索引效率逐渐下降。

B 树使用默认的填充因子 90，但是可以选择 10 到 100 之间的任何值。如果表示静态的，那么100是最好的最小化索引的物理大小的填充因子。但是对于大量更新的表，较小的填充因子是最好最小化页面分隔的需要。其他索引方法使用不同填充因子，但大致是相似的选择方式。默认的填充因子在方法之间有所不同。

tablespace

创建索引的表空间，如果没有指定，将使用默认的表空间。

predicate

部分索引的约束表达式。

## 注解

当在分区表上建立索引时，该索引将传播到由 HashData 数据库创建的所有子表。在 HashData 数据库创建的表上建立由分区表使用的索引是不支持的。

仅当索引列与 HashData 分布键列相同（或者是其超集）时，才允许 UNIQUE 索引。

在追加优化表上不允许使用 UNIQUE 索引。

在分区表上可以创建 UNIQUE 索引。但是，唯一性仅在分区内执行；分区之间之间不执行唯一性。例如，对于基于年份的分区和基于季度的自分区的分区表，仅在每个单独的季度分区上执行唯一性。在季度之间不执行唯一性。

默认情况下，索引不用于 IS NULL 字句。这种情况下使用索引的最好方法是使用 IS NULL 谓词创建部分索引。

bitmap 索引对于具有 100 到 100000 个不同值的列的效果最好。对于有超过 100000 个不同值的列， 位图索引的性能和空间效率下降。位图索引的大小与表中行数成比例，是索引列中不同值数量的数倍。

少于 100 个不同值的列通常不会从任何类型的索引中受益，例如，男性和女性只有两个不同值的性别列不是索引的良好候选。

HashData  的先前版本也有一个 R-tree 索引方法。 该方法已经被删除，因为它相对于 GIST 方法相比没有显著的优点。 如果指定了 USING rtree， 则 CREATE INDEX 将会将其解释为使用 USING gist。

更多关于 GiST 索引类型的信息，请参考 [PostgreSQL文档](https://www.postgresql.org/docs/8.3/static/indexes-types.html)。

HashData 中已经禁止使用 hash 和 GIN 索引。

## 例子

创建一个 B 树索引在 films 表的 title 列中：

```
CREATE UNIQUE INDEX title_idx ON films (title);
```

创建一个位图索引在 employee 表的 gender 列中:

```
CREATE INDEX gender_bmp_idx ON employee USING bitmap 
(gender);
```

创建一个索引在表达式 lower（title）上，允许有效的区分大小写的搜索：

```
CREATE INDEX lower_title_idx ON films ((lower(title)));
```

使用非默认的填充因子创建一个索引：

```
CREATE UNIQUE INDEX title_idx ON films (title) WITH 
(fillfactor = 70);
```

创建一个索引在 films 表的 code 列上，并且使索引驻留在表空间的索引空间中：

```
CREATE INDEX code_idx ON films(code) TABLESPACE indexspace;
```

## 兼容性

CREATE INDEX 是 HashData 数据库的语言扩展。SQL 标准中没有索引规定。

HashData 数据库不支持并发创建索引（CONCURRENTLY 关键字不支持）。

## 另见

[ALTER INDEX](./alter-index.md), [DROP INDEX](./drop-index.md), [CREATE TABLE](./create-table.md), [CREATE OPERATOR CLASS](./create-operator-class.md)

**上级话题：** [SQL命令参考](./README.md)
