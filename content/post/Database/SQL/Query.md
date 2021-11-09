

```mysql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
      [HIGH_PRIORITY]
      [STRAIGHT_JOIN]
      [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
      SQL_NO_CACHE [SQL_CALC_FOUND_ROWS]

    select_expr [, select_expr ...]

    [FROM table_references
      [PARTITION partition_list]
      [WHERE where_condition]
      [GROUP BY {col_name | expr | position}, ... [WITH ROLLUP]]

    [HAVING where_condition]
    [WINDOW window_name AS (window_spec)
        [, window_name AS (window_spec)] ...]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]

    [INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
      | INTO DUMPFILE 'file_name'
      | INTO var_name [, var_name]]
    [FOR {UPDATE | SHARE} [OF tbl_name [, tbl_name] ...] [NOWAIT | SKIP LOCKED]
      | LOCK IN SHARE MODE]]
```

### 常用基础关键词

- `select_expr [, select_expr ...]`

  每一条`select_expr`语句代表你想要从表中取出的字段值，这是必备的关键词，也就是你至少要写一条`select_expr`，如果要写多条`select_expr`的话，应该用`,`进行分割。

  并且`select_expr`可以使用`*`通配符来进行匹配，例如以下语句将会取出表中的所有字段。

  ```mysql
  SELECT * FROM tb_name;
  ```

- `FROM table_references`

  `table_references`指的是你想要提取的一张或多张表的表名。

  ```mysql
  SELECT col_name1, col_name2 FROM tb_name;
  ```

  如果对`table_references`指定了多个表名，那么意味着你在使用`JOIN`连接。

  - `[PARTITION partition_list]`

    在`FROM`语句中，可以使用`PARTITION`子句来指定查询的分区表。指定之后，查询将只从列出的分区表中查询数据。

  - `[WHERE where_condition]`

    `FROM`语句中，还有`WHERE`子句，可以指定查找符合`where_condition`表达式的数据。如果没有指定`WHERE`子句，那么将会把表中所有行的数据查询出来。

    例子：

    ```mysql
    SELECT col_name1 FROM tb_name WHERE col_name2=1;
    ```

    在`WHERE`的`where_condition`表达式中，可以使用任何MySQL支持的函数和运算符，聚合算法除外。

  - `[GROUP BY {col_name | expr | position}, ... [WITH ROLLUP]]`

    `FROM`中还有一个`GROUP BY`语句，用以分组。

- `[ORDER BY {col_name | expr | position} [ASC | DESC], ... [WITH ROLLUP]]`

  排序子句，用于将查询到的数据按一定顺序进行排列，`ORDER BY`可以通过列名或者列的别名来进行排序。例子如下所示：

  ```mysql
  SELECT college, region, seed FROM tournament
    ORDER BY region, seed;
  
  SELECT college, region AS r, seed AS s FROM tournament
    ORDER BY r, s;
  ```

  如果需要改变排序方式为降序的话，那么可以在`ORDER BY`语句后加上`DESC`(descending)，默认的排序方式是`ASC`(ascending)。

  ```mysql
  SELECT college, region, seed FROM tournament
    ORDER BY region, seed DESC;
  ```

- `[LIMIT {[offset,] row_count | row_count OFFSET offset}]`

  `LIMIT`子句主要用来限制查询结果的数量。`LIMIT`一般需要一个或两个非负整数参数来决定限制的范围和位置。

  - 当只有一个参数时，参数表示的是从查询结果的第一行开始返回的行数。例：

    ```mysql
    SELECT * FROM tb_name LIMIT 5;     # 返回前五行
    ```

  - 当传入两个参数时，第一个参数表示相对于结果第一行的偏移行数，第二个参数表示的是返回的行数。例：

    ```mysql
    SELECT * FROM tb_name LIMIT 5,10;  # 返回第6-15行
    ```

  - 如果你希望从某一行开始，返回这一行之后所有的结果，那么你可以把第二个参数设置的非常大。例：

    ```mysql
    SELECT * FROM tbl LIMIT 95,18446744073709551615;   #返回从96行开始所有的结果
    ```

### 修饰符

跟在`SELECT`后有许多可以影响查询结果的修饰符可以使用，例如`HIGH_PRIORITY`、`STRAIGHT_JOIN`等。

- `ALL`、`DISTINCT`

  这个修饰符指的是是否返回重复的查询结果。

  - `ALL`(默认)指的是，只要符合查询结果，就算有数据是重复的也全部返回。

  - `DISTINCT`指的是，如果查询结果中有重复的行，那么那些重复的数据将会被删除。`DISTINCTROW`是`DISTINCT`的同义词。

- `HIGH_PRIORITY`

  `HIGH_PRIORITY`能让`SELECT`语句拥有比`UPDATE`更高的优先级。

- `STARTGHT_JOIN`

  `STARTGHT_JOIN`会强制让优化器按照`FROM`字句中列出的顺序进行连接。

- `SQL_SMALL_RESULT`或`SQL_BIG_RESULT`

  这个修饰符可以与`GROUP BY`或者`DISTINCT`一起使用，来告诉优化器搜索的结果非常多还是非常少。如果使用`SQL_BIG_RESULT`，那么MySQL会直接使用基于磁盘的临时表来存储搜索结果，如果使用的是`SQL_SMALL_RESULT`，那么MySQL会使用基于内存的临时表来存储搜索结果。

**注：需要注意的是，`SELECT`子句的顺序都需要按照语法中给定的顺序来进行使用，`INTO`子句例外，它可以跟在`select_expr`列表后方。**

## 聚合分组

### 聚合函数

在MySQL中有一些对值的集合进行操作的函数可以使用，这被称为聚合函数，以下是常见的聚合函数：

| 聚合函数          | 描述                     |
| ----------------- | ------------------------ |
| `AVG()`           | 求平均值函数         |
| `COUNT()`         | 返回参数的行数           |
| `COUNT(DISTINCT)` | 返回去重之后的行数         |
| `GROUP_CONCAT()`  | 返回所有值拼接成的字符串 |
| `MAX()`           | 求最大值               |
| `MIN()`           | 求最小值               |
| `STD()`           | 求标准差               |
| `SUM()`           | 求和函数                 |

除非专门说明，否则这些聚合函数都会忽略列表中的`NULL`。

- `AVG([DISTINCT] expr) [over_clause]`

  返回`expr`的平均值，可以使用`DISTINCT`将结果先去重后再求平均值。例子：

  ```mysql
  mysql> SELECT student_name, AVG(test_score)
         FROM student
         GROUP BY student_name;
  ```

- `COUNT(expr) [over_clause]`

  `COUNT`会返回`expr`中值不为`NULL`的行数。但是`COUNT(*)`有一点特殊，其返回的是提取出来的总行数，不管其是否为`NULL。`

  ```mysql
  mysql> SELECT student.student_name,COUNT(*)
         FROM student,course
         WHERE student.student_id=course.student_id
         GROUP BY student_name;
  ```

- `GROUP_CONCAT(expr)`

  这个聚合函数会将`expr`中所有非`NULL`值拼接成一个字符串并返回。

  ```mysql
  mysql> SELECT student_name,
         GROUP_CONCAT(test_score)
         FROM student
         GROUP BY student_name;
  ```

### GROUP BY

`GROUP BY`是`SELECT`语句中的子句，其可以指定字段，将这个字段中值相同的行分为一组，所以这个字段基本上都与聚合函数一起使用 ，能够根据不同条件统计数据。例如：

```mysql
mysql> SELECT class, COUNT(name)
       FROM student
       GROUP BY class;
```

此例子会将学生通过班级分组，并统计出每个班级的人数。

需要注意的是，在使用了`GROUP BY`分组之后，那么前面`SELECT`查询的字段就只能使用分组的字段和聚合函数了，使用其他字段将会报错。

## JOIN

在MySQL中，支持在`SELECT`、`DELETE`和`UPDATE`的多表中使用`JOIN`语法。`JOIN`也被称为连接表达式，而连结又主要分为内连接以及外连接。

`JOIN`语法可以把两张不同的表按一定的条件进行拼接。

我将会使用以下两张表作为例子进行演示：

```mysql
mysql> SELECT * FROM a;
+------+-------+
| id   | name  |
+------+-------+
|    1 | Emma  |
|    2 | Jason |
+------+-------+
2 rows in set (0.00 sec)

mysql> SELECT * FROM b;
+------+------+
| id   | sex  |
+------+------+
|    1 | F    |
|    3 | M    |
+------+------+
2 rows in set (0.00 sec)
```

### 内连接

MySQL中的`JOIN`，`CROSS JOIN`，`INNER JOIN`三者是等价的，都被视为内连接。在直接使用的时候，又被称为无条件内连接或笛卡尔连接，其会把两张表里的数据完全相互连接，形成`M * N`条数据。

例子：

```mysql
mysql> SELECT * FROM a JOIN b;
+------+-------+------+------+
| id   | name  | id   | sex  |
+------+-------+------+------+
|    1 | Emma  |    1 | F    |
|    2 | Jason |    1 | F    |
|    1 | Emma  |    3 | M    |
|    2 | Jason |    3 | M    |
+------+-------+------+------+
4 rows in set (0.00 sec)
```

可以看到数据的连接是完全没有任何联系的，所以如果我们需要通过指定条件来限定连接的方式，就可以使用`ON`子句来设定连接条件。

这一种有条件的内连接是使用的最多的连接方式。

```mysql
mysql> SELECT * FROM a JOIN b ON a.id=b.id;
+------+------+------+------+
| id   | name | id   | sex  |
+------+------+------+------+
|    1 | Emma |    1 | F    |
+------+------+------+------+
1 row in set (0.00 sec)
```

### 外连接

在内连接中，可以看到如果没有符合`ON`字句的匹配条件，那么不符合的这些行将会被舍去。但是有一些情况下，我们希望保留下那些没有符合匹配条件的行，这个时候就可以使用外连接。

#### 左外连接

既然是要保留没有符合匹配条件的行，那么肯定是需要一个标准的，也就是保留哪张表的不匹配行。那么左外连接也就意味着将会以`JOIN`的左表为基准进行保留。

也就是说，在左外连接的过程中，左表中不符合匹配条件的行将会被保存下来，这些行的右表字段将使用`NULL`来填充。而右表中不匹配的字段将会被舍去。

```mysql
mysql> SELECT * FROM a LEFT JOIN b ON a.id=b.id;
+------+-------+------+------+
| id   | name  | id   | sex  |
+------+-------+------+------+
|    1 | Emma  |    1 | F    |
|    2 | Jason | NULL | NULL |
+------+-------+------+------+
2 rows in set (0.00 sec)
```

#### 右外连接

与左外连接相对应，将右表作为基准进行连接。

```mysql
mysql> SELECT * FROM a RIGHT JOIN b ON a.id=b.id;
+------+------+------+------+
| id   | name | id   | sex  |
+------+------+------+------+
|    1 | Emma |    1 | F    |
| NULL | NULL |    3 | M    |
+------+------+------+------+
2 rows in set (0.00 sec)
```

## 子查询

子查询也就是说把一个查询嵌套在另一个查询中，子查询也被称为内部查询，包含内部查询的则被称为外部查询。

外部查询需要是这些语句之一：`SELECT`、`INSERT`、`UPDATE`、`DELETE`或`DO`。

子查询的位置一般会在`SELECT`中、`FROM`后、`WHERE`中。

### 分类

子查询一般会被分为以下几类：

1. 标量子查询：返回单一值的标量，最简单的形式。

    是指子查询返回的是单一值的标量，如一个数字或一个字符串，也是子查询中最简单的返回形式。 可以使用 = > < >= <= <> 这些操作符对子查询的标量结果进行比较，通常子查询的位置在比较式的右侧。

    ```mysql
    SELECT * FROM article WHERE uid = (SELECT uid FROM user WHERE status=1 ORDER BY uid DESC LIMIT 1)
    SELECT * FROM t1 WHERE column1 = (SELECT MAX(column2) FROM t2)
    SELECT * FROM article AS t WHERE 2 = (SELECT COUNT(*) FROM article WHERE article.uid = t.uid)
    ```

2. 列子查询：返回的结果集是N行一列。

    指子查询返回的结果集是 N 行一列，该结果通常来自对表的某个字段查询返回。可以使用`IN`、`ANY`、`SOME`和`ALL`操作符，不能直接使用`=` `>` `<` `>=` `<=` `<>` 这些比较标量结果的操作符。

    ```mysql
    SELECT * FROM article WHERE uid IN(SELECT uid FROM user WHERE status=1)
    SELECT s1 FROM table1 WHERE s1 > ANY (SELECT s2 FROM table2)
    SELECT s1 FROM table1 WHERE s1 > ALL (SELECT s2 FROM table2)
    ```

3. 行子查询：返回的结果集是一行N列。

    指子查询返回的结果集是一行N列，该子查询的结果通常是对表的某行数据进行查询而返回的结果集。

    ```mysql
    SELECT * FROM table1 WHERE (1,2) = (SELECT column1, column2 FROM table2)
    SELECT * FROM article WHERE (title,content,uid) = (SELECT title,content,uid FROM blog WHERE bid=2)
    ```

4. 表子查询：返回的结果集是N行N列。

    指子查询返回的结果集是N行N列的一个表数据。

    ```mysql
    SELECT * FROM article WHERE (title,content,uid) IN (SELECT title,content,uid FROM blog)
    ```

