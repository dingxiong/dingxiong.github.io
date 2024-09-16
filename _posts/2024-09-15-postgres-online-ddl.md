---
layout: post
title: Postgres -- Online DDL
date: 2024-09-15 13:55 -0700
categories: [database, postgres]
tags: [postgres]
---

## Index

### Processes and Memory

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

BTW, we can use view `pg_stat_progress_create_index` to monitor index build
process.

### Create Index Locks

There are two ways to create an index: `CREATE INDEX <index name> on ...` and
`CREATE INDEX CONCURRENTLY <index name> on ...`. The former one holds a
`ShareLock` through out the index build process, which blocks other
[insert/update/delete operations](https://www.postgresql.org/docs/7.2/locking-tables.html).
So if building an index takes five hours, then the table is read-only for five
hours. This is only useful for bootstrapping or migrating a new database and is
disastrous for a online production database. That is usually why we take the
latter approach.

`CREATE INDEX CONCURRENTLY` uses three stages(transactions), three waits and
two full table scans. So it is definitely slower than the approach without the
`CONCURRENTLY` keyword, but most time it only holds a
`ShareUpdateExclusiveLock`, so table updates are not blocked during the
process.

Before we go to the details, we must cover an important concept
[Heap Only Tuples (HOT)](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/heap/README.HOT#L3).
This README file is very informative. Also, I like some Chinese blog posts with
vivid diagrams. For example
[this one](https://www.cnblogs.com/duanleiblog/p/14378565.html). Note sure if
the link is still valid ^\_^. Basically, in the world of MVCC, HOT means that
if a table update does not contain any index field change then the update can
be made only to the tuple page and not creating a new index version. There is
an redirect mechanism in the page for you to retrieve the updated version. This
optimization saves space and time. But it requires that the indices of this
table do not change. That is why it makes index creation complicated and we
need to process index creation in three stages. Let's take a closer look at
these three stages with some experiments.

The main code is
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/indexcmds.c#L531).

#### Stage 1: Create an Index

The code block for this part is
[this code range](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/indexcmds.c#L582-L1627).
I extract some important logic below.

```
...
lockmode = concurrent ? ShareUpdateExclusiveLock : ShareLock;
rel = table_open(tableId, lockmode);
...
	if (skip_build || concurrent || partitioned)
		flags |= INDEX_CREATE_SKIP_BUILD;
...

	indexRelationId =
		index_create(rel, indexRelationName, indexRelationId, parentIndexId,
					 parentConstraintId,
					 stmt->oldNumber, indexInfo, indexColNames,
					 accessMethodId, tablespaceId,
					 collationIds, opclassIds, opclassOptions,
					 coloptions, NULL, reloptions,
					 flags, constr_flags,
					 allowSystemTableMods, !check_rights,
					 &createdConstraintId);
...

	if (!concurrent)
	{
		/* Close the heap and we're done, in the non-concurrent case */
		table_close(rel, NoLock);

    ...
		return address;
	}

	/* save lockrelid and locktag for below, then close rel */
	heaprelid = rel->rd_lockInfo.lockRelId;
	SET_LOCKTAG_RELATION(heaplocktag, heaprelid.dbId, heaprelid.relId);
	table_close(rel, NoLock);

...

	PopActiveSnapshot();
	CommitTransactionCommand();
```

This stage does a lot of validation and preprocessing. The whole phase is
wrapped inside a single transaction. First, it determines the lock mode.
Non-CONCURRENTLY build uses `ShareLock` while CONCURRENTLY build uses
`ShareUpdateExclusiveLock`. So non-CONCURRENTLY build blocks table updates.
Note that we are referring to the lock on the table not the lock on the index.
Second, it sets the build flags. CONCURRENTLY build sets
`INDEX_CREATE_SKIP_BUILD`, which means the `index_create` function below only
creates the index but does not build the index content. Table is unlocked and
closed after `index_build`, and this is where Non-CONCURRENTLY build is
complete. For CONCURRENTLY build, it commits this transaction which marks the
end of phase one.

What is the end state of phase one?

- Non-CONCURRENTLY build is finished.
- CONCURRENTLY build has an empty index marked as `Invalid`.

Let's do a quick experiment. In terminal one, run below query

```
postgres=# select * from my_table;
 id |  name
----+---------
  1 | Alice
  2 | Bob
  3 | Charlie
  4 | David
(4 rows)

postgres=# begin;
BEGIN
postgres=*# update my_table set name = 'A' where id = 1;
UPDATE 1
postgres=*#
```

and in terminal two, run below query

```
postgres=# create index concurrently idx_test on my_table (name);
```

Basically, terminal one has an ongoing transaction and terminal two tries to
create a new index. You will see terminal two gets stuck. If you check the
table details, you see it has a new invalid index.

```
postgres=# \d my_table;
                                   Table "public.my_table"
 Column |         Type          | Collation | Nullable |               Default
--------+-----------------------+-----------+----------+--------------------------------------
 id     | integer               |           | not null | nextval('my_table_id_seq'::regclass)
 name   | character varying(50) |           |          |
Indexes:
    "my_table_pkey" PRIMARY KEY, btree (id)
    "idx_test" btree (name) INVALID
```

#### Stage 2: First Scan

The code block for this part is
[this code range](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/indexcmds.c#L1628-L1699).
It is not that long.

```
	StartTransactionCommand();
...

	WaitForLockers(heaplocktag, ShareLock, true);
...

	PushActiveSnapshot(GetTransactionSnapshot());

	index_concurrently_build(tableId, indexRelationId);

	PopActiveSnapshot();

	CommitTransactionCommand();
```

Phase 2 is also wrapped inside a single transaction. It first waits for a
`ShareLock` on this table. We need this step to make sure HOT chain is valid.
Refer to the HOT README file above for the reasoning behind.

Note that `WaitForLockers` only waits for locks but does not acquire locks. It
obtains a list of transactions which hold this lock at the function start and
waits until these transactions to disappear (either commit or rollback). It
does not add new transactions to the waiting list during waiting. Because it
does not acquire the lock, there is no problem with lock queue priority. I was
once skeptic that building index will need some metadata lock, so I am worried
whether some other process has higher priority to acquire this metadata lock.
In Msyql, the thread that is creating an index acquires a special lock which
has higher priority than normal xlock to solve this problem. In Postgres, we do
not have this problem at all because we DO NOT need to lock at all. We just
wait for the old transactions to die!

Then we build a snapshot, start the first scan path `index_concurrently_build`
and commit this transaction. We also mark this new index as writable but not
readable.

Continue with above experiment, let's check `pg_locks`. There is a process is
waiting for a `ShareLock`.

```
admincoin=# select *, relation::regclass from pg_locks;
   locktype    | virtualxid | transactionid  | virtualtransaction |  pid  |           mode           | granted | fastpath |          waitstart           |                       relation
---------------+------------+----------------+--------------------+-------+--------------------------+---------+----------+------------------------------+------------------------------------------------------
 virtualxid    | 4/322      |                | 5/62               | 11036 | ShareLock                | f       | f        | 2024-09-17 00:16:20.36455+00 |
```

The lock is not granted because there is an old ongoing transaction that is not
committed yet. If we check the index build progress, we will see

```
postgres=# select * from pg_stat_progress_create_index \gx
-[ RECORD 1 ]------+---------------------------------
command            | CREATE INDEX CONCURRENTLY
phase              | waiting for writers before build
```

The code for this phase is defined
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/catalog/system_views.sql#L1266).
The enum 1 corresponds to the
[code here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/indexcmds.c#L1645).
So far everything makes sense.

#### Stage 3: Second Scan

The code block for this part is
[this code range](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/indexcmds.c#L1700-L1759)
It is not that long either.

```
	StartTransactionCommand();

  pgstat_progress_update_param(PROGRESS_CREATEIDX_PHASE,
                              PROGRESS_CREATEIDX_PHASE_WAIT_2);

  WaitForLockers(heaplocktag, ShareLock, true);

	snapshot = RegisterSnapshot(GetTransactionSnapshot());
	PushActiveSnapshot(snapshot);

	/*
	 * Scan the index and the heap, insert any missing index entries.
	 */
	validate_index(tableId, indexRelationId, snapshot);

	limitXmin = snapshot->xmin;

	PopActiveSnapshot();
	UnregisterSnapshot(snapshot);

	CommitTransactionCommand();
```

This is the second `WaitForLockers`. It waits a `ShareLock` again. As the
comment says, this phase do a second scan to insert missing index entries from
the last stage.

Let's continue the experiment. Open terminal three, and run

```
postgres=# begin;
BEGIN
postgres=*# update my_table set name = 'B' where id = 1;
UPDATE 1
```

Go to terminal two, and run `rollback;`. So the first update is aborted. The
`CREATE INDEX CONCURRENTLY` command is still stuck. If you check `pg_locks`,
you will find that it is waiting for a `ShareLock`. This time it is the new
transaction we just started that was blocking it. Also, index build progress
shows

```
postgres=# select * from pg_stat_progress_create_index \gx
-[ RECORD 1 ]------+--------------------------------------
command            | CREATE INDEX CONCURRENTLY
phase              | waiting for writers before validation
```

It is slightly different changing from `writers before build` to
`writers before validation`. This message corresponds to an
[enum value 3](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/catalog/system_views.sql#L1270).
This is exactly the enum `PROGRESS_CREATEIDX_PHASE_WAIT_2` above. I know the
name is wrong!

#### Stage 4: Wait for Old Snapshots

The code block for this part is
[this code range](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/commands/indexcmds.c#L1760-L1799)

```
	StartTransactionCommand();

	/*
	 * The index is now valid in the sense that it contains all currently
	 * interesting tuples.  But since it might not contain tuples deleted just
	 * before the reference snap was taken, we have to wait out any
	 * transactions that might have older snapshots.
	 */
	pgstat_progress_update_param(PROGRESS_CREATEIDX_PHASE,
								 PROGRESS_CREATEIDX_PHASE_WAIT_3);
	WaitForOlderSnapshots(limitXmin, true);

	/*
	 * Index can now be marked valid -- update its pg_index entry
	 */
	index_set_state_flags(indexRelationId, INDEX_CREATE_SET_VALID);

	CacheInvalidateRelcacheByRelid(heaprelid.relId);

	UnlockRelationIdForSession(&heaprelid, ShareUpdateExclusiveLock);

	pgstat_progress_end_command();

```

This final stage does another wait, not for locks this time but for old
snapshots. Also, it marks this index as valid finally.

For the experiment above, a new transaction can no longer block this stage.
TODO: find a way to create a snapshot to block the phase.

### FAQ

After learning so much about how `CREATE INDEX` works. What are the FQAs people
may have?

#### Q1. How to kill a query?

```sql
-- gracefully cancel
SELECT pg_cancel_backend(<pid of the process>)

-- hard terminate
SELECT pg_terminate_backend(<pid of the process>)
```

#### Q2. For a blocked process, How to know which process is blocking it?

`pg_locks` provides an overall view of all locks in the database. However, it
is not straightforward to figure out the blocking relationship by ourselves.
Luckily, there is a function `pg_blocking_pids(<pid>)` to obtain this
information. For example,

```
=# select pg_blocking_pids(2267);
 pg_blocking_pids
------------------
 {3240}
(1 row)
```

`pg_blocking_pids` returns `integer[]`, and it is possible this list contains
duplicates. Quoted
[its documentation](https://www.postgresql.org/docs/16/functions-info.html)
below.

> When using parallel queries the result always lists client-visible process
> IDs (that is, pg_backend_pid results) even if the actual lock is held or
> awaited by a child worker process. As a result of that, there may be
> duplicated PIDs in the result.

This happens for example, one client issues an `create index` request, and
another clients tries to update the same table, then the result of
`pg_blocking_pids(2nd_client_pid)` contains the 1st client's pid multiple
times. This is because `create index` utilizes multiple processes. See the
explanation of `max_parallel_maintenance_workers` above.

Btw, a very useful query about pg_locks.

```
SELECT pl.*, psa.query FROM pg_locks pl LEFT JOIN pg_stat_activity psa ON pl.pid = psa.pid;
```

#### Q3. For non-CONCURRENTLY option, are writes all forbidden during index creation?

Yes. I did an experiment. If we do `CREATE INDEX CONCURRENTLY`, then there is
no such problem.

#### Q4. What locks are needed for create index CONCURRENTLY ?

It needs a `ShareUpdateExclusiveLock` for phase 1, and waits for `ShareLock`s
for phase 2 and 3. Also, it waits for old snapshots for phase 4.

#### Q5. What is lock queue priority?

Suppose we have a write query is running, and another write query is blocked.
Then a `create index` request comes and get blocked. Once the lock is released,
will the write request or the `create index` request get the lock?

There is no need to worry about it at all because it does not acquire locks. It
only wait for locks.

Wait. I need to be strict. It indeed acquires a `ShareUpdateExclusiveLock` at
phase 1. But according to
<https://www.postgresql.org/docs/7.2/locking-tables.html>,
SELECT/INSERT/UPDATE/DELETE does not conflict with this lock.
