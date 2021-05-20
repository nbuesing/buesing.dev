+++
title = "Kafka Summit Europe 2021 Recap, so far..."
date = "2021-05-19"
tags = [ "Kafka" ]
categories = [ "Apache Kafka", "Kafka Streams", "Kafka Summit" ]
+++

# Kafka Summit Europe 2021 Recap, so far...

This Kafka Summit, like the ones before it, was fantastic. Tons of content to choose from and an excellent virtual venue to use to consume that content. I highly recommend watching many of the sessions on demand. I am working through more content myself, but best to share what I have uncovered so far before I wait until I see everything I want to see.

## "Mistakes - I've Made a Few. Blunders in Event-driven Architecture"
###Simon Aubury, Thoughtworks

One of the best ways to learn is to review what you have done and evaluate what was done well and what you would have done differently. Simon is an excellent storyteller, and his insights are of great value. If you are looking to build an event-driven system (ideally in Kafka, but even if you are exploring other technologies), this is a must-watch session.

"Passive Aggressive Events" - worth watching for this call-out alone; it is one of many gems in this session.

## "On Track with Apache Kafka: Building a Streaming ETL Solution with Rail Data"
###Robin Moffat, Confluent

Simply put, this presentation is pragmatic, complete, and elegant. Robin always does a great job on provided comprehensive end-to-end demonstrations. I love the live demonstrations against the real-time rail data. Geospatial data is a great tool to understand event streaming; this presentation is excellent for anyone in their development journey with event streaming.

Robin refers to one of his examples as being hacky, but I find it quite pragmatic. Using curl and kafkacat and a few other standard UNIX commands to get data into Kafka is what developers want to do when proving out and validating their event pipeline. Robin works to help developers get there quickly and effectively.

## "Event-driven Applications with Kafka, Micronaut, and AWS Lambda"
### Dave Klein, Confluent

This is an excellent introduction to many technologies. Dave Klein is a software developer that has written many applications for enterprise companies. He has the background to provide and advocate for developing applications leveraging Kafka. Do you like cloud services and minimal software footprint; look here. If you want to build a Kafka microservice quickly, also look here.

Confluent Cloud, a sink connector for AWS lambda, a Micronaut application built from using the Micronaut Launch website to scaffold your application together quickly, all shown here in about 30 minutes. Lots of content to get you to an end-to-end-solution demonstration rather quickly.

## "Temporal-Joins in Kafka Streams & ksqlDB"
### Matthias Sax, Confluent

Kafka Streams is an event-based processing engine leveraging Kafka to make that possible. KsqlDB is an abstraction built on Kafka Streams. There is a lot you can do with this library for building your streaming solutions. However, building deterministic streaming applications is quite challenging, and Kafka Streams and the use of temporal semantics is one way to get there. As Matthias says, "Temporal joins are a key concept of stream processing."

Matthias's ability to distill temporal semantics is excellent. He provides clear examples to help you understand complicated scenarios. It feels like Matthais has built a fantastic progression of presentations over the last few years on Kafka Streams and the importance of understanding time when doing event processing; check those out too.

## "Advanced Change Data Streaming Patterns in Distributed Systems"
### Gunnar Morling, RedHat and Hans-Peter Grahsl, Netconomy\

Integration with legacy databases seems to be a cornerstone to librating data to be used in event-driven systems. Leveraging change-data-capture tooling to a database is the first and easy step for doing this process. Providing meaningful higher-level abstractions of that data is where it gets more complicated.

Outbox Pattern, Strangler fig Pattern, and Saga patterns are critical patterns to making change-data-capture successful. Both Gunnar and Hand-Peter provide an excellent overview. I have been a fan of Debeizum for a long time now, and glad it is used here to demonstrate these patterns.

## "Streaming With Structure"
### Kate Stanley and Salma Saeed, IBM

Data marshaling and governance are critical in building successful event-driven solutions. Understanding the use of a schema registry, how serializers work is essential. While developers typically do not need to write custom serializers, you need to know this. It is critical to have a core foundation to these concepts to support evolving applications, and Kate and Salma do an excellent job in providing you this foundation.

As indicated in this talk by Kate, do also check out the related presentation; it discusses an additional step to consider when adding structure content and the rules around that structure content to your organization.

## "Getting Started with AsyncAPI: How to Describe Your Kafka Cluster"
### Dale Lane and Salma Saeed, IBM

AsyncAPI is a mechanism to document your structure data so others in your organization can discover and leverage them accordingly.  If you have built an extensive set of topics and content within your Apache Kafka Ecosystem, you want to allow teams to use and leverage that data; instead of them going off and building it themselves. So documentation is vital and using standard ways of documenting saves time and helps prevent that documentation from becoming stale or out of date, since having a single unified way to document across your organization will help.

I know I am now going to find time to explore AsyncAPI.

## "Developing a Custom Kafka Connector? Make it Shine!"
### Igor Buzatović, Porche Digital Croatia d.o.o.

If you want to integrate a system with Kafka, it is best to use Kafka Connect, allowing organizations to bring that connection into an existing ecosystem. Igor does a great job of getting you up to speed in doing so.

In addition, Igor provides the best summary of the breakdown of partitions and tasks for both source connectors and sink connectors and how they are different.

Error handling with connectors is critical, and the call-out to these is essential. Even if you are not going to write a custom connector, you can learn how to configure connectors to run effectively in your ecosystem.

## "Not Your Mother's Kafka - Deep Dive into Confluent Cloud Infrastructure"
### Gwen Shapira, Confluent

Gwen pulls back the curtain and showcases some of what Confluent Cloud does to support their clients and meet those service level agreements. It is awe-inspiring.

Gwen always provides excellent references to the related KIPs (Kafka Improvement Proposals) related, when applicable. I find this extremely helpful for me to dive into something deeper or make sure I understand things.

Also, the slide on 100MB/s is incredible. I will not tell you where it is since you need to watch the entire presentation and not just this slide, but I found this quite eye-opening and will let it speak for itself.

## "Managing Multiple Event Types in a Single Topic with Schema Registry"
### Bill Bejeck, Confluent

Even before my involvement with Kafka, most of my messaging-based applications and RESTful applications were based on domain-driven design and bounded context. Bill showcases how to build successful Kafka-based applications where the message contract is essential. How do you manage messages based on the topic, the message (event), or both.

In addition to showcasing all of the above, examples are provided in both Avro and Protobuf. The differences between them are seen because of the excellent presentation in giving us both.

## "Look How Easy it is, to go From Events to User-facing Realtime Analytics!"
### Tim Berglund, Confluent and Neha Pawar, StarTree

The key to leveraging Kafka effectively is understanding leveraging other tools and systems with the event-streaming data. I enjoyed seeing how Apache Kafka integrates with other systems, in this case, Pinot.  Analytic Databases are a key component to many companies' toolset; making sure systems work and integrate are essential for successful enterprise solutions.

In a recent episode of [Between Chair and Keyboard](https://tanzu.vmware.com/developer/tv/bcak/28/), Nate Shutta interviews Tim Berglund.  Tim comments that you can do more by training 15 people than doing the work yourself (not sure on the exact number).  Well, it is safe to say I am one of those people he has trained, and I am confident in saying he has taught so many people in Apache Kafka. I love this Kafka Summit Session because it is his commitment to continue to do that education to others.

Thanks, Tim, for continuing to inspire, and thanks Naha for doing that for another technology.


## What's next

I will continue to explore the content; I hope to find another 11 sessions for me to watch and encourage others to explore.






