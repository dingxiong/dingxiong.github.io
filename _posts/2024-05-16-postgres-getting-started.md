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

## binary file

Postgres only supports importing data from text, csv and binary files. The
binary protocol is covered briefly
[here](https://www.postgresql.org/docs/current/sql-copy.html). The
(de)serialization protocol for each data type is defined by the `_send` and
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
The `00 00 00 08` designates another timestamp. Note, there are a lot `ff`
bytes follows. These `-1` values denote NULL values.
