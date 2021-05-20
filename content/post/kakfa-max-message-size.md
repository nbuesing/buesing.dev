+++
title = "Understanding Kafka's Max Message Size and RecordTooLargeException"
date = "2020-12-01"
tags = [ "Kafka", "Producer" ]
categories = [ "Apache Kafka", "Cluster" ]
+++

When using Apache Kafka it is important to understand the nuances of max message size, especially if you ever expect to have
to work with messages near the maximum message sizes of a topic. 
 
At the time of this article, Apache Kafka 2.6.0 is referenced as the current version.


## Key Takeaways

* How compression can be leveraged to handle messages larger than maximum message size.

* The differences between when a `RecordTooLargeException` is thrown from the client vs. thrown from the Broker.

* How changing just topic or broker setting is not enough.

* ...

## Log Fundamentals  

To better understand how to handle large messages and what to do to resolve issues with large messages, it is best to
have an understanding of some fundamentals around message format and batching.

### Message Format

Knowing how Kafka [Message Format](https://kafka.apache.org/documentation/#recordbatch) provides context to understanding the Kafka configurations. 
This format has remained unchanged it was introduced in Apache Kafka 0.11, and it has a header that is 61 bytes in length.  
See the [DefaultRecordBatch](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/common/record/DefaultRecordBatch.java
) class for more details.

```
		baseOffset: int64
		batchLength: int32
		partitionLeaderEpoch: int32
		magic: int8 (current magic value is 2)
		crc: int32
		attributes: int16
			bit 0~2 compresion (0=none, 1=gzip, 2=snappy, 3=lz4, 4=zstd)
			bit 3: timestampType
			bit 4: isTransactional (0=not transactional)
			bit 5: isControlBatch (0=not a control batch)
			bit 6~15: unused
		lastOffsetDelta: int32
		firstTimestamp: int64
		maxTimestamp: int64
		producerId: int64
		producerEpoch: int16
		baseSequence: int32
		records: [Record]
```

This format allows the broker to store the set of events as provided from the producer. This means if the producer was able to group 3 messages
together before sending the data to the broker, the broker stores those 3 messages together in the log file. Having the broker log file
handle the batch means the producer can compress the batch as a single unit (increasing compression performance in both CPU cycles and reduced message size), 
and it also means the broker does not have to decompress that single compressed message (provided the topic compression is set to the same as the producer or
set to honor whatever the producer sends).


### Batching

Configuration of the producer impacts batching. The settings of `linger.ms`, '`batch.size`, and `max.request.size` along 
with partition strategy impact batching. When keys are null (no order of messages is necessary), using _sticky partition strategy_, 
which is the default since Apache Kafka 2.5, improves batching.

If you have a topic with 3 partitions and produced 10 messages within the _linger window_, with the _round robin partition strategy_ 
you would most likely have messages 0,3,6,9 to one partition; 1,4,7 to the next partition, and 2,5,8 to the third partition. With _sticky partition strategy_
all 10 messages could be batch together to a single partition.  Again, provided the messages are smaller than `batch.size`, the time the producer needed
to create the messages was less than `linger.ms`, and the keys were all null.

### Verifying

From the brokers, uou can use `kafka-dump-log` to verify batching is working as expected. 

```
kafka-dump-log --files sample-0/00000000000000000000.log
```

Here is a message of three events produce to the broker from a single transfer of bytes from a producer.

```
baseOffset: 913 lastOffset: 915 count: 3 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 649418114 CreateTime: 1606676158150 size: 960094 magic: 2 compresscodec: NONE crc: 1052430509 isvalid: true
```

The values of `count`, `size`, and `compression` are of importance to quickly conclude if batching is indeed happening.

### Fundamental Summary

Having an understanding of how the client groups messages together helps with the understanding of the properties that impact messages size limits.

-----

## Large Messages

### Max Message Bytes

* A _topic_ has maximum message bytes. It either inherits the cluster's default of `message.max.bytes` or it 
is configured on the topic with `max.message.bytes`. As of Apache Kafka 2.5, the default value is `1048588`, 12 bytes larger than 1MiB.

* As mentioned earlier, the maximum message bytes applies to the batch of messages sent from the producer, not a given message in that batch.
Under default settings, a producer's batch will only contain 1 record, if that record is large (greater than `batch.size`).

* The producer has a configuration of `max.request.size`, and is used by the client to prevent messages of a given size being
send to the broker, in the first place. The default is 1 MiB (`1048576` bytes).

* Prior to 2.5, the `max.message.bytes` was 1MB (`100000` bytes) + 12 bytes for overhead, see [Issue 4203](https://issues.apache.org/jira/browse/KAFKA-4203) for further details.

### Topic 

* The maximum size of a message batch in a topic is controlled by the topic configuration `max.message.bytes`, if that is not
defined then the cluster's `message.max.bytes` is used.

* The applies to the bytes sent by the producer, if the topic's `compression.type` is `producer` or set to the same value as the producer,
it will be exactly the bytes sent (the nuances when message protocol differs is out of scope).

* If the producer sends the bytes with `gzip` compression, but the topic's compression is `uncompressed` then the max message size
will be checked on incoming message and when writing to disk.


### Producer Configuration

* Producer configuration of `max.request.size` and topic configuration of `max.message.bytes`.

* Producer's check of `max.request.size` is done prior to any compression.

; however, if a producer's configurations of
 `batch.size` and `max.request.size` properties are changed, it is possible.



### Producer Behavior

* The producer client will throw a `RecordTooLargeException`, if the uncompressed serialized bytes is greater than the 
producer configuration `max.request.size`. This is different than the broker's use of the `max.message.bytes` which is
used against the message as it is received and how it is stored (if storage compression format is different). 

* A message will only contain a single event, if a message is larger than `batch.size`.  The default `batch.size` is 16K (`16,384` bytes).
Even when producer. Even with [Optimizing Your Apache Kafka Deployment](https://www.confluent.io/white-paper/optimizing-your-apache-kafka-deployment/) of 200,000


### RecordTooLargeException

* The producer will throw a `RecordTooLargeException` if the serialized message, before compression, is greater than `max.request.size`.
The exception will be part of the callback or future and does not require any data transferred to the broker.

* The broker will throw a `RecordTooLargeExepction` if the serialized batch of records (as it would be compressed on the topic)
exceeds the producer's `max.message.bytes` (or cluster's `message.max.bytes`). To application developer using the producer client
library, the exception is handled the same way as if the client directly thew the exception, only the message of the exception is different.

### Large Messages Summary

* A large message exception can be thrown by either the client or the broker. With the default settings, it is extremely rare that
the broker would ever throw the exception. This is because the client checks against the uncompressed message, and the broker checks
against how it would be stored on the topic (usually compressed).  

* The broker is checking a batch of messages, but the client `batch.size` prevents large messages being batched with other messages.

...


## See It For Yourself

* Run producer with 1 message greater than the max request size but while the data is compressed.

```
gradle producer:run --args="--topic c1_gzip --compression gzip --max-request-size 300 --message-size 500"
```

```
org.apache.kafka.common.errors.RecordTooLargeException: 
The message is 586 bytes when serialized which is larger than 300, which is the value of the max.request.size configuration.
```

* Run the producer with 1 message greater than the max message size of the topic,

```
gradle producer:run --args="--topic c1_none --compression none --max-request-size 1100000 --message-size 1048577"
```

```
org.apache.kafka.common.errors.RecordTooLargeException:
The request included a message larger than the max message size the server will accept.
```

* Run the producer that is larger than max message size, if uncompressed...

```
gradle producer:run --args="--topic c1_gzip --compression gzip --max-request-size 1100000 --message-size 1048577"
```

NO ERRROR

*

```
 gradle producer:run --args="--topic c1_gzip --compression none --max-request-size 1200000 --message-size 1048577"
```

Even though data is compressed in flight,  
ERROR


Log.scala
```
          // re-validate message sizes if there's a possibility that they have changed (due to re-compression or message
          // format conversion)
          if (!ignoreRecordSize && validateAndOffsetAssignResult.messageSizeMaybeChanged) {
```


## Recommendations


* Use `producer` as compression type on a topic. This improves CPU utilization on broker. In addition, it also reduces a chance of getting
a message too large exception if a recompression results in a larger message (even though that would be rare).
  
* Setting the topic configuration of `max.message.bytes` to a larger value is insufficient. A producer will never send
the message greater than `max.request.size`.
  
* The client `max.request.size` is based on message prior to compression.

* Compression is based on key+headers+value...

















 
----------------------------

### 

* If you have JSON or highly verbose messages that have a high level of compression, you can adjust the producer's `max.request.size` 
to a large value and you ensure that compression is used; no modifications to the broker or topic would be needed.

* I have configured systems to handle uncompressed messages of 4MB in size and still manage to maintain the 1MiB message
size on the topic.

* The client configuratino of `max.request.size` is against 3 

### Advantages

What are the advantages of the producer batching events and the broker being able to handle the batch w/out having to break the batch apart 
into the individual messages?

* usually, compression of a batch of small messages is better than compressing the messages individually.

* if the topic honor the same compression that the producer uses, no CPU cycles on the brokers are needed to decompress
and re-compress the data.

* reduced the amount of bytes on disk needed to handle an event, since the header applies to the entire patch of events.

##


What is the benefits of having a record batches stored as a batch in the brokers?

The more a producer client can batch messages together, the less overhead; since the header is for the batch
of messages, not an individual message. Furthermore, compression of a batch of small messages has more advantages than
compressing each message.




## Batching

To better understand the complexities of large messages, it first helps to understand the batching of Kafka Producers. 
The [Message Format](https://kafka.apache.org/documentation/#messageformat) provides for multiple events to be sent together. 
To minimize work done in the broker and message overhead, the batch is stored as the producer sent it.  Furthermore, to improve
compression, the entire batch of messages is compressed together.  Additionally, if there is no change in compression from
the producer to the topic, no CPU processing is needed to uncompress the batch of messages.  

## Producer Batching

In order for the producer to successfully batch events together, the following needs to occur

* multiple messages need to be received by the client within prior to the producer sending the events; `linger.ms`  

* `batch.size`

* `max.request.size`

* sticky partitioner or keys hashed to the same partition


## How to handle messages larger than 1MB without changing the topic's maximum messages size


* Compression

* 

The example code [Max Message Size](https://github.com/nbuesing) 

https://issues.apache.org/jira/browse/KAFKA-8350



# Message Components

max.message.bytes -- topic


message.max.bytes -- broker


batch.size -- producer
max.request.size -- producer



# Message Batching

# Producer Settings

## max.batch.size

## max.request.size

## linger

## Compression

# RecordTooLargeException

# The Log File

# Why do I care

## What needs to be done to support large messages

## What does compression allow me to do

### 
