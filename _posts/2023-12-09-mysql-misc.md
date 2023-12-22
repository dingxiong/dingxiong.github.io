---
title: Mysql introduction
date: 2023-12-09 01:20 -0800
categories: [database, mysql]
tags: [mysql]
---

I am reading a good book "Database internals" by Alex Petrov.

Below are some useful papers I read

- [Ubiquitous b-tree](https://dl.acm.org/doi/10.115/356770.356776)

Some useful resources:

- Alibaba's database engine blogs http://mysql.taobao.org/monthly/

## Build

I did not find any official document about how to build mysql from source code.
But its test folder has a
[setup readme](https://github.com/mysql/mysql-server/blob/a246bad76b9271cb4333634e954040a970222e0a/mysql-test/std_data/log_corruption/README.txt),
which is quite useful.

**Generate Makefile and compilation database**

Mysql relies on boost at configuration stage.

```
cd mysql-server
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Debug -G "Unix Makefiles" -DCMAKE_EXPORT_COMPILE_COMMANDS=1 -DDOWNLOAD_BOOST=1  -DWITH_BOOST=./boost/ ..
cd ..
ln -s build/compile_commands.json compile_commands.json
```

Mysql build has a few dependencies. In ubuntu, you can install them as

```
apt-get install flex bison libudev-dev libssl-dev libncurses5-dev
```

**Build**

Do not run make install!

```
make -j8
```

## Post installation

There are
[a few steps](https://dev.mysql.com/doc/refman/8.0/en/postinstallation.html) to
follow after a fresh installation.

First, create mysql user and group

```
$ groupadd mysql
$ useradd -r -g mysql -s /bin/false mysql
```

This step can be omitted if the OS is MacOS.

Second, initialize the data directory. Suppose the build folder is
`/code/mysql-server/build`.

```
cd build
rm -rf data && mkdir data
bin/mysqld --initialize --user=mysql
```

The above command will print a temp root password in console.Now are ready to
start server

```
bin/mysqld --user=mysql
```

Open another terminal,

```
bin/mysql -u root -p
```

Type the temp password and then assign a new root password.

```
mysql>  ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
```

One thing I found interesting about MacOS. I already installed mysql using
homebrew and it is running. I assume the above process will fail because port
3306 is already used. However, it is not.

```bash
$ lsof -i tcp:3306
COMMAND   PID      USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
mysqld  62685 xiongding   21u  IPv6 0x26ae8b92b7c99911      0t0  TCP *:mysql (LISTEN)
mysqld  96366 xiongding  363u  IPv4 0x26ae8ba13830ee11      0t0  TCP localhost:mysql (LISTEN)

$ ps -ef |grep mysql
  502 96366 95551   0 Thu10AM ??         9:28.96 /opt/homebrew/opt/mysql/bin/mysqld --basedir=/opt/homebrew/opt/mysql --datadir=/opt/homebrew/var/mysql --plugin-dir=/opt/homebrew/opt/mysql/lib/plugin --log-error=xiong-MacBook-Pro-Zip.local.err --pid-file=xiong-MacBook-Pro-Zip.local.pid
  502 62685 50179   0 11:14AM ttys009    0:02.80 ./bin/mysqld --user=mysql
```

You can see that it works because one uses IPv4 and the other uses IPv6. By
default, Mysql server starts to listen to IPv6. If you want to use a different
port, then you can do

```bash
bin/mysqld --user=mysql --port 3307
bin/mysql -u root -ppassword -P 3307
```

Btw, mysqld does not response to `Ctl-c`, but it responses to SIGQUIT, which is
`Ctl-\`.

### Create example database and tables

Mysql provides sample data for an "employee" database. See
[doc](https://dev.mysql.com/doc/employee/en/employees-installation.html).

```bash
cd build
wget https://github.com/datacharmer/test_db/archive/refs/tags/v1.0.7.zip
unzip v1.0.7.zip
cd test_db-1.0.7/
../bin/mysql -t -u root -ppassword -P 3307 < employees.sql
```

Also, below is a sample table as well.

```
CREATE DATABASE demodb;

USE demodb;

CREATE TABLE t1 (
    id bigint NOT NULL,
    last_name varchar(255) NOT NULL,
    first_name varchar(255),
    age int,
    PRIMARY KEY (id),
    INDEX index_name (first_name, last_name)
);

insert into t1 (id, last_name, first_name, age) values (1, 'x1', 'y1', 18);
insert into t1 (id, last_name, first_name, age) values (2, 'x2', 'y2', 18);
```

## Debug

### Run in gdb

```
gdb --args bin/mysqld --user=mysql
(gdb) run&
```

Here, we added `&` to the run command because the main thread is daemon thread,
and it waits forever. We want to check the child threads. When a client
connects to the server, gdb will show a message like below

```
(gdb) [New Thread 0xffffe067edc0 (LWP 51367)]
```

### Run in lldb

As of Dec 17 2023, gdb still does not support MacOS Apple chip, so the only
debugger I can use is lldb.

```bash
$ lldb -- ./bin/mysqld --user=mysql --port 3307
(lldb) b sql/sql_optimizer.cc:5875
Breakpoint 1: where = mysqld`JOIN::estimate_rowcount() + 44 at sql_optimizer.cc:5875:37, address = 0x0000000100a2c4d8
(lldb) process launch -n
Process 4071 launched: '/Users/xiongding/code/mysql-server/build/bin/mysqld' (arm64)
```

### Use debug info

If Mysql is compiled with debug info, then when you connect to it, you see the
promp contains server version `xx-debug` as below. Also, a configuration
variable `debug` exists.

```bash
...
Server version: 8.2.0-debug Source distribution
...
mysql> show variables like 'debug';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| debug         |       |
+---------------+-------+
1 row in set (0.02 sec)
```

## Grammar

SQL is a nonregular language, but Mysql has a context-free grammar. See
[code](https://github.com/mysql/mysql-workbench/blob/6e135fb33942123c57f059139cbd787bea4f3f9b/library/parsers/grammars/MySQLParser.g4#L986).

## MISC

See a lot of `#ifndef UNIV_HOTBACKUP` in the codebase. This flag is never set.
See more https://jira.mariadb.org/browse/MDEV-11690 .

### Name conventions

Prefixes:

- `ut_`: utility
- `ha_`: handler
- `opt_`: optimization

## InnoDB

InnoDB source code tree has an interesting structure. Checkout
[this article](https://planet.mysql.com/entry/?id=17412) for more details.

Jeremy Cole is definitely an expert in InnoDB. He wrote a
[series blogs](https://blog.jcole.us/innodb/) about internals of InnoDB and
also is the author of `innodb_ruby`. This is most valuable references I found
on internet about InnoDB.

First, let's talk about a few core deta structures inside InnoDB.

- tablespace: fil_space_t

```
class page_id_t(m_space_no, m_page_no)
```

Blow terminology table can help you When reading the source code,

- sx lock: share/exclusive lock
-

### Locks

Next-key lock.

#### Metadata lock

There are a few good posts from Alibaba that explains MDL every well.

- http://mysql.taobao.org/monthly/2015/10/02
- http://mysql.taobao.org/monthly/2015/11/04/

## Data structures

### KEY

Key is just alias for index.

Code:
https://github.com/mysql/mysql-server/blob/7e1ce704209203da2bde727d5ce8b059d2c07c6c/sql/key.h#L121

```cpp
class KEY {
 public:
  /** Tot length of key */
  uint key_length{0};
  /** How many key_parts */
  uint user_defined_key_parts{0};
  /** How many key_parts including hidden parts */
  uint actual_key_parts{0};  //Xiong: This is the total number of fields in this index

  /**
    Array of AVG(number of records with the same field value) for 1st ... Nth
    key part. For internally created temporary tables, this member can be
    nullptr. This is the same information as stored in the above
    rec_per_key array but using float values instead of integer
    values. If the storage engine has supplied values in this array,
    these will be used. Otherwise the value in rec_per_key will be
    used.  @todo In the next release the rec_per_key array above
    should be removed and only this should be used.
  */
  rec_per_key_t *rec_per_key_float{nullptr};
  ...
};
```

One node about field `rec_per_key_float`. It is an array that contains the
average number of records with the same index prefix. For our example
`employee` database, the table `dept_emp` has below definition

```
| dept_emp | CREATE TABLE `dept_emp` (
  `emp_no` int NOT NULL,
  `dept_no` char(4) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`,`dept_no`),
  KEY `dept_no` (`dept_no`),
  CONSTRAINT `dept_emp_ibfk_1` FOREIGN KEY (`emp_no`) REFERENCES `employees` (`emp_no`) ON DELETE CASCADE,
  CONSTRAINT `dept_emp_ibfk_2` FOREIGN KEY (`dept_no`) REFERENCES `departments` (`dept_no`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

For the primary key, `actual_key_parts = 2`. so `rec_per_key_float` is a
two-element array. What are their values?

```
mysql> select * from mysql.innodb_index_stats where table_name = 'dept_emp' and index_name = 'PRIMARY';
+---------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
| database_name | table_name | index_name | last_update         | stat_name    | stat_value | sample_size | stat_description                  |
+---------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
| employees     | dept_emp   | PRIMARY    | 2023-12-12 12:18:05 | n_diff_pfx01 |     299527 |          20 | emp_no                            |
| employees     | dept_emp   | PRIMARY    | 2023-12-12 12:18:05 | n_diff_pfx02 |     331143 |          20 | emp_no,dept_no                    |
| employees     | dept_emp   | PRIMARY    | 2023-12-12 12:18:05 | n_leaf_pages |        731 |        NULL | Number of leaf pages in the index |
| employees     | dept_emp   | PRIMARY    | 2023-12-12 12:18:05 | size         |        737 |        NULL | Number of pages in the index      |
+---------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
4 rows in set (0.02 sec)
```

Field `n_diff_pfx[num]` refers the approximate number of different values for
this index with prefix length `num`. In our example, the primary key is
`(emp_no, dept_no)`, so there are approximate 299527 distinct `emp_no`, and
331143 distinct `(emp_no, dept_no)`. Also, the approximate total number of
records in this table is 331143 as well. Therefore

```
*rec_per_key_float = [299527/331143, 331143/331143] = [1.1055530887032, 1]
```

Let's check these numbers in debugger

```
(lldb) p keyinfo->rec_per_key_float[0]
(rec_per_key_t) 1.10555303
(lldb) p keyinfo->rec_per_key_float[1]
(rec_per_key_t) 1
```

Value of `rec_per_key_float` is populated by the database engine. For InnoDB,
the relevant code is
[here](https://github.com/mysql/mysql-server/blob/c12149baae15a972494a594f3eb9de2f9389a30e/storage/innobase/handler/ha_innodb.cc#L17749).

[TABLE_SHARE::keys](https://github.com/mysql/mysql-server/blob/b4523085027726e243f32a6bc855f7115c70c3b0/sql/table.h#L838)
defines the number of indices this table has.
[TABLE::key_info](https://github.com/mysql/mysql-server/blob/b4523085027726e243f32a6bc855f7115c70c3b0/sql/table.h#L1489)
points to the array of indices of this table. Note `TABLE_SHARE` also has a
field `key_info`, but they are not the same pointer. It is actually
[copied over](https://github.com/mysql/mysql-server/blob/eb0ec149d1ef371e858d59f5ae3f11da44adc3e3/sql/table.cc#L2825).

And below shows the indices of this table.

```
(lldb) p keyuse->table_ref->table->key_info[0]->name
(const char *) 0x00000001213a2cc0 "PRIMARY"
(lldb) p keyuse->table_ref->table->key_info[1]->name
(const char *) 0x00000001213a2cc8 "dept_no"
```

BTW, one thing to note is that the table definition is
[here](https://github.com/datacharmer/test_db/blob/3c7fa05e04b4c339d91a43b7029a210212d48e6c/employees.sql#L68).
Mysql automatically created an index `dept_no` for the foreign key constraint
`dept_emp_ibfk_2`, but not for `dept_emp_ibfk_1` because it smartly knows that
the primary key covers this index. See
[this post](https://dba.stackexchange.com/a/274424/254995).

> An index is generated for each FOREIGN KEY definition unless there is already
> an obviously adequate index in existence.

### Query block

Mysql code base is not easy to read because the mutual reference using
pointers. For example, `Query_block` and `JOIN` keeps a pointer to each other.

Best reference:
https://github.com/mysql/mysql-server/blob/eb86b4016060d426858cc09873d12492f1be396e/sql/query_term.h#L125

```
SELECT * FROM t1
JOIN t2 on ...
JOIN (select * from t3) t3t on ..
WHERE ...
group by ...
having ...
order by ...
```

```cpp
class Query_block : public Query_term {
...
  bool is_implicitly_grouped() const {
    return m_agg_func_used && group_list.elements == 0;
  }

...
};
```

Implicitly grouped: select max(emp_no) from employees;

```
class JOIN {
  JOIN_TAB *join_tab{nullptr};
  QEP_TAB *qep_tab{nullptr};
  JOIN_TAB **best_ref{nullptr};
};
```

`join_tab`:

`QEP_shared`

`QEP_shared_owner` seems badly implemented. Using a `shared_ptr` seems way more
cleaner.

`JOIN_TAB`

`QEP_TAB`

I frequently found that some comments in mysql codebase help me a lot to
understand its internals. Examples

- https://github.com/mysql/mysql-server/blob/d7255a34e726757df08659e5f295ac72b10a63c8/sql/sql_select.h#L352

## Useful commands

- [List long running transactions](https://blogs.oracle.com/mysql/post/mysql-80-how-to-display-long-transactions)

```
SELECT thr.processlist_id AS mysql_thread_id,
       p.thd_id,
       concat(PROCESSLIST_USER,'@',PROCESSLIST_HOST) User,
       Command,
       FORMAT_PICO_TIME(trx.timer_wait) AS trx_duration,
       current_statement as `latest_statement`
  FROM performance_schema.events_transactions_current trx
  INNER JOIN performance_schema.threads thr USING (thread_id)
  LEFT JOIN sys.processlist p ON p.thd_id=thread_id
 WHERE thr.processlist_id IS NOT NULL
   AND PROCESSLIST_USER IS NOT NULL
   AND trx.state = 'ACTIVE'
 GROUP BY thread_id, timer_wait
 ORDER BY TIMER_WAIT DESC LIMIT 20;

SELECT DATE_SUB(now(), INTERVAL (
         SELECT variable_value
           FROM performance_schema.global_status
           WHERE variable_name='UPTIME')-TIMER_START*10e-13 second) `start_time`,
       SQL_TEXT
  FROM performance_schema.events_statements_history
 WHERE nesting_event_id=(
               SELECT EVENT_ID
                 FROM performance_schema.events_transactions_current t
                 LEFT JOIN sys.processlist p ON p.thd_id=t.thread_id
                 WHERE conn_id=9251630)
 ORDER BY event_id;
```
