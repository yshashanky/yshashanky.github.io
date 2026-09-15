---
title: "From XML to Avro: Building SpecificRecord and Publishing a GenericRecord with Kafka"
date: 2026-09-15
description: "A practical look at converting XML payloads to Avro SpecificRecord and GenericRecord in Java, using StAX, Schema Registry, and Kafka for the final publish flow."
draft: false
---

When working with Kafka, the message you receive isn't always already in the format your application wants to work with.

In one of my recent data pipelines, the incoming Kafka payload was an XML string. The final Kafka topic, however, needed data represented using Avro and a schema managed through Confluent Schema Registry.

That meant there was a transformation in between:

```text
Kafka
  ↓
XML String
  ↓
XML Parsing
  ↓
Avro SpecificRecord
  ↓
GenericRecord
  ↓
Kafka
```

There are a few different technologies involved here—XML, StAX, Avro, GenericRecord, SpecificRecord, and Schema Registry. Individually, none of them is particularly difficult. The interesting part is how they fit together.

This article walks through that flow, focusing mainly on **how to take an XML payload and build an Avro SpecificRecord from it**.

---

## The problem with starting from XML

XML and Avro represent data differently.

An XML payload is essentially a hierarchy of elements:

```xml
<RootObject>
    <name>...</name>
    <resource>...</resource>

    <DataField>
        <name>...</name>
        <value>...</value>
    </DataField>

    <DataSet>
        <name>...</name>

        <DataField>
            <name>...</name>
            <value>...</value>
        </DataField>
    </DataSet>
</RootObject>
```

The actual structure in a production application can obviously be much larger and more deeply nested. The important point is that the XML contains a root object and child objects inside it.

Avro, on the other hand, works from a defined schema.

For the first transformation, I had an Avro schema available locally in the application. The corresponding Java class was generated from that schema, giving me an Avro `SpecificRecord` and its builder.

So rather than trying to construct the final Kafka payload directly from XML, I first converted the XML into a strongly defined Avro object.

---

## Why use StAX for the XML parsing?

For this transformation, I used the Java XML streaming API:

```java
XMLInputFactory
XMLStreamReader
XMLStreamConstants
```

The basic idea is simple.

Instead of treating the XML as a collection of Java objects immediately, the `XMLStreamReader` moves through the XML and reports events as it encounters them.

The basic pattern looks like this:

```java
while (reader.hasNext()) {
    int type = reader.next();

    if (type == XMLStreamReader.START_ELEMENT) {
        // process the element
    } else if (type == XMLStreamReader.END_ELEMENT) {
        // finish the object when required
    }
}
```

For the implementation, I use the element name to determine what needs to happen next.

For example, when the parser encounters a `DataSet`, I create its builder:

```java
DataSet.Builder rootSetBuilder = DataSet.newBuilder();
```

Then its attributes can be populated from the XML:

```java
rootSetBuilder.setName(...);
rootSetBuilder.setResource(...);
rootSetBuilder.setOperation(...);
```

Similarly, when a `DataField` starts, I create a `DataField.Builder` and populate its values as the corresponding XML elements are encountered.

The actual field names in the production XML are business-specific, so the examples here are intentionally simplified.

---

## Reading nested objects

This is where the streaming parser becomes a little more interesting.

The XML I was processing wasn't just a flat collection of fields. It contained nested objects, including collections of child objects.

For example:

```text
Root DataSet
 ├── DataField
 ├── DataField
 └── Nested DataSet
      ├── DataField
      └── DataField
```

The Avro schema represents these relationships using fields such as arrays.

While parsing the XML, I therefore maintain builders and lists for the objects currently being constructed.

A simplified version of the pattern is:

```java
DataSet.Builder rootSetBuilder = null;

List<DataSet> dataSets = new ArrayList<>();
List<DataField> dataFields = new ArrayList<>();

DataSet.Builder nestedSetBuilder = null;
List<DataField> nestedDataFields = new ArrayList<>();
```

When the parser encounters a start element, the appropriate builder is created or populated.

When it reaches the corresponding end element, the builder can be completed:

```java
DataField dataField = dataFieldBuilder.build();
dataFields.add(dataField);
```

For nested objects, the completed child objects are added to the appropriate collection.

The parser therefore isn't just reading values. It is keeping track of **where those values belong in the resulting Avro object structure**.

In my implementation, a depth value is also used to distinguish between root-level and nested objects.

---

## Mapping XML values into the SpecificRecord

For simple fields, the mapping is fairly direct.

Conceptually:

```text
XML element
     ↓
read value
     ↓
SpecificRecord builder
     ↓
set field
```

For example:

```java
if (type == XMLStreamReader.START_ELEMENT) {

    switch (reader.getLocalName()) {

        case "value" -> {
            if (dataFieldBuilder != null) {
                dataFieldBuilder.setValue(
                    reader.getElementText().trim()
                );
            }
        }

        // other elements...
    }
}
```

The important part is that the XML parser knows which builder is currently active.

At the end of an object, that builder is built and added to its parent structure.

Finally, once the XML stream has been completely processed:

```java
if (rootSetBuilder != null) {
    rootSetBuilder.setDataFields(dataFields);
    rootSetBuilder.setNestedDataSets(dataSets);

    return rootSetBuilder.build();
}
```

The result is the first `SpecificRecord`.

---

## Sanitizing the XML before parsing

There is another step before the XML reaches the stream reader.

The Kafka payload arrives as a `String`, and depending on how the payload is represented, there can be additional wrapping or escaping that needs to be handled before parsing it as XML.

For the root payload, my sanitization method does essentially three things:

```java
private String sanitizePayload(String payload) {
    String s = payload.trim();

    if (s.startsWith("\"")
            && s.endsWith("\"")
            && s.length() > 1) {

        s = s.substring(1, s.length() - 1);
    }

    return StringEscapeUtils.unescapeJava(s);
}
```

So the outer whitespace is removed, surrounding quotes are removed when present, and escaped Java characters are unescaped.

There is also a separate sanitization method for internal XML data.

That method first handles `null` or blank values, then trims the content. If the value begins with an XML declaration such as:

```xml
<?xml ...?>
```

the declaration is removed before the remaining XML is processed.

These are small steps, but they matter because the stream reader needs to receive valid XML rather than the string representation that happened to arrive in the message.

---

## Configuring XMLInputFactory

The XML reader is created through `XMLInputFactory`.

One part of my configuration looks like this:

```java
XMLInputFactory factory = XMLInputFactory.newFactory();

factory.setProperty(
    XMLInputFactory.SUPPORT_DTD,
    false
);

factory.setProperty(
    XMLInputFactory.IS_COALESCING,
    true
);

factory.setProperty(
    XMLInputFactory.IS_REPLACING_ENTITY_REFERENCES,
    false
);

factory.setProperty(
    XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES,
    false
);
```

Then the reader is created from the sanitized XML string:

```java
XMLStreamReader reader =
    factory.createXMLStreamReader(
        new StringReader(dataSet)
    );
```

The DTD and external-entity related settings are deliberately disabled in this configuration.

Once the reader has been created, the application can process the XML event by event.

---

## Why not build the GenericRecord immediately?

At this point, we have a `SpecificRecord`.

But the final Kafka payload uses a `GenericRecord`.

There is a reason for keeping the intermediate SpecificRecord.

A `SpecificRecord` has a generated Java representation based on a known Avro schema. That makes it convenient to build and work with from application code.

In my case, the XML-to-object transformation is much easier to express using the generated builders and strongly defined fields.

The final structure, however, is handled differently.

The target structure is relatively flat, with fields such as strings, timestamps, numbers and decimals at the root level.

So after creating the first SpecificRecord, I map its fields into a `GenericRecord`.

---

## SpecificRecord to GenericRecord

Creating the GenericRecord itself is straightforward.

The record is created against the target Avro schema:

```java
GenericRecord record =
    new GenericData.Record(schema);
```

Then the fields are populated individually:

```java
record.put(
    "firstName",
    specificRecord.get("firstName")
);

record.put(
    "paymentAmount",
    specificRecord.get("paymentAmount")
);

record.put(
    "paymentCurrency",
    specificRecord.get("paymentCurrency")
);
```

The actual implementation loops through the required fields and obtains their values from the first SpecificRecord.

So the transformation is essentially:

```text
SpecificRecord
      │
      ├── field A ──→ GenericRecord field A
      ├── field B ──→ GenericRecord field B
      ├── field C ──→ GenericRecord field C
      └── field D ──→ GenericRecord field D
```

There isn't a complicated conversion algorithm here. The important thing is that the GenericRecord is created using the **target schema** and then populated with values from the SpecificRecord.

---

## Where Schema Registry comes in

The target Avro schema isn't something the producer invents while publishing the message.

The schema for the target record is registered in **Confluent Schema Registry**.

The application retrieves the latest schema metadata using a cached Schema Registry client:

```java
CachedSchemaRegistryClient client =
    new CachedSchemaRegistryClient(url, 1000);

SchemaMetadata metadata =
    client.getLatestSchemaMetadata(
        subject + "-value"
    );

Schema schema =
    new Schema.Parser().parse(
        metadata.getSchema()
    );
```

That gives the application an Avro `Schema` object.

That schema is then used when creating the GenericRecord:

```java
GenericRecord record =
    new GenericData.Record(schema);
```

This is an important distinction:

**Schema Registry provides the schema. It does not provide the Java SpecificRecord class at runtime.**

The first SpecificRecord's Java class comes from the Avro schema available in the application. The target schema is retrieved from Schema Registry and used to construct the GenericRecord that will eventually be published.

---

## Publishing the GenericRecord to Kafka

Once the GenericRecord has been built, the final step is publishing it.

The application uses Spring Kafka's `KafkaTemplate`.

The value serializer is the Confluent `KafkaAvroSerializer`:

```properties
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer

schema.registry.url=<schema-registry-url>
```

One configuration that is important in this flow is:

```properties
auto.register.schemas=false
use.latest.version=true
```

I don't want the producer to automatically register a new schema as part of publishing. The schema is already managed through Schema Registry, so the application is working with the existing schema rather than asking the producer to register a new one as part of the publish operation.

The producer also has the Kafka connection, SSL and producer settings configured for the environment, including bootstrap servers, SSL truststore/keystore configuration, acknowledgements, retries, in-flight requests, `linger.ms`, and compression.

The final flow is therefore:

```text
XML String
    │
    ▼
Sanitize
    │
    ▼
XMLStreamReader
    │
    ▼
SpecificRecord
    │
    ▼
GenericRecord
    │
    │ + target Avro schema
    ▼
KafkaTemplate
    │
    ▼
KafkaAvroSerializer
    │
    ▼
Kafka
```

---

## SpecificRecord or GenericRecord?

This is probably the simplest way I would decide between them based on this implementation.

### SpecificRecord

Use it when the application knows the schema and you want a generated Java representation to work with.

In this pipeline, that makes the XML transformation easier because the application can work with generated builders and defined fields.

### GenericRecord

Use it when the record needs to be handled more dynamically against an Avro `Schema`.

In this pipeline, the GenericRecord is used at the final publishing stage, where the target schema comes from Schema Registry.

The distinction isn't that one is universally better than the other. They solve slightly different problems.

In this particular flow, using both gives a fairly clean separation:

```text
XML
 │
 ▼
SpecificRecord
 │
 │ easier application-side construction
 ▼
GenericRecord
 │
 │ target schema
 ▼
Kafka
```

---

## The complete picture

Putting everything together, the transformation looks like this:

```text
                    Confluent Schema Registry
                              │
                              │ latest target schema
                              ▼
Kafka XML ──→ Sanitize ──→ StAX Parser ──→ SpecificRecord
                                             │
                                             │ field mapping
                                             ▼
                                        GenericRecord
                                             │
                                             │ target schema
                                             ▼
                                        KafkaTemplate
                                             │
                                             ▼
                                            Kafka
```

The XML side and Avro side are doing different jobs.

The XML parser is responsible for understanding the incoming hierarchical structure.

The SpecificRecord gives the application a strongly defined Java representation to work with.

The GenericRecord provides the final record representation against the target schema.

And Schema Registry provides the schema that defines that target structure.

Once those responsibilities are separated, the transformation becomes much easier to reason about.

---

## Final thoughts

XML and Avro can initially feel like two completely different worlds. One is hierarchical and text-based, while the other is schema-driven and designed for structured serialization.

The important part is not trying to make one directly behave like the other.

Instead, treat the transformation as a series of small steps:

**clean the input → read the XML → build the SpecificRecord → map the required fields → create the GenericRecord with the target schema → publish it through Kafka.**

For me, the most useful part of this approach is that the XML parsing logic doesn't have to know anything about Kafka serialization, and the Kafka producer doesn't have to know how the original XML was structured.

Each step has one job.

And that makes the whole pipeline much easier to work with.
