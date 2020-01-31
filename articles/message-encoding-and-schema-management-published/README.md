# Kafka the afterthoughts: message encoding and schema management

[Apache Kafka](https://kafka.apache.org/) is a distributed streaming platform originally developed at Linkedin and later got open-sourced. [It’s designed](https://kafka.apache.org/documentation/#design) to handle high-throughput of data feed and generally used for a [broad spectrum of applications](https://kafka.apache.org/uses) that could be grouped in 2 classes:
 - Real-time data pipeline for asynchronous communication between multiple services
 - Real-time streaming applications that perform on the fly data transformations

This article is the start of a blog post series where I share my notes about working with Apache Kafka. In this first part, I will not dive in and explore all Kafka capabilities and how it works. I will rather share notes and thoughts, from my journey, about data management and usage of Apache Avro with Apache Kafka.

Topics Include:
 - Why schema management is challenging?
 - Why choose Apache Avro for encoding your data?
 - How does Avro play along with Kafka?

Target Audience:
 - You have a basic knowledge of Kafka
 - Looking for a better understanding of your technical choices
 - Looking for quick tips to start right with Kafka

Key Concepts:
 - Record: a data sent through Kafka
 - Producer: Any application writing records to Kafka
 - Consumer: Any application reading records from Kafka

!["kafka_core_apis"](https://miro.medium.com/max/1582/0*LFGlwm0HB_dndF49 "Overview of Kafka core functionalities")<center> *Overview of Kafka core functionalities* - [source](https://medium.com/race-conditions/kafka-the-afterthoughts-the-schema-management-7ea30e9518e4) </center>
 
## 1. How to choose your Kafka encoding protocol?

### Backward and Forward compatibility is not optional:
Kafka acts as a message broker moving data between multiple services (a set of producers and consumers). Thus, one of our first concerns was how to manage the data exchange between these different services in a way that drives compatibility and speed up the development of each service independently. As systems inevitably evolve and grow over time, new requirements are introduced and eventually also data schema requires changes. It becomes challenging over time to guarantee that any schema changes will not break the system. We have to think about which service should be upgraded first or if a legacy service will be able to read the latest data format being published to Kafka? Hence, we need an encoding protocol that helps us to drive compatibility across the system:

#### Backward compatibility:

We can say our system is [backward compatible](https://en.wikipedia.org/wiki/Backward_compatibility) if a consumer app using newer schema (version 2) will be able to read data produced using an older schema (version 1).
Let’s assume our producer team had to introduce new property required by the consumer app. The schema gets updated to version 2.

!["backward_compatibility_1"](https://miro.medium.com/max/1588/0*ArajBK8HwN2j0myu "Need to support backward-compatibility")
 

Both consumers and producers are upgraded with the new schema (version 2).

!["backward_compatibility_2"](https://miro.medium.com/max/1588/0*opZ9CXHYxlOTjC58 "Need to support backward-compatibility")

But later, for any strong serious reason you can think about, the producer team had to roll back the producer. And now our producer is publishing old data (version 1) while the consumer is expecting newer data (version 2).

!["backward_compatibility_3"](https://miro.medium.com/max/1588/0*if8WCKo9pekUZKeq "Need to support backward-compatibility")


The thing is, there is no way to change published messages in Kafka, messages only get deleted when the [retention period](https://www.cloudkarafka.com/blog/2018-05-08-what-is-kafka-retention-period.html) is over. If the schema does not guarantee that a new version of the schema can still read the old version, it will result in system downtime.

#### Forward compatibility:

We can say our system is [forward compatible](https://en.wikipedia.org/wiki/Forward_compatibility) if a consumer app using an old schema (version 1) will be able to read data produced using newer schema (version 2).

While backward compatibility support seems obvious to adopt, forward compatibility is a bit trickier because we deal with old software that had to handle a new requirement that it was not initially designed for. Let’s consider the 2 the following scenarios:

*Scenario 1*: 
We assume all services running using schema version 1. And as the business is growing, a new requirement introduced a new consumer app to build a certain dashboard.

!["forward_compatibility_1"](https://miro.medium.com/max/1588/0*WB4AETjJWjs9RHb- "Need to support forward-compatibility")


The new app requires schema changes (version 2). Let’s assume schema 2 is backward compatible. Being backward compatible, the new consumer app will be able to read both old and newly published data. But what if the new schema is not compatible with the legacy consumer, this will result in forcing the legacy consumer team to shift priorities and support the new schema first or postpone the release of new consumer app until all other consumer apps are ready for migrations.

!["forward_compatibility_2"](https://miro.medium.com/max/1598/0*PNQ1-Eeo8IRB4y6t "Need to support forward-compatibility")


This introduced deployment and cross-team dependencies, as well as migration downtime which harms the speed of development and reduces teams’ autonomy.

*Scenario 2*: 
Let’s assume we introduced a new schema (version 2) and migrated all consumer apps to support the new one.

!["forward_compatibility_3"](https://miro.medium.com/max/1588/0*e5iMkc6v61mbK71N "Need to support forward-compatibility")


But the legacy consumer team had a serious bug and had to roll back the service. Now the service uses the old schema (version 1). If the old schema can’t read what has been already published to Kafka using the new schema, it will break the legacy consumer app and cause downtime.
Need to support forward-compatibility

!["forward_compatibility_4"](https://miro.medium.com/max/1588/0*afBH7GRC4W6vAHmW "Need to support forward-compatibility")


⇒ One crucial thing to keep in mind is to try as much as possible to avoid deployment and cross-team dependencies. If you have to deploy a set of services before deploying some others or some service deployment can result in downtime probably we need to reconsider the system design. As we saw through the scenarios above having both backward and forward compatibility is not optional and will help our system to evolve safe and fast.

### Size matters:

As mentioned by Kafka LinkedIn core team, Kafka puts a [limit on the maximum size of a single message](https://www.slideshare.net/JiangjieQin/handle-large-messages-in-apache-kafka-58692297) that you can send: which defaults to 1MB. They explain that sending bigger sized messages is expensive to handle and cause memory pressure on the broker resulting in degradation in Kafka performance. Hence, it’s important that the schema encoding is compact enough to avoid hitting the message size limit.

!["message_size_effect"](https://miro.medium.com/max/2800/0*_uOLtmhll9svEnJS "Effect of Message Size on the Kafka throughput")<center> *Effect of Message Size on the Kafka throughput* - [source](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)</center>



## 2. Common encoding options:

> Encoding protocols fall into two major categories: protocols using an [IDL](https://en.wikipedia.org/wiki/Interface_description_language) and those that don’t. IDL-based encodings require schema definitions. They offer peace of mind with respect to data format and validation for consumers while sacrificing flexibility in the schema’s evolution. Non-IDL-based encodings are typically generic object serialization specifications, which define a compact format on top of a fixed-type system. They provide a flexible serialization mechanism but only give basic validation on types. ([source](https://eng.uber.com/trip-data-squeeze/))

The commonly adopted options with Kafka are [Avro](https://avro.apache.org/docs/current/) and [Protobuf](https://developers.google.com/protocol-buffers) which are IDL-based encoding protocol and JSON which is not.

### Why not simply JSON?

JSON has many advantages:
 - Widely supported
 - Human readable which makes it easy to debug and troubleshoot
 - Flexible as you want to introduce new data requirements
 - Everybody is familiar with it, so it doesn’t imply any learning curve.

JSON gets its advantage from being [schemaless](https://www.confluent.io/blog/how-i-learned-to-stop-worrying-and-love-the-schema-part-1/) and flexible. It could probably help making an easy start but it has major drawbacks: as JSON does not require a schema for encoding, it does not force the data validation and the published data could easily diverge from what was initially defined. It makes it hard to enforce compatibility rules as the system starts evolving. Also, the JSON requires field names to be included with every message which makes it less compact and data can easily grow in size.

⇒ It appears that clearly, JSON does not align with the requirements we mentioned in the previous section

### Avro vs Protobuf:

Avro and Protobuf are both binary encoding protocols and both require schema definition for encoding. So they share the same advantages of being schema-based:
Schemas are self-descriptive and because it’s required for encoding, it serves as a contract between services as well as between teams ([API-first approach](https://dzone.com/articles/an-api-first-development-approach-1))

The schema defines a set of rules of evolution that helps to check backward and forward compatibility when introducing schema changes. Services may have different versions of the schema without breaking: apps become more decoupled.
Schemas are separate from the data which results in a smaller encoding data size.

⇒ A recent [benchmarking](https://eng.uber.com/trip-data-squeeze/) done by Uber, which evaluates different combinations of encoding protocols and compression algorithms showed that Protobuf and Avro have quite similar performance when it comes to the encoding size.

![uber_benchmarking](https://miro.medium.com/max/2520/0*uycpgwnWFCrblXAE "Uber benchmarking of different encoding protocols")<center> *Uber benchmarking of different encoding protocols* - [source](https://eng.uber.com/trip-data-squeeze/)</center>

### Why Avro?

Avro and Protobuf both fit well in the picture and align with our requirements of the first section. But Avro has 2 slight advantages:
 - Support both dynamic and static typing
 - Designed with support to big data

Yet the main reason to choose Avro over Protobuf is more of a pragmatic decision since tools built around Kafka and more specifically the [Schema Registry](https://www.confluent.io/confluent-schema-registry/) currently has only [support for Apache Avro](https://www.confluent.io/blog/avro-kafka-data/).

> We have built tools for implementing Avro with Kafka or other systems as part of Confluent Platform. Most of our tools will work with any data format, but we do include a schema registry that specifically supports Avro. ([source](https://www.confluent.io/blog/avro-kafka-data/))

Now that we have decided on the encoding protocol. We will next talk about how the schema registry comes into play and how it works with Avro?

## 3. Using Avro with Kafka:

What Kafka does at the end of the day is just distributing a block of bytes. It does not have support for schemas out of the box and does not care what is the structure of the input data. Now that we have an encoding protocol that has support for compatibility rules, we still have to answer the 2 following questions:
 - How to enforce compatibility checks?
 - How to share the schema between the producer and consumer?
The answer to both questions is the Schema Registry.

### What is a Schema Registry?

Schema Registry is not officially part of the Apache Kafka project but has been introduced by the confluent team to solve [schema management challenges](https://docs.confluent.io/current/schema-registry/index.html).

> Schema evolution requires compatibility checks to ensure that the producer-consumer contract is not broken. This is where Schema Registry helps: it provides centralized schema management and compatibility checks as schemas evolve. ([source](https://docs.confluent.io/current/schema-registry/schema_registry_tutorial.html))

[Schema Registry is a crucial](https://www.confluent.io/blog/schema-registry-kafka-stream-processing-yes-virginia-you-really-need-one/) piece in the Kafka ecosystem. In easy words, the schema registry task is to ensure that whatever a producer is sending through Kafka, will not break the respective consumers. More technically it’s a web service providing an [HTTP interface](https://docs.confluent.io/3.0.0/schema-registry/docs/api.html#schemas) to store and retrieve schemas as well as performing [compatibility checks](https://docs.confluent.io/current/schema-registry/schema_registry_tutorial.html#schema-evolution-and-compatibility) against the previous schema versions.

### How does it work?

The schema registry stores Avro schemas under a unique name known as [subject](https://docs.confluent.io/current/schema-registry/index.html#schemas-subjects-and-topics). Each subject contains all versions related to the given schema where each version is identified by a unique ID. Specific schema version is represented as follow:

```javascript
{ 
   "id": 37,
   "version": 2,  
   "subject": "unique-subject-name",
   "schema": "{\"type\": \"string\"}"
}
```

Whenever a producer is about to publish to Kafka. The schema registry will check if the used schema is already registered. If not, the schema gets tested against the previous versions by applying the compatibility rules configured. The check happens more specifically at [the serializer level](https://docs.confluent.io/current/schema-registry/serializer-formatter.html). Only if the compatibility tests pass does the data get published to Kafka and the new schema version gets stored for later retrieval by the consumer apps.

![schema_registry](https://miro.medium.com/max/2254/0*RD1St-tEBS4nuuKu)<center> *How Confluent Schema Registry checks compatibility* - [source](https://www.confluent.io/confluent-schema-registry/)</center>

For the consumers to be able to know which schema is needed to decode the consumed record, the schema registry imposes a special [Avro message format](https://docs.confluent.io/current/schema-registry/serializer-formatter.html#wire-format). Instead of encoding the full schema with the data, the schema registry suggests encoding only the schema ID returned upon the schema registration. The consumer fetches the schema and caches it locally for future use.

![schema_registry_flow](https://miro.medium.com/max/2062/0*hRjrVGV5tuZYKPFW)<center> *Confluent Schema Registry for storing and retrieving Avro schemas* - [source](https://docs.confluent.io/current/schema-registry/index.html)</center>


## Summary:

In this first blog post we focused on the background of the protocol encoding choice and how it fulfills some design considerations to better work with Kafka:
 - To design a system where all services can be developed independently and autonomously, all code should be backward and forward compatible.
 - Keeping data size small is important for better Kafka performance
 - Schema Registry is your safeguard that will enforce compatibility across the system
 - Apache Avro with Schema Registry is a powerful combination that will save you time and overhead.
 - I have no strong reason against Protobuf but until the schema registry has support for it, I would stick with Avro.

## Next Steps:

Now that we have a better understanding of our technical choices, it’s time to get hands dirty and jump into code. Next, I will put Avro and Kafka into practice and show practical examples of Kafka Producer and Consumer and how to interact with the schema registry.
