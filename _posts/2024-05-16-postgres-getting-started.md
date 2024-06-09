---
layout: post
title: Postgres -- Getting Started
date: 2024-05-16 14:26 -0700
categories: [database, postgres]
tags: [postgres]
---

Postgres is wonderful piece of software!

## Set Up Postgres

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

## Various Binaries

### Import From Binary File

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

### Page Layout

Postgres has a good documentation about
[Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html).
The diagram in the
[source code comment](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/storage/bufpage.h#L29)
is super helpful too. Let's look at an example.

First, create the table and insert sample data.

```
CREATE TABLE my_table(
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    age INT
);

CREATE INDEX idx_name ON my_table (name);

INSERT INTO my_table (name, age) VALUES
('Alice', 30),
('bob', 25),
('Charlie', 35),
('Diana', 28),
('Edward', 40);
```

Second, Run `CHECKPOINT` and restart Postgres. Why? The DDL/DML changes above
are written to WAL, but WAL is not merged into the data files yet. The merging
process is done periodically and the frequency is controlled by some
configuration parameters. To manually trigger this process, we can create a
checkpoint and then restart Postgres such that it merges all changes before
this checkpoint to the data files. The corresponding data file is below

```
> sudo hexdump -C  /var/lib/postgresql/13/main/base/1638340/2481266
00000000  76 01 00 00 b0 f3 03 7f  00 00 00 00 2c 00 38 1f  |v...........,.8.|
00000010  00 20 04 20 00 00 00 00  d8 9f 50 00 b0 9f 48 00  |. . ......P...H.|
00000020  88 9f 50 00 60 9f 50 00  38 9f 50 00 00 00 00 00  |..P.`.P.8.P.....|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001f30  00 00 00 00 00 00 00 00  4d e3 07 00 00 00 00 00  |........M.......|
00001f40  00 00 00 00 00 00 00 00  05 00 03 00 02 09 18 00  |................|
00001f50  05 00 00 00 0f 45 64 77  61 72 64 00 28 00 00 00  |.....Edward.(...|
00001f60  4d e3 07 00 00 00 00 00  00 00 00 00 00 00 00 00  |M...............|
00001f70  04 00 03 00 02 09 18 00  04 00 00 00 0d 44 69 61  |.............Dia|
00001f80  6e 61 00 00 1c 00 00 00  4d e3 07 00 00 00 00 00  |na......M.......|
00001f90  00 00 00 00 00 00 00 00  03 00 03 00 02 09 18 00  |................|
00001fa0  03 00 00 00 11 43 68 61  72 6c 69 65 23 00 00 00  |.....Charlie#...|
00001fb0  4d e3 07 00 00 00 00 00  00 00 00 00 00 00 00 00  |M...............|
00001fc0  02 00 03 00 02 09 18 00  02 00 00 00 09 62 6f 62  |.............bob|
00001fd0  19 00 00 00 00 00 00 00  4d e3 07 00 00 00 00 00  |........M.......|
00001fe0  00 00 00 00 00 00 00 00  01 00 03 00 02 09 18 00  |................|
00001ff0  01 00 00 00 0d 41 6c 69  63 65 00 00 1e 00 00 00  |.....Alice......|
00002000
```

The first 24 bytes are `PageHeaderData`. You can see that

```
pd_lower = 0x002c
pd_upper = 0x1f38
pd_special = 0x2000
pd_pagesize = 0x2004 && 0xFF00 = 0x2000 = 8192
version = 0x2004 && 0x00FF = 4
```

Note that my MacBook M1, namely, AArch64, is a little-endian CPU, so we
interpret `2c 00` as `0x002c`. As you can see, the default page size in
Postgres is 8KB. Also this is the
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/storage/bufpage.h#L274)
to split page size and version.

The bytes between the end of `PageHeaderData` and `pd_lower` are
[ItemIdData](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/storage/itemid.h#L25).
They are pointers to the data tuples. The size of this part is
`0x002c - 24 = 20` bytes. The content is extracted as follows.

```
> sudo hexdump -C -s 24 -n 20 /var/lib/postgresql/13/main/base/1638340/2481266
00000018  d8 9f 50 00 b0 9f 48 00  88 9f 50 00 60 9f 50 00  |..P...H...P.`.P.|
00000028  38 9f 50 00                                       |8.P.|
```

Each ItemIdData takes four bytes, so we have five items in total corresponding
to the five tuples inserted. Let's take a look at one example `d8 9f 50 00`.
The definition of ItemIdData is copied below

```
typedef struct ItemIdData
{
  unsigned  lp_off:15,		/* offset to tuple (from start of page) */
            lp_flags:2,		/* state of line pointer, see below */
            lp_len:15;		/* byte length of tuple */
} ItemIdData;
```

How to only get the first 15 bits of the first 2 bytes? Actually, I am lost for
this niche detail. This is how I make sense of these bits: `d8 9f` should be
thought as `0x9fd8`, and `9 = 1001`, so I strip off the first bit of `9` to get
`0x1fd8`. For size `0x0050 = 1010000`, `5 = 101`, and I strip off the last bit
to get `100`, so the reals size is `0x0040`. So the first entry starts at
`0x1fd8` and takes 40 bytes.

```
> sudo hexdump -C -s 0x1f38 -n 40 /var/lib/postgresql/13/main/base/1638340/2481266
00001f38  4d e3 07 00 00 00 00 00  00 00 00 00 00 00 00 00  |M...............|
00001f48  05 00 03 00 02 09 18 00  05 00 00 00 0f 45 64 77  |.............Edw|
00001f58  61 72 64 00 28 00 00 00                           |ard.(...|
```

You can get the rest of the tuples. Forgive me! I need to read more about this
detail.

### Toast Storage

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
index_build
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

This is where `maintenance_work_mem` plays a role. The leader process and all
worker processes share a total `maintenance_work_mem` when scanning table and
doing tuple sort. The default value `maintenance_work_mem` is 64MB. Suppose we
have 3 workers, then each process can get 64/4=16MB memory. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsort.c#L1720)
and
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsort.c#L1827).
To make sure each process has at least 32MB, we cannot have more than 2
processes. That is one leader process and one worker process. Personally, I
think we should set `maintenance_work_mem` to at least 1GB.

One final mark about index_build process. I am curious to know how Postgres
forks these worker processes and then join them. What I found is this
[line](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsort.c#L1423).
Postgres builds some wrapper on top of the fork system call. After creating
this `ParallelContext`, it calls
[LaunchParallelWorkers(pcxt)](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsort.c#L1570)
and
[WaitForParallelWorkersToAttach(pcxt)](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsort.c#L1600).
All these makes sense. However, as you can see, it passes the function as a
string when creating the parallel context. So how does it look up the real
function by this string? The answer is
[LookupParallelWorkerFunction](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/transam/parallel.c#L1595).
If first tries to find the function from a dictionary
`InternalParallelWorkers`. If not found, then it calls `dlsym` to look up the
function from some shared library. I am shocked!

## Locale

Let's talk about locale in general before jumping into locales in Postgres.

We only talk about POSIX compliant systems. First, what is the locale of my
MacOS?

```
$ locale
LANG="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_CTYPE="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_ALL=
```

Locale has
[format](https://docs.oracle.com/cd/E23824_01/html/E26033/glmbx.html)

```
language[_territory][.codeset][@modifier]
```

In my case, language is `en`, territory or country is United States. `codeset`
is `UTF-8`. There is no modifier. `LC_COLLATE`, `LC_CTYPE` and etc are locale
categories. They specify various cultural conventions/behaviors of characters,
strings, time format, money format, and etc. Out of them, `LC_COLLATE` and
`LC_CTYPE` are most relevant to Postgres.

- `LC_COLLATE`: specifies the collation order. It controls how strings are
  compared and sorted.
- `LC_CTYPE`: `C` here stands for character. It specifies character
  classification and case conversion. For example, whether a character is digit
  or not? what is the corresponding upper case letter of a character?

In postgres, `LC_COLLATE` and `LC_CTYPE` are fixed once a database is created.
To check the current values,

```
admincoin=# show LC_COLLATE;
-[ RECORD 1 ]-------
lc_collate | C.UTF-8

admincoin=# show LC_CTYPE;
-[ RECORD 1 ]-----
lc_ctype | C.UTF-8
```

Note, somehow `show LC_COLLATE` does not work in Aurora Postgres. We can
directly query `pg_database`:

```
admincoin=# select datcollate, datctype from pg_database where datname = 'test';
-[ RECORD 1 ]-------
datcollate | C.UTF-8
datctype   | C.UTF-8
```

### How Does Collation Affect Index Layout?

As you can imagine, collation affects how tuples are compared it, so it affect
the page layout of indices. We can do a quick experiment. I am lazy to make a
new collation, but instead I choose to use type `CITEXT` to illustrate the
idea. `CITEXT` is case-insensitive text date type. It is equivalent to a
case-insensitive collation.

First, set up the schemas and data.

```
CREATE TABLE my_table(
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    age INT
);
CREATE INDEX idx_name ON my_table (name);

CREATE EXTENSION IF NOT EXISTS citext;
CREATE TABLE my_table_2(
    id SERIAL PRIMARY KEY,
    name CITEXT NOT NULL,
    age INT
);
CREATE INDEX idx_name_2 ON my_table_2 (name);

INSERT INTO my_table (name, age) VALUES
('Alice', 30),
('bob', 25),
('Charlie', 35),
('Diana', 28),
('Edward', 40);

INSERT INTO my_table_2 (name, age) VALUES
('Alice', 30),
('bob', 25),
('Charlie', 35),
('Diana', 28),
('Edward', 40);
```

You can see that I deliberately make `bob` all lower cases. Second, do a
`CHECKPOINT` and restart postgres. Let's check the data file content.

![citext_diff](/assets/images/pg_citex_bin_diff.png)

The left side is the data file of `idx_name` and the right side is the date
file of `idx_name_2`. There are two pages for each case. The first page is
special. It is called
[meta page](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/access/nbtree.h#L97).
This meta page contains information about the page number of the btree root,
the height of the btree, and etc. In our case, the root page number is 1 and
height is 1. It makes sense as there are only 5 tuples. The root page itself
can hold all of them.

Let's focus on the second page. The row `00003ff0` corresponds to
[BTPageOpaqueData](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/access/nbtree.h#L62).
The flag field is `03` meaning this is the root page and also a leaf page. Five
tuples are inserted to a page backwardly starting from larger offset. That is
why you see the names are from `Edward` to `Alice`. Let's check the difference
of the 2nd page for these two indices. Note, inside an index page, `ItemIdData`
array is sorted corresponding to the tuples they point to. This order is the
prerequisite that you can do a binary search inside a btree node. Whenever you
insert a new tuple in this page, db should maintain this invariant. You can see
[more detail](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/nbtree/nbtsearch.c#L337)
about how Postgres does binary search on this `ItemIdData` array.
