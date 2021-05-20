+++
title = "Truncating a Kafka Topic Without Deleting It"
date = "2020-09-10"
tags = [ "Kafka" ]
categories = [ "Apache Kafka" ]
+++

There are times I would like to clear out a Kafka topic without deleting and recreating it. I want to purge the data w/out having to stop and application.

Usually this is a result of me creating an incorrect Avro schema.
Of course, I would never blame Avro for making it difficult for me to want to use `String` instead of their internal `Utf8` class when leveraging Generic Record.
How many times do I specify my string type as `"string"` instead of `{ "type": "string", "avro.java.string": "String"}`? You think I would have learned by now.
In any case, I want to remove this record that is causing my Kafka Streams topology to fail and I do not want to have to stop my CDC connector.

So, here is how I truncate a Kafka topic (only in development and also with proper parental supervision).  

```
kafka-configs --bootstrap-server localhost:19092 --entity-type topics --entity-name TEST_IT --alter --add-config 'retention.ms=1, retention.bytes=1, segment.ms=1'
```

```
kafka-configs --bootstrap-server localhost:19092 --entity-type topics --entity-name TEST_IT --alter --delete-config 'retention.ms, retention.bytes, segment.ms'
```

kacc='kafka-avro-console-consumer --bootstrap-server localhost:19092 --property print.key=true --property key.separator=\| --key-deserializer=org.apache.kafka.common.serialization.BytesDeserializer --from-beginning --topic '

