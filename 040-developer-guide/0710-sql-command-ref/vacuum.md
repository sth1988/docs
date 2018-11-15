# VACUUM

垃圾收集并根据需要分析一个数据库。

## 概要

```
VACUUM [FULL] [FREEZE] [VERBOSE] [table]
VACUUM [FULL] [FREEZE] [VERBOSE] ANALYZE
              [table [(column [, ...] )]]
```

## 描述

VACUUM 收回由死亡元组占用的存储空间。在 HashData 数据库操作中，被删除或者被更新废弃的元组并没有在物理上从它们的表中移除，它们将一直存在直到一次 VACUUM 被执行。因此有必要周期性地做 VACUUM，特别是在频繁被更新的表上。

在不带任何参数的情况下，VACUUM 会处理当前用户具有清理权限的当前数据库中的每一个表。通过使用一个参数，VACUUM 可以只处理指定表。

VACUUM ANALYZE 对每一个选定的表执行一个 VACUUM 然后执行 ANALYZE。这是两种命令的一种方便的组合形式，可以用于例行的维护脚本。

VACUUM（不带 FULL）标记在表和索引中已经删除和废除的数据用于之后重用以及收回空间用于重用，只要空间位于一个表末端以及表上的排它锁能够容易地获取。在表开头和中间位置的没有使用的空间仍旧保留。在堆表中，这种形式命令能够并行读写表，因为没有获取排它锁。

使用追加优化表，VACUUM 首先通过清理索引来紧缩表，然后依次紧缩每个 Segment 文件，最后清理辅助关系同时更新统计信息。在每个 Segment 上，可见行从当前 Segment 文件复制到一个新的 Segment 文件，然后将当前的 Segment 文件计划为将删除同时将新的 Segment 文件设置为可用。一个追加优化表的简单 VACUUM 允许在 Segment 文件被紧缩时做 SELECT 、INSERT 、DELETE 以及 UPDATE 操作。不过，将会短暂地取得一个 Access Exclusive 锁以删除当前的 Segment 文件并且激活新的 Segment 文件。

VACUUM FULL 会做更多深入的处理，包括为了尝试把表紧缩成占用最少的磁盘块而在块间移动元组。这种形式会更慢一些并且在每个表被处理时都需要取得其上的 Access Exclusive 锁。Access Exclusive 锁确保持有者是唯一访问该表的事务。

**输出**

如果指定了 VERBOSE，VACUUM 会发出进度消息来表明当前正在处理哪个表。各种有关这些表的统计信息也会被打印出来。

## 参数

FULL

选择 "完全" 清理，它可以收回更多空间，并且需要更长时间和表上的排他锁。

FREEZE

指定 FREEZE 等价于参数 vacuum\_freeze\_min\_age 设置为 0 的 VACUUM。

VERBOSE

为每个表打印一份详细的清理活动报告。

ANALYZE

更新优化器用以决定最有效执行一个查询的方法的统计信息。

table

要清理的表的名称（可以有模式修饰）。缺省时是当前数据库中的所有表。

column

要分析的指定列的名称。缺省是所有列。

## 注解

VACUUM 不能再一个事务块中执行。

我们建议经常清理活动的生产数据库（至少每晚一次），以保证移除失效的行。在增加或删除了大量行之后， 对受影响的表执行 VACUUM ANALYZE 命令是一个很好的做法。 这样做将把最近的更改更新到系统目录，并且允许 HashData 数据库查询规划器在规划用户查询时做出更好的选择。

> 重要：在 PostgreSQL 中有一个单独叫做 autovacuum 守护进程 的可选服务进程，用来自动执行 VACUUM 和 ANALYZE 命令。该特征在 HashData 数据库中被禁止了。

VACUUM 会导致 I/O 流量的大幅度增加，这可能导致其他活动会话性能变差。因此，有时建议使用基于代价的清理延迟特性。

对于堆表，无效行被记录在一个叫做 **空闲空间映射** 地方。空闲空间映射必须足够大能够覆盖数据库中所有堆表的无效行。如果空间不足，那些从空闲空间映射溢出的无效行占用的空间将不能被一个 VACUUM 回收。

VACUUM 命令略过外部包。

VACUUM FULL 回收所有无效行空间，然后它需要获取一个排它锁在它当前处理的每个表上面，这是一个非常昂贵的操作，有可能会耗费很长的时间在那些大的，分布式的 HashData 数据库的表上。执行 VACUUM FULL 在维护数据库期间。

作为一个 VACUUM FULL 的替代方案，用户可以通过使用 CREATE TABLE AS 重新构建表，然后删除旧的表。

将空闲空间映射设置为适当的大小。通过下面的配置参数来配置空闲空间映射：

* max\_fsm\_pages
* max\_fsm\_relations

对于追加优化表， VACUUM 要求有足够的磁盘空间来容纳新的 Segment 文件在 VACUUM 执行过程中。如果在一个 Segment 文件中隐藏行的行数占总行数的比率小于一个阈值（默认值，10），那么 Segment 文件不会进行压缩。阈值可以通过 gp\_appendonly\_compaction\_threshold 来进行配置。VACUUM FULL 忽略阈值同时重写段文件而不考虑比率。可以通过 gp\_appendonly\_compaction 参数来关闭在追加优化表上 VACUUM 的使用。

在追加优化表被清理时如果检测到一个并发的可序列化事务，那么当前和后续的 Segment 文件都不会被紧缩。如果一个 Segment 文件已经被紧缩但是在删除原始 Segment 文件的事务中检测到在一个并发可序列化事务，则会跳过删除。这样在清理完成后，可能会留下一个或两个状态为 “awaiting drop” 的 Segment 文件。

## 示例

清理当前数据库下的所有表：

```
VACUUM;
```

只清理一张特定的表：

```
VACUUM mytable;
```

清理当前数据库下的所有表同时为查询优化器收集统计信息：

```
VACUUM ANALYZE;
```

## 兼容性

在 SQL 标准中没有 VACUUM。

## 另见

[ANALYZE](./analyze.md)

**上级话题：** [SQL命令参考](./README.md)
