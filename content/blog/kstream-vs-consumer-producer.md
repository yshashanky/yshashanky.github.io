---
title: "KStream vs Consumer-Producer: Choosing the Right Architecture"
date: 2026-09-09
description: "A practical experience of choosing between Kafka Streams and separate Kafka Consumer and Producer clients for a cross-cluster Kafka pipeline."
draft: false
---

When I first looked at the requirement for one of our Kafka pipelines, using KStream felt like the obvious choice.

The requirement itself was simple: consume a message from one Kafka topic, validate and process it, and publish the result to another Kafka topic.

The part that made it less simple was the infrastructure. The source topic and destination topic were on two completely separate Kafka clusters. They also had different security configurations.

That difference ended up changing the architecture.

## The requirement

The pipeline was processing transaction-related records for a banking use case, where reliability and data consistency were important.

The expected flow was straightforward:

1. Consume a message.
2. Validate and process it.
3. Transform it into the required output.
4. Publish it to the destination topic.

The pipeline was expected to handle around **80,000 messages per hour**.

At first, I thought Kafka Streams would fit this very well. A stream comes in, some processing happens, and another stream goes out. That is exactly the kind of flow Kafka Streams is designed to make convenient.

So we started with KStream.

## Where things became interesting

The source and destination were not just two topics with different names.

They belonged to different Kafka clusters. The consumer side had one set of connection and security requirements. The producer side had another.

For example, the source cluster used SASL + SSL, while the destination cluster used SSL. They also had their own bootstrap servers, certificates, keystores, truststores and related credentials.

We configured the Streams application and started testing.

The interesting part was that the consumer side was working. Messages were being consumed and the processing logic was running. But the producer was repeatedly failing to connect to the destination cluster.

The error we kept seeing was a connection timeout. It would retry the connection, but the result was the same connection timeout again.

We spent roughly three to four hours trying to get that approach working. One of the things that helped us narrow it down was testing the same kind of flow with topics from the same cluster. That worked.

Once the source and destination were on different clusters with their different configurations, we hit the problem again.

That gave us a much stronger signal that this wasn't simply a bad topic configuration or a temporary connectivity problem.

We went back through the Kafka Streams configuration model and some previous implementations/documentation. The issue was that the conventional Streams setup was not a good fit for what we were trying to do: independently connect the consumer side to one Kafka cluster and the producer side to another cluster with separate infrastructure and security configuration.

At that point, continuing to fight the configuration didn't make much sense.

We already had the answer we needed.

## The annoying part: we had already done the work

This was probably the most frustrating part of the whole thing.

We had already spent time implementing the KStream-based solution. The problem was not that the processing logic was wrong. The problem was that the architecture we had chosen didn't fit the infrastructure we actually had.

Changing the approach meant rewriting the code and going through the architecture approval process again. We were also working against a tight delivery timeline, so we had to put in some additional effort to complete the work on time.

It wasn't a huge schedule impact in the end, but it was avoidable rework.

If we had validated the Kafka cluster boundary before starting the implementation, we could have chosen the second approach from the beginning.

## Switching to a separate Consumer and Producer

The replacement was much more explicit.

Instead of using one Kafka Streams topology, we used a Spring Kafka consumer and a separate producer configuration in the same Spring Boot application.

The flow became:

```text
Kafka Cluster A
     |
     | @KafkaListener
     v
Consume
     |
     v
Validate / Process / Transform
     |
     v
KafkaTemplate
     |
     v
Kafka Cluster B
```

The important difference was that the consumer and producer were now independent clients.

The consumer had its own configuration for Cluster A.

The producer had its own configuration for Cluster B.

That made the separation much easier to reason about.

The listener itself was intentionally simple. It received a `ConsumerRecord`, passed it to the processing service, and worked with a `CompletableFuture` representing the asynchronous processing.

A simplified version of the listener looked like this:

```java
@KafkaListener(
    topics = "${kafka.consumer.topic}",
    groupId = "${spring.kafka.consumer.group-id}",
    containerFactory = "kafkaListenerContainerFactory"
)
public void consumeMessage(
        ConsumerRecord<String, String> message,
        Acknowledgment acknowledgment) {

    CompletableFuture<Void> processingFuture =
            asyncService.process(message);

    processingFuture.whenComplete((ignored, ex) -> {
        if (ex == null) {
            acknowledgment.acknowledge();
        } else {
            // Do not acknowledge the message
        }
    });
}
```

The actual implementation contains logging and additional error handling, but this is the important part of the flow.

We used **manual acknowledgment**. The message was not acknowledged immediately after the listener received it. We waited for the asynchronous processing to complete successfully.

Only then did we acknowledge it, after which Spring handled the corresponding offset commit.

That distinction mattered because publishing the message successfully was part of completing the processing.

## The producer was completely separate

For the producer, we used a dedicated `KafkaTemplate<String, GenericRecord>`.

The producer configuration had its own bootstrap servers and security properties for the destination cluster.

Some of the relevant configuration included:

```java
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, producerBootstrapServers);

props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, producerSecurityProtocol);

props.put(SslConfigs.SSL_TRUSTSTORE_LOCATION_CONFIG, truststoreLocation);

props.put(SslConfigs.SSL_TRUSTSTORE_PASSWORD_CONFIG, truststorePassword);

props.put(SslConfigs.SSL_KEYSTORE_LOCATION_CONFIG, keystoreLocation);

props.put(SslConfigs.SSL_KEYSTORE_PASSWORD_CONFIG, keystorePassword);
```

There were also producer settings for acknowledgments, retries, in-flight requests, linger and compression.

The producer was responsible for publishing the transformed `GenericRecord` to the appropriate destination topic.

That separation made debugging much easier too.

If the consumer had a problem, I could look at the Cluster A configuration.

If the producer had a problem, I could look at the Cluster B configuration.

There was less ambiguity about which client was responsible for which connection.

## What happens when publishing fails?

This was an important part of the implementation.

We didn't want to acknowledge the source message just because our application had successfully started processing it.

The producer send was asynchronous.

Conceptually, the processing looked like:

```java
return kafkaTemplate
        .send(publishTopic, consumerRecord.key(), payment)
        .thenAccept(result -> {
            // Publish succeeded
        })
        .handle((result, ex) -> {
            if (ex == null) {
                return CompletableFuture.completedFuture(null);
            }

            // Handle publish failure
            return handlePublishFailure(...);
        })
        .thenCompose(Function.identity());
```

The actual implementation also handled the transformation and topic selection before the send. For example, the processed transaction could be routed to one of the appropriate destination topics based on the resulting transaction state.

The important point is that **successful publishing and source acknowledgment were connected**.

If publishing completed successfully, the processing future completed successfully and the listener acknowledged the source message.

If publishing failed, the processing future did not complete successfully, so the source message was not acknowledged.

## Retry vs DLQ

We also didn't treat every failure as a retryable failure.

For producer/connectivity-related failures, we used retries.

The sequence was:

```text
Initial publish
      |
      X
      |
   wait 1s
      |
   Retry 1
      |
      X
      |
   wait 3s
      |
   Retry 2
      |
      X
      |
   wait 5s
      |
   Retry 3
      |
      X
      |
     DLQ
```

So there was an initial attempt followed by **three retries**.

The retry delays were:

- First retry: 1 second
- Second retry: 3 seconds
- Third retry: 5 seconds

The retries were primarily for producer/connectivity-related failures.

We didn't want to keep retrying a message when the message itself was the problem.

For example, if the data was corrupt, required fields were missing, a value was null where it shouldn't be, or the data type was incorrect, retrying the same message wasn't going to fix it.

Those cases went directly to the DLQ.

## There is an important limitation here

There is a subtle point about acknowledgments and offsets that is easy to miss. 

The source offset is acknowledged only after successful processing/publishing.

That gives us a useful guarantee: if the application fails before the source offset is acknowledged, the source record can be processed again.

But this does **not** give us exactly-once processing across two independent Kafka clusters.

Consider this sequence:

```text
1. Consume message from Cluster A
2. Publish successfully to Cluster B
3. Application fails
4. Source offset was not committed
5. Message is consumed again
6. It may be published to Cluster B again
```

So there is a possibility of duplicate publication if the destination publish succeeds but the source offset is not committed before the application fails.

That is an important distinction. The design gives us a practical at-least-once style processing behavior, but we should not describe it as exactly-once across the two clusters.

For this pipeline, that tradeoff was understood and handled as part of the design.

## What I would choose today

The main thing I took away from this was not that KStream is bad.

It isn't.

If the consumer and producer are working against the same Kafka cluster and the same general configuration, I would still prefer KStream for this kind of processing. It simplifies the implementation considerably. The consume-process-publish flow maps naturally to a stream topology, and there is less client configuration to manage directly.

The decision changes when the infrastructure changes.

If the source and destination are in different Kafka clusters and need independent connection and security configurations, I would choose a separate Consumer + Producer approach. In our case, that meant using `@KafkaListener` for the source and `KafkaTemplate` for the destination.

The lesson for me was fairly simple: **don't choose the abstraction before checking the infrastructure boundary.**

We already knew that the source and destination were in different clusters. What we didn't account for initially was what that meant for the Kafka Streams approach.

If I were starting the same pipeline today, I'd validate that first.

That would probably save the three or four hours of debugging — and more importantly, the code rewrite and second architecture approval that followed it.

That’s the whole story. Onward to the next surprise.