---
layout: post
title: Postgres -- Getting Started
date: 2024-05-16 14:26 -0700
categories: [database, postgres]
tags: [postgres]
---

Postgres is wonderful piece of software!

## Set up Postgres

Refer to <https://www.postgresql.org/docs/devel/installation.html> to build
from source.

```bash
mkdir build_dir
cd build_dir
../configure --without-icu
bear -- make -j6
cd .. && ln -s build_dir/compile_commands.json compile_commands.json
```

Postgres documents the progress of creating and starting a postgres database
[in detail](https://www.postgresql.org/docs/current/runtime.html). Basically,
it provides `initdb` and `postgres` commands to initialize data directory and
start a postmaster process. It also mentions the wrapper program `pg_ctl` which
can be used instead of running `initdb` or `postgres` directly. The source code
is
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/bin/pg_ctl/pg_ctl.c#L2459).
I am a little surprised by this wrapper because it uses
[system(cmd)](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/bin/pg_ctl/pg_ctl.c#L915)
or
[/bin/sh](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/bin/pg_ctl/pg_ctl.c#L496)
to run the wrapped command. Why not just call the relevant entry functions of
the wrapped command?

Things becomes a little more convoluted in Debian and Redhat. When you run
`apt install postgres`, it also installs a dependency
[postgresql-common](https://github.com/credativ/postgresql-common). Basically,
it is a wrapper on top of `pg_ctl` and enables us to manage different postgres
cluster wit the same or different versions. How does this shim layer work?
Let's see a few postgres commands.

```
> ll /usr/bin/createuser
lrwxrwxrwx 1 root root 37 Oct  2  2023 /usr/bin/createuser -> ../share/postgresql-common/pg_wrapper

> ll /usr/bin/psql
lrwxrwxrwx 1 root root 37 Oct  2  2023 /usr/bin/psql -> ../share/postgresql-common/pg_wrapper
```

So `gp_wrapper` is the shim layer. It is a Perl file that routes the original
command to the binary under
[/usr/lib/postgresql/_version_/bin](https://github.com/credativ/postgresql-common/blob/55cb147cff7d2d764c7733ecb8a220378cb83fb5/PgCommon.pm#L629).

Finally, we need some sample data.
[sakila](https://github.com/jOOQ/sakila/tree/main) is a popular data set.

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

### Configurations

The main configuration file is `postgresql.conf`. I thought Postgres reads it
as a plain text file and splits each line to a key value pair. Also, a line is
a comment if it starts with `#`. No way! Postgres always shows off in
unexpected ways. The configuration grammar is defined using
[Lex & Yacc](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/utils/misc/guc-file.l#L94)!

```
\n				ConfigFileLineno++; return GUC_EOL;
[ \t\r]+		/* eat whitespace */
#.*				/* eat comment (.* matches anything until newline) */

{ID}			return GUC_ID;
{QUALIFIED_ID}	return GUC_QUALIFIED_ID;
{STRING}		return GUC_STRING;
{UNQUOTED_STRING} return GUC_UNQUOTED_STRING;
{INTEGER}		return GUC_INTEGER;
{REAL}			return GUC_REAL;
=				return GUC_EQUALS;

.				return GUC_ERROR;

```

Integer, real number and string are recognized. "=" is parsed as equal token.
White spaces including tab and `\r` are ignored. Anything after `#` is
considered as comments and is ignored.

What if the same key shows up multiple times? The
[documentation](https://www.postgresql.org/docs/current/config-setting.html)
clearly says

> If the file contains multiple entries for the same parameter, all but the
> last one are ignored.

The code is
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/utils/misc/guc.c#L405).
It is funny that all of them are kept, but all except the last one have
`ignore=True`.

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

## Index

`CREATE INDEX` supports parallel processing starting from
[version 11.0](https://www.postgresql.org/docs/release/11.0/). This
[blog](https://www.cybertec-postgresql.com/en/postgresql-parallel-create-index-for-better-performance/)
shares performance improvement using this new feature. Configurations relevant
to this feature are `maintenance_work_mem` and
`max_parallel_maintenance_workers`.

How many processes are created during index creation? The entry point is
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/catalog/index.c#L2941)
and the call stack is

```
index_buid
  -> plan_create_index_workers
    -> compute_parallel_worker
```

Let's sort out the details. First, if the `parallel_workers` storage option is
set for the table, then the umber of worker process is just
`min(table.parallel_workers, max_parallel_maintenance_workers)`. If not, then
we get the number of pages of this table, and set
`#worker = log_3(#pages / min_parallel_table_scan_size)`. We can get the number
of pages from `pg_class.relpages`. Also, `min_parallel_table_scan_size` has
default value is `8MB`. Note that this parameter is stored as number of
[block size](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/utils/misc/guc_tables.c#L3511)
internally, so we can use the table storage size `pg_total_relation_size()` to
calculate the number of workers. For example, if table storage is less than
`3*8MB`, then no workers. If table storage is in range `[3*8, 3^2*8]MB`, then
one worker. It is very simple math. But this is not the end. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/optimizer/plan/planner.c#L6807).

This is where `maintenance_work_mem` plays a role. The main process and all
worker processes share a total `maintenance_work_mem` when scanning table and
doing tuple sort. The default value `maintenance_work_mem` is 64MB. Suppose we
have 3 workers, then each process can get 64/4=16MB memory. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsort.c#L1720)
and
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsort.c#L1827).
To make sure each process has at least 32MB, we cannot have more than 2
processes. That is one main process and one worker process. Personally, I feel
we should set `maintenance_work_mem` to at least 1GB.
