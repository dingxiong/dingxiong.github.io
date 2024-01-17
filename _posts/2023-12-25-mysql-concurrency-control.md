---
title: Mysql concurrency control
date: 2023-12-25 21:36 -0800
categories: [database, mysql]
tags: [mysql]
---

There are only two difficult problems in the traditional RDBMS domain: query
optimization and concurrency control. This blog discuss various aspects of the
latter in Mysql.

TODOs:

1. read relevant code around
   [acquire_lock](https://github.com/mysql/mysql-server/blob/4cc4db631e9b802a11646ffb814357e9f46761b2/sql/mdl.cc#L3360)

## Isolation levels

The bottom of
[Wikipedia: Isolation (database systems)](<https://en.wikipedia.org/wiki/Isolation_(database_systems)>)
has a table showing the relationship between isolation levels and the
corresponding consistent problems they solve. This part is fairly easy to
understand and we all know what `dirty read`, `unrepeatable read` and
`phantom read` mean. However, these are other consistent problems other than
these three, such as `lost update`.

Lost update is another term for
[write-write conflict](https://en.wikipedia.org/wiki/Write%E2%80%93write_conflict).
By the way, `unrepeatable read` is another term for
[read-write conflict](https://en.wikipedia.org/wiki/Read%E2%80%93write_conflict)
and `dirty read` is another term for
[write-read conflict](https://en.wikipedia.org/wiki/Write%E2%80%93read_conflict).
Alibaba cloud has
[a blog](https://www.alibabacloud.com/blog/comprehensive-understanding-of-transaction-isolation-levels_596894)
talking about other consistence issue. Besides `lost update`, there are
`read skew` and `write skew`.
[A Critique of ANSI SQL Isolation Levels](https://blog.acolyer.org/2016/02/24/a-critique-of-ansi-sql-isolation-levels/)
is a must read paper for anyone who is interested in isolation levels. It
classifies the consistence phenomenon as below.

- P0: dirty write
- P1: Dirty Read
- P2: Fuzzy Read (Non-Repeatable Read)
- P3: Phantom
- P4: Lost Update
- P4C: Cursor Lost Update
- A5A: Read Skew
- A5B: Write Skew

Note, P0 is dirty write, meaning that a transaction has updated a record but
has not committed yet, and a second transaction updates the same record. Dirty
write is different lost update. See the above paper for more explanation.

By the way, all isolation levels, including read uncommitted, prohibit dirty
write by using a row lock.

Two-phase locking (2PL) is a practical way to solve these consistent problems
to achieve serializable isolation level, but it comes with a performance
penalty. That is where `MVCC` comes to rescue. However, `MVCC` only solves the
read-write and write-read conflicts. `lost update` problem is still possible
under MVCC. When I first learn about MVCC. I was very confused that MVCC still
uses locks even though it is advertised as a non-locking concurrency control
method. After reading the paper above, I understand that no matter what
concurrency protocol RDBMS implements, it needs row lock to solve the dirty
write problem. So it is not surprising to see Mysql MVCC source code still use
locks. There is a
[stackoverflow post](https://stackoverflow.com/questions/30546187/why-does-mvcc-require-locking-for-dml-statements)
asking the same question.

Mysql uses MVCC + locks to achieve serializable isolation levels. According to
the Alibaba blog above, Mysql repeatable read isolation level is vulnerable to
both phantom read and lost update issues.

### Sorry, where does transaction start?

For a long time, I thought transactions start with a `BEGIN` statement. Yes,
syntactically correct, but there are some caveat about `repeatable read`
isolation level. According to Mysql
[official doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read),

> Consistent reads within the same transaction read the snapshot established by
> the first read.

This [Chinese article](https://developer.aliyun.com/article/928196) has a clear
description about how `ReadView` is used in repeatable-read isolation level.
Relevant code is
[here](https://github.com/mysql/mysql-server/blob/7e1ce704209203da2bde727d5ce8b059d2c07c6c/storage/innobase/include/read0types.h#L162).
Also, a tiny note about `ReadView::m_ids`. This is a list of open transactions
when this ReadView is created but excludes the creator transaction itself. See
more details in function
[ReadView::copy_trx_ids](https://github.com/mysql/mysql-server/blob/76a196a0c5df52c2b09b96ccfd9ddb34da95f67e/storage/innobase/read/read0read.cc#L353).

Below is sample backtrace I got when doing the first select in a transaction.

```
(lldb) bt
* thread #40, name = 'connection', stop reason = breakpoint 1.1
  * frame #0: 0x0000000102761e84 mysqld`MVCC::view_open(this=0x00006000001c2118, view=0x00000001227a9040, trx=0x00000001227a8fa0) at read0read.cc:505:7
    frame #1: 0x00000001029403e4 mysqld`trx_assign_read_view(trx=0x00000001227a8fa0) at trx0trx.cc:2320:20
    frame #2: 0x000000010283954c mysqld`row_search_mvcc(buf="", mode=PAGE_CUR_GE, prebuilt=0x000000011ecd36b0, match_mode=1, direction=0) at row0sel.cc:4808:7
    frame #3: 0x0000000102499d0c mysqld`ha_innobase::index_read(this=0x000000011eccec30, buf="", key_ptr="\U00000011'", key_len=4, find_flag=HA_READ_KEY_EXACT) at ha_innodb.cc:10618:13
    frame #4: 0x00000001002b8cd8 mysqld`handler::index_read_map(this=0x000000011eccec30, buf="", key="\U00000011'", keypart_map=1, find_flag=HA_READ_KEY_EXACT) at handler.h:5551:12
    frame #5: 0x00000001002a4e90 mysqld`handler::ha_index_read_map(this=0x000000011eccec30, buf="", key="\U00000011'", keypart_map=1, find_flag=HA_READ_KEY_EXACT) at handler.cc:3264:3
    frame #6: 0x00000001009b3a98 mysqld`read_const(table=0x000000011ecce220, ref=0x000000011ece5e70) at sql_executor.cc:3618:30
    frame #7: 0x00000001009b3338 mysqld`join_read_const_table(tab=0x000000011eccbb90, pos=0x000000011ece5f48) at sql_executor.cc:3522:13
    frame #8: 0x0000000100a2c248 mysqld`JOIN::extract_func_dependent_tables(this=0x000000011eccb478) at sql_optimizer.cc:5807:21
    frame #9: 0x0000000100a19bc0 mysqld`JOIN::make_join_plan(this=0x000000011eccb478) at sql_optimizer.cc:5342:9
    frame #10: 0x0000000100a15f18 mysqld`JOIN::optimize(this=0x000000011eccb478, finalize_access_paths=true) at sql_optimizer.cc:700:7
    frame #11: 0x0000000100b0ae28 mysqld`Query_block::optimize(this=0x000000011ecc7578, thd=0x000000011ebfc800, finalize_access_paths=true) at sql_select.cc:2043:13
    frame #12: 0x0000000100bdcccc mysqld`Query_expression::optimize(this=0x000000011ecc7490, thd=0x000000011ebfc800, materialize_destination=0x0000000000000000, create_iterators=true, finalize_access_paths=true) at sql_union.cc:1016:22
    frame #13: 0x0000000100b07e24 mysqld`Sql_cmd_dml::execute_inner(this=0x000000011eccaad8, thd=0x000000011ebfc800) at sql_select.cc:1026:13
    frame #14: 0x0000000100b06e9c mysqld`Sql_cmd_dml::execute(this=0x000000011eccaad8, thd=0x000000011ebfc800) at sql_select.cc:792:7
    frame #15: 0x0000000100a5ab90 mysqld`mysql_execute_command(thd=0x000000011ebfc800, first_level=true) at sql_parse.cc:4824:29
    frame #16: 0x0000000100a5248c mysqld`dispatch_sql_command(thd=0x000000011ebfc800, parser_state=0x0000000171759748) at sql_parse.cc:5479:19
    frame #17: 0x0000000100a4e19c mysqld`dispatch_command(thd=0x000000011ebfc800, com_data=0x000000017175ae30, command=COM_QUERY) at sql_parse.cc:2136:7
    frame #18: 0x0000000100a50a14 mysqld`do_command(thd=0x000000011ebfc800) at sql_parse.cc:1465:18
    frame #19: 0x0000000100d3c1b0 mysqld`handle_connection(arg=0x0000600002befd20) at connection_handler_per_thread.cc:303:13
    frame #20: 0x0000000102ee023c mysqld`pfs_spawn_thread(arg=0x000000011e7c31d0) at pfs.cc:3049:3
    frame #21: 0x000000018c4a2034 libsystem_pthread.dylib`_pthread_start + 136
```

OK, so with this knowledge, I have an interesting idea that we can use
repeatable-read isolation level and `select ... for update` clause to build a
serializable isolation level. The pseudo code is like

```
BEGIN;
SELECT * from t1 where id = 1 for update;
....
COMMIT;
```

We do a `select ... for update` every time we start a new transaction. There
are two implications:

1. Because the lock associated with `SELECT ... for update` is release after
   the transaction ends, so all concurrent transactions are actually executed
   in a sequential order.
2. Because this `SELECT ... for update` is the first read inside the
   transaction, so it establishes the snapshot to use.

Combining #1 and #2, we can conclude that transactions are executed in a
serializable order, and in this transaction chain, early transaction is visible
to later transactions.

## Transaction

Transactions are handled by storage engine. For InnoDB,

```
------------
TRANSACTIONS
------------
Trx id counter 8722
Purge done for trx's n:o < 8711 undo n:o < 0 state: running but idle
History list length 0
Total number of lock structs in row lock hash table 2
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 281480531941216, not started
0 lock struct(s), heap size 1192, 0 row lock(s)
---TRANSACTION 281480531938240, not started
0 lock struct(s), heap size 1192, 0 row lock(s)
---TRANSACTION 281480531937248, not started
0 lock struct(s), heap size 1192, 0 row lock(s)
---TRANSACTION 8721, ACTIVE 114 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1192, 2 row lock(s)
MySQL thread id 16, OS thread handle 6199619584, query id 282 localhost 127.0.0.1 root updating
update employees set hire_date = '1986-06-27' where emp_no = 10001
Trx read view will not see trx with id >= 8721, sees < 8720
------- TRX HAS BEEN WAITING 19 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 9 page no 5 n bits 408 index PRIMARY of table `employees`.`employees` trx id 8721 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 8; compact format; info bits 0
 0: len 4; hex 80002711; asc   ' ;;
 1: len 6; hex 000000002210; asc     " ;;
 2: len 7; hex 01000001d40151; asc       Q;;
```

### Two-phase commit

You may wonder what `xa` means in Mysql code base. It means
[extended architecuture](https://en.wikipedia.org/wiki/X/Open_XA).

## Misc

Error messages

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

They are defined in file
[messages_to_clients.txt](https://github.com/mysql/mysql-server/blob/dd9104fd63727e4d23239a571a8f61282cbbbdeb/share/messages_to_clients.txt#L0-L1).
This file is read by a utility file
[utilities/comp_err.cc](https://github.com/mysql/mysql-server/blob/b845ba26c825d8cf124b76c9738e88a9b0251eb0/utilities/comp_err.cc#L81)
and output to a lot of `errmsg.sys` files under different locale folder

```bash
2023-12-26 22:26:13 (bash) ~/code/mysql-server/build/share
$ tree
.
├── CMakeFiles
│   ├── CMakeDirectoryInformation.cmake
│   └── progress.marks
├── CTestTestfile.cmake
├── Makefile
├── bulgarian
│   └── errmsg.sys
├── cmake_install.cmake
├── czech
│   └── errmsg.sys
├── danish
│   └── errmsg.sys
├── dictionary.txt
├── dutch
│   └── errmsg.sys
├── english
│   └── errmsg.sys
├── estonian
│   └── errmsg.sys
├── french
│   └── errmsg.sys
├── german
│   └── errmsg.sys
├── greek
│   └── errmsg.sys
├── hungarian
│   └── errmsg.sys
├── italian
│   └── errmsg.sys
├── japanese
│   └── errmsg.sys
├── korean
│   └── errmsg.sys
├── norwegian
│   └── errmsg.sys
├── norwegian-ny
│   └── errmsg.sys
├── polish
│   └── errmsg.sys
├── portuguese
│   └── errmsg.sys
├── romanian
│   └── errmsg.sys
├── russian
│   └── errmsg.sys
├── serbian
│   └── errmsg.sys
├── slovak
│   └── errmsg.sys
├── spanish
│   └── errmsg.sys
├── swedish
```

Finally, function
[init_errmessage](https://github.com/mysql/mysql-server/blob/4cc4db631e9b802a11646ffb814357e9f46761b2/sql/derror.cc#L181)
load them.

### Some default values

- page size:
  [16KB](https://github.com/mysql/mysql-server/blob/9782dcbfc805a405cfefcbf1354e802dadfa2b33/storage/innobase/include/univ.i#L315).

## Update

I just came across this
[line](https://github.com/mysql/mysql-server/blob/7e1ce704209203da2bde727d5ce8b059d2c07c6c/sql/sql_update.h#L149).
Basically, it says we can update multiple rows in a single query, such as

```sql
UPDATE Books, Orders
SET Orders.Quantity = Orders.Quantity + 2,
    Books.InStock = Books.InStock - 2
WHERE
    Books.BookID = Orders.BookID
    AND Orders.OrderID = 1002;
```

Quite amazing!
