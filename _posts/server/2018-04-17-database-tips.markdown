---
layout: post
title:  "数据库小知识"
date:   2018-04-17 15:34:30 +0800
categories: server
---

* TOC
{:toc}


## SELECT ... FOR UPDATE




## 按照多个列排序

`ORDER BY` 是支持对多个列进行排序的，只需要按照顺序指定列名，并用逗号隔开即可。

```sql
SELECT prod_id, prod_price, prod_name
FROM products
ORDER BY prod_price, prod_name;
```

如果希望某些列采用降序排列，那么对每个需要降序的列都应该指定 `DESC`:

```sql
SELECT prod_id, prod_price, prod_name
FROM products
ORDER BY prod_price DESC, prod_name DESC;
```


## ORDER BY + LIMIT 获取最大或最小值

```sql
SELECT prod_price
FROM products
ORDER BY prod_price DESC
LIMIT 1;
```


## 使用计算字段

你可以对查询得到的列进行一些计算，得到一个新的列(称为字段 field):

```sql
-- 把名字和国家按照 名字(国家) 的格式拼接起来，形成一个新的字段
SELECT Concat(vend_name, '(', vend_country, ')')
FROM vendors
ORDER BY vend_name;
```

```sql
-- 数量 * 单价 形成一个新的字段
SELECT prod_id, quantity, item_price, quantity*item_price AS expanded_price
FROM orderitems
WHERE order_num = 20005;
```

上述例子中的 Concat 是 MySql 提供的数据处理函数。类似的函数还有：

- 文本处理类:
    - Length() : 获取字符串长度;
    - Locate() : 找出串的一个子串;
    - Lower() : 将串转换为小写;
    - Upper() : 将串转换为大写;
    - LTrim() : 去掉左边空格;
    - RTrim() : 去掉右边空格;
- 日期处理类:
    - Now(), CurDate(), CurTime() : 获取当前日期和时间;
    - AddDate(), AddTime() : 增加时间;
    - DateDiff() : 计算日期的差;
    - Date_Add() : 日期计算;
    - Date_Format() : 返回格式化后的日期;
    - Year(), Month(), Day(), DayOfWeek(), Hour(), Minute(), Second() 返回日期中的指定部分;
    - Date() : 返回日期部分;
    - Time() : 返回时间部分;
- 数值处理类:
    - Abs();
    - Sin(), Cos(), Tan();
    - Exp() 返回指数值;
    - Mod() 返回余数;
    - Pi()
    - Rand() 随机数;
    - Sqrt()

上述函数都是对一个具体的值进行计算，SQL 还提供了一些函数，能够对检索出的值进行汇总计算，这些函数叫做聚集函数 (aggregate function):
- AVG() 返回某列的平均数;
- COUNT() 返回某列的行数;
- MAX() 返回某列最大值;
- MIN() 返回某列最小值;
- SUM() 计算某列之和;

