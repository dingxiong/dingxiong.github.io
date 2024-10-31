---
layout: post
title: Postgres -- Privileges and Roles
date: 2024-10-26 21:48 -0700
categories: [database, postgres]
tags: [postgres]
---

We are all familiar with the concept of role based authorization. It is
designed around answering this question: is role X allowed to do Y on object Z?
Postgres is no exception.

## How to allow multiple users do `ALTER TABLE`?

## What for new tables?

Suppose `GRANT SELECT ON ALL TABLES IN SCHEMA public TO ro_user`, and then
what?

## Superuser

A lot of privilege checks are bypassed if the current user is a superuser.
Postgres refers to table
[pg_authid](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/utils/misc/superuser.c#L70)
to verify if a user is a superuser.

## Owner of a table

`pg_class`, `pg_tables` and `\dp <table_name>` all can show the owner of a
table.

## Membership

Get the membership:

```
SELECT r.rolname as role_name, u.rolname as member_name, m.*
FROM pg_roles r
JOIN pg_auth_members m ON r.oid = m.roleid
JOIN pg_roles u ON u.oid = m.member;
```

Q1. it is not permitted to grant membership in a role to PUBLIC.
https://www.postgresql.org/docs/current/role-membership.html

## Create Role

## Grant/Revoke

First, we need to figure out where ACL info is stored. Quoted from ChatGPT:

```
pg_class: Holds ACLs for tables, views, and indexes in the relacl column.
pg_namespace: Stores ACLs for schemas in the nspacl column.
pg_database: Contains ACLs for databases in the datacl column.
pg_proc: Stores ACLs for functions in the proacl column.
pg_type: Contains ACLs for data types in the typacl column.
```

### Grant Permissions to Tables

Let's walk through an example.

```
GRANT SELECT ON ALL TABLES IN SCHEMA schema_name TO PUBLIC;
```

This statement grants the read permission to all tables in schema
`schema_name`to public. First, `ALL TABLES IN SCHEMA` is parsed
[here](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/parser/gram.y#L7889).
`ACL_TARGET_ALL_IN_SCHEMA` instructs Postgres to find all tables in this
schema. See
[code](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/backend/catalog/aclchk.c#L428).
Then the call stack is

```
ExecuteGrantStmt
  -> ExecGrantStmt_oids
    -> ExecGrant_Relation
```

`ExecGrant_Relation` has a for loop that iterates all `istmt->objects`, namely,
all tables in this schema and change its definition in `pg_class` catalog
table. See below sample query result

```
=# select relacl from pg_class where relname = 'association' limit 1 \gx
-[ RECORD 1 ]-----------------------------
relacl | {pguser=arwdDxt/pguser,=r/pguser}
```

The format of this column is `grantee=privilege-abbreviation[*].../grantor`. An
empty grantee field in an aclitem stands for PUBLIC. `relacl` is an
[AclItem](https://github.com/postgres/postgres/blob/a3e6c6f929912f928fa405909d17bcbf0c1b03ee/src/include/utils/acl.h#L54)
array. It has a grantee id and grantor id. When grantee id is zero
(ACL_ID_PUBLIC), it means the grantee is PUBLIC.

We can use `has_table_privilege` to check whether a role has privilege on a
table. For example,

```
admincoin=> select  has_table_privilege('oneoff', 'public.user', 'SELECT');
 has_table_privilege
---------------------
 t
```

The call stack is

```
has_table_privilege_name_name
  -> pg_class_aclcheck
    -> pg_class_aclcheck_ext
      -> pg_class_aclmask_ext
```

## Implicit Role PUBLIC
