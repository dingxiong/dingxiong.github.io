---
layout: post
title: Postgres -- Getting Started
date: 2024-05-16 14:26 -0700
categories: [database, postgres]
tags: [postgres]
---

Postgres is wonderful piece of software!

## Set Up Postgres

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

### MacOS

Just use homebrew to install Postgres.

### Linux

```bash
# Install the default version
sudo apt-get intall postgres

# Install a newer version
sudo apt-get install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y
sudo apt-get install -y postgresql-16
```

Things becomes a little more convoluted in Debian and Redhat. When you run
`apt install postgres`, it also installs a dependency
[postgresql-common](https://github.com/credativ/postgresql-common). Basically,
it is a wrapper on top of `pg_ctl` and enables us to manage different postgres
cluster with the same or different versions. How does this shim layer work?
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

### Build From Source

Refer to <https://www.postgresql.org/docs/devel/installation.html> to build
from source.

```bash
mkdir build_dir
cd build_dir

# We need to provide installation prefix because the binary generated
# reference symbols in the default installation folder.
../configure --without-icu --prefix $HOME/code/postgres/build_dir/install_dir
# or for debug build
../configure --without-icu --enable-cassert --enable-debug CFLAGS="-ggdb -Og -O0 -g3 -fno-omit-frame-pointer" -prefix $HOME/code/postgres/build_dir/install_dir

# make
bear -- make -j6
cd .. && ln -s build_dir/compile_commands.json compile_commands.json
cd build_dir

# install
make install

# create a cluster
mkdir data
./install_dir/bin/initdb -D $HOME/code/postgres/build_dir/data

# start server using a different port
./install_dir/bin/postgres -D $HOME/code/postgres/build_dir/data -p 5433

# connect to sever and create test database
./install_dir/bin/psql postgres -p 5433
postgres=# create database test2;
```

After the first successful compilation, you probably only incremental
compilation afterward.

```
cd build_dir
make && make install
```

### Sample Data

Finally, we need some sample data.
[sakila](https://github.com/jOOQ/sakila/tree/main) is a popular data set.

```
git clone git@github.com:jOOQ/sakila.git --depth=1
psql -d test -f ~/code/sakila/postgres-sakila-db/postgres-sakila-schema.sql
psql -d test -f ~/code/sakila/postgres-sakila-db/postgres-sakila-insert-data.sql
psql test
```

### Debugger

https://github.com/tvondra/gdbpg

```
$ lldb -- ./install_dir/bin/postgres -D $HOME/code/postgres/build_dir/data -p 5433
(lldb) b sql/sql_optimizer.cc:5875
Breakpoint 1: where = mysqld`JOIN::estimate_rowcount() + 44 at sql_optimizer.cc:5875:37, address = 0x0000000100a2c4d8
(lldb) process launch -n
Process 4071 launched: '/Users/xiongding/code/mysql-server/build/bin/mysqld' (arm64)
```

I haven't make it work under debugger because some signal handler issue. Life
is short. Do not want to spend time making it work. Instead, there is a simple
trick to inspect internal states. It has a pretty print function that can print
out any node.

```
#include "nodes/print.h"
pprint(transform);
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

### How Are Data Types Serialized To Pages?

```
=# select oid, typname, typlen, typbyval, typalign, typstorage from pg_type where typname in ('varchar', 'int2', 'int4', 'char', 'bool', 'bytea', 'float8', 'text', 'uuid', 'timestamp', 'timestamptz', 'date');
 oid  |   typname   | typlen | typbyval | typalign | typstorage
------+-------------+--------+----------+----------+------------
   16 | bool        |      1 | t        | c        | p
   17 | bytea       |     -1 | f        | i        | x
   18 | char        |      1 | t        | c        | p
   21 | int2        |      2 | t        | s        | p
   23 | int4        |      4 | t        | i        | p
   25 | text        |     -1 | f        | i        | x
  701 | float8      |      8 | t        | d        | p
 1043 | varchar     |     -1 | f        | i        | x
 1082 | date        |      4 | t        | i        | p
 1114 | timestamp   |      8 | t        | d        | p
 1184 | timestamptz |      8 | t        | d        | p
 2950 | uuid        |     16 | f        | c        | p
```

https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/varatt.h#L142

https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/common/indextuple.c#L65

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

On the left side, the ItemIdData array has pointers
`[0x3fe0, 0x3fc0, 0x3fb0, 0x3fa0, 0x3fd0]`, which correspond to
`[Alice, Charlie, Diana, Edward, blob]`. On the right side, the ItemIdData
array has pointers `[0x3fe0, 0x3fd0, 0x3fc0, 0x3fb0, 0x3fa0]`, which correspond
to `[Alice, blob, Charlie, Diana, Edward]`. So the right side sorts tuples
case-insensitively.

### How Does Collation Affect Query Performance

Using the example above, what is the performance of case-insensitive query?

For `my_table_2`,

```
test2=# explain select * from my_table_2 where name = 'alice';
                               QUERY PLAN
-------------------------------------------------------------------------
 Bitmap Heap Scan on my_table_2  (cost=4.20..13.67 rows=6 width=40)
   Recheck Cond: (name = 'blob'::citext)
   ->  Bitmap Index Scan on idx_name_2  (cost=0.00..4.20 rows=6 width=0)
         Index Cond: (name = 'blob'::citext)
```

For `my_table`, we need to manually add the `lower` function.

```
test2=# explain select * from my_table where lower(name) = 'bob';
                        QUERY PLAN
-----------------------------------------------------------
 Seq Scan on my_table  (cost=0.00..12.10 rows=1 width=524)
   Filter: (lower((name)::text) = 'blob'::text)
```

You can see that with `lower`, Postgres does not use the name index. How to
make this query efficient? We can add an index

```
CREATE INDEX idx_name_lower on my_table (lower(name));
```

However, this change makes no difference probably because this table only has 5
tuples. Let's see a real life example with `object_column_index`.

```
admincoin=# \d object_column_index
                                    Table "public.object_column_index"
      Column       |            Type             | Collation | Nullable |             Default
-------------------+-----------------------------+-----------+----------+----------------------------------
 boolean_value     | boolean                     |           |          |
 column_name       | character varying(100)      |           |          |
 datetime_value    | timestamp(0) with time zone |           |          |
 float_value       | double precision            |           |          |
 id                | integer                     |           | not null | generated by default as identity
 int_value         | integer                     |           |          |
 long_value        | bigint                      |           |          |
 object_guid       | character varying(36)       |           | not null |
 object_type       | character varying(48)       |           | not null |
 organization_guid | character varying(36)       |           |          |
 string_value      | character varying(200)      |           |          |
Indexes:
    "object_column_index_pkey" PRIMARY KEY, btree (id)
    "idx_on_boolean_column" btree (object_type, column_name, organization_guid, boolean_value, object_guid)
    "idx_on_column_of_object" btree (object_guid, column_name)
    "idx_on_datetime_column" btree (object_type, column_name, organization_guid, datetime_value, object_guid)
    "idx_on_float_column" btree (object_type, column_name, organization_guid, float_value, object_guid)
    "idx_on_int_column" btree (object_type, column_name, organization_guid, int_value, object_guid)
    "idx_on_string_column" btree (object_type, column_name, organization_guid, string_value, object_guid)
    "uq_object_guid_and_column_name" UNIQUE, btree (object_guid, column_name)
```

A case-sensitive query

```
admincoin=# explain select * from object_column_index where object_type = 'a' and column_name = 'b' and organization_guid = 'c' and string_value  = 'alice';
-[ RECORD 1 ]------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Index Scan using idx_on_string_column on object_column_index  (cost=0.69..8.71 rows=1 width=151)
-[ RECORD 2 ]------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   Index Cond: (((object_type)::text = 'a'::text) AND ((column_name)::text = 'b'::text) AND ((organization_guid)::text = 'c'::text) AND ((string_value)::text = 'alice'::text))
```

A case-insensitive query

```
admincoin=# explain select * from object_column_index where object_type = 'a' and column_name = 'b' and organization_guid = 'c' and lower(string_value)  = 'alice';
-[ RECORD 1 ]-----------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Index Scan using idx_on_string_column on object_column_index  (cost=0.69..8.72 rows=1 width=151)
-[ RECORD 2 ]-----------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   Index Cond: (((object_type)::text = 'a'::text) AND ((column_name)::text = 'b'::text) AND ((organization_guid)::text = 'c'::text))
-[ RECORD 3 ]-----------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   Filter: (lower((string_value)::text) = 'alice'::text)
```

You see the additional filter stage. Let's add a lower index

```
create index idx_on_string_column_lower on object_column_index (object_type, column_name, organization_guid, lower(string_value), object_guid);
```

Then

```
admincoin=# explain select * from object_column_index where object_type = 'a' and column_name = 'b' and organization_guid = 'c' and lower(string_value)  = 'alice';
-[ RECORD 1 ]-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN | Index Scan using idx_on_string_column_lower on object_column_index  (cost=0.69..8.71 rows=1 width=151)
-[ RECORD 2 ]-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
QUERY PLAN |   Index Cond: (((object_type)::text = 'a'::text) AND ((column_name)::text = 'b'::text) AND ((organization_guid)::text = 'c'::text) AND (lower((string_value)::text) = 'alice'::text))
```

So it indeed uses the lower index!

## Replication

Postgres has a concept of logical replication. See these two chapters:

1. <https://www.postgresql.org/docs/current/logicaldecoding.html>
2. <https://www.postgresql.org/docs/current/logical-replication.html>
