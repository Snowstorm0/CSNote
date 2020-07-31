<!-- GFM-TOC -->
* [一、基础](#一基础)
* [二、创建表](#二创建表)
* [三、修改表](#三修改表)
* [四、插入](#四插入)
* [五、更新](#五更新)
* [六、删除](#六删除)
* [七、查询](#七查询)
* [八、排序](#八排序)
* [九、过滤](#九过滤)
* [十、通配符](#十通配符)
* [十一、计算字段](#十一计算字段)
* [十二、函数](#十二函数)
* [十三、分组](#十三分组)
* [十四、子查询](#十四子查询)
* [十五、连接](#十五连接)
* [十六、视图](#十六视图)
* [十七、存储过程](#十七存储过程)
* [十八、游标](#十八游标)
* [十九、触发器](#[十九触发器)
* [二十、事务管理](#二十事务管理)
* [二十一、字符集](#二十一字符集)
* [二十二、权限管理](#二十二权限管理)
* [参考资料](#参考资料)
<!-- GFM-TOC -->


# 一、基础

模式定义了数据如何存储、存储什么样的数据以及数据如何分解等信息，数据库和表都有模式。

主键的值不允许修改，也不允许复用（不能将已经删除的主键值赋给新数据行的主键）。

**SQL（Structured Query Language，结构化查询语言)**，标准 SQL 由 ANSI 标准委员会管理，从而称为 ANSI SQL。各个 DBMS 都有自己的实现，如 PL/SQL、Transact-SQL 等。

SQL 语句不区分大小写，但是数据库表名、列名和值是否区分依赖于具体的 DBMS 以及配置。

SQL 支持以下三种注释：

```sql
# 注释
SELECT *
FROM mytable; -- 注释
/* 注释1
   注释2 */
```

数据库创建与使用：

```sql
CREATE DATABASE test;
USE test;
```

# 二、创建表

**create table**

```sql
CREATE TABLE mytable (
  # int 类型，不为空，自增
  id INT NOT NULL AUTO_INCREMENT,
  # int 类型，不可为空，默认值为 1，不为空
  col1 INT NOT NULL DEFAULT 1,
  # 变长字符串类型，最长为 45 个字符，可以为空
  col2 VARCHAR(45) NULL,
  # 日期类型，可为空
  col3 DATE NULL,
  # 设置主键为 id
  PRIMARY KEY (`id`));
```

# 三、修改表

添加列 **add col**

```sql
ALTER TABLE mytable
ADD col CHAR(20);
```

删除列 **drop column**

```sql
ALTER TABLE mytable
DROP COLUMN col;
```

删除表 **drop table**

```sql
DROP TABLE mytable;
```

# 四、插入

普通插入 **insert into**

```sql
INSERT INTO mytable(col1, col2)
VALUES(val1, val2);
```

插入检索出来的数据 **select ……from**

```sql
INSERT INTO mytable1(col1, col2)
SELECT col1, col2
FROM mytable2;
```

将一个表的内容插入到一个新表 **create table**

```sql
CREATE TABLE newtable AS
SELECT * FROM mytable;
```

# 五、更新

**update**

```sql
UPDATE mytable
SET col = val
WHERE id = 1;
```

# 六、删除

**delete from**

```sql
DELETE FROM mytable
WHERE id = 1;
```

**truncate table**   可以**清空表**，也就是删除所有行。

```sql
TRUNCATE TABLE mytable;
```

使用更新和删除操作时一定要用 WHERE 子句，不然会把整张表的数据都破坏。可以先用 SELECT 语句进行测试，防止错误删除。

# 七、查询

## distinct

**select distinct** 用于返回**不重复的值**，**相同值只出现一次**。它作用于所有列，也就是说所有列的值都相同才算相同。

```sql
SELECT DISTINCT col1, col2
FROM mytable;
```

## limit

**限制返回的行数**。可以有两个参数，第一个参数为起始行，从 0 开始；第二个参数为返回的总行数。

返回前 5 行：

```sql
SELECT *
FROM mytable
LIMIT 5;
```

```sql
SELECT *
FROM mytable
LIMIT 0, 5;
```

返回第 3 \~ 5 行：

```sql
SELECT *
FROM mytable
LIMIT 2, 3;
```

## 组合查询

**UNION(union，并集)**:返回两个集合的并集。默认会去除相同行，如果需要保留相同行，使用 **UNION ALL**。每个查询必须包含相同的列、表达式和聚集函数。只能包含一个 ORDER BY 子句，并且必须位于语句的最后。

**INTERSECT(intersect，交集)**:返回两个集合的交集。

**EXCEPT(except，差集)**:比较两个查询的结果，返回左侧查询集合中不包含左右集合交集部分的非重复行（第一个表有，第二个表无）。

使用   UNION   来组合两个查询：

```sql
SELECT col
FROM mytable
WHERE col = 1
UNION
SELECT col
FROM mytable
WHERE col =2;
```

# 八、排序

-   **ASC**  ：升序 ascend（默认）
-   **DESC**  ：降序 descend 

可以按多个列进行排序，并且为每个列指定不同的排序方式：

```sql
SELECT *
FROM mytable
ORDER BY col1 DESC, col2 ASC;
```

# 九、过滤

**where**。不进行过滤的数据非常大，导致通过网络传输了多余的数据，从而浪费了网络带宽。因此尽量使用 SQL 语句来**过滤不必要的数据**，而不是传输所有的数据到客户端中然后由客户端进行过滤。

```sql
SELECT *
FROM mytable
WHERE col IS NULL;
```

下表显示了 WHERE 子句可用的操作符

|  操作符 | 说明  |
| :---: | :---: |
| = | 等于 |
| &lt; | 小于 |
| &gt; | 大于 |
| &lt;&gt; != | 不等于 |
| &lt;= !&gt; | 小于等于 |
| &gt;= !&lt; | 大于等于 |
| BETWEEN | 在两个值之间(between) |
| IS NULL | 为 NULL 值 |

应该注意到，NULL 与 0、空字符串都不同。

**AND 和 OR**   用于连接多个过滤条件。**优先处理 AND**，当一个过滤表达式涉及到多个 AND 和 OR 时，可以使用 () 来决定优先级，使得优先级关系更清晰。

**IN**   操作符用于匹配一组值，其后也可以接一个 SELECT 子句，从而匹配子查询得到的一组值。

**NOT**   操作符用于否定一个条件。

# 十、通配符

通配符也是用在过滤语句中，但它只能用于文本字段。

-   **%**   匹配 >=0 个任意字符；

-   **\_**   匹配 1 个任意字符；

-   **[ ]**   可以匹配集合内的字符，例如 [ab] 将匹配字符 a 或者 b。用脱字符 ^ 可以对其进行否定，也就是不匹配集合内的字符。

使用 **Like** 来进行通配符匹配。

```sql
SELECT *
FROM mytable
WHERE col LIKE '[^AB]%'; -- 不以 A 和 B 开头的任意文本
```

不要滥用通配符，通配符位于开头处匹配会非常慢。

# 十一、计算字段

在数据库服务器上完成数据的转换和格式化的工作往往比客户端上快得多，并且转换和格式化后的数据量更少的话可以减少网络通信量。

计算字段通常需要使用   **AS**   来取别名，否则输出的时候字段名为计算表达式。

```sql
SELECT col1 * col2 AS alias
FROM mytable;
```

**CONCAT()**   用于**连接(concat)**两个字段。许多数据库会使用空格把一个值填充为列宽，因此连接的结果会出现一些不必要的空格，使用 **TRIM()** 可以**去除(trim)首尾空格**。

```sql
SELECT CONCAT(TRIM(col1), '(', TRIM(col2), ')') AS concat_col
FROM mytable;
```

# 十二、函数

各个 DBMS （数据库管理系统，Database Management System）的函数都是不相同的，因此不可移植，以下主要是 MySQL 的函数。

## 汇总

|函 数 |说 明|
| :---: | :---: |
| AVG() | 返回**某列**的平均值 average |
| COUNT() | 返回某列的**行数**  count |
| MAX() | 返回某列的最大值 max |
| MIN() | 返回某列的最小值 min |
| SUM() |返回某列值之和 sum |

**AVG() 会忽略 NULL 行**。

使用 **DISTINCT** 可以汇总**不重复的值**。

```sql
SELECT AVG(DISTINCT col1) AS avg_col
FROM mytable;
```

## 文本处理

| 函数  | 说明  |
| :---: | :---: |
|  LEFT() | 左边的字符 left |
| RIGHT() | 右边的字符 right |
| LOWER() | 转换为小写字符 lower |
| UPPER() | 转换为大写字符 upper |
| LTRIM() | 去除左边的空格 left trim |
| RTRIM() | 去除右边的空格 right trim |
| LENGTH() | 长度 length |
| SOUNDEX() | 转换为语音值 soundex |

其中，  **SOUNDEX()**   可以将一个字符串转换为**描述其语音表示的字母数字模式**。

```sql
SELECT *
FROM mytable
WHERE SOUNDEX(col1) = SOUNDEX('apple')
```

## 日期和时间处理


- 日期格式：YYYY-MM-DD
- 时间格式：HH:<zero-width space>MM:SS

|函 数 | 说 明|
| :---: | :---: |
| ADDDATE() | 增加一个日期（天、周等） add date |
| ADDTIME() | 增加一个时间（时、分等） add time |
| CURDATE() | 返回当前日期 cur date |
| CURTIME() | 返回当前时间 cur time |
| DATE() |返回日期时间的日期部分 date|
| DATEDIFF() |计算两个日期之差 date diff|
| DATE_ADD() |高度灵活的日期运算函数 date add|
| DATE_FORMAT() |返回一个**格式化的日期或时间串** date for mat|
| DAY()| 返回一个日期的天数部分 day |
| DAYOFWEEK() |对于一个日期，返回对应的星期几 day of week|
| HOUR() |返回一个时间的小时部分 hour|
| MINUTE() |返回一个时间的分钟部分 minute|
| MONTH() |返回一个日期的月份部分 month|
| NOW() |返回当前日期和时间 now|
| SECOND() |返回一个时间的秒部分 second|
| TIME() |返回一个日期时间的时间部分 time|
| YEAR() |返回一个日期的年份部分 year|

```sql
mysql> SELECT NOW();
```

```
2018-4-14 20:25:11
```

## 数值处理

| 函数 | 说明 |
| :---: | :---: |
| SIN() | 正弦 sin |
| COS() | 余弦 cos |
| TAN() | 正切 tan |
| ABS() | 绝对值 abs |
| SQRT() | 平方根 sqrt |
| MOD() | 余数 mod |
| EXP() | 指数 exp |
| PI() | 圆周率 pi |
| RAND() | 随机数 rand |

# 十三、分组

**group by**。把具有**相同的数据值的行**放在同一组中。

可以对同一分组数据使用**汇总函数**进行处理，例如求分组数据的平均值等。

指定的分组字段除了能按该字段进行分组，也会自动按该字段进行排序。

```sql
SELECT col, COUNT(*) AS num
FROM mytable
GROUP BY col;
```

GROUP BY **自动按分组字段进行排序**，ORDER BY (**order by**)也可以按汇总字段来进行排序。

```sql
SELECT col, COUNT(*) AS num
FROM mytable
GROUP BY col
ORDER BY num;
```

WHERE(**where) 过滤行**，HAVING(**having) 过滤分组**，行过滤应当先于分组过滤。

```sql
SELECT col, COUNT(*) AS num
FROM mytable
WHERE col > 2
GROUP BY col
HAVING num >= 2;
```

分组规定：

- GROUP BY 子句出现在 WHERE 子句之后，ORDER BY 子句之前；
- 除了汇总字段外，SELECT 语句中的每一字段都必须在 GROUP BY 子句中给出；
- NULL 的行会单独分为一组；
- 大多数 SQL 实现不支持 GROUP BY 列具有可变长度的数据类型。

# 十四、子查询

**子查询就是嵌套在主查询中的查询**。子查询中只能返回一个字段的数据。子查询可以嵌套在主查询中所有位置，包括SELECT、FROM、WHERE、GROUP BY、HAVING、ORDER BY。

将子查询的结果作为 WHERE 语句的过滤条件：

```sql
SELECT *
FROM mytable1
WHERE col1 IN (SELECT col2
               FROM mytable2);
```

下面的语句可以检索出客户的订单数量，子查询语句会对第一个查询检索出的每个客户执行一次：

```sql
SELECT cust_name, (SELECT COUNT(*)
                   FROM Orders
                   WHERE Orders.cust_id = Customers.cust_id)
                   AS orders_num
FROM Customers
ORDER BY cust_name;
```

# 十五、连接

**join……on**。连接用于**连接多个表**，使用 JOIN 关键字，并且条件语句使用 ON 而不是 WHERE。

连接可以**替换子查询**，并且比子查询的效率一般会更快。

可以用 AS 给列名、计算字段和表名取别名，给表名取别名是为了简化 SQL 语句以及连接相同表。

## 内连接

**内连接**又称等值连接，使用 INNER JOIN (**inner join**)关键字。

```sql
SELECT A.value, B.value
FROM tablea AS A INNER JOIN tableb AS B
ON A.key = B.key;
```

可以不明确使用 INNER JOIN，而使用普通查询并在 WHERE 中将两个表中要连接的列用等值方法连接起来。

```sql
SELECT A.value, B.value
FROM tablea AS A, tableb AS B
WHERE A.key = B.key;
```

## 自连接

自连接可以看成**内连接的一种**，只是连接的表是自身而已。

一张员工表，包含员工姓名和员工所属部门，要找出与 Jim 处在同一部门的所有员工姓名。

子查询版本

```sql
SELECT name
FROM employee
WHERE department = (
      SELECT department
      FROM employee
      WHERE name = "Jim");
```

自连接版本

```sql
SELECT e1.name
FROM employee AS e1 INNER JOIN employee AS e2
ON e1.department = e2.department
      AND e2.name = "Jim";
```

## 自然连接

**nature join**。自然连接是把**同名列通过等值测试连接起来**的，同名列可以有多个。

内连接和自然连接的区别：**内连接提供连接的列**，而**自然连接**自动连接**所有同名列**。

```sql
SELECT A.value, B.value
FROM tablea AS A NATURAL JOIN tableb AS B;
```

## 交叉连接

**cross join**。交叉联接返回左表中的所有行，**左表中的每一行与右表中的所有行组合**。交叉联接也称作**笛卡尔积**。

## 外连接

外连接保留了没有关联的那些行。分为**左外连接(left outer join)**，**右外连接(right outer join)**以及**全外连接(full outer join)**。

①左外连接（*left outer join*）：返回指定**左表的全部行**+右表对应的行，如果左表中数据在右表中没有与其相匹配的行，则在查询结果集中显示为空值。 

②右外连接（*right outer join*）：与左外连接类似，是左外连接的反向连接。

②全外连接（*full outer join*）：完整外连接返回左表和右表中的**所有行**。左表在右表没有的地方显示NULL，右表在左表没有的地方显示NULL。（MYSQL不支持全外连接，适用于Oracle和DB2。）

检索所有顾客的订单信息，包括还没有订单信息的顾客。

```sql
SELECT Customers.cust_id, Customer.cust_name, Orders.order_id
FROM Customers LEFT OUTER JOIN Orders
ON Customers.cust_id = Orders.cust_id;
```

customers 表：

| cust_id | cust_name |
| :---: | :---: |
| 1 | a |
| 2 | b |
| 3 | c |

orders 表：

| order_id | cust_id |
| :---: | :---: |
|1    | 1 |
|2    | 1 |
|3    | 3 |
|4    | 3 |

结果：

| cust_id | cust_name | order_id |
| :---: | :---: | :---: |
| 1 | a | 1 |
| 1 | a | 2 |
| 3 | c | 3 |
| 3 | c | 4 |
| 2 | b | Null |

# 十六、视图

**create view**。**视图是虚拟的表**，本身不包含数据，也就不能对其进行索引操作。

**对视图的操作和对普通表的操作一样**。

视图具有如下好处：

- 简化复杂的 SQL 操作，比如复杂的连接；
- 只使用实际表的一部分数据；
- 通过只给用户访问视图的权限，保证数据的安全性；
- 更改数据格式和表示。

```sql
CREATE VIEW myview AS
SELECT Concat(col1, col2) AS concat_col, col3*col4 AS compute_col
FROM mytable
WHERE col5 = val;
```

# 十七、存储过程

存储过程可以看成是对**一系列 SQL 操作的批处理**。

使用存储过程的好处：

- 代码封装，保证了一定的安全性；
- 代码复用；
- 由于是预先编译，因此具有很高的性能。

命令行中创建存储过程需要自定义**分隔符(delimiter)**，因为命令行是以 ; 为结束符，而存储过程中也包含了分号，因此会错误把这部分分号当成是结束符，造成语法错误。

包含 in、out 和 inout 三种参数。

- IN 输入参数：表示调用者向过程传入值（传入值可以是字面量或变量）
- OUT 输出参数：表示过程向调用者传出值(可以返回多个值)（传出值只能是变量）
- INOUT 输入输出参数：既表示调用者向过程传入值，又表示过程向调用者传出值（值只能是变量）

给**变量赋值都需要用 select into 语句**。

每次只能给一个变量赋值，不支持集合的操作。

```sql
delimiter //

create procedure myprocedure( out ret int )
    begin
        declare y int;
        select sum(col1)
        from mytable
        into y;
        select y*y into ret;
    end //

delimiter ;
```

```sql
call myprocedure(@ret);
select @ret;
```

# 十八、游标

**declare……cursor**。在存储过程中使用游标可以对一个结果集进**行移动**遍历。

游标主要用于交互式应用，其中用户需要对数据集中的任意行进行浏览和修改。

使用游标的四个步骤：

1. **声明**游标，这个过程没有实际检索出数据；
2. **打开**游标；
3. **取出数据**；
4. **关闭**游标；

```sql
delimiter //
create procedure myprocedure(out ret int)
    begin
        declare done boolean default 0;

        declare mycursor cursor for
        select col1 from mytable;
        # 定义了一个 continue handler，当 sqlstate '02000' 这个条件出现时，会执行 set done = 1
        declare continue handler for sqlstate '02000' set done = 1;

        open mycursor;

        repeat
            fetch mycursor into ret;
            select ret;
        until done end repeat;

        close mycursor;
    end //
 delimiter ;
```

# 十九、触发器

**create trigger**。触发器会在某个表执行以下语句时而自动执行：**INSERT、UPDATE、DELETE**。

触发器必须指定在语句执行之前还是之后自动执行，之前执行使用 BEFORE (**before**)关键字，之后执行使用 AFTER (**after**)关键字。BEFORE 用于**数据验证和净化**，AFTER 用于**审计跟踪**，将修改记录到另外一张表中。

- INSERT 触发器包含一个名为 NEW  (new) 的虚拟表。


```sql
CREATE TRIGGER mytrigger AFTER INSERT ON mytable
FOR EACH ROW SELECT NEW.col into @result;

SELECT @result; -- 获取结果
```

- DELETE 触发器包含一个名为 OLD(**old**) 的虚拟表，并且是**只读**的。


- UPDATE 触发器包含一个名为 NEW (new) 和一个名为 OLD(old)  的虚拟表，其中 NEW 是可以被修改的，而 OLD 是只读的。


MySQL 不允许在触发器中使用 CALL 语句，也就是不能调用存储过程。

# 二十、事务管理

基本术语：

- **事务（transaction）**指一组 SQL 语句；
- **回退（rollback）**指撤销指定 SQL 语句的过程；
- **提交（commit）**指将未存储的 SQL 语句结果**写入数据库表**；
- **保留点（savepoint）**设置保存点，并和rollback结合使用，实现**回滚到指定保存点**。（与回退整个事务处理不同）。

不能回退 SELECT 语句，回退 SELECT 语句也没意义；也不能回退 CREATE 和 DROP 语句。

MySQL 的事务提交默认是**隐式提交**，**每执行一条语句就把这条语句当成一个事务然后进行提交**。当出现 START TRANSACTION (**start transaction**)语句时，会关闭隐式提交；当 COMMIT 或 ROLLBACK 语句执行后，事务会自动关闭，重新恢复隐式提交。

设置 autocommit 为 0 可以取消自动提交；autocommit 标记是针对每个连接而不是针对服务器的。

如果没有设置保留点，**ROLLBACK 会回退到 START TRANSACTION** 语句处；如果设置了保留点，并且在 ROLLBACK 中指定该保留点，则会回退到该保留点。

```sql
START TRANSACTION
// ...
SAVEPOINT delete1
// ...
ROLLBACK TO delete1
// ...
COMMIT
```

# 二十一、字符集

**character set**。一个**字符集（character set）**对应了一个默认的**校验规则（collation）**，用于指定数据集是如何排序的，以及字符串的比对规则。

基本术语：

- 字符集为字母和符号的集合；
- 编码为某个字符集成员的内部表示；
- 校对字符指定如何比较，主要用于**排序和分组**。

除了给表指定字符集和校对外，也可以给列指定：

```sql
CREATE TABLE mytable
(col VARCHAR(10) CHARACTER SET latin COLLATE latin1_general_ci )
DEFAULT CHARACTER SET hebrew COLLATE hebrew_general_ci;
```

可以在排序、分组时指定校对：

```sql
SELECT *
FROM mytable
ORDER BY col COLLATE latin1_general_ci;
```

# 二十二、权限管理

**use mysql**。MySQL 的账户信息保存在 **mysql 这个数据库中**。

```sql
USE mysql;
SELECT user FROM user;
```

**创建账户**  create user

新创建的账户没有任何权限。

```sql
CREATE USER myuser IDENTIFIED BY 'mypassword';
```

**修改账户名**   rename user

```sql
RENAME USER myuser TO newuser;
```

**删除账户**  drop user

```sql
DROP USER myuser;
```

**查看权限**  show grants for

```sql
SHOW GRANTS FOR myuser;
```

**授予权限**  grant ……to

账户用 username@host 的形式定义，username@% 使用的是默认主机名。

```sql
GRANT SELECT, INSERT ON mydatabase.* TO myuser;
```

**删除权限**  grant/revoke

GRANT (grant)和 REVOKE (revoke)可在几个层次上控制访问权限：

- 整个服务器，使用 GRANT ALL 和 REVOKE ALL；
- 整个数据库，使用 **ON database.\***；
- 特定的表，使用 **ON database.table**；
- 特定的列；
- 特定的存储过程。

```sql
REVOKE SELECT, INSERT ON mydatabase.* FROM myuser;
```

**更改密码**  set password for

必须使用 **Password() 函数**进行加密。

```sql
SET PASSWROD FOR myuser = Password('new_password');
```








<div align="center"><img width="320px" src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/githubio/公众号二维码-2.png"></img></div>
