---
layout: post
title: How to connect to zookeeper?
date: 2023-12-08 00:13 -0800
categories: [zookeeper, misc]
tags: [zookeeper, kafka, TLS]
---

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

We know that Kafka metadata such as locations of partitions is stored in
Zookeeper. Out of curiosity, I decide to take a look inside Zookeeper. However,
this seems not as simple as I thought. Before continuing to the subject, Let me
quickly summarize the set-up. We have a kafka-v3.3.1 cluster inside a AWS EKS
1.24 cluster. Kafka was installed by Strimzi controller. Below are relevant
pods

```
$ k get pods
NAME                                          READY   STATUS    RESTARTS      AGE
kafka-main-entity-operator-65d965684f-gnqq2   3/3     Running   4 (29d ago)   29d
kafka-main-kafka-0                            1/1     Running   0             16d
kafka-main-kafka-1                            1/1     Running   0             16d
kafka-main-zookeeper-0                        1/1     Running   0             16d
strimzi-cluster-operator-f696c85f7-rkwvz      1/1     Running   4 (14d ago)   30d
```

First, I read the Zookeeper official website and learn that `zkCli.sh` is the
command line client to use. However, I cannot find it anywhere inside the
Zookeeper pod. Then, I find this script
[zookeeper-shell.sh](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/bin/zookeeper-shell.sh#L23-L23).
Ah! Kafka team built their own Zookeeper command line client. This script needs
a `host:port` argument and an optional `-zk-tls-config-file`. This tls config
file is every important for the discussion below.

Ok, as naive as me. I cannot wait to try it out as below.

```
[kafka@kafka-main-zookeeper-0 kafka]$ bin/zookeeper-shell.sh localhost:2181
```

Damn it! I got below error from the zookeeper pod

```
2023-03-06 02:05:49,587 ERROR Unsuccessful handshake with session 0x0 (org.apache.zookeeper.server.NettyServerCnxnFactory) [nioEventLoopGroup-7-1]
2023-03-06 02:05:49,587 WARN Exception caught (org.apache.zookeeper.server.NettyServerCnxnFactory) [nioEventLoopGroup-7-1]
io.netty.handler.codec.DecoderException: io.netty.handler.ssl.NotSslRecordException: not an SSL/TLS record: 0000002d000000000000000000000000000075300000000000000000000000100000000000000000000000000000000000
```

It turns out that from strimzi-0.5.0,
[the communication with Zookeeper is encrypted via TLS](https://github.com/strimzi/strimzi-kafka-operator/issues/673).
I guess I need to spend some time, hopefully not hours to read code surrounding
`-zk-tls-config-file`.

The code that parse this tls config file is
[here](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/admin/ZkSecurityMigrator.scala#L128-L128)
and
[here](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/server/KafkaConfig.scala#L321-L321).
I am a newbie to TLS, I have no idea about the meaning of `keystore.type`,
`truststore.type` etc. But I definitely know that Kafka can successfully talk
to Zookeeper, so we can copy Kafka's configuration! Here is the
[code](https://github.com/strimzi/strimzi-kafka-operator/blob/f7f6e901810af1df1b9f437b7bd3d0c722a58510/cluster-operator/src/main/java/io/strimzi/operator/cluster/model/KafkaBrokerConfigurationBuilder.java#L188-L188)
of how strimzi build the config, and
[kafka_run.sh script](https://github.com/strimzi/strimzi-kafka-operator/blob/f7f6e901810af1df1b9f437b7bd3d0c722a58510/docker-images/kafka-based/kafka/scripts/kafka_run.sh#L52-L52),
dumps the config to file `/tmp/strimzi.properties`. Below is the Zookeeper
section in this file. I do not need to copy this section out to a new file.
Instead, I can use this whole file because irrelevant properties will be
ignored by
[Utils.loadProps](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/admin/ZkSecurityMigrator.scala#L128-L128).

```
##########
# Zookeeper
##########
zookeeper.connect=kafka-main-zookeeper-client:2181
zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
zookeeper.ssl.client.enable=true
zookeeper.ssl.keystore.location=/tmp/kafka/cluster.keystore.p12
zookeeper.ssl.keystore.password=[...]
zookeeper.ssl.keystore.type=PKCS12
zookeeper.ssl.truststore.location=/tmp/kafka/cluster.truststore.p12
zookeeper.ssl.truststore.password=[...]
zookeeper.ssl.truststore.type=PKCS12
```

Let's do it!

```
k cp kafka-main-kafka-0:/tmp/strimzi.properties ~/tmp/strimzi.properties
k cp kafka-main-kafka-0:/tmp/kafka/cluster.keystore.p12 ~/tmp/kafka/cluster.keystore.p12
k cp kafka-main-kafka-0:/tmp/kafka/cluster.truststore.p12 ~/tmp/kafka/cluster.truststore.p12

# I double checked that these 3 files do not exit in zookeeper pod, so we can safely copy them.
k cp ~/tmp/strimzi.properties kafka-main-zookeeper-0:/tmp/
k cp ~/tmp/kafka/ kafka-main-zookeeper-0:/tmp/kafka/
```

With this setup, Let's do it again.

```
[kafka@kafka-main-zookeeper-0 kafka]$ bin/zookeeper-shell.sh localhost:2181 -zk-tls-config-file ~/tmp/strimzi.properties
```

It failed with below server log :(.

```
2023-03-06 02:07:55,295 WARN Exception caught (org.apache.zookeeper.server.NettyServerCnxnFactory) [nioEventLoopGroup-7-1]
io.netty.handler.codec.DecoderException: javax.net.ssl.SSLHandshakeException: Received fatal alert: certificate_unknown
        at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:480)
        ...
        at java.base/java.lang.Thread.run(Thread.java:829)
Caused by: javax.net.ssl.SSLHandshakeException: Received fatal alert: certificate_unknown
        at java.base/sun.security.ssl.Alert.createSSLException(Alert.java:131)
        at java.base/sun.security.ssl.Alert.createSSLException(Alert.java:117)
        at java.base/sun.security.ssl.TransportContext.fatal(TransportContext.java:340)
        at java.base/sun.security.ssl.Alert$AlertConsumer.consume(Alert.java:293)
        at java.base/sun.security.ssl.TransportContext.dispatch(TransportContext.java:186)
        at java.base/sun.security.ssl.SSLTransport.decode(SSLTransport.java:172)
        at java.base/sun.security.ssl.SSLEngineImpl.decode(SSLEngineImpl.java:681)
        ...
```

Now, I said to myself: I need to deep dive into TLS. Blow material is a crush
course on TLS!

As we know, TLS stands for Transport Layer security. It is a way to build
secure connection between client and server such that no other body can
eavesdrop the communication. At the handshake stage, sever needs to present its
TLS certificate to client to assure client that the message is indeed from this
server. Optionally, server may also requests client to present its certificate.
This is called mTLS (mutual TLS). For more info, check out my other post
[security/tls.md](../security/tls.md).

By default, Zookeeper uses mTLS if TLS is enabled. See this
[code](https://github.com/apache/zookeeper/blob/df320056140a49423718ee6f0a7c35538281176e/zookeeper-server/src/main/java/org/apache/zookeeper/common/X509Util.java#L129-L141).
In our case, the error "seems" to be client side certificate is unknown. So if
we can disable client-side TLS, then the problem is resolved. The sad thing is
that Kafka does not provide this option for you.
[Code](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/server/KafkaConfig.scala#L321-L321)
omits
[client-side auth option](https://github.com/apache/zookeeper/blob/df320056140a49423718ee6f0a7c35538281176e/zookeeper-server/src/main/java/org/apache/zookeeper/common/X509Util.java#L163).
What a pity!

As you see from the above zookeeper config. There are two concepts `keystore`
and `truststore`. These two files belong to Java's TLS implementation.
Basically, `keystore` stores your own certifcates and public keys .
`truststore` stores others' certificates that you allow. Read more
[here](https://www.baeldung.com/java-keystore-truststore-difference). It is
slightly similar to ssh's `id_rsa` and `known_hosts`. Anyway, Java provides a
nice command line tool to inspect `keystore` and `truststore`.

```
[kafka@kafka-main-zookeeper-0 kafka]$ keytool -list -v -keystore /tmp/zookeeper/cluster.keystore.p12 -storepass <...>
...
Keystore type: PKCS12
Keystore provider: SUN

Your keystore contains 1 entry

Certificate[1]:
Owner: CN=kafka-main-zookeeper, O=io.strimzi
Issuer: CN=cluster-ca v0, O=io.strimzi
...
Extensions:

#1: ObjectId: 2.5.29.17 Criticality=false
SubjectAlternativeName [
  DNSName: kafka-main-zookeeper-client.kafka.svc
  DNSName: kafka-main-zookeeper-0.kafka-main-zookeeper-nodes.kafka.svc.cluster.local
  DNSName: *.kafka-main-zookeeper-nodes.kafka.svc
  DNSName: kafka-main-zookeeper-client
  DNSName: kafka-main-zookeeper-0.kafka-main-zookeeper-nodes.kafka.svc
  DNSName: kafka-main-zookeeper-client.kafka.svc.cluster.local
  DNSName: kafka-main-zookeeper-client.kafka
  DNSName: *.kafka-main-zookeeper-nodes.kafka.svc.cluster.local
  DNSName: *.kafka-main-zookeeper-client.kafka.svc
  DNSName: *.kafka-main-zookeeper-client.kafka.svc.cluster.local
]
...
```

Above shows the keystore inside the zookeeper node. Node the
`SubjectAlternativeName` section. At the TLS handshake stage, client will check
server's identity, namely, whether server's hostname is the same as stated in
the sever certificate. If not, then handshake fails. See
[rfc2818-section-3.1](https://datatracker.ietf.org/doc/html/rfc2818#section-3.1).
Quote from there:

> If a subjectAltName extension of type dNSName is present, that MUST be used
> as the identity.

Basically, `SubjectAlternativeName` allows one certificate can be used for
multiple different hosts. The relevant jdk implementation is
[here](https://github.com/openjdk/jdk/blob/dc08216f0ef55970c96df43bcc86ebd5792d486e/src/java.base/share/classes/sun/security/ssl/X509TrustManagerImpl.java#L457-L457).
Ah! So the issue is that client found the server's hostname and certificate
does not match. It is not the issue with client's certificate. At this moment,
we know how to resolve this issue: just replace `localhost` with one of the
alternative names above.

```
$ bin/zookeeper-shell.sh kafka-main-zookeeper-0.kafka-main-zookeeper-nodes.kafka.svc.cluster.local:2181 -zk-tls-config-file /tmp/strimzi.properties

Connecting to kafka-main-zookeeper-0.kafka-main-zookeeper-nodes.kafka.svc.cluster.local:2181
Welcome to ZooKeeper!
JLine support is disabled

WATCHER::
WatchedEvent state:SyncConnected type:None path:null

ls /
[admin, brokers, cluster, config, consumers, controller, controller_epoch, feature, isr_change_notification, latest_producer_id_block, log_dir_event_notification, zookeeper]

ls /brokers
[ids, seqid, topics]

ls /brokers/topics
[__consumer_offsets, __strimzi-topic-operator-kstreams-topic-store-changelog, __strimzi_store_topic, filebeat, xiong-test-kafka-producer]

get /brokers/topics/filebeat
{"partitions":{"0":[1,0],"1":[0,1],"2":[1,0],"3":[0,1]},"topic_id":"zeTNbNaeRS-h5yGiMZBOyg","adding_replicas":{},"removing_replicas":{},"version":3}
```

Finally, I can connect to Zookeeper!
