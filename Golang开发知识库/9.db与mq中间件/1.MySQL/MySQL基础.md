## MySQL基础

### 二、SQL简介

对数据库进行查询和修改操作的语言叫做 SQL（Structured Query Language，结构化查询语言）。SQL是专为数据库而建立的操作命令集，是一种功能齐全的数据库语言。

SQL 是一种数据库查询和程序设计语言，用于存取数据以及查询、更新和管理关系数据库系统。与其他程序设计语言（如 C语言、Java 等）不同的是，SQL 由很少的关键字组成，每个 SQL 语句通过一个或多个关键字构成。

在使用它时，只需要发出“做什么”的命令，“怎么做”是不用使用者考虑的。SQL功能强大、简单易学、使用方便，已经成为了数据库操作的基础，并且现在几乎所有的数据库(Oracle、DB2、Sybase、SQL Server )均支持sql。

SQL的规范：

> 1. 在数据库系统中，SQL语句不区分大小写(建议用大写) 。但字符串常量区分大小写。建议命令大写，表名库名小写；
>
> 2. SQL语句可单行或多行书写，以“;”结尾。关键词不能跨多行或简写。
>
> 3. 用空格和缩进以及折行来提高语句的可读性。子句通常位于独立行，便于编辑，提高可读性。
>
> ```sql 
> SELECT column1, column2, column3
> FROM table1
> WHERE column1 = 'value1'
>   AND column2 = 'value2'
>   AND column3 = 'value3'
> ```
>
> 4. 注释：
>
> ```sql
> -- 单行注释
> /*
> 多行注释
> */
> ```

### 三、数据库操作

![截屏2024-02-16 13.50.04](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-16%2013.50.04-8074458-8074459-8074460-8074589.png)

```sql
-- 1.创建数据库（在磁盘上创建一个对应的文件夹）
    create database [if not exists] db_name [character set xxx] 
   
-- 2.查看数据库
    show databases;  -- 查看所有数据库
    SHOW DATABASES LIKE '%test%';-- 查看名字中包含 test 的数据库
    show create database db_name; -- 查看数据库的创建方式

-- 3.修改数据库
    alter database db_name [character set xxx] 

-- 4.删除数据库
    drop database [if exists] db_name;
    
-- 5.使用数据库
    use db_name; -- 切换数据库  注意：进入到某个数据库后没办法再退回之前状态，但可以通过use进行切换
    select database(); --  查看当前使用的数据库
    
-- 数据库备份
   mysqldump -u username -p password database_name > backup.sql
```

> 使用 DROP DATABASE 命令时要非常谨慎，在执行该命令后，MySQL 不会给出任何提示确认信息。
>
> DROP DATABASE 删除数据库后，数据库中存储的所有数据表和数据也将一同被删除，而且不能恢复。因此最好在删除数据库之前先将数据库进行备份。

### 四、数据表操作

数据表是数据库的重要组成部分，每一个数据库都是由若干个数据表组成的。比如，在电脑中一个文件夹有若干excel文件。这里的文件夹就相当于数据库，excel文件就相当于数据表。

MySQL 的数据类型有大概可以分为 5 种，分别是整数类型、浮点数类型、日期和时间类型、字符串类型、二进制类型等。

#### 4.1、数据库数据类型

<img src="MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-16%2018.51.50-8080736.png" alt="截屏2024-02-16 18.51.50" style="zoom:50%;" />

![截屏2024-02-16 13.10.16](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-16%2013.10.16-8061482.png)

在 MySQL 中，布尔类型被称为 `BOOLEAN` 或 `BOOL` 类型。`BOOLEAN` 类型用于存储布尔值，即 `TRUE`（真）或 `FALSE`（假）。

MySQL 中的布尔类型可以有以下几种表示方式：

1. 整数类型：布尔类型可以表示为整数类型，其中 `0` 表示 `FALSE`，非零整数表示 `TRUE`。常见的表示 `TRUE` 的整数值为 `1`。
2. 字符串类型：布尔类型也可以表示为字符串类型，其中 `'0'` 表示 `FALSE`，而 `'1'` 表示 `TRUE`。这种表示方式更贴近人类可读的布尔值。

需要注意的是，尽管 MySQL 支持布尔类型的不同表示方式，但它并没有专门的存储布尔值的数据类型。在实际使用中，你可以选择使用 `TINYINT(1)` 或 `VARCHAR(1)` 等其他数据类型来存储布尔值，但约定使用 `0` 和 `1` 或 `'0'` 和 `'1'` 来表示布尔值。

![截屏2024-02-16 13.29.48](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-16%2013.29.48-8074023-8074024.png)

![截屏2024-02-16 13.30.01](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-16%2013.30.01-8074050-8074052.png)

BIT数据类型可以用来存储布尔值。

```sql
CREATE TABLE my_table (
    is_active BIT(1)
);
INSERT INTO my_table (is_active) VALUES (1); -- 存储真值
INSERT INTO my_table (is_active) VALUES (0); -- 存储假值
```

#### 4.2、创建数据表

```sql 
-- 语法
CREATE TABLE tab_name(
            field1 type [约束条件],
            field2 type,
            ...
            fieldn type    -- 一定不要加逗号，否则报错！
        )[character set utf8];
```

案例：

```sql 
 CREATE TABLE student(          
            name varchar(20),
            gender bit,
            age int,
            birth date,
            gpa double(8,2) unsigned, -- 平均绩点（无符号，不能为负值）
          )character set=utf8;
```

```sql 
-- show tables;
```

#### 4.3、约束

约束是一种限制，它通过限制表中的数据，来确保数据的完整性和唯一性。使用约束来限定表中的数据很多情况下是很有必要的。在 MySQL 中，约束是指对表中数据的一种约束，能够帮助数据库管理员更好地管理数据库，并且能够确保数据库中数据的正确性和有效性。例如，在数据表中存放年龄的值时，如果存入 200、300 这些无效的值就毫无意义了。因此，使用约束来限定表中的数据范围是很有必要的。

添加记录：

```sql
INSERT <表名> 字段1,...字段n VALUES (值1,...值n) ;
```

##### 【1】非空约束

非空约束用来约束表中的字段不能为空。比如，在用户信息表中，如果不添加用户名，那么这条用户信息就是无效的，这时就可以为用户名字段设置非空约束。

创建表时可以使用`NOT NULL`关键字设置非空约束

```sql 
CREATE TABLE user(
    name VARCHAR(22),
    age int
);
insert user (name) values ("yuan");
 
CREATE TABLE user(
    name VARCHAR(22),
    age int NOT NULL
);
 insert user (name) values ("yuan");
```

##### 【2】唯一约束

唯一约束（Unique Key）是指所有记录中字段的值不能重复出现。例如，为name字段加上唯一性约束后，每条记录的name值都是唯一的，不能出现重复的情况。

创建表时可以使用`UNIQUE`关键字设置唯一约束

例如，在用户信息表中，要避免表中的用户名重名，就可以把用户名列设置为唯一约束。

```sql 
CREATE TABLE user(
    name VARCHAR(22),
    age int
);
insert user (name) values ("yuan");
insert user (name) values ("yuan");

CREATE TABLE user(
    name VARCHAR(22) UNIQUE,
    age int
);
insert user (name) values ("yuan");
```

##### 【3】默认值约束

默认值约束用来约束当数据表中某个字段不输入值时，自动为其添加一个已经设置好的值。

创建表时可以使用`DEFAULT`关键字设置默认值约束

````sql 
CREATE TABLE user(
    name VARCHAR(22),
    gender bit
);
insert user (name) values ("yuan");

CREATE TABLE user(
    name VARCHAR(22),
    gender bit default 1
);
insert user (name) values ("yuan");

CREATE TABLE user(
    name VARCHAR(22),
    gender varchar(2) default "保密"
);
````

##### 【4】主键约束

主键约束是使用最频繁的约束。在设计数据表时，一般情况下，都会要求表中设置一个主键。主键是表的一个特殊字段，该字段能唯一标识该表中的每条信息。

```sql 
CREATE TABLE user(
    name VARCHAR(22),
    age int,
    gender varchar(1) default "男"
);
insert user (name,age) values ("yuan",18);
insert user (name,age) values ("rain",28);
insert user (name,age) values ("eric",23);
insert user (name,age) values ("yuan",18);

-- 没有唯一能标识该表中的每条记录的字段值

CREATE TABLE user(
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(20)
);
insert user (name) values ("yuan");
select * from user;
```

> 1. 一张表中最多只能有一个主键
> 2. 主键类型不一定必须是整型
> 3. 表中如果没有设置主键，默认设置NOT NULL和UNIQUE的字段为主键；此外，表中如果有多个NOT NULL和UNIQUE的字段，则按顺序将第一个设置NOT NULL和UNIQUE的字段设为主键。所以主键一定是非空且唯一，但非空且唯一的字段不一定是主键。

讲完约束，最后学生表就可以调整为

```sql
CREATE TABLE student(
            id int primary key auto_increment ,
            name varchar(20) not null,
            gender bit default 1,
            age int,
            birth date,
            gpa double(8,2) unsigned -- 平均绩点（无符号，不能为负值）
          )character set=utf8;
```

#### 4.4、查看表

```sql 
desc employee;    -- 查看表结构,等同于show columns from tab_name  
show tables 　　　　　　　　　　　-- 查看当前数据库中的所有的表
show create table tab_name      -- 查看当前数据库表建表语句 
```

#### 4.5、删除表

```sql 
DROP TABLE [IF EXISTS] 表名1 [ ,表名2, 表名3 ...]
```

#### 4.6、修改表结构

##### 修改表名和编码

* 修改表名  

  ```sql
  ALTER TABLE <旧表名> RENAME [TO] <新表名>；
  ```

* 修该表所用的字符集   

  ```sql
  ALTER TABLE <表名> CHARACTER SET <字符集名> 
  ```

##### 修改表字段

* 增加列(字段)

  ```sql
  ALTER TABLE <表名> ADD <新字段名><数据类型>[约束条件]［first｜after 字段名］;
  ```

* 增加多个字段

  ```sql
  alter table users2 
        add addr varchar(20),
        add age  int first,
        add birth varchar(20) after name;
  ```

* 删除某字段

  ```sql
  ALTER TABLE <表名> DROP <字段名>；
  ```

* 修改某字段类型

  ```sql
  ALTER TABLE <表名> MODIFY <字段名> <数据类型> [完整性约束条件]［first｜after 字段名］;
  ```

* 修改某字段名

  ```sql
  ALTER TABLE <表名> CHANGE <旧字段名> <新字段名> <新数据类型>  [完整性约束条件]［first｜after 字段名］;
  ```

### 五、表记录操作

#### 5.1、添加记录

INSERT 语句有两种语法形式，分别是 INSERT…VALUES 语句和 INSERT…SET 语句。

##### 【1】INSERT…VALUES语句

```sql 
INSERT INTO table_name (column1, column2, ...)
VALUES (value1, value2, ...);
```

> 1. INSERT 语句后面的列名称顺序可以不是 表定义时的顺序，即插入数据时，不需要按照表定义的顺序插入，只要保证值的顺序与列字段的顺序相同就可以。
> 2. 若向表中的所有列插入数据，则全部的列名均可以省略，直接采用 INSERT<表名>VALUES(…) 即可。
> 3. 使用 INSERT…VALUES 语句可以向表中插入一行数据，也可以插入多行数据；
>
> ````sql 
> INSERT [INTO] <表名> [ <列名1> [ , … <列名n>] ] VALUES (值1…,值n),
>                                                       (值1…,值n),
>                                                       ...
>                                                       (值1…,值n);
> -- 用单条 INSERT 语句处理多个插入要比使用多条 INSERT 语句更快。
> ````

案例：

```sql 
CREATE TABLE student
(
    id     int primary key auto_increment,
    name   varchar(20) not null,
    gender bit default 1,
    age    int,
    birth  date
)character set=utf8;

-- 单行插入
INSERT
student (name,gender,age,birth) VALUES
              ("yuan",1,18,"2002-11-12"),

-- 多行批量插入
INSERT student (name,gender,age,birth) VALUES

("张三",1,22,"2000-12-12"),
("李四",1,32,"1990-12-12"),
("王五",0,42,"1980-06-06");
```

##### 【2】 INSERT…SET语句

```sql 
INSERT INTO table_name
SET column1 = value1, column2 = value2, ...;
```

此语句用于直接给表中的某些列指定对应的列值，即要插入的数据的列名在 SET 子句中指定。对于未指定的列，列值会指定为该列的默认值。

#### 5.2、查询记录

标准语法：

```sql 
-- 查询语法：

   SELECT *|field1,filed2 ...   FROM tab_name
                  WHERE 条件
                  GROUP BY field
                  HAVING 筛选
                  ORDER BY field
                  LIMIT 限制条数


-- Mysql在执行sql语句时的执行顺序：
                -- from  where  select  group by  having  order by
```

准备数据：

```sql 
CREATE TABLE emp(
    id       INT PRIMARY KEY AUTO_INCREMENT,
    name     VARCHAR(20),
    gender   ENUM("male","female","other"),
    age      TINYINT,
    dep      VARCHAR(20),
    province VARCHAR(20),
    salary    DOUBLE(7,2),
    birthday DATE
) CHARACTER SET=utf8;


INSERT INTO emp (name, gender, age, dep, province, salary, birthday)
VALUES ("George", "male", 24, "教学部", "河北省", 6000, '1999-02-18'),
       ("danae", "male", 32, "运营部", "北京", 12000, '1992-07-12'),
       ("Sera", "male", 38, "运营部", "河北省", 7000, '1986-11-05'),
       ("Echo", "male", 19, "运营部", "河北省", 9000, '2002-03-28'),
       ("Abel", "female", 24, "销售部", "北京", 9000, '1999-09-10'),
       ("John", "male", 28, "教学部", "山东省", 8000, '1993-06-15'),
       ("Alice", "female", 32, "运营部", "北京", 10000, '1990-12-25'),
       ("Bob", "male", 24, "教学部", "河北省", 9000, '1999-08-03'),
       ("Cather", "female", 28, "销售部", "山东省", 6000, '1993-04-20'),
       ("David", "male", 34, "销售部", "山东省", 12000, '1987-09-01'),
       ("Emily", "female", 22, "教学部", "北京", 7000, '1999-01-08'),
       ("Frank", "male", 24, "教学部", "河北省", 9000, '1999-07-17'),
       ("Grace", "female", 32, "运营部", "北京", 8000, '1990-05-30'),
       ("Henry", "male", 38, "运营部", "河北省", 9000, '1984-03-12'),
       ("Ivy", "female", 19, "运营部", "河北省", 9000, '2002-08-22'),
       ("Jack", "male", 24, "销售部", "北京", 8000, '1999-10-05'),
       ("Kelly", "female", 28, "教学部", "山东省", 10000, '1993-02-28'),
       ("Leo", "male", 32, "运营部", "北京", 6000, '1990-11-11'),
       ("Megan", "female", 24, "教学部", "河北省", 12000, '1999-06-03'),
       ("Nick", "male", 34, "销售部", "山东省", 7000, '1987-07-08'),
       ("Olivia", "female", 22, "销售部", "山东省", 9000, '1999-03-18'),
       ("Peter", "male", 24, "教学部", "北京", 8000, '1999-12-01'),
       ("Queen", "female", 32, "运营部", "河北省", 9000, '1990-09-15'),
       ("Ryan", "male", 38, "运营部", "北京", 8000, '1984-11-20'),
       ("Sandy", "female", 19, "运营部", "河北省", 10000, '2002-05-07'),
       ("Tom", "male", 24, "销售部", "北京", 7000, '1999-04-27'),
       ("Uma", "female", 28, "教学部", "山东省", 9000, '1993-08-14'),
       ("Victor", "male", 32, "运营部", "北京", 6000, '1990-02-05'),
       ("Wendy", "female", 24, "教学部", "河北省", 12000, '1999-07-23'),
       ("Xander", "male", 34, "销售部", "山东省", 7000, '1987-12-16'),
       ("Yvonne", "female", 22, "销售部", "山东省", 9000, '1999-01-30'),
       ("Zack", "male", 24, "教学部", "北京", 8000, '1999-10-13');
```

##### 【1】查询字段（select）

```sql 
mysql> SELECT  * FROM emp;
mysql> SELECT  name,dep,salary FROM emp;
```

##### 【2】where语句

````sql 
-- where字句中可以使用：

      -- 比较运算符：
      > < >= <= <> != = 
      between 10 and 100 值在10到100之间
      in(10,20,30) 值是10或20或30
      like 'yuan%'
      /*
      pattern可以是%或者_，
      如果是%则表示任意多字符，此例如唐僧,唐国强
      如果是_则表示一个字符唐_，只有唐僧符合。两个_则表示两个字符：__
      */

      -- 正则 
                  SELECT * FROM emp WHERE emp_name REGEXP '^yu';
                  SELECT * FROM emp WHERE name REGEXP 'n$';  

     -- 逻辑运算符
                  在多个条件直接可以使用逻辑运算符 and or not
                                              
````

练习：

```sql 
-- 查询年纪大于24的员工
SELECT * FROM emp WHERE age > 24;

-- 查询年龄在20到30之间的员工
SELECT * FROM emp WHERE age between 20 and 30;

-- 查询工资等于7000，8000和9000的所有员工
SELECT * FROM emp WHERE salary in (7000,8000,9000);

-- 查询名字以A开头的员工
SELECT * FROM emp WHERE name like "A%";

-- 查询名字包含A的员工
SELECT * FROM emp WHERE name like "%A%";
SELECT * FROM emp WHERE REGEXP_LIKE(name, '.*A.*',"c");

-- 查询教学部的男老师信息
SELECT * FROM emp WHERE dep="教学部" AND gender="male";

-- 查询名字以A开头的员工并且工资大于等于10000的员工的姓名
SELECT * FROM emp WHERE name like "A%" and salary>=10000;

-- 查询年龄小于25或工资低于10000的员工
SELECT * FROM emp WHERE age<25 or  salary < 8000;


-- 日期相关的
-- 1990-01-01后出生的员工
SELECT * FROM emp WHERE birthday > '1990-01-01';

-- 查询生日在12月的员工
SELECT * FROM emp WHERE MONTH(birthday) = 12;

-- 查询所有魔羯座的员工
select * from emp where (month(birthday) = 12 and day(birthday) > 22)
					 or (month(birthday) = 1 and day(birthday) < 19)
```

##### 【3】order：排序

按指定的列进行，排序的列即可是表中的列名，也可以是select语句后指定的别名。

```sql 
-- 语法：
select *|field1,field2... from tab_name order by field [Asc|Desc]
 -- Asc 升序、Desc 降序，其中asc为默认值 ORDER BY 子句应位于SELECT语句的结尾。
```

练习：

````sql 
-- 按年龄从高到低进行排序
SELECT * FROM emp ORDER BY age DESC ;

-- 按工资从低到高进行排序
SELECT * FROM emp ORDER BY salary;

-- 先按工资排序，工资相同的按年龄排序
SELECT * FROM emp ORDER BY salary,age;
````

##### 【4】group by：分组查询

GROUP BY 语句根据某个列对结果集进行分组。分组一般配合着聚合函数完成查询。

**常用聚合(统计)函数**

- `max()`：最大值。
- `min()`：最小值。
- `avg()`：平均值。
- `sum()`：总和。
- `count()`：个数。

在MySQL的SQL执行逻辑中，where条件必须放在group by前面！也就是先通过where条件将结果查询出来，再交给group by去分组，完事之后进行统计，统计之后的查询用having。

练习：

```sql 
-- 查询男女员工各有多少人
SELECT gender,count(*) as employee_count FROM emp GROUP BY gender;

-- 查询教学部门的最高工资
SELECT dep, max(salary) as mas_salary FROM emp GROUP BY dep HAVING dep = '教学部';

-- 查询平均薪水超过8000的部门
SELECT dep, avg(salary) as avg_salary FROM emp GROUP BY dep HAVING avg(salary) >= 8000;

-- 查询每个组的员工姓名
SELECT dep,group_concat(name) as '该部门所有员工姓名' FROM emp GROUP BY dep;

-- 查询公司一共有多少员工(可以将所有记录看成一个组)
SELECT count(*) as '员工数量' FROM emp;

-- 每年出生的员工人数
SELECT year(birthday),count(*) as '该年出生的员工数量' from emp GROUP BY year(birthday);

-- 查询公司所有员工的平均工资
SELECT avg(salary) from emp;
```

###### where和having的区别

where和having都是筛选条件，where一般在group by前面用，它用于在数据分组之前对行进行过滤；having只能在group by后面用，它用于对分组后的结果进行过滤。

##### 【5】limit：记录条数限制

```sql 
SELECT 列1, 列2, ...
FROM 表名
LIMIT n OFFSET m;
-- 或者写成
LIMIT m, n  -- 这种写法在部分数据库中适用
-- 这里的 m 表示跳过的行数，n 表示返回的行数。


SELECT * from emp limit 10;	-- 跳过0条记录返回10条记录（返回前十条记录）
SELECT * from emp limit 2,5;	-- 跳过前2条显示接下来的5条记录，即返回第3~7条记录
SELECT * from emp limit 2,2;
```

##### 【6】distinct：查询去重

```sql 
-- 获取员工表中不重复的年龄值和薪水值，并按照相应的字段进行升序排序。
SELECT distinct age from emp order by age;
SELECT distinct salary from emp order by salary;
```

#### 5.3、更新记录

```sql 
UPDATE <表名> SET 字段1=值1 [,字段2=值2… ] [WHERE 子句 ]
```

案例:

```sql
-- 更新员工职位和工资
update emp set salary = 99999 where name = 'yuan';

-- 更改部门名称
update emp set dep = '用户增长部' where dep = '运营部';

-- 年龄大于35岁的员工薪资增加百分之十五
update emp set salary = salary * 1.15 where age > 35;

-- 薪资最高的五个人降薪百分之三十
update emp set salary = salary * 0.7 order by salary desc limit 5;
```

#### 5.4、删除记录

```sql 
DELETE FROM <表名> [WHERE 子句] [ORDER BY 子句] [LIMIT 子句]
```

> - `<表名>`：指定要删除数据的表名。
> - `ORDER BY` 子句：可选项。表示删除时，表中各行将按照子句中指定的顺序进行删除。
> - `WHERE` 子句：可选项。表示为删除操作限定删除条件，若省略该子句，则代表删除该表中的所有行。
> - `LIMIT` 子句：可选项。用于告知服务器在控制命令被返回到客户端前被删除行的最大值。

```sql
-- 删除薪资最高的五个人，相同薪资按年龄优先
-- 删除教学部年龄最大的男老师
```

### 今日作业

题目要求：

1. 创建一个名为`shop`的数据库
2. 在`shop`数据库创建一个名为 `orders` 的表，包含以下字段：
   - id (整数型，主键，自增)
   - customer_id (整数型，顾客ID)
   - order_date (日期型，订单日期)
   - order_time (时间型，订单时间)
   - total_amount (浮点型，订单总额)
   - status (字符串型，订单状态)
   - shipping_address (字符串型，配送地址)
   - payment_method (字符串型，支付方式)
   - payment_status (字符串型，支付状态)
3. 创建表：

```sql
CREATE TABLE orders
(
    id               INT PRIMARY KEY AUTO_INCREMENT,
    customer_id      INT,
    order_date       DATE,
    order_time       TIME,
    total_amount     DECIMAL(10, 2),
    status           VARCHAR(50),
    shipping_address VARCHAR(100),
    payment_method   VARCHAR(50),
    payment_status   VARCHAR(50)
)
```

4. 插入订单记录

```sql
INSERT INTO orders (order_id, customer_id, order_date, order_time, total_amount, status, shipping_address, payment_method, payment_status)
VALUES
(1, 1, '2024-05-07', '10:00:00', 100.50, '已完成', '北京市朝阳区', '微信支付', '已支付'),
(2, 2, '2024-05-07', '11:30:00', 56.20, '已发货', '上海市浦东新区', '支付宝', '已支付'),
(3, 3, '2024-05-07', '13:45:00', 220.00, '待发货', '广州市天河区', '微信支付', '已支付'),
(4, 4, '2024-05-07', '14:20:00', 75.80, '已取消', '深圳市福田区', '支付宝', '已支付'),
(5, 5, '2024-05-07', '16:10:00', 180.90, '已发货', '成都市武侯区', '支付宝', '已支付'),
(6, 1, '2024-05-07', '09:30:00', 50.00, '待付款', '北京市朝阳区', NULL, '未支付'),
(7, 2, '2024-05-08', '14:00:00', 120.00, '待付款', '上海市静安区', NULL, '未支付'),
(8, 3, '2024-05-08', '16:30:00', 89.50, '待付款', '广州市越秀区', NULL, '未支付'),
(9, 4, '2024-05-08', '18:45:00', 75.20, '待付款', '深圳市南山区', NULL, '未支付'),
(10, 5, '2024-05-08', '20:20:00', 200.00, '待付款', '成都市锦江区', NULL, '未支付'),
(11, 1, '2024-05-09', '11:15:00', 150.80, '已发货', '北京市海淀区', '微信支付', '已支付'),
(12, 2, '2024-05-09', '13:40:00', 95.60, '已发货', '上海市徐汇区', '支付宝', '已支付'),
(13, 3, '2024-05-09', '16:00:00', 180.00, '已完成', '广州市白云区', '微信支付', '已支付'),
(14, 4, '2024-05-09', '18:20:00', 60.50, '已完成', '深圳市龙岗区', '支付宝', '已支付'),
(15, 5, '2024-05-09', '20:45:00', 210.30, '已发货', '成都市高新区', '微信支付', '已支付'),
(16, 1, '2024-05-10', '09:10:00', 80.00, '已完成', '北京市朝阳区', '支付宝', '已支付'),
(17, 2, '2024-05-10', '12:30:00', 120.50, '待发货', '上海市浦东新区', '微信支付', '已支付'),
(18, 3, '2024-05-10', '15:15:00', 45.20, '已取消', '广州市天河区', '支付宝', '已支付'),
(19, 4, '2024-05-10', '17:40:00', 160.90, '待发货', '深圳市福田区', '微信支付', '已支付'),
(20, 5, '2024-05-10', '17:41:00', 78.90, '待发货', '深圳市福田区', '微信支付', '已支付'),
(21, 1, '2024-05-11', '10:30:00', 75.60, '待付款', '北京市朝阳区', NULL, '未支付'),
(22, 2, '2024-05-11', '12:45:00', 90.20, '已发货', '上海市静安区', '支付宝', '已支付'),
(23, 3, '2024-05-11', '15:00:00', 200.00, '已发货', '广州市越秀区', '微信支付', '已支付'),
(24, 4, '2024-05-11', '17:20:00', 65.80, '已取消', '深圳市南山区', '支付宝', '已支付'),
(25, 5, '2024-05-11', '19:40:00', 180.90, '待发货', '成都市锦江区', '微信支付', '已支付'),
(26, 1, '2024-05-12', '10:15:00', 150.80, '已完成', '北京市海淀区', '支付宝', '已支付'),
(27, 2, '2024-05-12', '12:30:00', 95.60, '已完成', '上海市徐汇区', '微信支付', '已支付'),
(28, 3, '2024-05-12', '15:10:00', 180.00, '已发货', '广州市白云区', '支付宝', '已支付'),
(29, 4, '2024-05-12', '17:35:00', 60.50, '已发货', '深圳市龙岗区', '微信支付', '已支付'),
(30, 5, '2024-05-12', '20:00:00', 210.30, '待付款', '成都市高新区', NULL, '未支付');
```

5. 完成相应查询

```sql
-- 查询订单表中总订单金额。
-- 查询订单表中订单总额最高的订单记录。
-- 查询订单ID为5的订单详情。
-- 查询客户ID为3的所有订单记录。
-- 查询订单状态为"已发货"的订单数量。
-- 查询总金额超过100元的订单数量。
-- 查询订单表中各个订单状态的订单数量。
-- 查询订单表中每个客户的订单数量。
-- 查询订单表中每个客户的总订单金额。
-- 查询订单表中订单总额超过平均订单金额的订单记录。
-- 查询订单表中最近一周内的订单记录。
-- 查询订单表中每个客户的最近一笔订单记录。
-- 查询订单日期为2024-05-07的所有订单记录。
-- 查询支付方式为"微信支付"的订单数量。
-- 查询订单总金额最高的订单信息。
-- 查询收货地址中包含"北京市"关键词的订单记录。
-- 查询每个客户的订单总数，并按订单总数降序排列。
-- 查询每个客户的订单总金额，并按订单总金额升序排列。
-- 查询订单日期为2024-05-10且总金额大于50的订单数量。
-- 查询订单状态不为"已取消"的订单数量。
-- 查询每个客户的订单平均金额，并按平均金额降序排列。
-- 查询支付方式为空值或为默认值的订单数量。
-- 删除客户ID为3且订单状态为"已取消"的所有订单记录
-- 更新订单状态为"待付款"且订单日期为2024-05-10的所有订单的收货地址为"上海市浦东区"
```

## MySQL基础（二）

![截屏2024-02-21 16.59.06](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-21%2016.59.06-8505967.png)

### 一、表的关联关系

在关系型数据库中，表之间可以通过关联关系进行连接和查询。关联关系是指两个或多个表之间的关系，通过共享相同的列或键来建立连接。常见的关联关系有三种类型：一对多关系，多对多关系以及一对一关系。

我们以选课系统为案例带大家深入解析关系表。

#### 1.1、一对多

##### （1）一对多的关系创建

* 在一对多关系中，一个表的记录可以与另一个表中的多个记录相关联，而另一个表的记录只能与一个表中的记录相关联（有且只有一个一对多）。
* 建立一对多的关系：**在‘多’的表中创建关联字段**

```sql
CREATE TABLE Students
(
    StudentID    INT PRIMARY KEY AUTO_INCREMENT,
    StudentName  VARCHAR(255) NOT NULL,
    gender       ENUM ('男','女','保密'),
    Age          INT,
    ClassName    VARCHAR(255),
    HeadTeacher  VARCHAR(255),
    StudentCount INT,
    AcademicYear VARCHAR(255)
);
```

插入模拟记录

```sql
INSERT INTO Students (StudentName, gender, Age, ClassName, HeadTeacher, StudentCount, AcademicYear)
VALUES ('张三', '男', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('李四', '女', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('王五', '男', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('赵六', '女', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('刘七', '男', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('陈八', '女', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('杨九', '男', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('周十', '女', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('吴十一', '男', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('郑十二', '女', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('黄十三', '男', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('许十四', '女', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('曾十五', '男', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('吕十六', '女', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('冯十七', '男', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('朱十八', '女', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('江十九', '男', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('何二十', '女', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('魏二十一', '男', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('孔二十二', '女', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('曹二十三', '男', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('秦二十四', '女', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('许二十五', '男', 17, '软件1班', '李老师', 30, '2022-2023'),
       ('韩二十六', '女', 16, '软件1班', '李老师', 30, '2022-2023'),
       ('田二十七', '男', 18, '软件1班', '李老师', 30, '2022-2023'),
       ('范二十八', '女', 17, '软件1班', '李老师', 30, '2022-2023');
```

在上述示例中，将班级相关的字段直接添加到学生表中会产生一些问题，比如可能会导致**冗余数据**。如果多个学生属于同一个班级，那么每个学生记录都会重复存储相同的班级信息。这样的冗余数据可能浪费存储空间并增加**数据的不一致性和更新的复杂性**。

为了解决这些问题，更好的做法是使用关联表来处理学生和班级之间的一对多关系。

```sql
-- 建立多对一的关系：在多的表中创建关联字段
CREATE TABLE Students
(
    StudentID    INT PRIMARY KEY AUTO_INCREMENT,
    StudentName  VARCHAR(255) NOT NULL,
    gender       ENUM ('男','女','保密'),
    Age          INT,
    ClassID 	 INT
);

CREATE TABLE Classes (
    ID INT PRIMARY KEY,
    ClassName VARCHAR(50),
    HeadTeacher VARCHAR(50),
    StudentCount INT,
    AcademicYear VARCHAR(10)
);
```

插入模拟记录：

```sql
-- 插入班级记录
INSERT INTO Classes (ID, ClassName, HeadTeacher, StudentCount, AcademicYear)
VALUES (1, '软件1班', '李老师', 30, '2022-2023'),
       (2, '软件2班', '王老师', 35, '2022-2023'),
       (3, '计算机1班', '张老师', 28, '2022-2023'),
       (4, '计算机2班', '刘老师', 32, '2022-2023'),
       (5, '电子1班', '陈老师', 31, '2022-2023');


-- 插入学生记录
INSERT INTO Students (StudentID, StudentName, gender, Age, ClassID)
VALUES (1, '张三', '男', 18, 1),
       (2, '李思琪', '女', 17, 1),
       (3, '王伟', '男', 16, 1),
       (4, '赵雅芝', '女', 18, 1),
       (5, '刘德华', '男', 17, 1),
       (6, '陈小姐', '女', 16, 1),
       (7, '杨明', '男', 18, 1),
       (8, '周慧', '女', 17, 1),
       (9, '钱振华', '男', 16, 1),
       (10, '孙艺珍', '女', 18, 1),
       (11, '周润发', '男', 17, 1),
       (12, '吴莫愁', '女', 16, 1),
       (13, '郑源', '男', 18, 1),
       (14, '王菲', '女', 17, 1),
       (15, '李小龙', '男', 16, 2),
       (16, '张曼玉', '女', 18, 2),
       (17, '赵本山', '男', 17, 2),
       (18, '钱钟书', '女', 16, 3),
       (19, '孙红雷', '男', 18, 3),
       (20, '周杰伦', '女', 17, 3),
       (21, '吴京', '男', 16, 4),
       (22, '郑少秋', '女', 18, 4),
       (23, '王祖贤', '男', 17, 4),
       (24, '李连杰', '女', 16, 5),
       (25, '张学友', '男', 18, 5),
       (26, '赵薇', '女', 17, 5),
       (27, '钱飞', '男', 16, 5),
       (28, '孙俪', '女', 18, 5),
       (29, '李宇春', '男', 17, 2),
       (30, '王力宏', '女', 16, 3);
```

##### （2）一对多的典型案例解析

* 部门和员工

  ```sql
  -- 创建部门表
  CREATE TABLE Departments (
  
  );
  
  -- 创建员工表
  CREATE TABLE Employees (
    
  );
  ```

* 作者和博客

  ```sql
  -- 创建作者表
  CREATE TABLE Authors (
  
  );
  
  -- 创建博客表
  CREATE TABLE Blogs (
  
  );
  ```

* 博客和评论

  ```sql
  -- 创建博客表
  CREATE TABLE Blogs (
  
  );
  
  -- 创建评论表
  CREATE TABLE Comments (
  
  );
  ```

* 书籍和出版社

  ```sql
  -- 创建出版社表
  CREATE TABLE Publishers (
   
  );
  
  -- 创建书籍表
  CREATE TABLE Books (
  
  );
  ```

* 用户和订单

  ```sql
  -- 创建用户表
  CREATE TABLE Users (
   
  );
  
  -- 创建订单表
  CREATE TABLE Orders (
    
  );
  ```

* 供应商和产品

  ```sql
  -- 创建供应商表
  CREATE TABLE Suppliers (
   
  );
  
  -- 创建产品表
  CREATE TABLE Products (
    
    
    supplier_id   int
  );
  ```

##### （3）一对多的子查询

```sql
-- 案例1: 查询李小龙的班级名称和班级人数以及班主任名字
-- 查询李小龙的班级ID
SELECT ClassID
FROM Students
WHERE StudentName = '李小龙';
-- 使用李小龙的班级ID查询班级名称、班级人数和班主任名字
SELECT ClassName, StudentCount, HeadTeacher
FROM Classes
WHERE ID = (SELECT ClassID FROM Students WHERE StudentName = '李小龙');


-- 案例2：查询软件1班所有学生的姓名
SELECT StudentName
FROM Students
WHERE ClassID = (SELECT ID FROM Classes WHERE ClassName = '软件1班');
```

> 子查询是 MySQL 中比较常用的查询方法，通过子查询可以实现多表关联查询。子查询指将一个查询语句嵌套在另一个查询语句中。

#### 1.2、多对多

多对多，在数据库中也比较常见，可以理解为是一对多和多对一的组合。

* 学生和课程的多对多关系：
  - 学生表（Students）：存储学生的信息。
  - 课程表（Courses）：存储课程的信息。
  - 选课表（Enrollment）：作为中间表，存储学生和课程之间的关联关系。
* 作者和书籍的多对多关系：
  - 作者表（Authors）：存储作者的信息。
  - 书籍表（Books）：存储书籍的信息。
  - 作者-书籍关系表（Author_Book）：作为中间表，存储作者和书籍之间的关联关系。
* 用户和角色的多对多关系：
  - 用户表（Users）：存储用户的信息。
  - 角色表（Roles）：存储角色的信息。
  - 用户-角色关系表（User_Role）：作为中间表，存储用户和角色之间的关联关系。
* 商品和订单的多对多关系：
  - 商品表（Products）：存储商品的信息。
  - 订单表（Orders）：存储订单的信息。
  - 订单-商品关系表（Order_Product）：作为中间表，存储订单和商品之间的关联关系。
* 标签和文章的多对多关系：
  - 标签表（Tags）：存储标签的信息。
  - 文章表（Articles）：存储文章的信息。
  - 文章-标签关系表（Article_Tag）：作为中间表，存储文章和标签之间的关联关系。

当在学生表和课程表中直接使用`course_id`和`student_id`来表示多对多关系时，会导致数据冗余。这意味着课程信息将在课程表中重复存储，而每个学生选修的课程也将在学生表中重复存储。

<img src="MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-22%2019.56.26.png" alt="截屏2024-02-22 19.56.26" style="zoom:50%;" />

要实现多对多，一般都需要有一张中间表（也叫关联表），将两张表进行关联，形成多对多的形式。

<img src="MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-22%2021.04.53-8608385.png" alt="截屏2024-02-22 21.04.53" style="zoom:50%;" />

如果将学生和课程的关联关系直接存储在学生表和课程表中，那么数据将如下所示：

```sql 
-- 建立多对一的关系：在多的表中创建关联字段
CREATE TABLE Students
(
    StudentID    INT PRIMARY KEY AUTO_INCREMENT,
    StudentName  VARCHAR(255) NOT NULL,
    gender       ENUM ('男','女','保密'),
    Age          INT,
    ClassID INT
);

CREATE TABLE Courses (
    CourseID INT PRIMARY KEY,
    CourseName VARCHAR(255),
    Instructor VARCHAR(255),
    Description VARCHAR(255),
    Credits INT,
    Classroom VARCHAR(255),
    Periods VARCHAR(255)
);

-- 两张表各建一个关联字段数据冗余，管理低效
-- 更高效建立多对多关系的答案是第三张关系表
CREATE TABLE StudentCourses (
    ID INT PRIMARY KEY AUTO_INCREMENT,
    StudentID INT,
    CourseID INT
);
```

添加课程记录：

```sql
-- 插入课程记录
INSERT INTO Courses (CourseID, CourseName, Instructor, Description, Credits, Classroom, Periods)
VALUES (1, '高等数学', '王老师', '本课程介绍高等数学的基本概念与方法', 4, 'A101', '周一、周三 9:00-10:30'),
       (2, '线性代数', '李老师', '本课程介绍线性代数的基本理论与应用', 3, 'B202', '周二、周四 13:30-15:00'),
       (3, '计算机程序设计', '张老师', '本课程教授常用的计算机程序设计语言和方法', 3, 'C301', '周一、周三 13:30-15:00'),
       (4, '数据库系统', '刘老师', '本课程介绍数据库系统的原理与应用', 3, 'D402', '周二、周四 9:00-10:30'),
       (5, '操作系统', '陈老师', '本课程介绍操作系统的基本原理与设计', 3, 'E503', '周一、周三 10:30-12:00'),
       (6, '离散数学', '赵老师', '本课程介绍离散数学的基本概念与证明方法', 3, 'F604', '周二、周四 10:30-12:00'),
       (7, '数据结构与算法', '钱老师', '本课程介绍常用数据结构和算法的设计与分析', 3, 'G705', '周一、周三 15:00-16:30'),
       (8, '人工智能', '孙老师', '本课程介绍人工智能的基本原理与应用', 4, 'H806', '周二、周四 15:00-16:30'),
       (9, '网络技术', '周老师', '本课程介绍计算机网络的基本概念与技术', 3, 'I907', '周一、周三 16:30-18:00'),
       (10, '软件工程', '吴老师', '本课程介绍软件工程的基本原理与实践', 3, 'J1008', '周二、周四 16:30-18:00');
```

建立多对多关系：

```sql
-- 学生John Doe选修了数学和英语课程
INSERT INTO StudentCourses (StudentID, CourseID)
VALUES (1, 1); -- 学生ID为1，课程ID为1

INSERT INTO StudentCourses (StudentID, CourseID)
VALUES (1, 2); -- 学生ID为1，课程ID为2

-- 学生Jane Smith选修了英语和历史课程
INSERT INTO StudentCourses (StudentID, CourseID)
VALUES (2, 2); -- 学生ID为2，课程ID为2

INSERT INTO StudentCourses (StudentID, CourseID)
VALUES (2, 4); -- 学生ID为2，课程ID为4

-- 学生Tom Johnson选修了数学和物理课程
INSERT INTO StudentCourses (StudentID, CourseID)
VALUES (3, 1); -- 学生ID为3，课程ID为1

INSERT INTO StudentCourses (StudentID, CourseID)
VALUES (3, 5); -- 学生ID为3，课程ID为5
```

多对多的子查询：

```sql
-- 查询张三选修课程的名字和学分
SELECT StudentID
FROM Students
WHERE StudentName = '张三';

SELECT CourseID
FROM StudentCourses
WHERE StudentID = (
    1
);

SELECT CourseName, Credits
FROM Courses
WHERE CourseID IN (
	1,2
);

-- 嵌套版
SELECT CourseName, Credits
FROM Courses
WHERE CourseID IN (
	SELECT CourseID
	FROM StudentCourses
	WHERE StudentID = (
		SELECT StudentID
		FROM Students
		WHERE StudentName = '张三'
	)
);

-- 查询选修高等数学的学生的姓名
select CourseID from Courses where CourseName='高等数学';
select StudentID from StudentCourses where CourseID=（1）;
select StudentName from Students where StudentID in (1,3);
```

#### 1.3、一对一

一对一关系（One-to-One Relationship）指的是两个实体之间存在一种对应关系，其中一个实体的每个记录只能对应另一个实体的一条记录，而另一个实体的每个记录也只能对应一个实体的记录。在一对一关系中，关联字段在每个表中都加上了唯一约束，以确保关联的唯一性。

一对一是将数据表“垂直切分”。也就是 A 表的一条记录对应 B 表的一条记录。

1. 学生和学生详情的一对一关系：

   - 学生（Students）：存储用户的基本信息
   - 学生详情表（StudentDetails）：存储学生地址、联系方式等。每个学生只有一条详情记录。

   ![image-20240222234840634](MySQL%E5%9F%BA%E7%A1%80.assets/image-20240222234840634-8616922-8617247.png)

2. 员工和薪资信息的一对一关系：

   - 员工表（Employees）：存储员工的基本信息，如员工号、姓名、职位等。
   - 薪资信息表（Salary）：存储员工的薪资信息，如基本工资、津贴等。每个员工在薪资信息表中只有一条记录，与员工表中的对应记录关联。

   <img src="MySQL%E5%9F%BA%E7%A1%80.assets/image-20240222235205268-8617126.png" alt="image-20240222235205268" style="zoom: 50%;" />

一对一创建表的优点：

* 将两个实体分开存储在不同的表中，可以更好地保持数据的规范性和完整性。每个表都可以有自己的约束和验证规则，确保数据的一致性和有效性。

* 使用一对一关系表，可以更容易地扩展和修改数据模型。
* 数据隐私和安全性

**通过唯一约束的关联字段建立一对一**。

```sql
/*
CREATE TABLE Students
(
    StudentID    INT PRIMARY KEY AUTO_INCREMENT,
    StudentName  VARCHAR(255) NOT NULL,
    gender       ENUM ('男','女','保密'),
    Age          INT,
    ClassID INT
);*/

-- 创建学生联系信息表
CREATE TABLE StudentDetails
(
    ID          INT PRIMARY KEY,
    PhoneNumber VARCHAR(255),
    Email       VARCHAR(255),
    Address     VARCHAR(255),
    StudentID   INT UNIQUE -- 区别于一对多，关联字段加唯一约束！ 
);
```

插入记录：

```sql
INSERT INTO StudentDetails (ID, PhoneNumber, Email, Address, StudentID)
VALUES (1, '1234567890', '张三@example.com', '北京市朝阳区建国门外大街123号', 1),
       (2, '2345678901', '李思琪@example.com', '上海市黄浦区南京东路456号', 2),
       (3, '3456789012', '王伟@example.com', '广东省深圳市福田区华强北路789号', 3),
       (4, '4567890123', '赵雅芝@example.com', '四川省成都市武侯区锦江宾馆321号', 4),
       (5, '5678901234', '刘德华@example.com', '湖北省武汉市江汉区解放大道654号', 5);
```

一对一子查询：

```sql
-- 查询张三的手机号和邮箱
select Email, PhoneNumber
from StudentDetails
where StudentID = (select StudentID from Students where StudentName = '张三')


-- 查询手机号为5678901234的学生姓名和年龄
SELECT StudentName, Age
FROM Students
WHERE StudentID = (SELECT StudentID
                   FROM StudentDetails
                   WHERE PhoneNumber = '5678901234')
```

### 二、JOIN关联查询

#### 【1】笛卡尔积查询

笛卡尔积（Cartesian product）是指两个集合之间的所有可能的组合。在关系型数据库中，笛卡尔积是指两个表之间的所有可能的行组合。

如果有两个表A和B，每个表包含n和m行，那么它们的笛卡尔积将产生n * m行。每一行是表A中的一行与表B中的一行的组合。

````sql 
SELECT * FROM Students,Classes;
````

**注意：没有设定条件的内连接与笛卡尔积没有区别；另外，在两个表都不为空表的前提下，没有设定条件的外连接与笛卡尔积外观上没有区别（逻辑上不一样）**

- 笛卡尔积（内连接无条件）：结果是两表行的 “纯粹数学组合”，不依赖任何表的 “主导地位”，仅体现 “所有可能的搭配”。
- 外连接（左 / 右）：结果是 “以主表为基础，强制保留主表所有行后，再与另一表的所有行组合”，隐含 “主表行必须全部出现” 的逻辑（即使另一表为空，主表行也会保留，如前例）。

举一个具体的例子来理解笛卡尔积：

假设表 A（学生表）有 2 行数据：

| 学生 ID | 姓名 |
| ------- | ---- |
| 1       | 张三 |
| 2       | 李四 |

表 B（课程表）有 3 行数据：

| 课程 ID | 课程名 |
| ------- | ------ |
| 101     | 数学   |
| 102     | 英语   |
| 103     | 物理   |

此时，表 A 和表 B 的笛卡尔积会产生 `2 * 3 = 6` 行数据，每一行是表 A 的一行与表 B 的一行的所有可能组合：

| 学生 ID | 姓名 | 课程 ID | 课程名 |
| ------- | ---- | ------- | ------ |
| 1       | 张三 | 101     | 数学   |
| 1       | 张三 | 102     | 英语   |
| 1       | 张三 | 103     | 物理   |
| 2       | 李四 | 101     | 数学   |
| 2       | 李四 | 102     | 英语   |
| 2       | 李四 | 103     | 物理   |

可以看到，笛卡尔积会无差别地将两个表中的所有行进行组合，不考虑任何逻辑关联（比如 “学生是否选修了这门课”）。在实际业务中，通常会通过 `JOIN ... ON` 条件过滤掉无关的组合，只保留有逻辑关联的记录。

#### 【2】内连接

- **作用**：只保留两个表中**匹配条件的记录**，不匹配的记录会被过滤掉。
- 适用场景：需要查询两个表中存在关联关系的数据。例如：
  - 查询 “有订单的客户”（客户表与订单表通过客户 ID 关联，只保留有订单的客户）。
  - 查询 “选修了课程的学生”（学生表与选课表关联，只保留有选课记录的学生）。

查询两张表中都有的关联数据,相当于利用条件从笛卡尔积结果中筛选出了正确的结果。

案例1（多对一拼接）：

```sql 
-- 查询两张表中都有的关联数据，相当于利用条件从笛卡尔积结果中筛选出了正确的结果。
SELECT * From Students,Classes
where Students.ClassID = Classes.ID;

-- 或者
SELECT * FROM Students
INNER JOIN Classes ON Students.ClassID = Classes.ID;

SELECT * FROM Classes c
INNER JOIN Students s ON s.ClassID = c.ID;
```

![截屏2024-02-21 23.28.52](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-21%2023.28.52-8529346.png)

案例2（多对多拼接）：

```sql 
SELECT * FROM Students
INNER JOIN StudentCourses ON Students.StudentID = StudentCourses.StudentID;
```

![image-20250426173405008](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426173405008.png)

```sql  
SELECT * FROM Students
INNER JOIN StudentCourses ON Students.StudentID = StudentCourses.StudentID
INNER JOIN Courses ON StudentCourses.CourseID = Courses.CourseID;
```

![image-20250426173618897](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426173618897.png)

**踩坑记录**

如果我们把StudentCourses表的顺序改一下

![image-20250426192634248](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426192634248.png)

这时再用内连接查询

```sql
-- 和上面的sql一样，可以发现顺序是以StudentCourses这个表的StudentID作为标准的
SELECT * FROM Students
INNER JOIN StudentCourses ON Students.StudentID = StudentCourses.StudentID
INNER JOIN Courses ON StudentCourses.CourseID = Courses.CourseID;
```

![image-20250426192753444](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426192753444.png)

#### 【3】左连接和右连接

外连接又分为**左外连接（LEFT JOIN）**、**右外连接（RIGHT JOIN）** 和**全外连接（FULL JOIN）**，核心是**保留某一侧表的所有记录**，另一侧不匹配的记录用`NULL`填充。

- **左外连接（LEFT JOIN）**：保留左表的所有记录，右表中无匹配的记录用`NULL`填充。

  适用场景：需要**左表的全部数据**，即使右表中没有对应记录。例如：“查询所有员工及其部门信息，没有部门的员工也要显示”（员工表为左表，部门表为右表）。

- **右外连接（RIGHT JOIN）**：保留右表的所有记录，左表中无匹配的记录用`NULL`填充。

  适用场景：与左连接相反，需要**右表的全部数据**。例如：“查询所有部门及其员工，没有员工的部门也要显示”（部门表为右表，员工表为左表）。

- **全外连接（FULL JOIN）**：保留两个表的所有记录，双方不匹配的部分用`NULL`填充（部分数据库如 MySQL 不直接支持，需用`UNION`模拟）。

  适用场景：需要**两个表的全部数据**，无论是否匹配。例如：“查询所有客户和所有订单，包括没有订单的客户和没有对应客户的订单”。

左外连接又称为左连接，使用 **LEFT OUTER JOIN** 关键字连接两个表，并使用 ON 子句来设置连接条件。

````sql 
SELECT *
FROM 表1
LEFT JOIN 表2
ON 表1.连接字段 = 表2.连接字段;
````

上述语法中，“表1”为基表，“表2”为参考表。左连接查询时，可以查询出“表1”中的所有记录和“表2”中匹配连接条件的记录。如果“表1”的某行在“表2”中没有匹配行，那么在返回结果中，“表2”的字段值均为空值（NULL）。

```sql
-- 将几个学生的ClassID设置为空
SELECT *
FROM Students
LEFT JOIN Classes ON Students.ClassID = Classes.ID;
```

![截屏2024-02-21 23.54.00](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-21%2023.54.00-8530856.png)

```sql 
-- 将某个班级处理成没有学生
SELECT * FROM Classes
Left JOIN Students ON Students.ClassID = Classes.ID;
```

![image-20250426183855119](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426183855119.png)

```sql
-- 这两种的结果其实是一样的，只不过顺序不一样
SELECT * FROM Students
RIGHT JOIN Classes ON Students.ClassID = Classes.ID;
```

![image-20250426183934893](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426183934893.png)

```sql
-- 以Students表作为主表，顺序也是以StudentCourses这个表的StudentID作为标准的，后面选课的人则是以Students表的StudentID的顺序进行排序的
select * from Students
left join StudentCourses on StudentCourses.StudentID = Students.StudentID
left join Courses on  Courses.CourseID = StudentCourses.CourseID;
```

![image-20250426193924413](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426193924413.png)

```sql
-- 以Courses表作为主表，顺序也是以StudentCourses这个表的CoursesID作为标准的，后面没有人选的课程则是以Courses表的CoursesID的顺序进行排序的
select * from Courses
left join StudentCourses on StudentCourses.CourseID = Courses.CourseID
left join Students on  Students.StudentID = StudentCourses.StudentID;
```

![image-20250426193952548](MySQL%E5%9F%BA%E7%A1%80.assets/image-20250426193952548.png)

右外连接又称为右连接，右连接是左连接的反向连接。使用 **RIGHT OUTER JOIN** 关键字连接两个表，并使用 ON 子句来设置连接条件。

与左连接相反，右连接以“表2”为主表，“表1”为参考表。右连接查询时，可以查询出“表2”中的所有记录和“表1”中匹配连接条件的记录。如果“表2”的某行在“表1”中没有匹配行，那么在返回结果中，“表1”的字段值均为空值（NULL）。

![截屏2024-02-21 23.59.59](MySQL%E5%9F%BA%E7%A1%80.assets/%E6%88%AA%E5%B1%8F2024-02-21%2023.59.59-8531213.png)

> **全连接（FULL JOIN）：** 全连接返回左表和右表中的所有行，无论是否有匹配。如果某个表的行没有匹配，结果集中对应的列将包含NULL值。

### 三、外键约束

外键约束是关系数据库中的一种机制，用于维护表之间的关联关系和数据完整性。外键约束经常和主键约束一起使用，用来确保数据的一致性。

外键约束（FOREIGN KEY）是表的一个特殊字段，经常与主键约束一起使用。对于两个具有关联关系的表而言，相关联字段中主键所在的表就是主表（父表），外键所在的表就是从表（子表）。

外键用来建立主表与从表的关联关系，为两个表的数据建立连接，约束两个表中数据的一致性和完整性。主表删除某条记录时，从表中与之对应的记录也必须有相应的改变。一个表可以有一个或多个外键，外键可以为空值，若不为空值，则每一个外键的值必须等于主表中主键的某个值。

比如上面的书籍管理案例，若删除一个`ID=1`的班级记录，没有任何影响，但是，学生表中`ClassID = 1` 的记录班级ID字段就没有意义了。

#### 【1】创建表时设置外键约束

````sql 
[CONSTRAINT <外键名>]
FOREIGN KEY 字段名 [，字段名2，…]
REFERENCES <主表名> 主键列1 [，主键列2，…]
````

> 定义外键时，需要遵守下列规则：
>
> 1. 先有主表，再有子表。
>
> 2. 必须为主表定义主键。
>
> 3. 主键不能包含空值，但允许在外键中出现空值。
> 4. `外键中列的数据类型必须和主表主键中对应列的数据类型相同`。

例如：

```sql 
CREATE TABLE Classes2
(
    ID           INT PRIMARY KEY,
    ClassName    VARCHAR(50),
    HeadTeacher  VARCHAR(50),
    StudentCount INT,
    AcademicYear VARCHAR(10)
);

CREATE TABLE Students2
(
    StudentID   INT PRIMARY KEY AUTO_INCREMENT,
    StudentName VARCHAR(255) NOT NULL,
    gender      ENUM ('男','女','保密'),
    Age         INT,
    ClassID     INT,
    Constraint FK_Students_Classes FOREIGN KEY (ClassID) REFERENCES Classes2 (ID)
);

CREATE TABLE Courses2
(
    ID    INT PRIMARY KEY,
    CourseName  VARCHAR(255),
    Instructor  VARCHAR(255),
    Description VARCHAR(255),
    Credits     INT,
    Classroom   VARCHAR(255),
    Periods     VARCHAR(255)
);

CREATE TABLE StudentCourses2
(
    StudentID INT,
    CourseID  INT,
    CONSTRAINT FK_StudentCourses_Students
        FOREIGN KEY (StudentID)
            REFERENCES Students2 (StudentID),
  
    CONSTRAINT FK_StudentCourses_Courses
        FOREIGN KEY (CourseID)
            REFERENCES Courses2 (ID)
);
```

#### 【2】添加删除外键约束

````sql 
-- （1）添加外键约束
ALTER TABLE <数据表名> ADD CONSTRAINT <外键名>
FOREIGN KEY(<列名>) REFERENCES <主表名> (<列名>);

-- 给学生表ClassID字段添加外键约束
ALTER TABLE Students ADD CONSTRAINT class_fk
FOREIGN KEY(ClassID) REFERENCES Classes(ID);
SHOW CREATE TABLE Students;

-- 尝试删除一个出版社记录
DELETE
FROM Classes
where ID = 1;  -- 报错

-- （2）删除外键约束
ALTER TABLE <表名> DROP FOREIGN KEY <外键约束名>;
drop index 外键约束名 on <表名>; -- 同时将索引删除 

-- 将学生表ClassID字段的外键约束和索引删除
ALTER TABLE Students DROP FOREIGN KEY class_fk;
Drop Index class_fk on Students; -- 同时将索引删除
````

#### 【3】INNODB支持的ON语句

在 MySQL 中，InnoDB 是一种常用的存储引擎，它提供了一些 ON 语句来定义外键约束和触发器。以下是 InnoDB 支持的 ON 语句：

1. `ON DELETE RESTRICT`：如果关联的主表中的数据被删除，且从表中存在对应的数据，则禁止删除操作。
2. `ON DELETE CASCADE`：当关联的主表中的数据被删除时，自动删除从表中对应的数据。
3. `ON DELETE SET NULL`：当关联的主表中的数据被删除时，从表中对应的数据将被设置为 NULL。
4. `ON DELETE SET DEFAULT`：当关联的主表中的数据被删除时，从表中对应的数据将被设置为默认值。
5. `ON UPDATE RESTRICT`：如果关联的主表中的数据更新，且从表中存在对应的数据，则禁止更新操作。
6. `ON UPDATE CASCADE`：当关联的主表中的数据更新时，自动更新从表中的对应数据。
7. `ON UPDATE SET NULL`：当关联的主表中的数据更新时，从表中对应的数据将被设置为 NULL。
8. `ON UPDATE SET DEFAULT`：当关联的主表中的数据更新时，从表中对应的数据将被设置为默认值。

这些 ON 语句可以在创建或修改外键约束时使用，以定义在不同操作（更新或删除）发生时，从表中的数据应该如何处理。

> 请注意，这些 ON 语句只适用于 InnoDB 存储引擎。

```sql
# 把学生表和课程表的外键约束规则改为关联删除
ALTER TABLE Students ADD CONSTRAINT class_fk
FOREIGN KEY(ClassID) REFERENCES Classes(ID)
ON DELETE CASCADE;
```

### 四、MySQL内置函数

MySQL内置函数是MySQL数据库提供的一组函数，用于在SQL查询中执行各种操作和计算。以下是一些常用的MySQL内置函数：

1. 字符串函数：
   - `CONCAT(str1, str2, ...)`：将多个字符串连接成一个字符串。
   - `UPPER(str)`：将字符串转换为大写。
   - `LOWER(str)`：将字符串转换为小写。
   - `SUBSTRING(str, start, length)`：返回字符串的子串。
   - `LENGTH(str)`：返回字符串的长度。
   - `TRIM(str)`：去除字符串两端的空格。
   - `REPLACE(str, old, new)`：将字符串中的某个子串替换为新的子串。
2. 数值函数：
   - `ABS(x)`：返回一个数的绝对值。
   - `ROUND(x, d)`：将一个数四舍五入到指定的小数位数。
   - `FLOOR(x)`：向下取整，返回不大于给定数的最大整数。
   - `CEILING(x)`：向上取整，返回不小于给定数的最小整数。
   - `RAND()`：返回一个0到1之间的随机数。
3. 日期和时间函数：
   - `NOW()`：返回当前日期和时间。
   - `CURDATE()`：返回当前日期。
   - `CURTIME()`：返回当前时间。
   - `DATE_FORMAT(date, format)`：将日期格式化为指定的格式。
   - `DATEDIFF(date1, date2)`：计算两个日期之间的天数差。
   - `DATE_ADD(date, INTERVAL expr unit)`：在日期上添加指定的时间间隔。
4. 聚合函数：
   - `COUNT(expr)`：计算满足条件的行数。
   - `SUM(expr)`：计算指定列的总和。
   - `AVG(expr)`：计算指定列的平均值。
   - `MIN(expr)`：计算指定列的最小值。
   - `MAX(expr)`：计算指定列的最大值。

这只是MySQL内置函数的一部分，还有许多其他函数可用于字符串处理、数值计算、日期时间操作、聚合计算等。

### 五、Python操作MySQL

![img](MySQL%E5%9F%BA%E7%A1%80.assets/78de60e5339447d92d3724a4625ca212.png)

#### 【1】pymysql模块

安装：

```python
pip install pymysql
```

基本语法

```python
import pymysql

# 连接到MySQL数据库
conn = pymysql.connect(host="10.0.0.53",
                       port=3306,
                       user="tonydu01",
                       password="123123",
                       database="mysql-day2"
                       )

# 创建游标对象
cursor = conn.cursor()

# 执行SQL查询
query = "select * from Classes"
cursor.execute(query)

# 获取查询结果
results = cursor.fetchall()
# mysql结果默认会处理成元组   
print(type(results))
for row in results:
    print(row)

# 关闭游标和数据库连接
cursor.close()
conn.close()

# 输出
<class 'tuple'>
((1, '软件1班', '李老师', 30, '2022-2023'), (2, '软件2班', '王老师', 35, '2022-2023'), (3, '计算机1班', '张老师', 28, '2022-2023'), (4, '计算机2班', '刘老师', 32, '2022-2023'), (5, '电子1班', '陈老师', 31, '2022-2023'))


# 改成字典模式
import pymysql
from pymysql import cursors

conn = pymysql.connect(host="10.0.0.53",
                       port=3306,
                       user="tonydu01",
                       password="123123",
                       database="mysql-day2",
                       cursorclass=cursors.DictCursor
                       )

cursor = conn.cursor()

query = "select * from Classes"
cursor.execute(query)

results = cursor.fetchall()
print(type(results))
print(results)

cursor.close()
conn.close()

# 输出
<class 'list'>
[{'ID': 1, 'ClassName': '软件1班', 'HeadTeacher': '李老师', 'StudentCount': 30, 'AcademicYear': '2022-2023'}, {'ID': 2, 'ClassName': '软件2班', 'HeadTeacher': '王老师', 'StudentCount': 35, 'AcademicYear': '2022-2023'}, {'ID': 3, 'ClassName': '计算机1班', 'HeadTeacher': '张老师', 'StudentCount': 28, 'AcademicYear': '2022-2023'}, {'ID': 4, 'ClassName': '计算机2班', 'HeadTeacher': '刘老师', 'StudentCount': 32, 'AcademicYear': '2022-2023'}, {'ID': 5, 'ClassName': '电子1班', 'HeadTeacher': '陈老师', 'StudentCount': 31, 'AcademicYear': '2022-2023'}]
```

在上述示例中，首先使用`pymysql.connect()`函数连接到MySQL数据库。您需要提供数据库的连接参数，如主机名、用户名、密码和数据库名称。

然后，使用连接对象的`cursor()`方法创建一个游标对象。游标用于执行SQL查询并获取结果。

执行SQL查询时，可以使用游标对象的`execute()`方法传递SQL查询语句。在示例中，我们执行了一个简单的SELECT语句来获取表中的所有数据。

接下来，使用游标对象的`fetchall()`方法获取查询结果。`fetchall()`方法返回一个包含所有行的列表，每行又是一个元组。

最后，使用`cursor.close()`关闭游标对象，使用`conn.close()`关闭数据库连接。

请注意，在实际使用中，可能需要根据具体情况进行错误处理、事务处理和参数化查询等操作，以确保代码的正确性和安全性。

案例：创建user表，插入用户记录：

```python
import pymysql
from pymysql import cursors

def create_users_table(cursor):
    sql = """
        CREATE TABLE users(
        id INT PRIMARY KEY AUTO_INCREMENT,
        username VARCHAR(50),
        password VARCHAR(50)
        )
    """
    cursor.execute(sql)

def insert_users(cursor, conn):
    user = input("请输入用户名：")
    pwd = input("请输入密码：")
    sql = f"INSERT INTO users(username,password) VALUES('{user}','{pwd}')"
    cursor.execute(sql)
    conn.commit()	# 执行增删改操作时务必要提交

def main():
    conn = pymysql.connect(host="10.0.0.53",
                           port=3306,
                           user="tonydu01",
                           password="123123",
                           database="mysql-day2",
                           cursorclass=cursors.DictCursor
                           )

    cursor = conn.cursor()

    create_users_table(cursor)
    insert_users(cursor, conn)

    cursor.close()
    conn.close()

if __name__ == '__main__':
    main()
```

#### 【2】SQL注入

SQL注入是一种常见的安全漏洞，它发生在应用程序未正确验证和处理用户输入数据时。攻击者可以通过在输入中插入恶意的SQL代码，利用这个漏洞来执行未经授权的数据库操作。

下面是一个简单的示例，展示了一个存在SQL注入漏洞的代码：

```python
import pymysql


def login(username, password):
    conn = pymysql.connect(
        host='localhost',
        user='root',
        password='yuan0316',
        database='db_day02'
    )

    cursor = conn.cursor()

    # 构造SQL查询语句
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
    print("query:",query)
    # 执行SQL查询
    cursor.execute(query)

    # 获取查询结果
    result = cursor.fetchall()

    cursor.close()
    conn.close()

    if len(result) > 0:
        print("登录成功")
    else:
        print("登录失败")


# 用户输入作为参数传递给登录函数
username = input("请输入用户名：")
password = input("请输入密码：")

login(username, password)
```

在上述示例中，我们定义了一个`login`函数，该函数接收用户名和密码作为参数。在函数内部，我们使用`pymysql`库连接到MySQL数据库，并构造了一个包含用户名和密码的SQL查询语句。

然而，代码中存在SQL注入漏洞。如果用户输入恶意的数据，例如在用户名输入框中输入

`' or 1=1;-- `，那么构造的查询语句将变为：

```python
SELECT * FROM users WHERE username = '' or 1=1;-- ' AND password = '111'
```

这将导致查询条件始终为真，绕过了实际的身份验证过程，从而允许攻击者以任意用户登录。

要修复SQL注入漏洞，可以使用参数化查询或输入验证来过滤和转义用户输入数据，确保它们不会被当作SQL代码执行。以下是修复SQL注入漏洞的示例代码：

```python
# 使用参数化查询构造SQL语句
query = "SELECT * FROM users WHERE username = %s AND password = %s"
params = (username, password)
# 执行SQL查询
cursor.execute(query, params)
```

在修复后的代码中，我们使用了参数化查询的方式来构造SQL语句。将用户输入作为参数传递给`execute()`方法，而不是直接将它们插入到SQL查询语句中。这样可以确保用户输入的数据被正确地转义和处理，防止SQL注入攻击的发生。

### 六、SQL 窗口函数使用文档

#### 一、什么是窗口函数？

窗口函数（Window Function）是 SQL 中用于**对一组行（称为 “窗口”）进行计算**的函数。与普通聚合函数（如 `SUM()`、`MAX()`）不同，窗口函数不会将多行数据合并为一行，而是**保留每行的原始数据**，同时在每行后附加计算结果。

**核心特点**：

- 不减少原表的行数（与 `GROUP BY` 聚合的区别）。
- 可以在一行中同时显示原始数据和基于 “窗口” 的统计结果。
- 支持对 “窗口” 进行灵活划分（分组、排序、限定范围）。

#### 二、基本语法

```sql
窗口函数名(参数) OVER (
    [PARTITION BY 列名1, 列名2, ...]  -- 可选：将数据按列分组，每组单独计算
    [ORDER BY 列名1 [ASC/DESC], 列名2 [ASC/DESC], ...]  -- 可选：对分组内的数据排序
    [ROWS/RANGE BETWEEN 边界1 AND 边界2]  -- 可选：限定窗口的范围（滑动窗口）
)
```

- **`OVER()`**：关键字，用于定义 “窗口”（即计算范围），所有窗口函数必须跟在 `OVER()` 后。
- **`PARTITION BY`**：类似 `GROUP BY`，将数据划分为多个 “分区”（组），窗口函数在每个分区内独立计算。若不指定，则整个表视为一个分区。
- **`ORDER BY`**：对分区内的行排序，影响依赖顺序的函数（如排名函数、累计计算）。
- **`ROWS/RANGE BETWEEN`**：定义窗口的具体范围（如 “前 3 行到当前行”），默认范围为 “从分区起始到当前行”。

#### partion by 和 group by 有什么区别？

假设我们有一份 “学生成绩表”，数据如下：

| 学生 ID | 班级 | 科目 | 成绩 |
| ------- | ---- | ---- | ---- |
| 1       | 1 班 | 数学 | 90   |
| 2       | 1 班 | 数学 | 85   |
| 3       | 2 班 | 数学 | 95   |
| 4       | 2 班 | 数学 | 88   |

**1. GROUP BY：分组后 “折叠数据”（求每个班级的数学平均分）**

```sql
SELECT 
  班级, 
  AVG(成绩) AS 班级平均分  -- 聚合计算：每个组只输出1个结果
FROM 学生成绩表
GROUP BY 班级;  -- 按“班级”分组（1班为1组，2班为1组）
```

**结果（仅 2 行，每组 1 行）**：

| 班级 | 班级平均分 |
| ---- | ---------- |
| 1 班 | 87.5       |
| 2 班 | 91.5       |

> 逻辑：按 “班级” 相同值分组后，**折叠每组数据**，只保留 “班级” 和 “该组的平均分”，原表的 4 行数据最终只剩 2 行（组的数量）。

**2. PARTITION BY：分区后 “保留原行”（求每个学生在班级内的成绩排名）**

```sql
SELECT 
  学生ID, 
  班级, 
  成绩,
  RANK() OVER (PARTITION BY 班级 ORDER BY 成绩 DESC) AS 班级内排名  -- 窗口计算：每行都有结果
FROM 学生成绩表;  -- 按“班级”分区（1班为1区，2班为1区）
```

**结果（4 行，与原数据行数一致）**：

| 学生 ID | 班级 | 成绩 | 班级内排名 |
| ------- | ---- | ---- | ---------- |
| 1       | 1 班 | 90   | 1          |
| 2       | 1 班 | 85   | 2          |
| 3       | 2 班 | 95   | 1          |
| 4       | 2 班 | 88   | 2          |

> 逻辑：按 “班级” 相同值分区后，**不折叠数据**，保留原表所有 4 行数据，只是给每行增加了 “其所在班级内的排名”（排名基于分区内的成绩计算）。

**关键结论**

1. **分组 / 分区逻辑一致**：无论是 `GROUP BY 班级` 还是 `PARTITION BY 班级`，都是将 “班级字段值相同” 的行归为一组 / 一区（1 班的 2 行是一组，2 班的 2 行是另一组），**绝不会 “每条数据单独分组”**。
2. 核心差异在 “数据保留”：
   - `GROUP BY` 是 “**聚合后折叠**”：用聚合函数（SUM/AVG）将组内数据压缩为 1 行，丢失原行细节；
   - `PARTITION BY` 是 “**分区后保留**”：用窗口函数（RANK/ROW_NUMBER）给每行计算 “组内属性”，不丢失原行细节。

#### 三、常用窗口函数分类及示例

假设存在表 `sales`，数据如下：

| sale_id | user_id | sale_date  | amount |
| ------- | ------- | ---------- | ------ |
| 1       | 101     | 2023-01-01 | 100    |
| 2       | 101     | 2023-01-02 | 150    |
| 3       | 102     | 2023-01-01 | 200    |
| 4       | 101     | 2023-01-03 | 50     |
| 5       | 102     | 2023-01-02 | 300    |

##### 1. 排名函数

用于对分区内的行进行排名，常用函数：

- **`ROW_NUMBER()`**：为每行分配唯一序号（1,2,3...），即使值相同也不重复。
- **`RANK()`**：排名可能重复，重复后跳过后续序号（如 1,1,3...）。
- **`DENSE_RANK()`**：排名可能重复，重复后不跳过后续序号（如 1,1,2...）。

**示例**：按 `user_id` 分组，对每个用户的销售额按 `amount` 降序排名：

```sql
SELECT 
  sale_id,
  user_id,
  amount,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn,
  RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rk,
  DENSE_RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS dr
FROM sales;
```

**结果**：

| sale_id | user_id | amount | rn   | rk   | dr   |
| ------- | ------- | ------ | ---- | ---- | ---- |
| 2       | 101     | 150    | 1    | 1    | 1    |
| 1       | 101     | 100    | 2    | 2    | 2    |
| 4       | 101     | 50     | 3    | 3    | 3    |
| 5       | 102     | 300    | 1    | 1    | 1    |
| 3       | 102     | 200    | 2    | 2    | 2    |

##### 2. 聚合窗口函数

将普通聚合函数（`SUM`、`AVG`、`MAX`、`MIN` 等）作为窗口函数使用，用于计算分区内的累计 / 分组统计。

**示例 1**：按 `user_id` 分组，计算每个用户的累计销售额：

```sql
SELECT 
  sale_id,
  user_id,
  amount,
  SUM(amount) OVER (PARTITION BY user_id ORDER BY sale_date) AS user_total,  -- 累计到当前行的总和
  AVG(amount) OVER (PARTITION BY user_id) AS user_avg  -- 整个分区的平均值（无ORDER BY时）
FROM sales;
```

**结果**（用户 101 的累计销售额）：

| sale_id | user_id | amount | user_total | user_avg |                     |
| ------- | ------- | ------ | ---------- | -------- | ------------------- |
| 1       | 101     | 100    | 100        | 100      | （100+150+50)/3=100 |
| 2       | 101     | 150    | 250        | 100      | （100+150=250）     |
| 4       | 101     | 50     | 300        | 100      | （250+50=300）      |

`user_total`是否为固定值，取决于窗口函数中是否包含`ORDER BY`：

- 有`ORDER BY`：按排序后的顺序计算 “滚动累计”（每行值不同）。
- 无`ORDER BY`：计算 “整个分区的总和”（同分区内所有行值相同）

**示例 2**：滑动窗口（最近 2 天的销售额总和）：

```sql
SELECT 
  sale_date,
  amount,
  SUM(amount) OVER (
    ORDER BY sale_date 
    ROWS BETWEEN 1 PRECEDING AND CURRENT ROW  -- 范围：前1行到当前行（共2行）
  ) AS slide_sum
FROM sales
WHERE user_id = 101;
```

**结果**：

| sale_date  | amount | slide_sum |                |
| ---------- | ------ | --------- | -------------- |
| 2023-01-01 | 100    | 100       | （只有当前行） |
| 2023-01-02 | 150    | 250       | （100+150）    |
| 2023-01-03 | 50     | 200       | （150+50）     |

##### 3. 偏移函数

用于获取分区内某一行相对于当前行的偏移数据（如前一行、后一行）。

- **`LAG(列名, n)`**：获取当前行的前第 `n` 行数据（默认 `n=1`）。
- **`LEAD(列名, n)`**：获取当前行的后第 `n` 行数据（默认 `n=1`）。
- **`FIRST_VALUE(列名)`**：获取分区内第一行的指定列值。
- **`LAST_VALUE(列名)`**：获取分区内当前窗口最后一行的指定列值。

**示例**：获取每个用户的上一笔销售额和下一笔销售额：

```sql
SELECT 
  sale_id,
  user_id,
  sale_date,
  amount,
  LAG(amount) OVER (PARTITION BY user_id ORDER BY sale_date) AS prev_amount,  -- 上一笔
  LEAD(amount) OVER (PARTITION BY user_id ORDER BY sale_date) AS next_amount  -- 下一笔
FROM sales;
```

**结果**（用户 101）：

| sale_id | user_id | sale_date  | amount | prev_amount | next_amount |                      |
| ------- | ------- | ---------- | ------ | ----------- | ----------- | -------------------- |
| 1       | 101     | 2023-01-01 | 100    | NULL        | 150         | （第一行无上一笔）   |
| 2       | 101     | 2023-01-02 | 150    | 100         | 50          |                      |
| 4       | 101     | 2023-01-03 | 50     | 150         | NULL        | （最后一行无下一笔） |

#### 四、窗口函数的使用场景

1. **排名需求**：如 “查询每个班级的成绩前三名”“销售额 Top N 的用户”。
2. **累计统计**：如 “每月累计销售额”“用户的历史消费总额”。
3. **滑动分析**：如 “最近 7 天的平均活跃用户数”“连续 3 个月的业绩变化”。
4. **行与行对比**：如 “计算当天与前一天的销售额差值”“找出连续两天销售额下降的日期”。
5. **分组内的首 / 尾值**：如 “用户的第一笔订单金额”“每个部门的最高工资记录”。

#### 五、注意事项

1. **兼容性**：窗口函数支持主流数据库（MySQL 8.0+、PostgreSQL、SQL Server、Oracle），但 MySQL 5.x 及以下版本不支持。

2. 性能优化：

   - `PARTITION BY` 和 `ORDER BY` 的字段建议建立索引，减少排序和分组的开销。
   - 避免在超大表上使用无 `PARTITION BY` 的窗口函数（会对全表排序，性能较差）。

3. 与 `GROUP BY` 的区别：

   - `GROUP BY` 会将分组后的多行合并为一行（聚合后行数减少）。

- 窗口函数保留所有行，仅附加计算结果（行数不变）。

#### 六、常用函数速查表

| 函数类别     | 函数名            | 作用                         |
| ------------ | ----------------- | ---------------------------- |
| 排名函数     | `ROW_NUMBER()`    | 生成唯一序号（无重复）       |
|              | `RANK()`          | 排名可重复，重复后跳号       |
|              | `DENSE_RANK()`    | 排名可重复，重复后不跳号     |
| 聚合窗口函数 | `SUM(列)`         | 分区内的累计 / 总和          |
|              | `AVG(列)`         | 分区内的平均值               |
|              | `MAX(列)/MIN(列)` | 分区内的最大 / 最小值        |
| 偏移函数     | `LAG(列, n)`      | 获取前第 n 行数据            |
|              | `LEAD(列, n)`     | 获取后第 n 行数据            |
|              | `FIRST_VALUE(列)` | 分区内第一行的列值           |
|              | `LAST_VALUE(列)`  | 分区内当前窗口最后一行的列值 |

通过窗口函数，可高效实现复杂的数据分析需求，代码更简洁，逻辑更清晰，是 SQL 进阶的必备技能。实际使用时，建议结合 `EXPLAIN` 分析执行计划，优化性能。



### 七、索引

在数据库中索引最核心的作用是：**加速查找**。  例如：在含有300w条数据的表中查询，无索引需要700秒，而利用索引可能仅需1秒。

```
mysql> select * from big where password="81f98021-6927-433a-8f0d-0f5ac274f96e";
+----+---------+---------------+--------------------------------------+------+
| id | name    | email         | password                             | age  |
+----+---------+---------------+--------------------------------------+------+
| 11 | wu-13-1 | w-13-1@qq.com | 81f98021-6927-433a-8f0d-0f5ac274f96e |    9 |
+----+---------+---------------+--------------------------------------+------+
1 row in set (0.70 sec)

mysql> select * from big where id=11;
+----+---------+---------------+--------------------------------------+------+
| id | name    | email         | password                             | age  |
+----+---------+---------------+--------------------------------------+------+
| 11 | wu-13-1 | w-13-1@qq.com | 81f98021-6927-433a-8f0d-0f5ac274f96e |    9 |
+----+---------+---------------+--------------------------------------+------+
1 row in set (0.00 sec)

mysql> select * from big where name="wu-13-1";
+----+---------+---------------+--------------------------------------+------+
| id | name    | email         | password                             | age  |
+----+---------+---------------+--------------------------------------+------+
| 11 | wu-13-1 | w-13-1@qq.com | 81f98021-6927-433a-8f0d-0f5ac274f96e |    9 |
+----+---------+---------------+--------------------------------------+------+
1 row in set (0.00 sec)
```

在开发过程中会为哪些 经常会被搜索的列 创建索引，以提高程序的响应速度。例如：查询手机号、邮箱、用户名等。



#### 7.1 索引原理

为什么加上索引之后速度能有这么大的提升呢？ 因为索引的底层是基于B+Tree的数据结构存储的。

![image-20210526160040895](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526160040895.png)

![image-20210526155746811](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526155746811.png)

![image-20210526160519425](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526160519425.png)

很明显，如果有了索引结构的查询效率比表中逐行查询的速度要快很多且数据量越大越明显。

B+Tree结构连接：https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html

数据库的索引是基于上述B+Tree的数据结构实现，但在创建数据库表时，如果指定不同的引擎，底层使用的B+Tree结构的原理有些不同。

- myisam引擎，非聚簇索引（数据 和 索引结构 分开存储）

- innodb引擎，聚簇索引（数据 和 主键索引结构存储在一起）



##### 7.1.1 非聚簇索引（mysiam引擎）

```sql
create table 表名(
    id int not null auto_increment primary key, 
    name varchar(32) not null,
    age int
)engine=myisam default charset=utf8;
```


![image-20210526160040895](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526160040895.png)

![image-20210526155746811](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526155746811.png)

![image-20210526155118552](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526155118552.png)



##### 7.1.2 聚簇索引（innodb引擎）

```sql
create table 表名(
    id int not null auto_increment primary key, 
    name varchar(32) not null,
    age int
)engine=innodb default charset=utf8;
```


![image-20210526160040895](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526160040895.png)

![image-20210526155746811](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526155746811.png)

![image-20210526160519425](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526160519425.png)

![image-20210526155250801](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526155250801.png)

在MySQL文件存储中的体现：

```bash
root@192 userdb # pwd
/usr/local/mysql/data/userdb

root@192 userdb # ls -l
total 1412928
-rw-r-----  1 _mysql  _mysql       8684 May 15 22:51 big.frm，表结构。
-rw-r-----  1 _mysql  _mysql  717225984 May 15 22:51 big.ibd，数据和索引结构。
-rw-r-----  1 _mysql  _mysql       8588 May 16 11:38 goods.frm
-rw-r-----  1 _mysql  _mysql      98304 May 16 11:39 goods.ibd
-rw-r-----  1 _mysql  _mysql       8586 May 26 10:57 t2.frm，表结构
-rw-r-----  1 _mysql  _mysql          0 May 26 10:57 t2.MYD，数据
-rw-r-----  1 _mysql  _mysql       1024 May 26 10:57 t2.MYI，索引结构
```

上述 聚簇索引 和 非聚簇索引 底层均利用了B+Tree结构结构，只不过内部数据存储有些不同罢了。

在企业开发中一般都会使用 innodb 引擎（内部支持事务、行级锁、外键等特点），在MySQL5.5版本之后默认引擎也是innodb。

```sql
mysql> show create table users \G;
*************************** 1. row ***************************
       Table: users
Create Table: CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(32) DEFAULT NULL,
  `password` varchar(64) DEFAULT NULL,
  `ctime` datetime DEFAULT NULL,
  `age` int(11) DEFAULT '5',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

ERROR:
No query specified

mysql> show index from users \G;
*************************** 1. row ***************************
        Table: users
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 3
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE   -- 虽然显示BTree，但底层数据结构基于B+Tree。
      Comment:
Index_comment:
1 row in set (0.00 sec)

ERROR:
No query specified

mysql>
```

innodb引擎，一般创建的索引：聚簇索引。

#### 7.2 常见索引

在innodb引擎下，索引底层都是基于B+Tree数据结构存储（聚簇索引）。

![image-20210526170708409](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526170708409.png)

在开发过程中常见的索引类型有：

- 主键索引：加速查找、不能为空、字段的值不能重复，索引只有一个。 + 联合主键索引
- 唯一索引：加速查找、字段的值不能重复，索引可以有多个。  + 联合唯一索引
- 普通索引：加速查找，索引可以有多个。 + 联合索引

##### 7.2.1 主键和联合主键索引

```sql
create table 表名(
    id int not null auto_increment primary key,   -- 主键
    name varchar(32) not null
);

create table 表名(
    id int not null auto_increment,
    name varchar(32) not null,
    primary key(id)
);

create table 表名(
    id int not null auto_increment,
    name varchar(32) not null,
    primary key(列1,列2)          -- 如果有多列，称为联合主键（不常用且myisam引擎支持）
);
```

```sql
alter table 表名 add primary key(列名);
```

```sql
alter table 表名 drop primary key;
```

注意：删除索引时可能会报错，自增列必须定义为键。

```
ERROR 1075 (42000): Incorrect table definition; there can be only one auto column and it must be defined as a key

alter table 表 change id id int not null;
```

```sql
create table t7(
    id int not null,
    name varchar(32) not null,
    primary key(id)
);

alter table t6 drop primary key;
```

##### 7.2.2 唯一和联合唯一索引

```sql
create table 表名(
    id int not null auto_increment primary key,
    name varchar(32) not null,
    email varchar(64) not null,
    unique ix_name (name),
    unique ix_email (email),
);

create table 表名(
    id int not null auto_increment,
    name varchar(32) not null,
    unique (列1,列2)               -- 如果有多列，称为联合唯一索引。
);
```

```sql
create unique index 索引名 on 表名(列名);
```

```sql
drop unique index 索引名 on 表名;
```

##### 7.2.3 索引和联合索引

```sql
create table 表名(
    id int not null auto_increment primary key,
    name varchar(32) not null,
    email varchar(64) not null,
    index ix_email (email),
    index ix_name (name),
);

create table 表名(
    id int not null auto_increment primary key,
    name varchar(32) not null,
    email varchar(64) not null,
    index ix_email (name,email)     -- 如果有多列，称为联合索引。
);
```

```sql
create index 索引名 on 表名(列名);
```

```sql
drop index 索引名 on 表名;
```

在项目开发的设计表结构的环节，大家需要根据业务需求的特点来决定是否创建相应的索引。

#### 案例：博客系统

![image-20210531164453161](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210531164453161.png)

- 每张表id列都创建 自增 + 主键。
- 用户表
  - 用户名 + 密码 创建联合索引。
  - 手机号，创建唯一索引。
  - 邮箱，创建唯一索引。
- 推荐表
  - user_id和article_id创建联合唯一索引。



#### 7.3 操作表

在表中创建索引后，查询时一定要命中索引。

![image-20210526170700985](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526170700985.png)

![image-20210526155746811](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526155746811.png)

在数据库的表中创建索引之后优缺点如下：

- 优点：查找速度快、约束（唯一、主键、联合唯一）
- 缺点：插入、删除、更新速度比较慢，因为每次操作都需要调整整个B+Tree的数据结构关系。

所以，在表中不要无节制的去创建索引啊。。。

在开发中，我们会对表中经常被搜索的列创建索引，从而提高程序的响应速度。

![image-20210526170718219](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526170718219.png)

```sql
CREATE TABLE `big` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `name` varchar(32) DEFAULT NULL,
    `email` varchar(64) DEFAULT NULL,
    `password` varchar(64) DEFAULT NULL,
    `age` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),                       -- 主键索引
    UNIQUE KEY `big_unique_email` (`email`),  -- 唯一索引
    index `ix_name_pwd` (`name`,`password`)     -- 联合索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

一般情况下，我们针对只要通过索引列去搜搜都可以 `命中` 索引（通过索引结构加速查找）。

```sql
select * from big where id = 5;
select * from big where id > 5;
select * from big where email = "wupeiqi@live.com";
select * from big where name = "武沛齐";
select * from big where name = "kelly" and password="ffsijfs";
...
```

但是，还是会有一些特殊的情况，让我们无法命中索引（即使创建了索引），这也是需要大家在开发中要注意的。

![image-20210526170718219](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210526170718219.png)

- 类型不一致

  ```sql
  select * from big where name = 123;		-- 未命中
  select * from big where email = 123;	-- 未命中
  
  -- 特殊的主键：
  	select * from big where id = "123";	-- 命中
  ```

- 使用不等于

  ```sql
  select * from big where name != "武沛齐";				-- 未命中
  select * from big where email != "wupeiqi@live.com";  -- 未命中
  
  -- 特殊的主键：
  	select * from big where id != 123;	-- 命中
  ```

- or，当or条件中有未建立索引的列才失效。

  ```sql
  select * from big where id = 123 or password="xx";			-- 未命中
  select * from big where name = "wupeiqi" or password="xx";	-- 未命中
  -- 特别的：
  	-- 注：and优先级高于or
  	select * from big where id = 10 or password="xx" and name="xx"; -- 命中
  ```

- 排序，当根据索引排序时候，选择的映射如果不是索引，则不走索引。

  ```sql
  select * from big order by name asc;     -- 未命中
  select * from big order by name desc;    -- 未命中
  
  -- 特别的主键：
  	select * from big order by id desc;  -- 命中
  ```

- like，模糊匹配时。

  ```sql
  select * from big where name like "%u-12-19999";	-- 未命中
  select * from big where name like "_u-12-19999";	-- 未命中
  select * from big where name like "wu-%-10";		-- 未命中
  
  -- 特别的：
  	select * from big where name like "wu-1111-%";	-- 命中
  	select * from big where name like "wuw-%";		-- 命中
  ```

- 使用函数

  ```sql
  select * from big where reverse(name) = "wupeiqi";  -- 未命中
  
  -- 特别的：
  	select * from big where name = reverse("wupeiqi");  -- 命中
  ```

- 最左前缀，如果是联合索引，要遵循最左前缀原则。

  ```sql
  如果联合索引为：(name,password)
      name and password       -- 命中
      name                 	-- 命中
      password                -- 未命中
      name or password       	-- 未命中
  ```


常见的无法命中索引的情况就是上述的示例。

对于大家来说会现在的最大的问题是，记不住，哪怎么办呢？接下来看执行计划。

#### 7.4 执行计划

MySQL中提供了执行计划，让你能够预判SQL的执行（只能给到一定的参考，不一定完全能预判准确）。

```
explain + SQL语句;
```

![image-20210527074105599](MySQL%E5%9F%BA%E7%A1%80.assets/image-20210527074105599.png)

其中比较重要的是 type，他他SQL性能比较重要的标志，性能从低到高依次：`all < index < range < index_merge < ref_or_null < ref < eq_ref < system/const` 

- ALL，全表扫描，数据表从头到尾找一遍。(一般未命中索引，都是会执行权标扫描)

  ```sql
  select * from big;
  
  特别的：如果有limit，则找到之后就不在继续向下扫描.
  	select * from big limit 1;
  ```

- INDEX，全索引扫描，对索引从头到尾找一遍

  ```sql
  explain select id from big;
  explain select name from big;
  ```

- RANGE，对索引列进行范围查找

  ```sql
  explain select * from big where id > 10;
  explain select * from big where id in (11,22,33);
  explain select * from big where id between 10 and 20;
  explain select * from big where name > "wupeiqi" ;
  ```

- INDEX_MERGE，合并索引，使用多个单列索引搜索

  ```sql
  explain select * from big where id = 10 or name="武沛齐";
  ```

- REF，根据 索引 直接去查找（非键）。

  ```sql
  select *  from big where name = '武沛齐';
  ```

- EQ_REF，连表操作时常见。

  ```sql
  explain select big.name,users.id from big left join users on big.age = users.id;
  ```

- CONST，常量，表最多有一个匹配行,因为仅有一行,在这行的列值可被优化器剩余部分认为是常数,const表很快。

  ```sql
  explain select * from big where id=11;					-- 主键
  explain select * from big where email="w-11-0@qq.com";	-- 唯一索引
  ```

- SYSTEM，系统，表仅有一行(=系统表)。这是const联接类型的一个特例。

  ```sql
   explain select * from (select * from big where id=1 limit 1) as A;
  ```

其他列：

```
id，查询顺序标识

z，查询类型
    SIMPLE          简单查询
    PRIMARY         最外层查询
    SUBQUERY        映射为子查询
    DERIVED         子查询
    UNION           联合
    UNION RESULT    使用联合的结果
    ...
    
table，正在访问的表名

partitions，涉及的分区（MySQL支持将数据划分到不同的idb文件中，详单与数据的拆分）。 一个特别大的文件拆分成多个小文件（分区）。

possible_keys，查询涉及到的字段上若存在索引，则该索引将被列出，即：可能使用的索引。
key，显示MySQL在查询中实际使用的索引，若没有使用索引，显示为NULL。例如：有索引但未命中，则possible_keys显示、key则显示NULL。

key_len，表示索引字段的最大可能长度。(类型字节长度 + 变长2 + 可空1)，例如：key_len=195，类型varchar(64)，195=64*3+2+1

ref，连表时显示的关联信息。例如：A和B连表，显示连表的字段信息。

rows，估计读取的数据行数（只是预估值）
	explain select * from big where password ="025dfdeb-d803-425d-9834-445758885d1c";
	explain select * from big where password ="025dfdeb-d803-425d-9834-445758885d1c" limit 1;
filtered，返回结果的行占需要读到的行的百分比。
	explain select * from big where id=1;  -- 100，只读了一个1行，返回结果也是1行。
	explain select * from big where password="27d8ba90-edd0-4a2f-9aaf-99c9d607c3b3";  -- 10，读取了10行，返回了1行。
	注意：密码27d8ba90-edd0-4a2f-9aaf-99c9d607c3b3在第10行
	
extra，该列包含MySQL解决查询的详细信息。
    “Using index”
    此值表示mysql将使用覆盖索引，以避免访问表。不要把覆盖索引和index访问类型弄混了。
    “Using where”
    这意味着mysql服务器将在存储引擎检索行后再进行过滤，许多where条件里涉及索引中的列，当（并且如果）它读取索引时，就能被存储引擎检验，因此不是所有带where子句的查询都会显示“Using where”。有时“Using where”的出现就是一个暗示：查询可受益于不同的索引。
    “Using temporary”
    这意味着mysql在对查询结果排序时会使用一个临时表。
    “Using filesort”
    这意味着mysql会对结果使用一个外部索引排序，而不是按索引次序从表里读取行。mysql有两种文件排序算法，这两种排序方式都可以在内存或者磁盘上完成，explain不会告诉你mysql将使用哪一种文件排序，也不会告诉你排序会在内存里还是磁盘上完成。
    “Range checked for each record(index map: N)”
    这个意味着没有好用的索引，新的索引将在联接的每一行上重新估算，N是显示在possible_keys列中索引的位图，并且是冗余的。
```

#### 小结

上述索引相关的内容讲的比较多，大家在开发过程中重点应该掌握的是：

- 根据情况创建合适的索引（加速查找）。
- 有索引，则查询时要命中索引。

