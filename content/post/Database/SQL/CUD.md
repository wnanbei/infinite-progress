## INSERT

使用 `INSERT` 语句可以快速便利的插入一条或多条数据到表中。

**语法**：

```mysql
INSERT [LOW_PRIORITY | DELAYED | HIGH_PRIORITY] [IGNORE]
    [INTO] tb_name
    [PARTITION (partition_name [, partition_name] ...)]
    [(col_name [, col_name] ...)]
    {VALUES | VALUE} (value_list) [, (value_list)] ...
    [ON DUPLICATE KEY UPDATE assignment_list]
```

**关键字**：

- `tb_name` - 插入的表名。

- `[(col_name [, col_name] ...)]`

  要插入数据的字段名列表，如果你需要指定插入某几个字段的数据，那么你需要将你插入的字段名列出。并且在插入指的时候需要顺序一一对应。

  ```mysql
  INSERT INTO tb_name (column1,column2...) VALUES (value1,value2,...);
  ```

  没有插入值的字段将自动填充NULL或默认值。

- `{VALUES | VALUE} (value_list) [, (value_list)] ...`

  插入数据的值。在`col_name`没有指定插入的字段名时，需要将所有字段的值按照表中字段的顺序一一插入。`AUTO_INCREMENT`字段可以忽略，不用插入。

  ```mysql
  INSERT INTO tb_name VALUES (value1,value2,value3,...);
  ```

  如果不希望写定字段名，又希望某些字段插入的数据为默认值的话，那么可以使用`DEFAULT(col_name)`来生成默认值。

  如果需要一次性插入多行数据，那么需要用逗号分割。

  ```mysql
  INSERT INTO tb_name (column1,column2...)
  VALUES (value1,value2,...),
         (value1,value2,...);
  ```

- `[ON DUPLICATE KEY UPDATE assignment_list]`

  在插入数据时使用这个选项，那么意味着，如果插入的数据与现有的`UNIQUE`或者`PRIMARY KEY`字段重复时，将会更新旧的数据，而不是抛出重复的异常。

- `[PARTITION (partition_name [, partition_name] ...)]` - 控制插入分区表。

- `[IGNORE]` - 忽略掉插入语句引发的异常，只记录警告。

- `[LOW_PRIORITY | DELAYED | HIGH_PRIORITY]` - 插入语句的优先级。

## INSERT SELECT

`INSERT ... SELECT`是带有`SELECT`字句的`INSERT`语句，可以将`SELECT`查询到的多行结果插入到表中。

### 语法

```mysql
INSERT [LOW_PRIORITY | HIGH_PRIORITY] [IGNORE]
    [INTO] tbl_name
    [PARTITION (partition_name [, partition_name] ...)]
    [(col_name [, col_name] ...)]
    SELECT ...
    [ON DUPLICATE KEY UPDATE assignment_list]
```

**例子**：

```mysql
INSERT INTO tb_name2 (fld_id)
  SELECT tb_name1.fld_order_id
  FROM tb_name1 WHERE tb_name1.fld_order_id > 100;
```

## UPDATE

### 语法

```mysql
UPDATE [LOW_PRIORITY] [IGNORE] table_reference
    SET assignment_list
    [WHERE where_condition]
    [ORDER BY ...]
    [LIMIT row_count]
```

### 关键字

- `table_reference` - 要更新的表名

- `SET`

  必备参数，`SET`决定了更新哪些字段，更新成什么值。如下所示：

  ```mysql
  col_name1 = value1,
  col_name2 = value2,
  ```

- `[WHERE where_condition]`

  如果没有`WHERE`子句的话，那么`UPDATE`将会更新表中所有行的数据。

  如果给定了`WHERE`子句，那么`UPDATE`将会根据`WHERE`子句后面的表达式来决定更新哪些行的数据。

  例子：

  ```mysql
  UPDATE tb_name
  SET col_name1 = value1,col_name2 = value2
  WHERE id=3;
  ```

- `[ORDER BY ...]`

  在没有指定`WHERE`子句时，`UPDATE`将更新所有行。如果此时指定了`ORDER BY ...`子句，那么`UPDATE`将按照`ORDER BY ...`指定的顺序进行更新。

- `[LIMIT row_count]`

  一般与`ORDER BY ...`子句配合使用，`[LIMIT row_count]`将会限制更新的条数。

- `[IGNORE]` - 忽略掉更新语句引发的异常，只记录警告。

- `[LOW_PRIORITY]` - 降低更新语句的优先级。

## DELETE

### 语法

```mysql
DELETE [LOW_PRIORITY] [QUICK] [IGNORE] FROM tb_name
    [PARTITION (partition_name [, partition_name] ...)]
    [WHERE where_condition]
    [ORDER BY ...]
    [LIMIT row_count]
```

### 关键词

- `[WHERE where_condition]`

  如果没有`WHERE`子句的话，那么`DELETE`将会删除表中所有数据。

  如果给定了`WHERE`子句，那么`DELETE`将会根据`WHERE`子句后面的表达式来决定删除哪些行的数据。

- `[ORDER BY ...]`

  在没有指定`WHERE`子句时，`DELETE`将删除所有行。如果此时指定了`ORDER BY ...`子句，那么`DELETE`将按照`ORDER BY ...`指定的顺序进行删除。

- `[LIMIT row_count]`

  一般与`ORDER BY ...`子句配合使用，`[LIMIT row_count]`将会限制删除的条数。

  `[PARTITION (partition_name [, partition_name] ...)]` - 控制删除分区表。

- `[IGNORE]` - 忽略掉删除语句引发的异常，只记录警告。

- `[LOW_PRIORITY]` - 降低删除语句的优先级。

**例子**：

```mysql
DELETE FROM tb_name WHERE col_name=3;
```

## TRUNCATE

`TRUNCATE`语句主要用于清空一张表的内容。当我们需要删除一张表时，应该使用`DROP`，当我们想要删除表中一部分数据的时候，我们应该使用`DELETE`。

而当我们希望保存这张表但是删除里面所有数据的时候，我们就可以使用`TRUNCATE`。

### 语法

```mysql
TRUNCATE [TABLE] tbl_name;
```

`TRUNCATE`语句的语法非常简单，也与`DELETE`删除所有行非常类似，但它们主要有以下区别：

- `TRUNCATE`实际上时删除并重新创建表，所以一般来说会比`DELETE`删除所有行快很多，特别是在面对一张非常大的表的时候。
- `TRUNCATE`无法回滚。
- 对于`InnoDB`表，如果表中有`FOREIGN KEY`约束，那么`TRUNCATE`语句将会失败。
- 对于`AUTO_INCREMENT`的值将会重新刷新为初始值。
