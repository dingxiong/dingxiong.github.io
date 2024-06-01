---
layout: post
title: Postgres -- Getting Started
date: 2024-05-16 14:26 -0700
categories: [database, postgres]
tags: [postgres]
---

Refer to <https://www.postgresql.org/docs/devel/installation.html> to build
from source.

```bash
mkdir build_dir
cd build_dir
../configure --without-icu
bear -- make -j6
cd .. && ln -s build_dir/compile_commands.json compile_commands.json
```

We need some sample data. [sakila](https://github.com/jOOQ/sakila/tree/main) is
a popular data set.

```
git clone git@github.com:jOOQ/sakila.git --depth=1
psql -d test -f ~/code/sakila/postgres-sakila-db/postgres-sakila-schema.sql
psql -d test -f ~/code/sakila/postgres-sakila-db/postgres-sakila-insert-data.sql
psql test

test=# insert into actor (first_name, last_name) values ('x', 'y'), ('z', 'w') returning actor_id, last_update;
 actor_id |        last_update
----------+----------------------------
      204 | 2024-06-01 00:16:58.206651
      205 | 2024-06-01 00:16:58.206651
(2 rows)

INSERT 0 2
```

## binary file

Postgres only supports importing data from text, csv and binary files. The
binary protocol is covered briefly
[here](https://www.postgresql.org/docs/current/sql-copy.html). Basically,

1. Each row starts with a 16-bit integer denoting the number of fields in this
   row.
2. Following this 16-bit integer is a sequence of fields. Each field starts
   with a 32-bit integer denoting the size of the field, and then is followed
   by the binary data of this field.

See code
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/copyfromparse.c#L1015).
The (de)serialization protocol for each data type is defined by the `_send` and
`_recv` functions inside the `adt` folder, i.e., Abstract Data Types. See
[timestamp example](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/utils/adt/timestamp.c#L258).
Basically, timestamp is just a uint64 that stores the microseconds from
[postgres epoch date](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/datatype/timestamp.h#L235),
i.e., 2000-01-01. Note, it is NOT Unix epoch date. Below is an part of binary
postgres data file.

```
00000560  3a 20 31 7d 00 00 00 04  00 00 00 00 00 00 00 08  |: 1}............|
00000570  00 02 4d 04 9e 82 c7 00  00 00 00 08 00 02 ae 50  |..M............P|
00000580  b9 9b 4a 00 ff ff ff ff  ff ff ff ff ff ff ff ff  |..J.............|
```

The `00 00 00 08` in the first line says the next field has 8 bytes. The value
follows is
`00 02 4d 04 9e 82 c7 00 = 647632188000000 (ms) = 1990 5:49:48 PM Unix epoch time = 2020 5:49:48 PM Postgres epoch time`.
The `00 00 00 08` in the second line designates another timestamp. Note, there
are a lot `ff` bytes follows. These `-1` values denote NULL values.

## Toast storage

TODO: learn it

## pgloader

`pgloader` is a great tool to migrate data from another database to postgres.
Using Mysql -> postgres as an example, the main logic is

```
process-source-and-target (main.lisp#main)
  -> load-data (api.lisp#process-source-and-target)
    -> lisp-code-for-loading-from-mysql (api.lisp#load-data)
      -> copy-database (command-mysql.lisp#lisp-code-for-loading-from-mysql)
        -> copy-from (migrate-database.lisp#copy-database)
          -> queue-raw-data (copy-data.lisp#copy-from)
            -> map-rows (copy-data.lisp#queue-raw-data)
              -> (mysql.lisp#map-rows)
```

There are lots of things going along the chain. First, it takes a
producer-consumer model using channels provided by the `lparallel` package,
which is every similar to Golang. It is true that Lisp is a multi-paradigm
programming language. Second, the `map-rows`
[implementation](https://github.com/dimitri/pgloader/blob/2079646c81f565b5e9edba627d14cbf63af2dbdd/src/sources/mysql/mysql.lisp#L120)
inside `mysql.lisp` actually uses batching. The batch size is controlled by
configuration parameter
[rows-per-range](https://github.com/dimitri/pgloader/blob/2079646c81f565b5e9edba627d14cbf63af2dbdd/src/sources/mysql/mysql.lisp#L46).
So we are sure pgloader is not stupid to cause obvious OOM. Third, pgload has
configuration parameter controlling whether we only want to copy data or also
need to create tables beforehand. See
[doc](https://github.com/dimitri/pgloader/blob/2079646c81f565b5e9edba627d14cbf63af2dbdd/docs/index.rst#L144).
Last, data is copied as binary as described in the above binary protocol. The
code is
[here](https://github.com/dimitri/pgloader/blob/2079646c81f565b5e9edba627d14cbf63af2dbdd/src/pg-copy/copy-format.lisp#L15-L16).
