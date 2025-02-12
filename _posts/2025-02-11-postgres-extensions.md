---
layout: post
title: Postgres -- Extensions (Draft)
date: 2025-02-11 15:07 -0800
categories: [database, postgres]
tags: [postgres, pg_repack]
---

## pg_repack

### What locks used

https://github.com/reorg/pg_repack/blob/306b0d4f7f86e807498ac00baec89ecd33411398/bin/pg_repack.c#L1599

### How to clean up things

```
DROP EXTENSION IF EXISTS pg_repack CASCADE;
```

`CASCADE` will drop the `repack` schema, so all temporary tables, sequences,
indices will all be dropped.
