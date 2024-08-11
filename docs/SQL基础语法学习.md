# SQL基础语法学习

## 一、SQL介绍

### 1.1 基本介绍

SQL（Structured Query Language）是一种用于管理和操作**<font color='cornflowerblue'>关系数据库的标准语言</font>**，包括数据查询、数据插入、数据更新、数据删除、数据库结构创建和修改等功能。。

一个数据库通常包含**一个或多个表**，成为数据库表，每个表有一个名字标识（例如:"Websites"），表包含**带有数据的记录（行）**，每个列成为**字段**。在数据库上执行的大部分工作都由 SQL 语句完成。

<img src="G:\code\study\CppStudy\docs\figures\SQL.png" alt="img" style="zoom:33%;" />

### 1.2 SQL 语句后面的分号？

某些数据库系统要求在每条 SQL 语句的末端使用分号。

分号是在数据库系统中分隔每条 SQL 语句的标准方法，这样就可以在对服务器的相同请求中执行一条以上的 SQL 语句。

## 二、SQL基本语法

### 2.1 SELECT 查询

SELECT 语句用于从数据库中选取数据。

### 2.2 INSERT INTO 插入

INSERT INTO 语句用于向表中插入新记录。

INSERT INTO 语句可以有两种编写形式：

- 第一种形式无需指定要插入数据的列名，只需提供被插入的值即可：

  ```mysql
  INSERT INTO table_name
  VALUES (value1,value2,value3,...)
  ```

- 第二种形式需要指定列名及被插入的值：

  ```mysql
  INSERT INTO table_name (column1,column2,column3,...)
  VALUES (value1,value2,value3,...)
  ```

**参数说明：**

- **table_name**：需要插入新记录的表名。
- **column1, column2, ...**：需要插入的字段名。
- **value1, value2, ...**：需要插入的字段值。

### 2.3 UPDATE 更新

UPDATE 语句用于更新表中已存在的记录。语法如下：

```mysql
UPDATE table_name
SET column1 = value1, column2 = value2, ...
WHERE condition
```

参数说明：

- **table_name**：要修改的表名称。
- **column1, column2, ...**：要修改的字段名称，可以为多个字段。
- **value1, value2, ...**：要修改的值，可以为多个值。
- **condition**：修改条件，用于指定哪些数据要修改。

### 2.4 DELETE 删除

DELETE 语句用于删除表中的行。语法如下：

```mysql
DELETE FROM table_name
WHERE condition
```

### 2.5 ORDER BY 排序

ORDER BY 关键字用于对结果集按照一个列或者多个列进行排序。

ORDER BY 关键字默认按照升序对记录进行排序。如果需要按照降序对记录进行排序，您可以**使用 DESC 关键字**。

```mysql
SELECT column1, column2, ...
FROM table_name
ORDER BY column1, column2, ... ASC|DESC
```

- **column1, column2, ...**：要排序的字段名称，可以为多个字段。
- **ASC**：表示按升序排序。
- **DESC**：表示按降序排序。

### 2.6 CREATE DATABASE 创建数据库

SQL CREATE DATABASE语句用于创建数据库。创建数据库的基本语法可以通过以下方式给出：

```mysql
CREATE DATABASE database_name;
```

创建数据库不会**选择使用它**。因此，在继续之前，我们必须选择带有**该USE语句的目标数据库**。例如，该`USE database_name`命令将*演示*数据库设置为所有将来所有命令的**目标数据库**。

### 2.7 CREATE TABLE 创建表

SQL CREATE TABLE语句用于创建表。创建表的基本语法可以通过以下方式给出：

```mysql
CREATE TABLE table_name (column1_name data_type constraints, column2_name data_type constraints, ...)

# 如果不存在则创建表
CREATE TABLE IF NOT EXISTS table_name (...)
```

### 2.8 CONSTARINTS 字段约束

constraints（约束）是一些对于这个字段的约束（也称为***修饰符***），例如：

- `NOT NULL`约束确保该字段不能接受一个NULL值。
- `PRIMARY KEY`约束标记对应的字段作为表的主键。
- `AUTO_INCREMENT`属性是标准SQL的MySQL扩展，它告诉MySQL如果未指定该值，则将前一个值增加1来自动为该字段分配一个值。仅适用于数字字段。
- `UNIQUE`约束确保一列的每一行必须具有唯一值。
- `DEFAULT`约束指定列的默认值
- `FOREIGN` 创建外键
- `CHECK` 限制可以放置在列中的值。

### 2.9 TRUNCATE TABLE 清空表

TRUNCATE TABLE语句从表中**删除所有行，但表结构及其列，约束，索引等保持不变**。要删除表及其数据，可以使用该DROP TABLE语句，但是需谨慎操作。

```mysql
TRUNCATE TABLE table_name
```

DELETE和TRUNCATE TABLE似乎具有相同的效果，但是它们的工作方式不同。这是这两个语句之间的一些主要区别：

- RUNCATE TABLE语句删除并重新创建表，并使任何自动增量值都重置为其初始值（通常为1）。
- DELETE可让您根据**可选WHERE子句过滤要删除的行**，TRUNCATE TABLE而不支持WHERE子句则仅**删除所有行**。
- TRUNCATE TABLE与DELETE相比，它更快并且使用的系统资源更少，因为**DELETE扫描表以生成受影响的行数，然后逐行删除行**，并为每个删除的行在数据库日志中记录一个条目，而TRUNCATE TABLE**只删除所有行而不提供任何其他信息**。

### 2.10 DROP TABLE 删除表

DROP TABLE语句永久删除表中的所有数据，以及在数据字典中定义表的元数据。DROP TABLE删除一个或多个表。可以使用以下语法：

```mysql
DROP TABLE talbe1_name, table2_name, ...
```

> **警告：**删除数据库或表是不可逆的。因此，在使用DROP语句时要小心，因为数据库系统通常不会显示任何警告，例如“您确定删除吗？”。它将立即删除数据库或表及其所有数据。



## 三、SQL高级语法

### 3.1 SELECT LIMIT语句

SELECT LIMIT 语句用于在 SQL 中**限制返回的结果集中的行数**， 它通常用于只需要查询前几行数据的情况，尤其在数据集非常大时，可以显著提高查询性能。

```mysql
SELECT column1, column2, ...
FROM table_name
LIMIT number;
```

number：指定返回的行数。

### 3.2 LIKE 操作符

LIKE 操作符用于在 WHERE 子句中搜索列中的指定模式，是进行**模糊查询的关键字**，它允许我们**根据模式匹配来选择数据，通常与 `%` 和 `_` 通配符一起使用**。

```mysql
SELECT column1, column2, ...
FROM table_name
WHERE column_name LIKE pattern

SELECT column1, column2, ...
FROM table_name
WHERE column_name NO LIKE pattern
```

参数说明：

- **column1, column2, ...**：要选择的字段名称，可以为多个字段。如果不指定字段名称，则会选择所有字段。
- **table_name**：要查询的表名称。
- **column**：要搜索的字段名称。
- **pattern**：搜索模式。

**通配符**

- `%`：匹配任意字符（包括零个字符）。
- `_`：匹配单个字符。

### 3.3 SQL通配符

在 SQL 中，通配符与 SQL LIKE 操作符或SQL REGEXP 操作符一起使用。SQL 通配符用于搜索表中的数据。

在 SQL 中，可使用以下通配符：

| 通配符                         | 描述                       |
| :----------------------------- | :------------------------- |
| %                              | 替代 0 个或多个字符        |
| _                              | 替代一个字符               |
| [*charlist*]                   | 字符列中的任何单一字符     |
| [^*charlist*] 或 [!*charlist*] | 不在字符列中的任何单一字符 |

例：

```mysql
# 选取 name 以 "G"、"F" 或 "s" 开始的所有网站
SELECT * FROM Websites
WHERE name REGEXP '^[GFs]';  # ^: 匹配字符串的开始位置。    # [GFs]: 匹配单个字符，该字符可以是 'G'、'F' 或 's'。
```

### 3.3 IN 操作符

IN 操作符允许您在 WHERE 子句中规定多个值。

```mysql
SELECT column1, column2, ...
FROM table_name
WHERE column IN (value1, value2, ...)
```

参数说明：

- **column1, column2, ...**：要选择的字段名称，可以为多个字段。如果不指定字段名称，则会选择所有字段。
- **table_name**：要查询的表名称。
- **column**：要查询的字段名称。
- **value1, value2, ...**：要查询的值，可以为多个值。

### 3.4 BETWEEN 操作符

BETWEEN 操作符选取介于两个值之间的数据范围内的值，这些值可以是数值、文本或者日期。

```
SELECT column1, column2, ...
FROM table_name
WHERE column BETWEEN value1 AND value2
```

参数说明：

- column1, column2, ...：要选择的字段名称，可以为多个字段。如果不指定字段名称，则会选择所有字段。
- table_name：要查询的表名称。
- column：要查询的字段名称。
- value1：范围的起始值。
- value2：范围的结束值。

### 3.4 AS 别名

通过使用 SQL AS操作符，可以为表名称或列名称指定别名。

**列的 SQL 别名语法**

```mysql
SELECT column_name AS alias_name
FROM table_name;
```

**表的 SQL 别名语法**

```mysql
SELECT column_name(s)
FROM table_name AS alias_name;
```

### 3.5 JOIN 连接

JOIN 子句用于把来自两个或多个表的行结合起来，**基于这些表之间的共同字段**。也可写为INNER JOIN。

![SQL INNER JOIN](G:\code\study\CppStudy\docs\figures\img_innerjoin.gif)

```mysql
SELECT column_name
FROM table1
JOIN table2
ON table1.column_name=table2.column_name;
```

**参数说明：**

- columns：要显示的列名。
- table1：表1的名称。
- table2：表2的名称。
- column_name：表中用于连接的列名。

### 3.6 LEFT JOIN 左连接

LEFT JOIN 关键字从左表（table1）返回所有的行，即使右表（table2）中没有匹配。如果**右表中没有匹配，则结果为 NULL**。

![SQL LEFT JOIN](G:\code\study\CppStudy\docs\figures\img_leftjoin.gif)

```mysql
SELECT column_name
FROM table1
LEFT JOIN table2
ON table1.column_name = table2.column_name
```

### 3.7 RIGHT JOIN 左连接

RIGHT JOIN 关键字从右表（table2）返回所有的行，即使左表（table1）中没有匹配。如果**左表中没有匹配，则结果为 NULL**。

![SQL RIGHT JOIN](G:\code\study\CppStudy\docs\figures\img_rightjoin.gif)

```mysql
SELECT column_name
FROM table1
RIGHT JOIN table2
ON table1.column_name=table2.column_name
```

### 3.8 FULL JOIN

FULL OUTER JOIN 关键字**只要左表（table1）和右表（table2）其中一个表中存在匹配**，则返回行。FULL OUTER JOIN 关键字**结合了 LEFT JOIN 和 RIGHT JOIN 的结果**。

![SQL FULL OUTER JOIN](G:\code\study\CppStudy\docs\figures\img_fulljoin.gif)

```mysql
SELECT column_name
FROM table1
FULL JOIN table2
ON table1.colunmn_name = table2.column_name
```

### 3.8 CROSS JOIN 笛卡尔积

`CROSS JOIN` 用于生成两个表的笛卡尔积。结果是两个表中每一行的组合。如果第一个表有 m 行，第二个表有 nnn 行，那么结果表将有 m×n 行。

```mysql
SELECT *
FROM table1
CROSS JOIN table2
```

**与FULL JOIN的区别总结**

- **`CROSS JOIN`**:
  - 生成两个表的笛卡尔积，即每一行与另一表的每一行组合。
  - 没有连接条件。
  - 结果行数为两个**表行数的乘积**。
- **`FULL JOIN`**:
  - 返回两个表中所有匹配的和不匹配的行。
  - 使用连接条件进行匹配。
  - 没有匹配的行会以 `NULL` 填充对应的列。
  - 结果行数为两个表中**行数最多的表的行数之和**。

### 3.8 UNION 组合

UNION操作符用于将两个或多个SELECT查询的结果合并到一个结果集中。UNION操作不同于使用合并两个表中的列的连接。union运算符**将两个源表中的所有行放在一个结果表中，从而创建一个新表**。

以下是使用UNION组合两个SELECT查询的结果集的基本规则：

- 在所有查询中，列的**数量和顺序**必须相同。
- 相应列的**数据类型必须兼容**。

```mysql
SELECT column_list
FROM table1_name
UNION 
SELECT column_list
FROM table2_name
```

### 3.9 GROUP BY 分组

GROUP BY子句与**SELECT语句**和**聚合函数**结合使用，以按通用列值将行分为不同的组，相同组的在一起，可以使用聚合函数操作。

```mysql
SELECT column1_name, SUM(column2_name)
FROM table_name
GROUP BY column2_name
```

### 3.10 HAVING 过滤组

HAVING子句通常与GROUP BY子句一起使用，以**指定组或集合的过滤条件**。HAVING子句**只能与SELECT语句一起使用**。

```mysql
SELECT t1.dept_name, count(t2.emp_id) AS total_employees
FROM departments AS t1 LEFT JOIN employees AS t2
ON t1.dept_id = t2.dept_id
GROUP BY t1.dept_name
HAVING total_employees = 0;		# 过滤掉不满足的组，目的：找没有员工的部门的名称
```





## 四、SQL中的一般函数

> - **聚合函数**: 操作一组数据，并返回一个单一的聚合结果。
> - **一般函数**: 操作每个单独的数据值，并返回相应的结果。

### 4.1 DATE_FORMAT 时间格式输出

`DATE_FORMAT(date, format)`: 用于以不同的格式显示日期/时间数据。`date` 参数是合法的日期，`format` 规定日期/时间的输出格式。

fomat格式，这里是一些常用的占位符以及它们代表的含义：

- `%Y`：4位数的年份（例如，2024）。
- `%m`：2位数的月份（例如，01 到 12）。
- `%d`：2位数的日（例如，01 到 31）。
- `%H`：2位数的小时（24小时制，00 到 23）。
- `%i`：2位数的分钟（00 到 59）。
- `%s`：2位数的秒（00 到 59）。
- `%w`：星期几（0=星期日，1=星期一，...，6=星期六）。
- `%a`：简写的星期几（Sun, Mon, ..., Sat）。
- `%b`：简写的月份（Jan, Feb, ..., Dec）。

### 4.2 DATE_SUB 从一个日期值中减去指定的时间间隔

`DATE_SUB` 函数用于从一个日期值中减去指定的时间间隔。这个函数在日期和时间操作中非常有用，比如计算某个日期之前的日期。

```mysql
DATE_SUB(date, INTERVAL expr unit)
```

- `date`: 这是一个合法的日期表达式（如日期、日期时间或时间戳）。
- `INTERVAL expr unit`: 指定要减去的时间间隔，其中 `expr` 是一个数字，`unit` 是时间单位（如 `DAY`, `MONTH`, `YEAR` 等）

### 4.3 DATE_ADD 从一个日期值中加上指定的时间间隔

`DATE_ADD` 函数用于从一个日期值中加上指定的时间间隔。这个函数在日期和时间操作中非常有用，比如计算某个日期之后的日期。

```mysql
DATE_ADD(date, INTERVAL expr unit)
```

- `date`: 这是一个合法的日期表达式（如日期、日期时间或时间戳）。
- `INTERVAL expr unit`: 指定要加上的时间间隔，其中 `expr` 是一个数字，`unit` 是时间单位（如 `DAY`, `MONTH`, `YEAR` 等）





## 五、SQL中的聚合函数

> - **聚合函数**: 操作一组数据，并返回一个单一的聚合结果。
> - **一般函数**: 操作每个单独的数据值，并返回相应的结果。

### 5.1 MIN 最小值

`MIN` 函数在 SQL 中用于返回一组值中的最小值。你可以在 `SELECT` 语句中使用它来**获取特定列中的最小值**。下面是一些示例和你的查询中的可能用法：

假设我们有一个表 `Employees`，包含以下数据：

| id   | name  | salary |
| ---- | ----- | ------ |
| 1    | Alice | 5000   |
| 2    | Bob   | 6000   |
| 3    | Carol | 7000   |
| 4    | Dave  | 8000   |

```mysql
SELECT MIN(salary) AS min_salary
FROM Employees;
```

这个查询将返回：

| min_salary |
| ---------- |
| 5000       |





## 六、SQL视图 View

### 6.1 什么是视图？

SQL视图（View）是一**种虚拟表**，它基于SQL查询的结果集定义。视图实际上**不包含任何数据**。而是**<font color='red'>存储SQL查询（尤其是复杂的SQL查询）</font>**。当你查询视图时，数据库系统会**动态执行视图中定义的查询**，并返回结果。

通过允许用户通过视图访问数据，而不是直接授予整个基表访问权限，视图也可以用作**安全机制**。

视图的优点和用途包括：

- **简化复杂查询**: 将复杂的SQL查询封装在视图中，可以简化后续的查询。例如，多个表的连接和筛选条件可以放在视图中，用户查询视图即可得到所需的数据。
- **数据安全性**: 可以通过视图限制用户访问特定的表或字段。用户只能查询视图，而不能直接访问底层表，从而保护敏感数据。
- **数据抽象**: 视图提供了一种数据抽象层，用户可以通过视图来查看数据，而不必了解底层表的结构。
- **提高重用性**: 可以将常用的查询逻辑放在视图中，方便在不同的查询中重用。

### 6.2 SQL创建视图

使用 `CREATE VIEW` 语句创建视图。其基本语法如下：

```mysql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

创建一个名为 `view_name` 的视图，基于一个 `SELECT` 查询的结果集定义。当你查询视图时，数据库系统会**动态执行视图中定义的查询**，并返回结果。

### 6.3 SQL替换现有视图



