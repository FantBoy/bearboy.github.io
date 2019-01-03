---
layout: post
filename: common_sql_optimization_strategy
title: 常见sql优化策略
datetime: 2018-04-20 10:56:02
description: 

comments: true
tags:
 - MySQLdb优化
 
categories:
 - MySQLdb
 
---

## 常见sql优化策略

 - 去掉不必要的查询和搜索。很多查询条件是可有可无的，能从源头上避免的多余功能尽量砍掉，这是最简单粗暴的解决方案。

 - 合理使用索引和复合索引。建索引是SQL优化中最有效的手段
  > 查找、删除、更新以及排序时常用的字段可以适当建立索引
  > 单条查询不能同时使用多个索引，只能使用一个索引
  > 查询条件较多时，可以使用多个字段合并的复合索引
  > 使用复合索引时，查询条件的字段顺序需要与复合索引的字段顺序保持一致

 - 谨慎使用not in等可能无法使用索引的条件
  > 当出现`not in`，`!=`，`like '%xx%'`，`is null`等条件时，索引无效，使用这些条件的时候，请放到能有效使用索引的条件的右边
  > 设计表结构时，尽可能用int类型代替varchar类型，int类型部分时候可以通过大于或小于代替`!=`等条件
  > 把所有可能的字段设置为`not null`，并设置默认值，避免在where字句中出现`is null`的判断。

 - 不要在where子句中的`=`左边进行函数、算术运算或其他表达式运算，否则系统将无法正确使用索引。尽可能少用MySQL的函数，类似`Now()`完全可以通过程序实现并赋值，部分函数也可以通过适当的建立冗余字段来间接替代

 - 在where条件中使用or，可能导致索引无效。可用 `union all` 或者 `union` （会过滤重复数据，效率比前者低） 代替，或程序上直接分开两次获取数据再合并，确保索引的有效利用

 - 不使用`select * `，倒不是能提高查询效率，主要是减少输出的数据量，提高传输速度。

 - 避免类型转换，这里所说的“类型转换”是指where子句中出现字段的类型和传入的参数类型不一致的时候发生的类型转换

 - 分页查询的优化。页数比较多的情况下，如`limit 10000,10 `影响的结果集是10010行，查询速度会比较慢
  > 推荐的解决方案是：先只查询主键`select id from table where .. order by .. limit 10000,10`（搜索条件和排序请建立索引），再通过主键去获取数据






















