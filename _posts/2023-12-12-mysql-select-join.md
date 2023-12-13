---
title: Mysql select & join
date: 2023-12-12 11:52 -0800
categories: [database, mysql]
tags: [mysql, join]
---

QEP: query execution plan

## Join

### straight join

It is kind of option that forbid reordering tables when joining. According to
[official doc](https://dev.mysql.com/doc/refman/8.0/en/join.html),

> This can be used for those (few) cases for which the join optimizer processes
> the tables in a suboptimal order.

### Semi-join

TLDR: semi-join is a optimization step that transforms a sub-select into a
join.

This is a good blog about this topic
https://roylyseng.blogspot.com/2012/04/semi-join-in-mysql-56.html

The different semijoin strategy constants are defined
[here](https://github.com/mysql/mysql-server/blob/eb86b4016060d426858cc09873d12492f1be396e/sql/sql_const.h#L212).

Relevant code
[here](https://github.com/mysql/mysql-server/blob/9912892feecfa8f0450bb97b74ebaf37d16e375c/sql/sql_planner.cc#L4183-L4184).

### Choose table order

const table: table with zero or one row. See
[code](https://github.com/mysql/mysql-server/blob/2a57e948ca9b238262161ae854119f60c8fd347e/sql/sql_optimizer.cc#L5629).

Why const table is special? Because const table can be put at the most outside
order.

Questions:

1. what is the cost? how is it calculated. For const table, it is zero.
   - https://dev.mysql.com/doc/refman/8.0/en/cost-model.html
