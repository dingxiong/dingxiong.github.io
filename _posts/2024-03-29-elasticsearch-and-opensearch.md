---
layout: post
title: Elasticsearch and Opensearch
date: 2024-03-29 15:26 -0700
categories: [elk]
tags: [elk, elasticsearch, opensearch]
---

## Node discovery

Zen discovery

Reference:
https://www.elastic.co/guide/en/elasticsearch/reference/current/discovery-hosts-providers.html

Snapshot related logic is mainly contained in class.
[SnapshotsService](https://github.com/opensearch-project/OpenSearch/blob/5b4b4aa4c282d06a93de72a5c07a54a1524b04ff/server/src/main/java/org/opensearch/snapshots/SnapshotsService.java#L2693-L2693).
One thing to note when snapshotting is happening, you cannot delete index.
Otherwise, you will see
[SnapshotInProgressException](https://github.com/opensearch-project/OpenSearch/blob/5b4b4aa4c282d06a93de72a5c07a54a1524b04ff/server/src/main/java/org/opensearch/cluster/metadata/MetadataDeleteIndexService.java#L150-L150).

## Workflow

What is running inside an ES node?

```
elasticsearch@elasticsearch-master-0:~$ ps auxww
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
elastic+     1  0.0  0.0   2500   592 ?        Ss   13:43   0:00 /bin/tini -- /usr/local/bin/docker-entrypoint.sh eswrapper
elastic+     7  0.0  0.5 2599688 89504 ?       Sl   13:43   0:09 /usr/share/elasticsearch/jdk/bin/java -Xms4m -Xmx64m -XX:+UseSerialGC -Dcli.name=server -Dcli.script=/usr/share/elasticsearch/bin/elasticsearch -Dcli.libs=lib/tools/server-cli -Des.path.home=/usr/share/elasticsearch -Des.path.conf=/usr/share/elasticsearch/config -Des.distribution.type=docker -cp /usr/share/elasticsearch/lib/*:/usr/share/elasticsearch/lib/cli-launcher/* org.elasticsearch.launcher.CliToolLauncher
elastic+   170 71.3 32.0 428683128 5197536 ?   Sl   13:43 128:31 /usr/share/elasticsearch/jdk/bin/java -Des.networkaddress.cache.ttl=60 -Des.networkaddress.cache.negative.ttl=10 -Djava.security.manager=allow -XX:+AlwaysPreTouch -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -XX:-OmitStackTraceInFastThrow -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Dlog4j2.formatMsgNoLookups=true -Djava.locale.providers=SPI,COMPAT --add-opens=java.base/java.io=ALL-UNNAMED -Des.cgroups.hierarchy.override=/ -XX:+UseG1GC -Djava.io.tmpdir=/tmp/elasticsearch-7239405004551249449 -XX:+HeapDumpOnOutOfMemoryError -XX:+ExitOnOutOfMemoryError -XX:HeapDumpPath=data -XX:ErrorFile=logs/hs_err_pid%p.log -Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m -Xmx4g -Xms4g -XX:MaxDirectMemorySize=2147483648 -XX:G1HeapRegionSize=4m -XX:InitiatingHeapOccupancyPercent=30 -XX:G1ReservePercent=15 -Des.distribution.type=docker --module-path /usr/share/elasticsearch/lib --add-modules=jdk.net -m org.elasticsearch.server/org.elasticsearch.bootstrap.Elasticsearch
elastic+   191  0.0  0.0 114336  6164 ?        Sl   13:43   0:00 /usr/share/elasticsearch/modules/x-pack-ml/platform/linux-x86_64/bin/controller
```

There are two java processes. The
[CliToolLauncher class](https://github.com/elastic/elasticsearch/blob/5e1c859dc82843df708226e9d62c3f272c6bf6a9/distribution/tools/cli-launcher/src/main/java/org/elasticsearch/launcher/CliToolLauncher.java#L51-L51)
is the entry point. It loads `server-cli` which is process
[bootstrap.Elasticsearch](https://github.com/elastic/elasticsearch/blob/5e1c859dc82843df708226e9d62c3f272c6bf6a9/server/src/main/java/org/elasticsearch/bootstrap/Elasticsearch.java#L60-L60).
Why it has this design? Because ES also provides other
[command line tools](https://www.elastic.co/guide/en/elasticsearch/reference/current/commands.html)
which also use `CliToolLauncher` as an entry point.

Another thing to note is the memory usage. Elasticsearch uses 428G virtual
memory and 5G physical memory. Underneath, Lucene uses `mmap` to make searching
faster. So, ES official guideline is to use
[50%](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/advanced-configuration.html#set-jvm-heap-size)
memory for JVM heap, so the rest can be used for Lucene `mmap`. The
contributors of Lucene are even more aggressive:
[only reserve 25% for JVM heap](https://www.youtube.com/watch?v=hgF0jNxKrrg&t=1378s).
Here is
[relevant code](https://github.com/apache/lucene/blob/main/lucene/core/src/java/org/apache/lucene/store/MappedByteBufferIndexInputProvider.java#L121)
for Lucene memory mapped indices.

## Shards

According to
[this elastic doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html),
ES searches run on a single thread per shard, so making make shard smaller
leads to faster search, but in reality, more shards mean more overhead, so the
optimal size of a shard is 50GB. You can also use below query to see the size
of thread pool which determines the maximal parallelism.

```
GET /_cat/thread_pool/search?v=true&h=node_name,name,core,largest,max,qs,size,type
```

ES data stream is a good way to control shard size.

## Storage

Endpoint `/_cat/indices` tells basic information about ES indices. Example
below.

```
health status index                        uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   filebeat-7.10.0-2023.02.19   pCBMkxMzQ1CAwoOkkyLY3g   1   1   43446487            0    118.6gb         59.5gb
green  open   filebeat-7.10.0-2023.02.20   xdKlPEsiQqu4Bt6NimFifA   1   1   45494980            0    128.6gb         64.4gb
```

Take note of above `uuid` field. The index files are stored on disk at location
`data/indices<index_uuid>`. Inside this folder, you will see files with various
extensions: `.tid`, `.cfs`, and etc. There are Lucene index files. Check out
[this description](https://lucene.apache.org/core/9_5_0/core/org/apache/lucene/codecs/lucene95/package-summary.html)
if you are interested.

## Misc

How to get ES version?

```
/usr/share/elasticsearch/bin/elasticsearch --version
```

The Elasticsearch helm chat has a configuration `replicas` which means the
number of nodes, not the `index.number_of_replicas`. The number of replicas of
each shard is by default one unless changed.
