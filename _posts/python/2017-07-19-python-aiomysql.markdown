---
layout: post
title:  "Python 使用 aiomysql"
date:   2017-07-19 10:32:05 +0800
categories: python
---

* TOC
{:toc}


本文介绍如何在 python3 中使用 aiomysql 库来访问 mysql 数据库。

简单来说，要操作数据库，首先需要连接到数据库中，然后对其执行增删改查等操作。


## MySQL Workbench 操作

### 连接到 MySQL 数据库

在 Windows 上操作数据库，可以通过 MySQL Workbench 来完成，图形化的界面比较简单一些。

打开 MySQL Workbench 之后，会显示如下界面，其中已经包含了我之前建立好的一个在本机上的数据库，名为 localhost ：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench0.png )

如果数据库是安装在其它机器上的，也可以通过 Workbench 来管理，这需要点击上图中的 "+" 图标，打开数据库添加窗口：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench1.png )

添加连接名称，机器的 IP 及端口，用户名密码，然后点击 Test Connection 测试一下能否连接上，可以的话点确定，这样就连接到了另一台机器上的数据库。

### 创建 schema

一个 MySQL 中可能会有好几个数据库实例，在 MySQL 中这被称为 schema (方案)，在创建数据库表之前，首先要创建一个 schema。

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench2.png )

点击工具栏上的 create schema 按钮，就能打开创建新 schema 的 tab 页，填写 schema 名称后点击 Apply, 弹出以下确认窗口：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench3.png )

可以看到实际上是执行了以下 MySQL 指令：

```sql
CREATE SCHEMA `test` ;
```

创建好的 schema 会显示在左侧导航栏中：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench4.png )

### 创建 Table

一个 schema 中可能有多个数据库表，我们之前创建的 test schema 是空的，现在创建一个出来：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench5.png )

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench6.png )

你可以在这里填写 table 的名称，并设置表的每一列。

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench7.png )

可以看到实际上是执行了以下 MySQL 指令：

```sql
CREATE TABLE `test`.`user` (
  `idnew_table` INT NOT NULL,
  `email` VARCHAR(256) NULL,
  `address` VARCHAR(256) NULL,
  `phonenumber` VARCHAR(256) NULL,
  PRIMARY KEY (`idnew_table`));
```

如果创建 Table 后发现有些表结构需要修改，比如需要把 id 设为自增长，那么可以点击如下按钮打开修改 Tab 页：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench8.png )

### 插入数据

MySQL 中插入数据的语法如下：

```sql
INSERT [LOW_PRIORITY | DELAYED | HIGH_PRIORITY] [IGNORE]
    [INTO] tbl_name [(col_name,...)]
    VALUES ({expr | DEFAULT},...),(...),...
    [ ON DUPLICATE KEY UPDATE col_name=expr, ... ]

INSERT [LOW_PRIORITY | DELAYED | HIGH_PRIORITY] [IGNORE]
    [INTO] tbl_name
    SET col_name={expr | DEFAULT}, ...
    [ ON DUPLICATE KEY UPDATE col_name=expr, ... ]

INSERT [LOW_PRIORITY | HIGH_PRIORITY] [IGNORE]
    [INTO] tbl_name [(col_name,...)]
    SELECT ...
    [ ON DUPLICATE KEY UPDATE col_name=expr, ... ]
```

看起来很乱是不是，把参数去掉之后就很简化了：

```sql
-- 向表中插入数据
INSERT INTO tbl_name (col_name, ...) VALUES ({expr | default}, ...)

-- 向表中插入数据，采用 key=value 的形式
INSERT INTO tbl_name set col_name={expr | default}, ...

-- 向表中插入数据，该数据通过查询其他表得到
INSERT INTO tbl_name (col_name, ...) SELECT ...
```

例如以下几个例子：

```sql
INSERT INTO user (id, email, address, phonenumber) VALUES (1, "test123@outlook.com", "毛里求斯", "12344445555");

INSERT INTO user set email="hahaha123@outlook.com", phonenumber="12345678910";

INSERT INTO user (email, address) SELECT email,address FROM vipuser;
```

在 Workbench 里点击如下按钮，就会打开一个编辑页面，可以输入上述指令来测试效果：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench9.png )

注意每个 SQL 语句之间用分号 `;` 分隔。

### 删除数据

MYSQL 中删除语句的语法如下：

```sql
DELETE [LOW_PRIORITY] [QUICK] [IGNORE] FROM tbl_name  
[WHERE where_definition]  
[ORDER BY ...]  
[LIMIT row_count] 

-- 简化版
DELETE FROM tbl_name WHERE where_definition
```

需要注意的地方主要是 `where_definition` 这个怎么填写。主要有一下几种方式：

```sql
-- 通过 key=value 的形式来指定要删除的内容
DELETE FROM user WHERE id=1;
DELETE FROM user WHERE phonenumber="12345678910";

-- 可以通过 AND, OR 等操作符来进行 与或操作
DELETE FROM user WHERE email="test123@outlook.com" OR phonenumber="12345678910";
DELETE FROM user WHERE address="毛里求斯" AND phonenumber="12344445555";

-- 可以通过 LIKE 进行模糊匹配
-- 删除所有电话号码以 186 开头的行
DELETE FROM user WHERE phonenumber LIKE '186%';
-- 删除所有电话号码符合 186xxxx1234 模式的行
DELETE FROM user WHERE phonenumber LIKE '186____1234'
-- 删除所有地址中包括 % 符号的行, 这里的 % 需要用 / 进行转义
DELETE FROM user WHERE address LIKE '%/%%'

-- 可以通过 REGEXP 进行正则查找
DELETE FROM user WHERE phonenumber REGEXP '^186'
DELETE FROM user WHERE phonenumber REGEXP '1234$'
DELETE FROM user WHERE address REGEXP '^[0-9]'
```

对于 LIKE 语句，只有三种特殊符号：
- `%` 表示任意多个任意字符；
- `_` 表示一个任意字符；
- `/` 是转义字符；

对于 REGEXP 语句，其规则跟普通的正则语法类似，这里就不介绍了，以后需要可以再确认；

执行上述语句时有可能出现错误：Error Code: 1175. You are using safe update mode and you tried to update a table without a WHERE that uses a KEY column To disable safe mode, toggle the option in Preferences -> SQL Queries and reconnect.

这是因为 MySQL 运行在 safe-update 模式下，不能根据主键意外的查找条件来删除数据，你可以通过 Workbench 菜单的 Edit --> Preferences 菜单项打开选项，然后取消勾选 Safe Updates 选项：

![]( {{site.url}}/asset/python-aiomysql-mysql-workbench10.png )

**注意，设置该选项之后需要重新连接到数据库上。**

除了 DELETE 语句，还可以使用 TRUNCATE 语句，该语句的作用是删除表中的所有数据：

```sql
TRUNCATE TABLE user;
```

### 修改数据

MySQL 中修改数据的语法如下：

```sql
UPDATE [LOW_PRIORITY] [IGNORE] tbl_name
    SET col_name1=expr1 [, col_name2=expr2 ...]
    [WHERE where_definition]
    [ORDER BY ...]
    [LIMIT row_count]
```

其中 SET, WHERE 的用法之前已经介绍过了，举例如下：

```sql
UPDATE user SET phonenumber="12345678910" WHERE phonenumber LIKE "186%"
```

### 查询数据

MySQL 中查询数据的语法如下：

```sql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
      [HIGH_PRIORITY]
      [STRAIGHT_JOIN]
      [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
      [SQL_CACHE | SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
    select_expr, ...
    [INTO OUTFILE 'file_name' export_options
      | INTO DUMPFILE 'file_name']
    [FROM table_references
    [WHERE where_definition]
    [GROUP BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [HAVING where_definition]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC] , ...]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [PROCEDURE procedure_name(argument_list)]
    [FOR UPDATE | LOCK IN SHARE MODE]]
```

简化版：

```sql
SELECT col_name, ... FROM tbl_name WHERE where_definition
```

举例：

```sql
-- 查询 user 表中的所有数据
SELECT * FROM user;

-- 查询 user 表中的 phonenumber, address 这两列的所有数据
SELECT phonenumber, address FROM user;

-- 查询 user 表中电话号码以 186 开头的所有数据
SELECT * FROM user WHERE phonenumber LIKE '186%'

-- 查询 user 表中的所有数据，然后按照电话号码进行升序排序
SELECT * FROM user ORDER BY phonenumber

-- 查询 user 表中的所有数据，然后按照地址进行分组
SELECT * FROM user GROUP BY address
```

以上介绍了 MySQL Workbench 的操作方法，以及增删改查 SQL 语句的语法。

### 使用 .sql 文件

数据库的创建操作如果是手动操作的话，以后再操作起来比较麻烦。最好是保存到文件里，需要的话通过 .sql 文件来创建。例如：

```sql
-- create_user.sql

drop database if exists user;

create database user;

use user;

CREATE TABLE user (
    `id` varchar(50) not null,
    `email` varchar(50) not null,
    `address` varchar(50) not null,
    unique key `idx_email` (`email`),
    primary key (`id`)
) engine=innodb default charset=utf8;
```

在命令行里执行该脚本，即可创建对应的 schema 和 table：

```s
$ mysql -u root -p < schema.sql
```


## aiomysql 如何使用

未完待续...