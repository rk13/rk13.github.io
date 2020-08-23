---
layout: post
title:  "Generic KafkaDeserializationSchema for FLINK"
date:   2020-08-01 01:00:00
---

![stream of events](/resources/fabrizio-chiagano-YhnODmrg8hY-unsplash.jpg)
<span align="center">Photo by <a href="https://unsplash.com/@fabriziochiagano?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Fabrizio Chiagano</a> on <a href="https://unsplash.com/s/photos/digital?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


# Stream processing and Apache Kafka

Stream processing and [Apache Kafka](https://kafka.apache.org/) is Klarna's essential piece of infrastructure that enables engineering [data-driven culture](https://www.forbes.com/sites/forbestechcouncil/2020/01/22/why-a-data-driven-culture-matters-and-how-to-get-there/) and the ability to move forward with our new projects quickly.

At Klarna decision services, we rely heavily on Kafka to process millions of data points streaming to us every day. We need to be able to provide easy to use ad-hoc analytics, aggregate these data points as they stream into our systems and also run arbitrary transformations to compute features and variables used later in the decisioning process, alert events and anomalies.


# Apache Flink and AWS Kinesis Data Analytics

[Apache Flink](https://flink.apache.org/) is a scalable and fault-tolerant processing framework for streams of data based on the idea that it should not be hard to express computations (like AVG or GROUP BY) while still be able to scale indefinitely, in a fault-tolerant manner.

In its core, it is the JVM based framework that was developed specifically for stateful computations over data streams. The project itself has an active open source community, both used and contributed by many companies that require to process large amounts of data.

![alt text](https://flink.apache.org/img/flink-home-graphic.png)

[Kinesis Data Analytics](https://aws.amazon.com/kinesis/data-analytics/) was [announced](https://aws.amazon.com/about-aws/whats-new/2019/11/you-can-now-run-fully-managed-apache-flink-applications-with-apache-kafka/) by AWS In November 2019. 

It is an AWS managed runtime environment for Apache Flink applications. Since that time using Apache Flink on AWS Kinesis Data Analytics is getting momentum for us given it important features:
- supports checkpoints and snapshots, that allows easy recovery from the offset that we reached in Kafka
- treats batch data sources and streaming data sources same way, so we can use the same implementation for backfill and streaming parts of our pipeline
- low latency computations with autoscaling options by AWS Kinesis Data Analytics
- Java-based on out-of-the-box integration with [Apache Kafka](https://kafka.apache.org/), [Kinesis Data Streams](https://aws.amazon.com/kinesis/data-streams/) and [Apache Avro](http://avro.apache.org/).


# Flink connector for Kafka

In this article, I will be sharing our experience and key learnings of using Amazon Kinesis Data Analytics and Flink for processing Kafka events encoded using Apache Avro and discuss how to implement custom deserialization schema in Flink and why you might need that. 

As the driving use case consider the solution that is responsible for a) ingesting trace events from different decision services via data consumption from different Kafka topics, b) making some non-sophisticated data massaging, and c) persisting the data for later ad-hoc analysis.

![flink-use-case](/resources/2020-08-flink-use-case1.png)

Flink provides Apache Kafka connector for reading data from and writing data to Kafka topics out of box using [FlinkKafkaConsumer](https://ci.apache.org/projects/flink/flink-docs-stable/dev/connectors/kafka.html).

In order to use `FlinkKafkaConsumer` you normally configure 
* The topic name / list of topic names
* A `DeserializationSchema` for deserializing the data from Kafka
* Properties for the Kafka consumer, e.g.

```java
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "");
properties.setProperty("group.id", "");
```

Example of using `FlinkKafkaConsumer` in you application
```java
DataStream<String> stream = env.addSource(
new FlinkKafkaConsumer<>(
   "topic", 
new SimpleStringSchema(), 
properties));
```

In order to define how to turn the binary data in Kafka into Java/Scala objects you need to select the implementation of [DeserializationSchema](https://ci.apache.org/projects/flink/flink-docs-release-1.9/api/java/org/apache/flink/streaming/util/serialization/DeserializationSchema.html)


Flink provides the following schemas out of the box
* [JsonNodeDeserializationSchema](https://ci.apache.org/projects/flink/flink-docs-master/api/java/org/apache/flink/formats/json/JsonNodeDeserializationSchema.html) (from `org.apache.flink:flink-json` library) that turns the serialized JSON into an ObjectNode object
* [AvroDeserializationSchema](https://ci.apache.org/projects/flink/flink-docs-master/api/java/org/apache/flink/formats/avro/AvroDeserializationSchema.html) (from `org.apache.flink:flink-avro` library) that reads data serialized with Avro using a statically provided Avro schema
* [ConfluentRegistryAvroDeserializationSchema](https://ci.apache.org/projects/flink/flink-docs-master/api/java/org/apache/flink/formats/avro/registry/confluent/ConfluentRegistryAvroDeserializationSchema.html) (from `org.apache.flink:flink-avro-confluent-registry` library) that reads data serialized with Avro using a statically provided reader schema and lookups the writer's schema in [schema-registry](https://docs.confluent.io/current/schema-registry/index.html).


# Custom DeserializationSchema

The main challenges that required us to implement custom deserialization schema for our use case were the following:
* One service can emit events using different (thus backward compatible) type versions into the same topic
* Multiple different services can emit events into the same Kafka topic using [different trace types](https://www.confluent.io/blog/multiple-event-types-in-the-same-kafka-topic)
* All data points from trace event should be persisted in the database, so it is not technically feasible to stick to particular reader schema
* Existing https://issues.apache.org/jira/browse/FLINK-11030 ("Cannot use Avro logical types with ConfluentRegistryAvroDeserializationSchema") affecting us as producers use [Avro logical types](http://avro.apache.org/docs/1.9.0/spec.html#Logical+Types).


Luckily Flink provides good abstraction for custom deserialization logic in the form of 
[KafkaDeserializationSchema](https://ci.apache.org/projects/flink/flink-docs-master/api/java/org/apache/flink/streaming/connectors/kafka/KafkaDeserializationSchema.html) interface that you need to implement in order to provide you custom logic how to turn the binary data from Kafka into Java/Scala objects.

```java
public interface KafkaDeserializationSchema<T> extends Serializable, ResultTypeQueryable<T> {
  /**
   * Method to decide whether the element signals the end of the stream. If
   * true is returned the element won't be emitted.
   */
  boolean isEndOfStream(T nextElement);
  /**
   * Deserializes the Kafka record.
   */
  T deserialize(ConsumerRecord<byte[], byte[]> record) throws Exception;
}
```

For our particular use case if was enough to use widely adopted `io.confluent.kafka.serializers.KafkaAvroDeserializer` from [io.confluent:kafka-avro-serializer](https://docs.confluent.io/current/schema-registry/serdes-develop/serdes-avro.html) library.

```java
class GenericKafkaDeserializationSchema implements KafkaDeserializationSchema<GenericRecord> {

   private KafkaAvroDeserializer deserializer;

   @Override
   public GenericRecord deserialize(ConsumerRecord<byte[], byte[]> consumerRecord) {
       return (GenericRecord) deserializer.deserialize(consumerRecord.topic(), consumerRecord.value());
   }

   @Override
   public boolean isEndOfStream(GenericRecord nextElement) {
       return false;
   }
```

# Additional optimizations

You should strongly consider using Schema Registry Client with client-side caching, e.g.[CachedSchemaRegistryClient](https://github.com/confluentinc/schema-registry/blob/master/client/src/main/java/io/confluent/kafka/schemaregistry/client/CachedSchemaRegistryClient.java.

Also, be aware of potential internal optimizations for `KafkaAvroDeserializer` related to caching `DatumReader` instances (version not yet released at the moment) fixed in the scope of
[https://github.com/confluentinc/schema-registry/issues/1515]. The actual gains from the fix vary somewhat based on hardware and Java version used but are generally [between ~3x and ~8x](https://github.com/confluentinc/schema-registry/issues/1515#issue-646876399). The cause is expensive DatumReader and DatumWriter objects being instantiated per record serialized or deserialized due to a lack of caching. The result is a lot of wasted CPU resources as well as potentially capping pipeline throughput.

# Conclusion
* Flink provides good out-of-box primitives to work with Kafka and Avro
* Flink provides good abstraction for custom deserialization logic 
* We can process large amounts of data efficiently
* You can alway find ways to optimize
