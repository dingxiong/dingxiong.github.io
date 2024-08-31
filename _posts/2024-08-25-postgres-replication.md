---
layout: post
title: Postgres -- Replication
date: 2024-08-25 08:03 -0700
categories: [database, postgres]
tags: [postgres]
---

## WAL (Write Ahead Log)

### Tree locations

```
postgres=# select pg_current_wal_flush_lsn(), pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_flush_lsn | pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------------+--------------------+---------------------------
 0/176BED8                | 0/176BED8          | 0/176BED8

postgres=# select pg_walfile_name_offset('0/176BED8');
       pg_walfile_name_offset
------------------------------------
 (000000010000000000000001,7782104)
```

### Min SLN

https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/transam/xlog.c#L2690

### wal sender

wake up WAL sender
https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/transam/xlog.c#L2499

1. when a replication slot is created, does it use the oldest wal location or
   the newest?
2. Who maintains the offset?

## Replication

Most replication code is inside
[src/backend/replication folder](https://github.com/postgres/postgres/tree/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication).
There are two types of replication: physical and logical. Physical replication
means raw WALs are streamed to the client, which is good for master-slave
database replication. Logical replication means each changed tuple is streamed
to client. This model is better suited for business consumers. Think of
physical and logical replication as L4 and L7 layer in the network stack.

Physical replication does not have too much things to tune. While logical
replication has more flexibility. `pg_publication` defines the relevant
configurations for logical replications.

Replication uses the COPY sub-protocol underneath. See more details of
server-client protocol
[here](https://www.postgresql.org/docs/16/protocol.html).

### Logical slots creation

Both physical and logical replication need to define replication slots. The
data can be retrieved from table `pg_replication_slots`. Note, this info is not
backed by a relational database table. But instead they are persisted to disk
`pg_replslot/<slot_name>/state` directly.

For example.

```
postgres=# select pg_create_logical_replication_slot('my_slot', 'pgoutput');
```

Disk content

```
$ ll data/pg_replslot/my_slot/
total 8
drwx------  3 xiongding  staff    96B Aug 26 06:31 ..
-rw-------  1 xiongding  staff   200B Aug 26 06:31 state
drwx------  3 xiongding  staff    96B Aug 26 06:31 .
```

And at postmaster starts, all slots on disk are loaded to memory. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/slot.c#L1898).

### Logical replication

There are two ways to use logical replication: pull model and push model. In
pull mode, we only make a connection to db as necessary to pull new changes. In
push mode, we keep a connection active all the time and wait for new changes to
show up.

#### Pull Mode

Let's do a simple experiment to explore this approach (this example comes from
[this blog](https://medium.com/@film42/getting-postgres-logical-replication-changes-using-pgoutput-plugin-b752e57bfd58)).
First, create a table, set up replication slot and publication.

```sql
CREATE TABLE demo_table (id integer);
select pg_create_logical_replication_slot('my_slot', 'pgoutput');
CREATE PUBLICATION my_publication FOR ALL TABLES;
```

Then we insert some data to the table and check the replication data.
`pgoutput` has two formats to encode replication tuples: `text(t)` and
`binary(b)`. By default, binary format is
[disabled](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/pgoutput/pgoutput.c#L287).

**Binary Format**

```
INSERT INTO demo_table (id) values (1);

SELECT * FROM pg_logical_slot_get_binary_changes('my_slot', NULL, NULL, 'proto_version', '1', 'publication_names', 'my_publication', 'binary', 'true');
    lsn    | xid |                                      data
-----------+-----+--------------------------------------------------------------------------------
 0/A1C0A78 | 772 | \x42000000000a1c0ab80002bf3c1fa2794400000304
 0/A1C0A78 | 772 | \x52000040287075626c69630064656d6f5f7461626c65006400010069640000000017ffffffff
 0/A1C0A78 | 772 | \x49000040284e0001620000000400000001
 0/A1C0AE8 | 772 | \x4300000000000a1c0ab8000000000a1c0ae80002bf3c1fa27944
```

The first row is `BEGIN` and the last row is `COMMIT`. Not sure what the second
row is. The third row is the insert itself.

```
\x 49 00004028 4e 0001   62  00000004   00000001
    I  OID     N nattrs  b   val_size    value
```

**Text Format**

```
INSERT INTO demo_table (id) values (1);

SELECT * FROM pg_logical_slot_get_binary_changes('my_slot', NULL, NULL, 'proto_version', '1', 'publication_names', 'my_publication');
    lsn    | xid |                                      data
-----------+-----+--------------------------------------------------------------------------------
 0/A1C0828 | 771 | \x42000000000a1c0a100002bf3c1de9ff8a00000303
 0/A1C0828 | 771 | \x52000040287075626c69630064656d6f5f7461626c65006400010069640000000017ffffffff
 0/A1C0828 | 771 | \x49000040284e0001740000000131
 0/A1C0A40 | 771 | \x4300000000000a1c0a10000000000a1c0a400002bf3c1de9ff8a
```

```
\x 49 00004028 4e 0001   74  00000001    31
    I  OID     N nattrs  t   num_chars  value
```

the hex value of character `1` in ascii table is `31`. See detailed code
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/logical/proto.c#L852).

**Date Field**

Also, when working with Debezium, the date field output confused me a little
bit. Let's see one example.

```
CREATE TABLE important_dates (
    event_date DATE
);
INSERT INTO important_dates (event_date) VALUES ('2024-01-01');

SELECT * FROM pg_logical_slot_get_binary_changes('my_slot', NULL, NULL, 'proto_version', '1', 'publication_names', 'my_publication', 'binary', 'true');
    lsn    | xid |                                                   data
-----------+-----+----------------------------------------------------------------------------------------------------------
 0/A1D6EA0 | 774 | \x42000000000a1d6ee00002bf3c4b500b3800000306
 0/A1D6EA0 | 774 | \x520000402c7075626c696300696d706f7274616e745f646174657300640001006576656e745f64617465000000043affffffff
 0/A1D6EA0 | 774 | \x490000402c4e000162000000040000223e
 0/A1D6F10 | 774 | \x4300000000000a1d6ee0000000000a1d6f100002bf3c4b500b38
```

Note that `0x223e = 8766 ~= 24 * 365`. So pgoutput uses
[postgres epoch date](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/datatype/timestamp.h#L235).

Let's see what Debezium got. Below is some sample date I get from Debezium

```
  {
    "type": "int32",
    "optional": true,
    "name": "io.debezium.time.Date",
    "version": 1,
    "field": "start_date"
  },
          ...
  "payload": {
    "before": null,
    "after": {
      "id": 1,
      "name": "Project Alpha",
      "start_date": 19723,
      "end_date": 19904
    },
```

I was curious why the date is `19723 ~= (30 + 24) * 365`. This is because after
Debezium got the postgres epoch based date, it converts it to
[is.debezium.time.Date](https://github.com/debezium/debezium/blob/493e1e23b0633a4e4d990e43733e249343599af5/debezium-core/src/main/java/io/debezium/time/Date.java#L16-L16)
and as its comment says it then converts the date to a int32 integer relative
to Unix epoch. So everything makes sense. No bug! hum~

#### Push mode

Follow the same setup in the pull mode, we can use `pg_recvlogical` to
continuously stream the changes.

```
$ ./install_dir/bin/pg_recvlogical --start --slot=my_slot -p 5433 -f - -d postgres -o proto_version=1 -o publication_names=my_publication
B̈́Gv
I@Nt6
C ̈́Gv
```

Above output is from an update

```
postgres=# INSERT INTO demo_table (id) values (6);
```

First, we cannot use `psql` to do the work because `psql` does not support
`START_REPLICATION` protocol.

Let's see what happened internally. Let's start from the client side first. The
main code of `pg_recvlogical` is
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/bin/pg_basebackup/pg_recvlogical.c#L213).
Basically, it has two steps.

1. Start replication by sending
   `START_REPLICATION SLOT my_slot LOGICAL 0/190D020 (proto_version '1', publication_names 'my_publication');`
   to the server.
2. Start the loop to receive CopyData messaeges. The loop stops either there is
   an error or the specified
   [EndPos](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/bin/pg_basebackup/pg_recvlogical.c#L93)
   is reached. Note, there should not be CopyDone messages from the server
   because in this mode only Client should send CopyDone to the server. When
   there is no CopyData message from the server, the client is idling by
   issuing a
   [select](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/bin/pg_basebackup/pg_recvlogical.c#L416)
   to the underlying socket.

On the server side, once it receives the `START_REPLICATION` command from the
client, it first creates a
[logical_decoding_ctx](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/walsender.c#L1483),
and then sends a `CopyBothResponse` message to the client, which is required by
the
[protocol](https://www.postgresql.org/docs/16/protocol-replication.html#PROTOCOL-REPLICATION-START-REPLICATION):

> The server can reply with an error, for example if the requested section of
> WAL has already been recycled. On success, the server responds with a
> CopyBothResponse message, and then starts to stream WAL to the frontend.

After that, the server enters the `WalSndLoop`, keeps streaming WALs to the
client, and stops after receiving a `CopyDone` message from the client. When
there is no new WALs, the server process enters a
[wait state](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/walsender.c#L2886).
This wait is implemeted using epoll or kevent depending on the underlying OS,
and it waits for two types of events: socket events from the client and
conditional variable events of new WAL. Every time when a new WAL is flushed to
disk, this conditional variable is woke up. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/access/transam/xlog.c#L2499).
Note, many processes are running at the same time ^\_^.

```
$ ps -ef | grep -i postgres
  502 61643 86413   0 12:37PM ??         0:00.53 postgres: background writer
  502 61644 86413   0 12:37PM ??         0:00.52 postgres: walwriter
  502 61645 86413   0 12:37PM ??         0:00.77 postgres: autovacuum launcher
  502 61933 86413   0 12:48PM ??         0:00.56 postgres: walsender xiongding postgres [local] START_REPLICATION
```

One special note about the start `lsn`. When the start `lsn` provided in the
`START_REPLICATION` command is older than slot's `confirmed_flush_lsn`, it is
reset to `confirmed_flush_lsn`. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/logical/logical.c#L567).
When the start `lsn` is newer than slot's `confirmed_flush_lsn`, then WALs
between `[confirmed_flush_lsn, provided lsn)` are ignored. The subtlety is that
the walsender process still processes the WALs in range
`[confirmed_flush_lsn, provided lsn]`, but these records are thrown away in
streaming. The core code is
[this function](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/logical/snapbuild.c#L433).
The corresponding stack path is

```
PostgresMain
  -> exec_replication_command
    -> WalSndLoop
      -> XLogSendLogical
        -> ogicalDecodingProcessRecord
          -> heap_decode
            -> SnapBuildProcessChange
              -> ReorderBufferSetBaseSnapshot
                -> AssertTXNLsnOrder
                  -> SnapBuildXactNeedsSkip
```

Also, when creating a new replication slot by
`pg_create_logical_replication_slot`, it will set the `confirmed_flush_lsn` to
`pg_current_wal_insert_lsn()`. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/slotfuncs.c#L150).
So basically, a new slot always starts from the lastest `lsn`. See below
example.

```
postgres=# select pg_create_logical_replication_slot('my_slot2', 'pgoutput');
 pg_create_logical_replication_slot
------------------------------------
 (my_slot2,0/190D798)

postgres=# select * from pg_replication_slots;
 slot_name |  plugin  | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase |        inactive_since         | conflicting | invalidation_reason | failover | synced
-----------+----------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------+-------------------------------+-------------+---------------------+----------+--------
 my_slot2  | pgoutput | logical   |      5 | postgres | f         | f      |            |      |          759 | 0/190D760   | 0/190D798           | reserved   |               | f         | 2024-08-30 08:40:11.831745-07 | f           |                     | f        | f

postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/190D798
```

During the while loop, the client sends
[Standby Status Update](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/bin/pg_basebackup/pg_recvlogical.c#L144)
messages to the server to inform the written and flushed `lsn`. On the server
side, the corresponding `lsn` in-memory fields are
[updated](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/walsender.c#L2462)
and the corresponding slot is
[updated and persisted in Disk](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/walsender.c#L2484).

```
postgres=# select * from pg_replication_slots\gx
...
catalog_xmin        | 759
restart_lsn         | 0/190D728
confirmed_flush_lsn | 0/190D760
...
```

`confirmed_flush_lsn` is the latest lsn flushed on the client side.
`restart_lsn` is the min lsn needed for this slot. The two values are not equal
and `restart_lsn` is definitely no larger than `confirmed_flush_lsn` because
walsender may still need WAL older than `confirmed_flush_lsn` in order to
calculate the required information. Also, notice that the last step of updating
replication slot is to recalculate the global
[replicationSlotMinLSN = min of restart_lsn among all replication slots](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/replication/logical/logical.c#L1906).
This value is used by postgres to decide the oldest WAL to keep.

OK. so far we understand how client and server keep track of streaming
checkpoints.
