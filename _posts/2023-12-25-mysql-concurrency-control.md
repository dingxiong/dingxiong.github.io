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
