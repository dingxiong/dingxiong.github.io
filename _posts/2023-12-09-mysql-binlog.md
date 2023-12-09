---
title: Mysql binlog
date: 2023-12-09 00:20 -0800
categories: [database, mysql]
tags: [mysql, binlog]
---

## Configurations

Mysql binary log is rotated. The threshold is controlled by `max_binlog_size`
option. AWS RDS uses a stored procedure to control the retention period of
binary logs `CALL mysql.rds_show_configuration;`.

Binlog can have a few different format: `STATEMENT`, `ROW` or `MIXED`.

```
mysql> show variables like 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
```

## How to view binary logs

There are two ways to view the binary logs. But first we need to list all
available binary logs.

```
mysql> show binary logs;
+---------------+-----------+-----------+
| Log_name      | File_size | Encrypted |
+---------------+-----------+-----------+
| binlog.000001 |  87502922 | No        |
| binlog.000002 |       180 | No        |
| binlog.000003 |  13855131 | No        |
| binlog.000004 |  43809965 | No        |
+---------------+-----------+-----------+
```

### View binary logs inside mysql shell

`SHOW BINLOG EVENTS`

### Use mysqlbinlog command

Note the `--stop-never` parameter can keep the connection open and stream new
log after log rotation.

### Programmatically subscribe to bin logs

[Binlog network stream](https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_replication.html#sect_protocol_replication_binlog_stream)
allows application to programmatically subscribe bin logs. After your
application registers itself to the master db, it can receive bin logs.
[mysql-binlog-connector-java](https://github.com/shyiko/mysql-binlog-connector-java/blob/dd710a5466381faa57442977b24fceff56a0820e/src/main/java/com/github/shyiko/mysql/binlog/BinaryLogClient.java#L541-L582)
has an example implementation of using binlog network stream to obtain a
continuous stream of bin logs.
