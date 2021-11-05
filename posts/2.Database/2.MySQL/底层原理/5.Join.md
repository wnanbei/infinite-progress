# Join

## 执行流程

在 MySQL 中，不同的 Join 语句会使用不同的算法扫描数据。

直接使用 Join 时，MySQL 的优化器会自动选定 Join 的驱动表，如果使用 `straight_join` 可以指定前表为驱动表。

### 1. Index Nested-Loop Join

当可以用上被驱动表的索引时，会使用 `Index Nested-Loop Join` 算法，简称 `NLJ`。

执行流程：

1. 遍历并取出 t1 驱动表中符合条件的数据。
2. 根据取出数据的 ON 的条件值，去 t2 被驱动表中查找满足条件的数据。

由于查询 t2 被驱动表使用的是索引，所以 NLJ 的性能良好，比拆分成多条单表 SQL 性能更好。

### 2. Block Nested-Loop Join

当无法使用被驱动表上的索引时，需要全表扫描被驱动表，此时 MySQL 会使用 `Block Nested-Loop Join`，简称 `BNL`。

执行流程：

1. 把表 t1 需要用到的数据列读入线程内存 `join_buffer` 中。
2. 扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。

使用 BNL 与直接全表扫描，扫描的行数基本是一样的，但是 BNL 在内存中进行判断，性能会更好。

由于有可能 join_buffer 放不下驱动表，BNL 会将驱动表分块放入 join_buffer，对比完被驱动表以后换下一块驱动表，这样会导致 Join 性能降低很多，如果遇到这种情况，可以增大 `join_buffer_size` 的大小。

## 准则

### 1. 是否可以使用 Join

1. 如果可以使用 NLJ，则可以使用 Join 语句，比拆分成多条单表 SQL 性能更好。
2. 如果只能使用 BNL，则会占用大量系统资源，此情况避免使用 Join。

### 2. Join 使用准则

1. Join 关联字段必须建索引。
2. 使用小表为驱动表。

## 优化

### 1. MRR

### 2. BKA