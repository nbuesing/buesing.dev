+++
title = "Abstract: Synchronous Commands Over Apache Kafka"
date = "2020-07-11"
tags = [ "Kafka", "RESTful", "CQRS" ]
categories = [ "Apache Kafka", "Kafka Streams", "Kafka Summit" ]
+++

You have built an event-driven system leveraging Apache Kafka. Now you face the challenge of integrating traditional synchronous request-response capabilities, such as user interaction, through an HTTP web service.

There are various techniques, each with advantages and disadvantages.  This talk discusses multiple options on how to do a request-response over Kafka — showcasing producers and consumers using single and multiple topics, and more advanced considerations using the interactive queries of ksqlDB and Kafka Streams.

Advanced considerations discussed:

* What a consumer rebalance means to your active request-responses.
* Discuss options for blocking for the async response in the web-service.
* How can the CQRS (Command Query Responsibility Segregation) be leveraged with the interactive state stores of Kafka Streams and ksqlDB?
* Interactive queries of the ksqlDB and Kafka Streams state stores are not available during a rebalance. What is the active Kafka development happening that will make interactive queries a more feasible option?
* Would a custom state store help with rebalancing limitations?
* Can custom partitioning be used for proper routing, and what impacts could that have to the other services in your ecosystem?

We will explore the above considerations with an interactive quiz application built using Apache Kafka, Kafka Streams, and ksqlDB. With a proper implementation in place, your request-response application can scale and be performant along with handling all of the requests.

### Kafka Summit Schedule

* My session will be presented on Monday, August 24th, 4:00 PM CDT (2:00 PM PDT)
* See complete [Kafka Summit 2020 Schedule](https://events.kafka-summit.org/2020-schedule) for any updates.
