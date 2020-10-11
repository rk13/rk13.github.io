---
layout: post
title:  "Implementing GenericKafkaDeserializationSchema for Apache Flink"
date:   2020-08-01 01:00:00
---

![stream of events](/resources/fabrizio-chiagano-YhnODmrg8hY-unsplash.jpg)

<span>Photo by <a href="https://unsplash.com/@fabriziochiagano?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Fabrizio Chiagano</a> on <a href="https://unsplash.com/s/photos/digital?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>


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

# Important performance optimizations. Flink Serialization

Flink job exchanges data records between its operators. Records need to be serialized to bytes first, because the records may not only be sent to another instance in the same JVM but instead to a separate process. Also, Flink's off-heap state-backend is based on a local embedded RocksDB instance which is implemented in native C++ code and thus also needs transformation into bytes on every state access. Please read [Flink Serialization Tuning post](https://flink.apache.org/news/2020/04/15/flink-serialization-tuning-vol-1.html) for detailed explanation why wire and state serialization alone can easily cost a lot of your job’s performance if not executed correctly.

Apache Flink’s out-of-the-box serialization can be roughly divided into the following groups:
* *Flink-provided special serializers* - for basic types
* *POJOs* - a public, standalone class with a public no-argument constructor and all non-static, non-transient fields in the class hierarchy either public or with a public getter- and a setter-method
* *Generic types* - user-defined data types that are not recognized as a POJO and then serialized via [Kryo](https://github.com/EsotericSoftware/kryo).

Flink offers built-in support for the Apache Avro serialization framework (currently using version 1.8.2) by adding the [org.apache.flink:flink-avro] dependency into your job. However, it is important to note that Avro’s *GenericRecord* types cannot, unfortunately, be used automatically since they require the user to specify a schema.
Unfortunately in our case, it is not possible because we need to use *GenericRecord* in and custom *DeserializationSchema* exactly due to lack of strictly defined Avro schema.

**Without type information, Flink will fall back to Kryo for GenericRecord serialization which would serialize the schema into every record, over and over again. As a result, the serialized form will be bigger and more costly to create. We have observed a huge performance drop in terms of increased CPU load when dealing with records backed by complex Avro schemas.**

Since Avro’s Schema class is not serializable, it can not be sent around as is. You can work around this by converting it to a String and parsing it back when needed or you can improve your *KafkaDeserializationSchema* by returning
[Tuple2](https://flink.apache.org/news/2020/04/15/flink-serialization-tuning-vol-1.html#tuple-data-types) or [Row](https://flink.apache.org/news/2020/04/15/flink-serialization-tuning-vol-1.html#tuple-data-types) entries instead. 

The example below demonstrates the optimizations done in one of our projects where *KafkaDeserializationSchema* functionality was combined with the mapper operator to avoid *GenericRecord* deserialization happening between those operators. 

Flink comes with a predefined set of tuple types that all have a fixed length and contain a set of strongly-typed fields of potentially different types. This certainly is a (performance) advantage when working with tuples instead of POJOs. Row types are mainly used by the Table and SQL APIs of Flink. A Row groups an arbitrary number of objects together similar to the tuples. Because exact field types are missing Row type information should be provided by the operator as well for effective serialization

```java
    @Override
    public Tuple2<Boolean, Row> deserialize(ConsumerRecord<byte[], byte[]> consumerRecord) {
        try {
            GenericRecord deserializedRecord = (GenericRecord) deserializer
                    .deserialize(consumerRecord.topic(), consumerRecord.value());
            return mapper.map(deserializedRecord);
        } catch (Throwable t) {
            LOGGER.warn("Error deserializing generic Avro record. {}", t.getMessage());
            return null;
        }
    }

    @Override
    public TypeInformation<Tuple2<Boolean, Row>> getProducedType() {
        return Types.TUPLE(Types.BOOLEAN, Types.ROW(Types.STRING,
                Types.SQL_TIMESTAMP,
                Types.STRING,
                Types.STRING,
                Types.POJO(PGobject.class),
                Types.STRING,
                Types.STRING,
                Types.STRING,
                Types.STRING,
                Types.SQL_TIMESTAMP));
    }
```


# Additional optimizations. KafkaAvroDeserializer

You should strongly consider using Schema Registry Client with client-side caching, e.g.[CachedSchemaRegistryClient](https://github.com/confluentinc/schema-registry/blob/master/client/src/main/java/io/confluent/kafka/schemaregistry/client/CachedSchemaRegistryClient.java).

Also, be aware of potential internal optimizations for `KafkaAvroDeserializer` related to caching `DatumReader` instances (version not yet released at the moment) fixed in the scope of [this issue](https://github.com/confluentinc/schema-registry/issues/1515). The actual gains from the fix vary somewhat based on hardware and Java version used but are generally [between ~3x and ~8x](https://github.com/confluentinc/schema-registry/issues/1515#issue-646876399). The cause is expensive DatumReader and DatumWriter objects being instantiated per record serialized or deserialized due to a lack of caching. The result is a lot of wasted CPU resources as well as potentially capping pipeline throughput.

# Conclusion
* Flink provides good out-of-box primitives to work with Kafka and Avro
* Flink provides good abstraction for custom deserialization logic 
* We can process large amounts of data efficiently
* You can alway find ways to optimize
