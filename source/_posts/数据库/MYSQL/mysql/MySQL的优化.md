---
title: MySQL的优化
categories:
  - 数据库
  - MYSQL
  - mysql
tags:
  - 数据库
  - MYSQL
  - mysql
abbrlink: f1e1e97a
date: 2018-06-28 09:15:20
---
### 1. MySQL优化的三个方面

- #### 索引优化

- #### 表结构的优化

- #### SQL慢查询的优化

![图](https://github.com/mxsm/document/blob/master/image/database/MySQL%E7%9A%84%E4%BC%98%E5%8C%96%E4%B8%89%E4%B8%AA%E6%96%B9%E9%9D%A2.jpg?raw=true)

### 2. 索引优化

#### 2.1什么是索引

索引：简单的说，相当于图书的目录，可以帮助用户快速的找到需要的内容。索引的目的在于提高查询效率，与我们查询图书所用的目录是一个道理：先定位到章，然后定位到该章下的一个小结，然后找到页数。相似的例子还有：查字典，查地图等。

#### 2.2 索引的类型

- **普通索引**

  ```
  是最基本的索引，它没有任何限制。
  ```

- **唯一索引**

  ```
  与前面的普通索引类似，不同的就是：索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。
  ```

- **组合索引**

  ```
  指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。
  ```

- **主键索引**

  ```
  是一种特殊的唯一索引，一个表只能有一个主键，不允许有空值。一般是在建表的时候同时创建主键索引
  ```

- **全文索引(InnoDB不支持)**

  ```
  主要用来查找文本中的关键字，而不是直接与索引中的值相比较。fulltext索引跟其它索引大不相同，它更像是一个搜
  索引擎，而不是简单的where语句的参数匹配。fulltext索引配合match against操作使用，而不是一般的where语
  句加like。它可以在create table，alter table ，create index使用，不过目前只有char、varchar，text
  列上可以创建全文索引。值得一提的是，在数据量较大时候，现将数据放入一个没有全局索引的表中，然后再用CREATE 
  index创建fulltext索引，要比先为一张表建立fulltext然后再将数据写入的速度快很多。
  ```

#### 2.3 索引的优化

- 只要列中含有NULL值，就最好不要在此例设置索引，复合索引如果有NULL值，此列在使用时也不会使用索引。(所以在建表的过程中尽可能的不要有NULL的字段，设置相应的默认值)
- 尽量使用短索引，如果可以，应该制定一个前缀长度
- 对于经常在where子句使用的列，最好设置索引，这样会加快查找速度
- 对于有多个列where或者order by子句的，应该建立复合索引(这个应该是在开发后期页面展示的视图基本上确定了)
-  对于like语句，以%或者‘-’开头的不会使用索引，以%结尾会使用索引
- 尽量不要在列上进行运算（函数操作和表达式操作）(比如 **`from_unixtime(create_time) = ’2014-05-29’`** 这样的操作应该尽量避免，但是可以写成下面这样  **`create_time= unix_timestamp(’2014-05-29’)`**)
- 尽量不要使用not in和<>操作

### 3. SQL慢查询优化

SQL慢查询优化的步骤：

![图](https://github.com/mxsm/document/blob/master/image/database/SQL%E6%85%A2%E6%9F%A5%E8%AF%A2%E4%BC%98%E5%8C%96%E6%AD%A5%E9%AA%A4.jpg?raw=true)

#### 3.1 如何捕获低效SQL

- **slow_query_log**

  这个参数设置为ON，可以捕获执行时间超过一定数值的SQL语句。

- **long_query_time**

  当SQL语句执行时间超过此数值时，就会被记录到日志中，建议设置为1或者更短

- **slow_query_log_file**

  记录日志的文件名。

- **log_queries_not_using_indexes**

  这个参数设置为ON，可以捕获到所有未使用索引的SQL语句，尽管这个SQL语句有可能执行得挺快。

#### 3.2 慢查询优化的基本步骤

- **先运行看看是否真的很慢，注意设置SQL_NO_CACHE**

  SELECT SQL_NO_CACHE yz_id FROM dev_item_backlog WHERE yz_id NOT IN ('xZeT3PHc','WcUVNWEq');

- **where条件单表查，锁定最小返回记录表**

  查询语句的where都应用到表中返回的记录数最小的表开始查起，单表每个字段分别查询，看哪个字段的区分度最高

- **explain查看执行计划**

- **order by limit 形式的sql语句让排序的表优先查**

- **加索引时参照建索引的几大原则**

#### 3.3 优化原则

- **查询时，能不要 * 就不用 *，尽量写全字段名**
- **大部分情况连接效率远大于子查询换一句话说就是 少用子查询**
- **多使用explain和profile分析查询语句**
- **查看慢查询日志，找出执行时间长的sql语句优化**
- **多表连接时，尽量小表驱动大表，即小表 join 大表**
- **在千万级分页时使用limit**
-  **对于经常使用的查询，可以开启缓存**

### 4. 数据库表的优化

- **表的字段尽可能用NOT NULL，或者是最好字段都不存在NULL。因为NULL不走索引。所以最好设置非NULL的默认值**

- **将表拆分，垂直拆分或者水平拆分**

  