---
layout: post
title: Postgres -- Vacuum
date: 2024-12-04 23:13 -0800
categories: [database, postgres]
tags: [postgres]
---

The purpose of vacuum:

1. recover disk space.
2. Update planner statistics
3. Prevent transaction id wraparound failure

   - how to find the max and min XID in postgres
   - how is vacuum affected by XID?
   - how dd dashboard show low row count after failover
   - how does it set FrozenTransactionId
   - how MVCC prevents vacuum, and only subset of tables or all tables?

How does vacuum compare XID? See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/transam/transam.c#L280)
Wrapp around happens

## Autovacuum

### When will autovacuum run?

Autovacuum is controlled by two parameters

- autovacuum_vacuum_threshold
- autovacuum_vacuum_scale_factor

When the number of dead tuples go beyond threshold
`vac_base_thresh + vac_scale_factor * reltuples`, autovacuum runs. See the code
[here](https://github.com/postgres/postgres/blob/e4c8865196f6ad6bb3473bcad1d2ad51147e4513/src/backend/postmaster/autovacuum.c#L3053).
The number of dead tuples can be found inside table `pg_stat_all_tables`.

```
admincoin=> select relname, n_dead_tup from pg_stat_all_tables where relname = 'event_log' \gx
-[ RECORD 1 ]---------
relname    | event_log
n_dead_tup | 3
```

Meanwhile, we can check vacuum progress from table `pg_stat_progress_vacuum`.

## Lock needed

Vacuum requires `ShareUpdateExclusiveLock`, which does not block usual CRUD
operations, but this lock type conflicts with itself, so it there is at most
one vacuum for a table at a given time. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/vacuum.c#L2020).
Meanwhile, vacuum blocks `ALTER TABLE` command.

I understand that it requires `ShareUpdateExclusiveLock` at the beginning and
release it at the end. But during the process, it needs to scan disk blocks,
what locks are needed then? Suppose another transaction is locking a tuple,
such as update or select for update, what will this vacuum process do?

Ok. Let's answer the first question. What locks are needed when scanning
blocks? When scanning blocks, vacuum needs to hold
[cleanup lock](https://github.com/postgres/postgres/blob/e4c8865196f6ad6bb3473bcad1d2ad51147e4513/src/backend/access/heap/vacuumlazy.c#L927).
The terminolgy is subtle. To be honest, I still do not know whether cleanup
lock is a buffer pin or a buffer lock. No matter what, the lock/pin is released
quickly after finishing scanning the page. In practice, I do see any real case
that we need to worry that this lock/pin is held for two long.

The answer to the second question depends on the version of Postgres. For
version <= 14, vacuum skips all buffers that it cannot acquire the cleanup lock
immediately. For version >= 15. It waits "infinitely". This is the changing
[commit](https://github.com/postgres/postgres/commit/44fa84881fff4529d68e2437a58ad2c906af5805#diff-3198152613d9a28963266427b380e3d4fbbfabe96a221039c6b1f37bc575b965L1057)
with a long
[discussion](https://www.postgresql.org/message-id/flat/CA%2BTgmoZYri_LUp4od_aea%3DA8RtjC%2B-Z1YmTc7ABzTf%2BtRD2Opw%40mail.gmail.com#34b6b03d6d8340b186a19f72c0b7a698)
about why the author made this change. From my point of view, waiting
"infinitely" seems better because if you simply skip, then we just invoke the
vacuum process repeatably without any real progress. Also, in any practical
situation, this wait is very short. I tried to simulate a long buffer pin with
the help of chatgpt for about 3 hours without any luck. We can tell the
difference between the old and new behavior from the vacuum logs. Below is an
example log of the old behavior.

```
INFO:  vacuuming "public.large_table"
INFO:  table "large_table": index scan bypassed: 1 pages from table (0.12% of total) have 1 dead item identifiers
INFO:  table "large_table": found 0 removable, 119 nonremovable row versions in 2 out of 834 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 767
Skipped 1 page due to buffer pins, 0 frozen pages.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s.
INFO:  vacuuming "pg_toast.pg_toast_16441"
INFO:  table "pg_toast_16441": found 0 removable, 0 nonremovable row versions in 0 out of 0 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 767
Skipped 0 pages due to buffer pins, 0 frozen pages.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s.
VACUUM
```

You see that it skips 1 page due to buffer pins.

### Logs

`log_autovacuum_min_duration` controls the threshold of long-running autovacuum
actions. The code is
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/heap/vacuumlazy.c#L599).
An example logs is as following

```
automatic vacuum of table "admincoin.public.object_column_index": index scans: 0
	pages: 0 removed, 14094016 remain, 978387 scanned (6.94% of total)
	tuples: 0 removed, 823163314 remain, 10252998 are dead but not yet removable
	removable cutoff: 345727619, which was 19811910 XIDs old when operation ended
	frozen: 27 pages from table (0.00% of total) had 372 tuples frozen
	index scan bypassed: 96277 pages from table (0.68% of total) have 1027041 dead item identifiers
	I/O timings: read: 0.000 ms, write: 0.000 ms
	avg read rate: 0.000 MB/s, avg write rate: 0.000 MB/s
	buffer usage: 1870163 hits, 0 misses, 0 dirtied
	WAL usage: 0 records, 0 full page images, 0 bytes
	system usage: CPU: user: 6.09 s, system: 0.00 s, elapsed: 21.81 s
```

Find last vacuum time of each table

```
SELECT relname AS table_name,
       last_autovacuum,
       last_vacuum
FROM pg_stat_user_tables
WHERE last_autovacuum IS NOT NULL OR last_vacuum IS NOT NULL;
```
