---
title: Tech diary
date: 2023-12-10 23:39 -0800
categories: [diary]
tags: [diary]
---

Some random thoughts or learnings.

## 2023-12-24 Sun

I read a few videos from Prof. Andy. I just feel that the two most difficult
problems in RDMS are query optimization and concurrency control.

## 2023-12-16 Sat

I need code folding in my blog, and this post
https://github.com/cotes2020/jekyll-theme-chirpy/issues/436 perfectly solved my
problem.

## 2023-12-15 Fri

This is really a nice series about Mysql Internals
https://www.zhihu.com/column/c_1453135878944567296

## 2023-12-10 Sun

When I was watching Prof. Andy's CMU class talk
[Query Execution 1](https://www.youtube.com/watch?v=I0QdCSu06_o&list=PLSE8ODhjZXjaKScG3l0nuOiDTTqpfnWFf&index=12),
one part stroke me

> Expression trees are flexible but slow. JIT compilation can (sometimes) speed
> them up.

This is very interesting. This reminds me of the days when we wrote the
execution engine for the matching service DSL. I should definitely spend some
time reading Postgres JIT.

## 2023-11-30 Thu

TIL: parquet-tools

## 2023-11-25 Sat

I finally set up my tech blog https://dingxiong.github.io/

##gc 2023-11-24 Fri

Tmux on MacOS drove me crazy. It reorders PATH variable. See
https://superuser.com/a/583502 .

## 2023-11-21 Tue

I finally understand Python async/await.

## 2023-10-26 Thu

TIL: how to calculate `C_n^k % p`: https://codeforces.com/blog/entry/78873

## 2023-10-16 Mon

LC 1388 blows my mind. I should make it an exercise for my kid in future.

## 2023-10-11 Wed

I learnt quite a few things about Java today: c1, c2 compilier, ZGC, etc. Most
useful things can be found at https://openjdk.org/jeps .

## 2023-10-07 Sat

TIL: asof join.

TIL: When I read Clickhouse documents, I found a good blog post series from
Tilak.
[Vectorized and Compiled Queries](https://medium.com/@tilakpatidar/vectorized-and-compiled-queries-part-3-807d71ec31a5).

## 2023-09-30 Sat

TIL: Java playground https://dev.java/playground/

TIL: Java virtual thread
https://inside.java/2021/05/10/networking-io-with-virtual-threads/ This is
interesting because I hate async/await. This is as easy as Golang goroutine but
is stackless. I am kind of liking Java again!

## 2023-09-14 Thu

TIL: the meaning of each field inside `/proc/meminfo`.

## 2023-09-02 Sat

I spent almost one week studying the source code of Debezium and learnt a lot
of details in it. Then I asked myself: if I need to read the source code of
every tool I use, then I will burn out. I admit that reading source code helps
me in the long run, but I do not have the energy cause I have a little baby. I
would spend more time in family instead of self-improvement.

## 2023-08-29 Tue

TIL: kafka advertised.listeners from this post
https://stackoverflow.com/questions/52996028/accessing-local-kafka-from-within-services-deployed-in-local-docker-for-mac-inc

## 2023-08-22 Tue

TIL: a good sql parser in Java https://github.com/apache/calcite

## 2023-08-16 Wed

I always feel overwhelmed by the new Apache projects in the data area. Today I
read a great post
https://www.clouddatainsights.com/real-time-olap-databases-and-streaming-databases-a-comparison/
that answers some of my confusions.

## 2023-08-10 Thu

Uff. I spent almost one week diving deep into TLS and finally get the blog post
out.

## 2023-07-30 Sun

I was bored yesterday and tried to dig a bit more into distributed databases,
so I found a Youtube video titled
[The Design & Architecture of CockroachDB Beta](https://www.youtube.com/watch?v=p8aJuk7TJJA).
Before watching it, I have a few questions on top of my mind:

1. How strong consistency is achieved? We are all familiar with the
   master-slave architecture, but in this architecture, only the master node is
   the source of truth. A distributed db should not use this architecture
   because we should not always route the traffic to the same node. So how does
   CockroachDB solve this issue?
2. How distributed transaction is achieved? two-phase commit?
3. I already heard that CockroachDB is built on top of RocksDB. How does it
   implement a SQL layer on top a KV store? And especially how secondary index
   is constructed?
4. Benchmark data compared to Mysql/PostgresDB.

After watching the video, I think my first question is resolved. It is based on
the Raft consensus algorithm and data replication. Basically, each write will
replicate data to multiple notes, and only majority of the node group thinks it
is done, it will return to the client. In this way, we have multiple source of
truth and any node has this replica can answer a strong-consistent query. Then,
I am more curious about the performance. This video does not answer my fourth
question though.

This video mentioned how transaction is implemented, but it is only briefly
mentioned. My first intuition is that this process has so many race conditions
that can break transaction. Also, it does not give many details about isolation
levels. My third question is briefly explained too, but I still do not know how
index is constructed and the performance.

Anyway, I need to read more.

I am back after surfing the internet for the whole afternoon! There are quite a
few blogs and videos discussing transactions in CockroachDB, but none of them
show the details I desire, so I was still lost until I read this paper
[RAFT based Key-Value Store with Transaction Support](http://www.scs.stanford.edu/20sp-cs244b/projects/RAFT%20based%20Key-Value%20Store%20with%20Transaction%20Support.pdf).
I would admit that I love short papers! This paper is not about CockroachDB,
but I believe CockroachDB uses the same idea. Basically, it is Raft + 2PC. The
transaction coordinators form a Raft group. The data shards form a group of
Raft groups. The transaction coordinator leader uses 2PC(two-phase commit)
algorithm to coordinate data shards leaders to commit a transaction.

So far, we figured out how distributed transaction works. The paper also
provides some benchmark data. Write transaction can take up to 5 seconds given
heavy loads! I am more curious to see CockroachDB's benchmark. We are still
left with the 3rd and 4th questions.

OK. After digging around for some more time, It is 11:41pm PST. I found this
[post](https://www.cockroachlabs.com/blog/sql-in-cockroachdb-mapping-table-data-to-key-value-storage/)
from CockroachDB team. It is well written. Now I understand how data is stored
in the kv store and how secondary indices are implemented. But how about joins?

Btw, I came across quite a few databases I probably need to dig deep into.

- RocksDB
- [BadgerDB](https://github.com/dgraph-io/badger)
- [BoltDB](https://github.com/boltdb/bolt). This is archived. There is a Hacker
  news post saying that the author of BadgerDB was disappointed by the
  performance of BoltDB, so he/she invented BadgerDB.
- [pebble](https://github.com/cockroachdb/pebble)
- TiKV

## 2023-06-20 Mon

TIL: https://wg21.link/index.html is the place to search all C++ Standards
Committee Papers. It is a really a wonderful place for me to dig into the
history of some topics.

## 2023-06-15 Thu

TIL: LLVM does have its libc++ library, but its libc library is not ready to
use. So it will use whatever libc library provided by OS: glibc, apple libc,
Musl, etc.

See more details in this post.
https://stackoverflow.com/questions/59019932/what-standard-c-library-does-clang-use-glibc-its-own-or-some-other-one

Also learnt the difference between `abort` and `exit`:
https://stackoverflow.com/questions/397075/what-is-the-difference-between-exit-and-abort

## 2023-06-10 Sat

TIL: https://github.com/haoNoQ/clang-analyzer-guide/releases

## 2023-06-07 Wed

Last time I said LLVM static analyzer is my next focus. But I found it is so
hard. I spent two days reading Prof. Anders Moller's book
[Static Program Analysis](https://cs.au.dk/~amoeller/spa/), and some companying
videos, called "DECA I" on Youtube. But still I found it not easy to follow.

Meanwhile, Prof. Claire Le Goues's course
[Program Analysis](https://clairelegoues.com/teaching.html) seems quite
interesting. I think it is better to learn it by doing projects.

## 2023-06-04 Sun

LLVM static analyzer will be my next focus.

## 2023-06-01 Thu

TIL:

> The final period or comma goes inside the quotation marks, even if it is not
> a part of the quoted material, unless the quotation is followed by a
> citation. If a citation in parentheses follows the quotation, the period
> follows the citation.

Grammarly helps!

## 2023-05-27 Sat

TIL: Split Infinitives

I started using grammarly and it pointed me a potential mistake like
`to easily use`. According to
https://www.niu.edu/writingtutorial/grammar/split-infinitives.shtml,

> An infinitive is a verb preceded by the word to: (to write, to examine, to
> take, to cooperate). When an adverb appears between to and the verb itself,
> we get a split infinitive.

## 2023-05-24 Wed

TIL: The Inverted Pyramid Structure

## 2023-05-16 Tue

Definitely will never read the memory paper from Ulrich. :)

## 2023-05-14 Sun

After reading a lot of ELF in the last 3 weeks. I will pick up mold source
code. I found that I can attach vscode to k8s. I will do that I want to dig
into x86 binary files.

But before that, I think I'd better read Ulrich Drepper's another paper first:
[What Every Programmer Should Know About Memory](https://www.akkadia.org/drepper/cpumemory.pdf).

## 2023-05-07 Sun

TIL: `static` keyword in C means local file scope. Thanks to Ulrich for his
wonderful paper "How to Write Shared Libraries".

## 2023-04-30 Sun

TIL: Ulrich Drepper is the author of glibc. He wrote quite a few good papers
about linux, core system etc. See the [list](https://www.akkadia.org/drepper/).

I will spend some time reading them. Some interesting onces:

- https://www.akkadia.org/drepper/
- https://www.akkadia.org/drepper/dsohowto.pdf

## 2023-04-29 Sat

TIL: The first C++ compiler is called
[Cfront](https://en.wikipedia.org/wiki/Cfront). Basically, it just translate
C++ to C.

## 2023-04-26 Wed

`r/ExperiencedDevs` is a really good sub reddit!

## 2023-04-19 Wed

Today I spent a little bit time reading dynamo paper
https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html. I did not
finish reading all.

## 2023-04-06 Thu

I got interested in ELF recently. After finishing major part of my boring Kafka
work, I decide to take some time to dive into ELF. And during this process, I
found this blog series about
[linker](https://www.airs.com/blog/page/4?s=Linkers). But why to study linker
for ELF though? Because these two are closed related. Without some linker
knowledge, I probably will never appreciate the value of the various sections
inside ELF. Ian Lance Taylor is great engineer and researcher. No doubt that I
will learn a lot from his blog posts.

## 2023-04-02 Sun

Today I finished following the
[kilo](https://viewsourcecode.org/snaptoken/kilo/) project! So fantastic! I
learnt a lot about terminal editor and picked up some C knowledge. The author
Salvatore Sanfilippo is also the author of Redis. I already followed his blog
on `feedly`. Hopefully, I will read more update from him in future.

Again, excited!

## 2023-03-30 Thu

Today I read a great blog about termios
https://blog.nelhage.com/2009/12/a-brief-introduction-to-termios/ Not only
termios, but it also helps me to differentiate console, terminal, shell. I may
read Nelson's other posts in future. I notice he was a staff engineer in stripe
working on Ruby static checker. That is well deserved! I wish I have such
strong technical capability one day!

## 2023-03-19 Sun

I built a wails app recently, and today I tried to package it as a .pkg file in
MacOS. This is a terrible experience. First, in order to publish it to Apple
store, you need to pay $99 per year for a developer account. Ok, I do not want
to pay. How about just making it only work for myself, namely, adding it to
`Launchpad`? I copied the full release folder to `/Applications`. Man, the app
immediately crashed every time I clicked the icon. Sure, I guess probably
launchpad virtual env does not allow port forwarding. But where are the app
logs? Nowhere! From below snippet, you see that fd 0, 1, 2 all point to
`/dev/null`. There are some posts online talking about how to log to syslog and
then use `Console` to view the logs. At this moment, I lost all my patience and
gave up. This reminded me of the dark days of configuring a windows server.

Fuck it! Both windows and macos suck!

```
$ ps -ef |grep wails
  502 24870     1   0  1:21PM ??         0:00.03 /bin/bash /Applications/wails-example.app/Contents/MacOS/wails-example
  502 24878  1703   0  1:21PM ttys017    0:00.00 grep wails
(website-py3_11_0) 2023-03-19 13:21:41 (bash) /Applications/wails-example.app/Contents/MacOS
$ lsof -p 24870
COMMAND   PID      USER   FD   TYPE DEVICE SIZE/OFF                NODE NAME
bash    24870 xiongding  cwd    DIR   1,13      640                   2 /
bash    24870 xiongding  txt    REG   1,13  1326688 1152921500312422323 /bin/bash
bash    24870 xiongding    0r   CHR    3,2      0t0                 335 /dev/null
bash    24870 xiongding    1u   CHR    3,2     0t72                 335 /dev/null
bash    24870 xiongding    2u   CHR    3,2      0t0                 335 /dev/null
bash    24870 xiongding  255r   REG   1,13      131           110694417 /Applications/wails-example.app/Contents/MacOS/wails-exampl
```
