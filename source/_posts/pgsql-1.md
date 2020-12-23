---
layout: post
title: 关于pgsql的索引
date: 2020-12-18 10:00:00
tags: 
- pgsql
categories:
- pgsql
---

主要注意一般jsonb的索引都是gin索引，gin是倒排索引的实现: https://www.cnblogs.com/flying-tiger/p/6704931.html

当然我们也可以为jsonb的某一列建立btree索引：

```sql
--btree--
create index if not exists idx_customers on danube.config_data ((config_info -> 'customers'::text));
--gin--
create index if not exists idx_customers on danube.config_data using gin (config_info, (config_info->'customers'));
```

PostgreSQL 9种索引的原理和应用场景: http://mysql.taobao.org/monthly/2019/04/08/

关于pgsql的文章 https://github.com/digoal/blog/blob/master/README.md

总体感觉pgsql的实现要比mysql复杂得多，功能也更强大，比如json的支持（直接代替文档数据库），gin索引的支持，物化视图的支持（多表查询的利器），还有fdw功能（把别的库的表当成自己的用，直接解决跨库join的问题），主从物理复制（流复制，实例复制，基于wal日志，一致性强于mysql的binlog逻辑复制）。

mysql也就胜在简单，以及做分库分表和ha的这些中间件的生态更齐全吧。性能看网上说都估计都差不多。

感觉pgsql有个大问题就是学习资料还是太少了，公司生产中用pgsql碰到过几次死锁，后来都是找dba搞定的，自己搞不定。

有时间我会再仔细整理这部分。