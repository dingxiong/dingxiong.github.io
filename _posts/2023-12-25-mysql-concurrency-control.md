---
title: Mysql concurrency control
date: 2023-12-25 21:36 -0800
categories: [database, mysql]
tags: [mysql]
---

There are only two difficult problems in the traditional RDMS domain: query
optimization and concurrency control. This blog discuss various aspects of the
latter in Mysql.

TODOs:

1. read relevant code around
   [acquire_lock](https://github.com/mysql/mysql-server/blob/4cc4db631e9b802a11646ffb814357e9f46761b2/sql/mdl.cc#L3360)

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
