---
title: Kafka binary protocol
date: 2023-11-26 21:47 -0800
categories: [kakfa, binary-protocol]
tags: [kafka, nc]
---

## Kafka protocol

Kafka has its own binary protocol built on top of TCP to allow clients to talk
to it. See the
[official documentation](https://kafka.apache.org/0100/protocol.html). There
are less than twenty request types (ApiKey) in total, so it is not hard to code
them up. A natural question is why Kafka needs a customized protocol instead of
using existing protocol like HTTP, AMQP. The bottom of the official document
above answers this question :)

In this post, I illustrate how this binary protocol works by playing with the
metadata endpoint.

In kafka protocol, request and response share the same format:

```
RequestOrResponse => Size (RequestMessage | ResponseMessage)
  Size => int32
```

And request format is

```
RequestMessage => ApiKey ApiVersion CorrelationId ClientId RequestMessage
  ApiKey => int16
  ApiVersion => int16
  CorrelationId => int32
  ClientId => string
  RequestMessage => MetadataRequest | ProduceRequest | FetchRequest | OffsetRequest | OffsetCommitRequest | OffsetFetchRequest
```

Especially, metadata request takes form

```
Metadata Request (Version: 0) => [topics]
  topics => STRING
```

Let's build the request step by step.

First, we do not know the total size of the request body, so let's put it side.
The request message has a few fixed header fields. Metadata ApiKey is 3 and we
choose verion 0. For the rest, we use random values.

- ApiKey: 3
- ApiVersion: 0
- CorrelationId: 1
- ClientId: test

The body of meta request consists of a list of topics. If the list is empty,
then kafka returns metadata for all topics. For simplicity, we leave this list
empty. Based on this setup, we have the binary request as below

```
[0 0 0 18, 0 3, 0 0, 0 0 0 1, 0 4, 116 101 115 116, 0 0 0 0]

0 0 0 18 => totol bytes
0 3 => ApiKey
0 0 => ApiVersion
0 0 0 1 => CorrelationId
0 4 => length of ClientId
116 101 115 116 => ClientId: test
0 0 0 0 => size 0 meaning the length of topics list is zero
```

For readability, I use comma `,` to separate each field in the binary. Note
that total size 18 does not include itself. Also, Kafka protocol uses
big-endian encoding.

We can use `nc` to send this request to kafka. Assume we are inside a kafka
broker and the port is 9092.

```
echo -ne '\x00\x00\x00\x12\x00\x03\x00\x00\x00\x00\x00\x01\x00\x04\x74\x65\x73\x74\x00\x00\x00\x00' | nc localhost 9092 -x ~/response.txt
```

The above `\x00\x00...` is the hex representation of the byte array above.
Check out this
[post](https://unix.stackexchange.com/questions/82561/convert-a-hex-string-to-binary-and-send-with-netcat)
to learn how to do the convention.

OK. Let's check the response! The response has format

```
Response => CorrelationId ResponseMessage
CorrelationId => int32
ResponseMessage => MetadataResponse | ProduceResponse | FetchResponse | OffsetResponse | OffsetCommitResponse | OffsetFetchResponse
```

and specific to metadata ApiKey,

```
Metadata Response (Version: 0) => [brokers] [topic_metadata]
  brokers => node_id host port
    node_id => INT32
    host => STRING
    port => INT32
  topic_metadata => topic_error_code topic [partition_metadata]
    topic_error_code => INT16
    topic => STRING
    partition_metadata => partition_error_code partition_id leader [replicas] [isr]
      partition_error_code => INT16
      partition_id => INT32
      leader => INT32
      replicas => INT32
      isr => INT32
```

Below is what we get.

```
[kafka@kafka-main-kafka-0 bin]$ cat -e ~/response.txt
[0000]   00 00 00 12 00 03 00 00   00 00 00 01 00 04 74 65   ........ ......te$
[0010]   73 74 00 00 00 00                                   st....$
[0000]   00 00 08 80 00 00 00 01   00 00 00 02 00 00 00 00   ........ ........$
[0010]   00 35 6B 61 66 6B 61 2D   6D 61 69 6E 2D 6B 61 66   .5kafka- main-kaf$
[0020]   6B 61 2D 30 2E 6B 61 66   6B 61 2D 6D 61 69 6E 2D   ka-0.kaf ka-main-$
[0030]   6B 61 66 6B 61 2D 62 72   6F 6B 65 72 73 2E 6B 61   kafka-br okers.ka$
[0040]   66 6B 61 2E 73 76 63 00   00 23 84 00 00 00 01 00   fka.svc. .#......$
[0050]   35 6B 61 66 6B 61 2D 6D   61 69 6E 2D 6B 61 66 6B   5kafka-m ain-kafk$
[0060]   61 2D 31 2E 6B 61 66 6B   61 2D 6D 61 69 6E 2D 6B   a-1.kafk a-main-k$
[0070]   61 66 6B 61 2D 62 72 6F   6B 65 72 73 2E 6B 61 66   afka-bro kers.kaf$
[0080]   6B 61 2E 73 76 63 00 00   23 84 00 00 00 04 00 00   ka.svc.. #.......$
[0090]   00 15 5F 5F 73 74 72 69   6D 7A 69 5F 73 74 6F 72   ..__stri mzi_stor$
[00a0]   65 5F 74 6F 70 69 63 00   00 00 01 00 00 00 00 00   e_topic. ........$
[00b0]   00 00 00 00 01 00 00 00   02 00 00 00 01 00 00 00   ........ ........$
[00c0]   00 00 00 00 02 00 00 00   00 00 00 00 01 00 00 00   ........ ........$
[00d0]   08 66 69 6C 65 62 65 61   74 00 00 00 04 00 00 00   .filebea t.......$
[00e0]   00 00 00 00 00 00 01 00   00 00 02 00 00 00 01 00   ........ ........$
[00f0]   00 00 00 00 00 00 02 00   00 00 00 00 00 00 01 00   ........ ........$
[0100]   00 00 00 00 02 00 00 00   01 00 00 00 02 00 00 00   ........ ........$
[0110]   01 00 00 00 00 00 00 00   02 00 00 00 01 00 00 00   ........ ........$
[0120]   00 00 00 00 00 00 03 00   00 00 00 00 00 00 02 00   ........ ........$
[0130]   00 00 00 00 00 00 01 00   00 00 02 00 00 00 00 00   ........ ........$
[0140]   00 00 01 00 00 00 00 00   01 00 00 00 00 00 00 00   ........ ........$
[0150]   02 00 00 00 00 00 00 00   01 00 00 00 02 00 00 00   ........ ........$
[0160]   00 00 00 00 01 00 00 00   37 5F 5F 73 74 72 69 6D   ........ 7__strim$
[0170]   7A 69 2D 74 6F 70 69 63   2D 6F 70 65 72 61 74 6F   zi-topic -operato$
[0180]   72 2D 6B 73 74 72 65 61   6D 73 2D 74 6F 70 69 63   r-kstrea ms-topic$
[0190]   2D 73 74 6F 72 65 2D 63   68 61 6E 67 65 6C 6F 67   -store-c hangelog$
[01a0]   00 00 00 01 00 00 00 00   00 00 00 00 00 01 00 00   ........ ........$
...
```

The first two rows are the request, which are saved by `nc` and they are not
part of the response. Response starts from line 3. The first int32
`00 00 08 80 => 2176` is the total size of the response. `00 00 00 01` is the
CorrelationId. The rest is broker and topic metadata. `00 00 00 02` means there
are two brokers. Interpreting the rest binary is pretty hard in a shell or by
eye-spying, so it is better to write a program to systematically decode it. All
kafka client needs to do it. For example, check out
[sarama's deocder](https://github.com/Shopify/sarama/blob/main/real_decoder.go).
I also write a simple program just to decode metadata response. See this
[link](https://github.com/dingxiong/career/blob/master/tutorials/go-kafka-sarama/pkg/metadata.go).

In the last, I hope you know how kafka protocol works after reading this
article :)
