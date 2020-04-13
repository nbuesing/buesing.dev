+++
title = "Building a Mix Protocol Apache Kafka Cluster"
date = "2020-04-12"
tags = [ "SSL", "Kafka", "Security", "ACLs", "security.inter.broker.protocol", "PLAINTEXT" ]
categories = [ "Apache Kafka", "Cluster" ]
+++

## Introduction

Goal: build a multi-protocol Apache Kafka Clusters for __SSL Client Authentication__ for all clients while leveraging
__PLAINTEXT__ for inter broker communication.

There are many tutorials and articles on setting up Apache Kafka Clusters with different security options. However, I
have not seen a mix-protocol cluster with __SSL__ for encryption and client authentication and __PLAINTEXT__ for broker communication; so I decided to build one.

## tl;tr

1. Setup broker configuration of `super.users` for __SSL__ authenticated users, not for __PLAINTEXT__.
1. Start brokers with __SSL__ inter broker communication.
1. create full-access ACLs for user `User:ANONYMOUS`, but _only_ for the IPs of your brokers.
1. Change brokers to use __PLAINTEXT__ inter broker communication.
1. Do a rolling restart.

## Challenges 

The main challenges in setting up the cluster were: 

* Kafka broker authorizers are not protocol specific.

* Unlike __SASL__, __SSL__ authorization cannot be done over __PLAINTEXT__; it has to be over __SSL__ encryption.

* SSL Certificates are tricky, and those were addressed in [What I learned about SSL Certificates when building a Secured Kafka Cluster](/post/kafka-ssl-certificates). 

* While ACLs can allow resource access to a given principal from a given set of hosts, the broker's `super.users` is only a principal
setting.

## Configuration

* __SSL__

  Each broker needs to be configured with the truststore and keystore referencing the appropriate jks files.

  ```
  ssl.key.credentials = kafka.key
  ssl.keystore.credentials = kafka.key
  ssl.keystore.filename = broker-{ID}.keystore.jks
  ssl.truststore.credentials = kafka.key
  ssl.truststore.filename = kafka.server.truststore.jks
  ```

  Note: [Confluent platform docker images](https://github.com/confluentinc/cp-docker-images) has minor changes over a standard
  deployment; as in the use of `ssl.*.credentials` and files with those credentials instead of `ssl.*.passwords`.

* __Protocols and Listeners__

  The settings for protocols and listeners are standard. Both __SSL__ and __PLAINTEXT__ are needed.

  __[listener.security.protocol.map](https://kafka.apache.org/documentation/#brokerconfigs#:~:text=listener%2Esecurity%2Eprotocol%2Emap,-%3A)__

  The default value for this broker property is just fine, but if you prefer to only have configured what is needed; 
  consider setting it to:

  ```
  listener.security.protocol.map = PLAINTEXT:PLAINTEXT,SSL:SSL
  ```

  __[advertised.listeners](https://kafka.apache.org/documentation/#brokerconfigs#:~:text=advertised%2Elisteners,-%3A)__

  Each broker needs to established its list of listeners and advertises them in a way that the clients and other brokers can 
  directly access them. Standard 9092 and 9093 ports for `PLAINTEXT` and `SSL` used respectively.

  ```
  advertised.listeners = PLAINTEXT://broker-{ID}:9092,SSL://broker-{ID}:9093
  ```

* __Authorizer__

  __[authorizer.class.name](https://kafka.apache.org/documentation/#brokerconfigs#:~:text=authorizer%2Eclass%2Ename,-%3A)__

  To enable [client authorization](https://kafka.apache.org/documentation/#security_authz), set `authorizer.class.name`
  to `kafka.security.authorizer.AclAuthorizer`, the provided implementation of `org.apache.kafka.server.authorizer.Authorizer`. 

  ```
  authorizer.class.name = kafka.security.authorizer.AclAuthorizer
  ```

* __SSL Client Authentication__

  __[ssl.client.auth](https://kafka.apache.org/documentation/#brokerconfigs#:~:text=ssl%2Eclient%2Eauth,-%3A)__

  Set `ssl.client.auth` to `required` or `requested`.

  ```
  ssl.client.auth = required
  ```

  While you could set this to `requested` that would still allow SSL clients to not support client authentication. Unless
  you have a specific reason to need to do this, make sure you set this to `required`.

* __No ACL Found__

  [allow.everyone.if.no.acl.found](https://kafka.apache.org/documentation/#:~:text=allow%2Eeveryone%2Eif%2Eno%2Eacl%2Efound )

  Be sure to exclude the property `allow.everyone.if.no.acl.found` or set to `false`.  If you set this to `true`,
  it allows full access to a resource if no ACLs are found.

  ```
  allow.everyone.if.no.acl.found = false
  ```

* __Super Users__

  __[super.users](https://kafka.apache.org/documentation/#brokerconfigs#:~:text=super%2Eusers)__

  Do not add `User:ANONYMOUS` to `super.users`.  There is no way to restrict privileges to a given host. Instead, only add the 
  users of the `SSL` certificates for your brokers and the user you will use to create ACLs. This ensures that the cluster 
  is always secure. In addition to each broker's certificate added here; add in the root certificate so `kafka-acls` can be used to create the proper ACL.  

  The `super.users` configuration it should not be used for 
  authentication for __PLAINTEXT__. When SSL authorization is enabled, the user presented over __PLAINTEXT__ will always be `User:ANONYMOUS`.
  If you add this user to `super.users` anyone will be able to access your cluster (unless network restrictions prevent it).

  ```
  super.users = User:CN=root;User:CN=broker-1;User:CN=broker-2;User:CN=broker-3;User:CN=broker-4
  ```

* __Inter Broker Protocol__

  __[security.inter.broker.protocol](https://kafka.apache.org/documentation/#brokerconfigs#:~:text=security%2Einter%2Ebroker%2Eprotocol,-%3A)__

  This is a tricky setting. First, start the cluster with __SSL__.
   
  `security.inter.broker.protocol = SSL`
  
  Once the ACL for the brokers is created it change it to __PLAINTEXT__.
  
  `security.inter.broker.protocol = PLAINTEXT`

  The cluster will need to be restarted (following a proper rolling restart) for the change to take effect.

## Creating the Secure Cluster

With all these pieces in place, the process is straightforward.

1. Start Cluster
1. Create ACLs for a given superuser, `User:ANONYMOUS` only from IPs of brokers.
1. Change inter broker protocol on all the brokers from `SSL` to `PLAINTEXT`
1. Perform a rolling restart of the cluster

An alternate approach considered was temporarily adding __User:ANONYMOUS__ to the `super.users`, and remove it after the ACLs
are configured; to use this approach, ensure the cluster is not accessible until `User:ANONYMOUS` is part of `super.users`. 

## Example Cluster

The [Kafka SSL Cluster](https://github.com/nbuesing/kafka-ssl-cluster) repository provides an example cluster. It deploys on 
a single machine leveraging `docker-compose`. It showcases this configuration, it is not a production-ready cluster.

See Confluent's [quickstart documentation](https://docs.confluent.io/current/quickstart/ce-docker-quickstart.html) to understand
the differences in some of the configurations leveraging the Confluent Platform docker images.

Use the `jumphost` container to access the cluster.  Scripts to easily create the ACLs and test the cluster are mounted to this 
container.

Connect to that cluster using `docker exec`.
```
docker exec -it kafka_jumphost bash
cd /opt/bin
```

#### Example Code

An Example configuration is available at the GitHub repository [Kafka SSL Cluster](https://github.com/nbuesing/kafka-ssl-cluster).
It includes scripts to create certificates, script to swap changing protocol from __SSL__ to __PLAINTEXT__, a 
rolling restart script, and a dashboard. Please see its documentation for additional details.

To run commands to create ACLs, you will need a configuration like the following.

```
security.protocol=SSL
ssl.key.password=dev_cluster_secret
ssl.keystore.location=/certs/root.keystore.jks
ssl.keystore.password=dev_cluster_secret
ssl.truststore.location=/certs/kafka.server.truststore.jks
ssl.truststore.password=dev_cluster_secret
```

#### Super User ACL

Once the cluster is up and running with the security inter broker protocol of __SSL__, create an ACL based __superuser__
that can only be used by the hosts of the brokers.  With this, the unauthenticated user of `User:ANONYMOUS` has full operational
access to the cluster, but only from the hosts hosting the brokers. 

```
kafka-acls \
 --bootstrap-server broker-1:9093,broker-2:9093,broker-3:9093,broker-4:9093 \
 --command-config ./config/adminclient-config.conf \
 --add \
 --force \
 --allow-principal User:ANONYMOUS \
 --allow-host 10.5.0.101 \
 --allow-host 10.5.0.102 \
 --allow-host 10.5.0.103 \
 --allow-host 10.5.0.104 \
 --operation All \
 --topic '*' \
 --cluster
```

## Performance Testing

Now that we know it is feasible, is a mixed protocol cluster worth it? While using `docker-compose` to run a cluster on 
one machine is not the best way to property test; it does give insights on where it could lead.

#### acks=1

With `acks=1` theoretically, I would expect no change. However, with less encryption going on between brokers, the CPU
load would go down, so a small improvement is expected. My tests show a __7%__ improvement.

| security.inter.broker.protocol | records/sec | MB/sec | ms avg latency |
| --- | --- | --- | --- | --- | 
| SSL | 29,338.56 | 27.98 | 970.06 |
| PLAINTEXT | 31,512.17 | 30.06 | 926.85 |

#### acks=all 

With `acls=all` a larger improvement is expected. This is because the time for the insync replicates to replicate should be faster. 
The test shows a __44%__ improvement; which is greater than what I was expecting. I will need to move this to a realistic 
configuration and do additional testing. This result is rather promising and justifies doing more realistic cluster testing.

| security.inter.broker.protocol | records/sec | MB/sec | ms avg latency |
| --- | --- | --- | --- | --- | 
| SSL | 12,097.04 | 11.54 | 2,557.13 |
| PLAINTEXT | 17,418.82 | 16.62 | 1,732.00 |


### The Test

Here is the performance script I ran with the ACLs generated for the topic.

#### performance script

```
export BOOTSTRAP_SERVERS=broker-1:9093,broker-2:9093,broker-3:9093,broker-4:9093

kafka-producer-perf-test \
  --num-records 500000 \
  --record-size 1000 \
  --throughput -1 \
  --producer.config ./config/producer-config.conf \
  --producer-props \
    bootstrap.servers=${BOOTSTRAP_SERVERS} \
    acks=all \
    request.timeout.ms=60000 \
    retries=2147483647 \
  --topic $1
```

#### ACLs

```
kafka-acls \
 --bootstrap-server broker-1:9093,broker-2:9093,broker-3:9093,broker-4:9093 \
 --command-config ./config/adminclient-config.conf \
 --add \
 --allow-principal User:CN=jumphost \
 --allow-host '*' \
 --operation READ \
 --operation WRITE \
 --topic $1

kafka-acls \
 --bootstrap-server broker-1:9093,broker-2:9093,broker-3:9093,broker-4:9093 \
 --command-config ./config/adminclient-config.conf \
 --add \
 --allow-principal User:CN=jumphost \
 --allow-host '*' \
 --operation ALL \
 --group '*' 
```

## Conclusion

I hope this gives you some insights into Kafka security and possible configurations you could explore for performance
improvements. For me it was the journey to create a cluster configuration I hadn't seen before; giving me the opportunity
to fully understand the security configurations of Apache Kafka.

