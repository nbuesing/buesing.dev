+++
title = "The process in collecting the versioning information"
date = "2022-02-26"
tags = [  ]
categories = [ "Apache Kafka", "Version" ]
+++

This is a quick overview of how the data in the versioning documentation is built.

<!--more-->

# Version Documents

* [Apache Kafka](/post/kafka-versions)
* [Confluent Community](/post/confluent-community-versions).


# Libraries

The library versions are based on introspection of the distribution tarballs, not through release notes.


This pulls in the majority of the dependencies for Apache Kafka

```
for i in $(ls -dr kafka_2.13*) $(ls -dr kafka_2.12*) $(ls -dr kafka_2.11*) $(ls -dr kafka_2.10*) $(ls -dr kafka_2.9*) $(ls -dr kafka_2.8*); do

gzcat $i | tar tfv - | (echo $i; grep -E 'scala-library|rocksdbjni-|zstd-jni|lz4-java|snappy-java|jackson-core|slf4j-log4j|zookeeper-[23]' | cut -d\/ -f2-; echo "")

done
```

For Confluent Community, the query is similar. Some libraries exist in multiple locations. 
For example jackson is in Schema Registry and Kafka. 
Sometimes the library versioning is slightly different, so more care to identify the library being used by the core component.

# .X releases

Confluent and Apache do bug fix releases independently, but Confluent typically merges the updates to the Apache distribution into their own.
Comparing commits and merge dates is a manual process, but can be used to figure out the differences in the bug fixes releases.

* [https://github.com/apache/kafka](https://github.com/apache/kafka)

* [https://github.com/confluentinc/kafka](https://github.com/confluentinc/kafka)