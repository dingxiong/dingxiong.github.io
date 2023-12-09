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

## post installation

There are
[a few steps](https://dev.mysql.com/doc/refman/8.0/en/postinstallation.html) to
follow after a fresh installation.

### Create mysql user and group

```
$ groupadd mysql
$ useradd -r -g mysql -s /bin/false mysql
```

### Initialize the data directory

Suppose the build folder is `/code/mysql-server/build`.

```
cd build
rm -rf data && mkdir data
bin/mysqld --initialize --user=mysql
```

The above command will print a temp root password in console.

### Start server

```
bin/mysqld --user=mysql
```

### Change root password

Open another terminal,

```
bin/mysql -u root -p
```

Type the temp password and then assign a new root password.

```
mysql>  ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
```

### Create example database and tables

```
CREATE DATABASE demodb;

USE demodb;

CREATE TABLE test_example_1 (
    id bigint NOT NULL,
    last_name varchar(255) NOT NULL,
    first_name varchar(255),
    age int,
    PRIMARY KEY (id),
    INDEX index_name (first_name, last_name)
);

insert into test_example_1 (id, last_name, first_name, age) values (1, 'x1', 'y1', 18);
insert into test_example_1 (id, last_name, first_name, age) values (2, 'x2', 'y2', 18);
```

## Run in gdb

```
gdb --args bin/mysql --user=mysql
(gdb) run&
```

Here, we added `&` to the run command because the main thread is daemon thread,
and it waits forever. We want to check the child threads. When a client
connects to the server, gdb will show a message like below

```
(gdb) [New Thread 0xffffe067edc0 (LWP 51367)]
```

# Name conventions

A lot of files/functions start with `ut_`, which stands for `utility`.

# MISC

See a lot of `#ifndef UNIV_HOTBACKUP` in the codebase. This flag is never set.
See more https://jira.mariadb.org/browse/MDEV-11690 .

# InnoDB

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

## Useful commands

```
show global variables like '%...%'
show global status like '%..%'
```

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
