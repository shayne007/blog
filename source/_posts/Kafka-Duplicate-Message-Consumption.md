---
title: Kafka Duplicate Message Consumption
date: 2025-06-10 14:15:57
tags: [kafka]
categories: [kafka]
---
# Understanding and Mitigating Duplicate Consumption in Apache Kafka

Apache Kafka is a distributed streaming platform renowned for its high throughput, low latency, and fault tolerance. However, a common challenge in building reliable Kafka-based applications is dealing with **duplicate message consumption**. While Kafka guarantees "at-least-once" delivery by default, meaning a message might be delivered more than once, achieving "exactly-once" processing requires careful design and implementation.

This document delves deeply into the causes of duplicate consumption, explores the theoretical underpinnings of "exactly-once" semantics, and provides practical best practices with code showcases and illustrative diagrams. It also integrates interview insights throughout the discussion to help solidify understanding for technical assessments.

## The Nature of Duplicate Consumption: Why it Happens

Duplicate consumption occurs when a Kafka consumer processes the same message multiple times. This isn't necessarily a flaw in Kafka but rather a consequence of its design principles and the complexities of distributed systems. Understanding the root causes is the first step towards mitigation.

**Interview Insight:** A common interview question is "Explain the different delivery semantics in Kafka (at-most-once, at-least-once, exactly-once) and where duplicate consumption fits in." Your answer should highlight that Kafka's default is at-least-once, which implies potential duplicates, and that exactly-once requires additional mechanisms.

### Consumer Offset Management Issues

Kafka consumers track their progress by committing "offsets" – pointers to the last message successfully processed in a partition. If an offset is not committed correctly, or if a consumer restarts before committing, it will re-read messages from the last committed offset.

* **Failure to Commit Offsets:** If a consumer processes a message but crashes or fails before committing its offset, upon restart, it will fetch messages from the last *successfully committed* offset, leading to reprocessing of messages that were already processed but not acknowledged.
* **Auto-commit Misconfiguration:** Kafka's `enable.auto.commit` property, when set to `true`, automatically commits offsets at regular intervals (`auto.commit.interval.ms`). If processing takes longer than this interval, or if a consumer crashes between an auto-commit and message processing, duplicates can occur. Disabling auto-commit for finer control without implementing manual commits correctly is a major source of duplicates.

**Showcase: Incorrect Manual Offset Management (Pseudo-code)**

```java
// Consumer configuration: disable auto-commit
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-consumer-group");
props.put("enable.auto.commit", "false"); // Critical for manual control

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("my-topic"));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("Processing message: offset = %d, key = %s, value = %s%n",
                              record.offset(), record.key(), record.value());
            // Simulate processing time
            Thread.sleep(500);

            // ! DANGER: Offset commit placed after potential failure point or not called reliably
            // If an exception occurs here, or the application crashes, the offset is not committed.
            // On restart, these messages will be re-processed.
        }
        consumer.commitSync(); // This commit might not be reached if an exception occurs inside the loop.
    }
} catch (WakeupException e) {
    // Expected exception when consumer is closed
} finally {
    consumer.close();
}
```

### Consumer Failures and Rebalances

Kafka consumer groups dynamically distribute partitions among their members. When consumers join or leave a group, or if a consumer fails, a "rebalance" occurs, reassigning partitions.

* **Unclean Shutdowns/Crashes:** If a consumer crashes without gracefully shutting down and committing its offsets, the partitions it was responsible for will be reassigned. The new consumer (or the restarted one) will start processing from the last *committed* offset for those partitions, potentially reprocessing messages.
* **Frequent Rebalances:** Misconfigurations (e.g., `session.timeout.ms` too low, `max.poll.interval.ms` too low relative to processing time) or an unstable consumer environment can lead to frequent rebalances. Each rebalance increases the window during which messages might be reprocessed if offsets are not committed promptly.

**Interview Insight:** "How do consumer group rebalances contribute to duplicate consumption?" Explain that during a rebalance, if offsets aren't committed for currently processed messages before partition reassignment, the new consumer for that partition will start from the last committed offset, leading to reprocessing.

### Producer Retries

Kafka producers are configured to retry sending messages in case of transient network issues or broker failures. While this ensures message delivery (`at-least-once`), it can lead to the broker receiving and writing the same message multiple times if the acknowledgement for a prior send was lost.

**Showcase: Producer Retries (Conceptual)**

{% mermaid sequenceDiagram %}
    participant P as Producer
    participant B as Kafka Broker

    P->>B: Send Message (A)
    B-->>P: ACK for Message A (lost in network)
    P->>B: Retry Send Message (A)
    B->>P: ACK for Message A
    Note over P,B: Broker has now received Message A twice and written it.
{% endmermaid %}

### "At-Least-Once" Delivery Semantics

By default, Kafka guarantees "at-least-once" delivery. This is a fundamental design choice prioritizing data completeness over strict non-duplication. It means messages are guaranteed to be delivered, but they *might* be delivered more than once. Achieving "exactly-once" requires additional mechanisms.

## Strategies for Mitigating Duplicate Consumption

Addressing duplicate consumption requires a multi-faceted approach, combining Kafka's built-in features with application-level design patterns.

**Interview Insight:** "What are the different approaches to handle duplicate messages in Kafka?" A comprehensive answer would cover producer idempotence, transactional producers, and consumer-side deduplication (idempotent consumers).

### Producer-Side Idempotence

Introduced in Kafka 0.11, **producer idempotence** ensures that messages sent by a producer are written to the Kafka log *exactly once*, even if the producer retries sending the same message. This elevates the producer-to-broker delivery guarantee from "at-least-once" to "exactly-once" for a single partition.

* **How it Works:** When `enable.idempotence` is set to `true`, Kafka assigns a unique Producer ID (PID) to each producer. Each message is also assigned a sequence number within that producer's session. The broker uses the PID and sequence number to detect and discard duplicate messages during retries.
* **Configuration:** Simply set `enable.idempotence=true` in your producer configuration. Kafka automatically handles retries, acks, and sequence numbering.

**Showcase: Idempotent Producer Configuration (Java)**

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("enable.idempotence", "true"); // Enable idempotent producer
props.put("acks", "all"); // Required for idempotence
props.put("retries", Integer.MAX_VALUE); // Important for reliability with idempotence

Producer<String, String> producer = new KafkaProducer<>(props);

try {
    for (int i = 0; i < 10; i++) {
        String key = "message-key-" + i;
        String value = "Idempotent message content " + i;
        ProducerRecord<String, String> record = new ProducerRecord<>("idempotent-topic", key, value);
        producer.send(record, (metadata, exception) -> {
            if (exception == null) {
                System.out.printf("Message sent successfully to topic %s, partition %d, offset %d%n",
                                  metadata.topic(), metadata.partition(), metadata.offset());
            } else {
                exception.printStackTrace();
            }
        });
    }
} finally {
    producer.close();
}
```

**Interview Insight:** "What is the role of `enable.idempotence` and `acks=all` in Kafka producers?" Explain that `enable.idempotence=true` combined with `acks=all` provides exactly-once delivery guarantees from producer to broker for a single partition by using PIDs and sequence numbers for deduplication.

### Transactional Producers (Exactly-Once Semantics)

While idempotent producers guarantee "exactly-once" delivery to a *single partition*, **transactional producers** (also introduced in Kafka 0.11) extend this guarantee across *multiple partitions and topics*, as well as allowing atomic writes that also include consumer offset commits. This is crucial for "consume-transform-produce" patterns common in stream processing.

* **How it Works:** Transactions allow a sequence of operations (producing messages, committing consumer offsets) to be treated as a single atomic unit. Either all operations succeed and are visible, or none are.
    * **Transactional ID:** A unique ID for the producer to enable recovery across application restarts.
    * **Transaction Coordinator:** A Kafka broker responsible for managing the transaction's state.
    * **`__transaction_state` topic:** An internal topic used by Kafka to store transaction metadata.
    * **`read_committed` isolation level:** Consumers configured with this level will only see messages from committed transactions.

* **Configuration:**
    * Producer: Set `transactional.id` and call `initTransactions()`, `beginTransaction()`, `send()`, `sendOffsetsToTransaction()`, `commitTransaction()`, or `abortTransaction()`.
    * Consumer: Set `isolation.level=read_committed`.

**Showcase: Transactional Consume-Produce Pattern (Java)**

```java
// Producer Configuration for Transactional Producer
Properties producerProps = new Properties();
producerProps.put("bootstrap.servers", "localhost:9092");
producerProps.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
producerProps.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
producerProps.put("transactional.id", "my-transactional-producer-id"); // Unique ID for recovery

KafkaProducer<String, String> transactionalProducer = new KafkaProducer<>(producerProps);
transactionalProducer.initTransactions(); // Initialize transaction

// Consumer Configuration for Transactional Consumer
Properties consumerProps = new Properties();
consumerProps.put("bootstrap.servers", "localhost:9092");
consumerProps.put("group.id", "my-transactional-consumer-group");
consumerProps.put("enable.auto.commit", "false"); // Must be false for transactional commits
consumerProps.put("isolation.level", "read_committed"); // Only read committed messages
consumerProps.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
consumerProps.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> transactionalConsumer = new KafkaConsumer<>(consumerProps);
transactionalConsumer.subscribe(Collections.singletonList("input-topic"));

try {
    while (true) {
        ConsumerRecords<String, String> records = transactionalConsumer.poll(Duration.ofMillis(100));
        if (records.isEmpty()) {
            continue;
        }

        transactionalProducer.beginTransaction(); // Start transaction
        try {
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("Consumed message: offset = %d, key = %s, value = %s%n",
                                  record.offset(), record.key(), record.value());

                // Simulate processing and producing to another topic
                String transformedValue = record.value().toUpperCase();
                transactionalProducer.send(new ProducerRecord<>("output-topic", record.key(), transformedValue));
            }

            // Commit offsets for consumed messages within the same transaction
            transactionalProducer.sendOffsetsToTransaction(
                new HashMap<TopicPartition, OffsetAndMetadata>() {{
                    records.partitions().forEach(partition ->
                        put(partition, new OffsetAndMetadata(records.lastRecord(partition).offset() + 1))
                    );
                }},
                transactionalConsumer.groupMetadata().groupId()
            );

            transactionalProducer.commitTransaction(); // Commit the transaction
            System.out.println("Transaction committed successfully.");

        } catch (KafkaException e) {
            System.err.println("Transaction aborted due to error: " + e.getMessage());
            transactionalProducer.abortTransaction(); // Abort on error
        }
    }
} catch (WakeupException e) {
    // Expected on consumer close
} finally {
    transactionalConsumer.close();
    transactionalProducer.close();
}
```

**Mermaid Diagram: Kafka Transactional Processing (Consume-Transform-Produce)**

{% mermaid sequenceDiagram %}
    participant C as Consumer
    participant TP as Transactional Producer
    participant TXC as Transaction Coordinator
    participant B as Kafka Broker (Input Topic)
    participant B2 as Kafka Broker (Output Topic)
    participant CO as Consumer Offsets Topic

    C->>B: Poll Records (Isolation Level: read_committed)
    Note over C,B: Records from committed transactions only
    C->>TP: Records received
    TP->>TXC: initTransactions()
    TP->>TXC: beginTransaction()
    loop For each record
        TP->>B2: Send Transformed Record (uncommitted)
    end
    TP->>TXC: sendOffsetsToTransaction() (uncommitted)
    TP->>TXC: commitTransaction()
    TXC-->>B2: Mark messages as committed
    TXC-->>CO: Mark offsets as committed
    TP-->>TXC: Acknowledge Commit
    alt Transaction Fails
        TP->>TXC: abortTransaction()
        TXC-->>B2: Mark messages as aborted (invisible to read_committed consumers)
        TXC-->>CO: Revert offsets
    end
{% endmermaid %}

**Interview Insight:** "When would you use transactional producers over idempotent producers?" Emphasize that transactional producers are necessary when atomic operations across multiple partitions/topics are required, especially in read-process-write patterns, where consumer offsets also need to be committed atomically with output messages.

### Consumer-Side Deduplication (Idempotent Consumers)

Even with idempotent and transactional producers, external factors or application-level errors can sometimes lead to duplicate messages reaching the consumer. In such cases, the consumer application itself must be designed to handle duplicates, a concept known as an **idempotent consumer**.

* **How it Works:** An idempotent consumer ensures that processing a message multiple times has the same outcome as processing it once. This typically involves:
    * **Unique Message ID:** Each message should have a unique identifier (e.g., a UUID, a hash of the message content, or a combination of Kafka partition and offset).
    * **State Store:** A persistent store (database, cache, etc.) is used to record the IDs of messages that have been successfully processed.
    * **Check-then-Process:** Before processing a message, the consumer checks if its ID already exists in the state store. If it does, the message is a duplicate and is skipped. If not, the message is processed, and its ID is recorded in the state store.

**Showcase: Idempotent Consumer Logic (Pseudo-code with Database)**

```java
// Assuming a database with a table for processed message IDs
// CREATE TABLE processed_messages (message_id VARCHAR(255) PRIMARY KEY, kafka_offset BIGINT, processed_at TIMESTAMP);

Properties consumerProps = new Properties();
consumerProps.put("bootstrap.servers", "localhost:9092");
consumerProps.put("group.id", "my-idempotent-consumer-group");
consumerProps.put("enable.auto.commit", "false"); // Manual commit is crucial for atomicity
consumerProps.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
consumerProps.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
consumer.subscribe(Collections.singletonList("my-topic"));

DataSource dataSource = getDataSource(); // Get your database connection pool

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> record : records) {
            String messageId = generateUniqueId(record); // Derive a unique ID from the message
            long currentOffset = record.offset();
            TopicPartition partition = new TopicPartition(record.topic(), record.partition());

            try (Connection connection = dataSource.getConnection()) {
                connection.setAutoCommit(false); // Begin transaction for processing and commit

                // 1. Check if message ID has been processed
                if (isMessageProcessed(connection, messageId)) {
                    System.out.printf("Skipping duplicate message: ID = %s, offset = %d%n", messageId, currentOffset);
                    // Crucial: Still commit Kafka offset even for skipped duplicates
                    // So that the consumer doesn't keep pulling old duplicates
                    consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(currentOffset + 1)));
                    connection.commit(); // Commit the database transaction
                    continue; // Skip to next message
                }

                // 2. Process the message (e.g., update a database, send to external service)
                System.out.printf("Processing new message: ID = %s, offset = %d, value = %s%n",
                                  messageId, currentOffset, record.value());
                processBusinessLogic(connection, record); // Your application logic

                // 3. Record message ID as processed
                recordMessageAsProcessed(connection, messageId, currentOffset);

                // 4. Commit Kafka offset
                consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(currentOffset + 1)));

                connection.commit(); // Commit the database transaction
                System.out.printf("Message processed and committed: ID = %s, offset = %d%n", messageId, currentOffset);

            } catch (SQLException | InterruptedException e) {
                System.err.println("Error processing message or committing transaction: " + e.getMessage());
                // Rollback database transaction on error (handled by try-with-resources if autoCommit=false)
                // Kafka offset will not be committed, leading to reprocessing (at-least-once)
            }
        }
    }
} catch (WakeupException e) {
    // Expected on consumer close
} finally {
    consumer.close();
}

// Helper methods (implement based on your database/logic)
private String generateUniqueId(ConsumerRecord<String, String> record) {
    // Example: Combine topic, partition, and offset for a unique ID
    return String.format("%s-%d-%d", record.topic(), record.partition(), record.offset());
    // Or use a business key from the message value if available
    // return extractBusinessKey(record.value());
}

private boolean isMessageProcessed(Connection connection, String messageId) throws SQLException {
    String query = "SELECT COUNT(*) FROM processed_messages WHERE message_id = ?";
    try (PreparedStatement ps = connection.prepareStatement(query)) {
        ps.setString(1, messageId);
        ResultSet rs = ps.executeQuery();
        rs.next();
        return rs.getInt(1) > 0;
    }
}

private void processBusinessLogic(Connection connection, ConsumerRecord<String, String> record) throws SQLException {
    // Your actual business logic here, e.g., insert into another table
    String insertSql = "INSERT INTO some_data_table (data_value) VALUES (?)";
    try (PreparedStatement ps = connection.prepareStatement(insertSql)) {
        ps.setString(1, record.value());
        ps.executeUpdate();
    }
}

private void recordMessageAsProcessed(Connection connection, String messageId, long offset) throws SQLException {
    String insertSql = "INSERT INTO processed_messages (message_id, kafka_offset, processed_at) VALUES (?, ?, NOW())";
    try (PreparedStatement ps = connection.prepareStatement(insertSql)) {
        ps.setString(1, messageId);
        ps.setLong(2, offset);
        ps.executeUpdate();
    }
}
```

**Mermaid Diagram: Idempotent Consumer Flowchart**

{% mermaid flowchart TD %}
    A[Start Consumer Poll] --> B{Records Received?};
    B -- No --> A;
    B -- Yes --> C{For Each Record};
    C --> D[Generate Unique Message ID];
    D --> E{Is ID in Processed Store?};
    E -- Yes --> F[Skip Message, Commit Kafka Offset];
    F --> C;
    E -- No --> G[Begin DB Transaction];
    G --> H[Process Business Logic];
    H --> I[Record Message ID in Processed Store];
    I --> J[Commit Kafka Offset];
    J --> K[Commit DB Transaction];
    K --> C;
    J -.-> L[Error/Failure];
    H -.-> L;
    I -.-> L;
    L --> M[Rollback DB Transaction];
    M --> N[Re-poll message on restart];
    N --> A;
{% endmermaid %}

**Interview Insight:** "Describe how you would implement an idempotent consumer. What are the challenges?" Explain the need for a unique message ID and a persistent state store (e.g., database) to track processed messages. Challenges include managing the state store (scalability, consistency, cleanup) and ensuring atomic updates between processing and committing offsets.

### Smart Offset Management

Proper offset management is fundamental to minimizing duplicates, even when full "exactly-once" semantics aren't required.

* **Manual Commits (`enable.auto.commit=false`):** For critical applications, manually committing offsets using `commitSync()` or `commitAsync()` *after* messages have been successfully processed and any side effects (e.g., database writes) are complete.
    * `commitSync()`: Synchronous, blocks until commit is acknowledged. Safer but slower.
    * `commitAsync()`: Asynchronous, non-blocking. Faster but requires handling commit callbacks for errors.
* **Commit Frequency:** Balance commit frequency. Too frequent commits can add overhead; too infrequent increases the window for reprocessing in case of failures. Commit after a batch of messages, or after a significant processing step.
* **Error Handling:** Implement robust exception handling. If processing fails, ensure the offset is *not* committed for that message, so it will be re-processed. This aligns with at-least-once.
* **`auto.offset.reset`:** Understand `earliest` (start from beginning) vs. `latest` (start from new messages). `earliest` can cause significant reprocessing if not handled carefully, while `latest` can lead to data loss.

**Interview Insight:** "When should you use `commitSync()` vs `commitAsync()`? What are the implications for duplicate consumption?" Explain `commitSync()` provides stronger guarantees against duplicates (as it waits for confirmation) but impacts throughput, while `commitAsync()` is faster but requires explicit error handling in the callback to prevent potential re-processing.

## Best Practices for Minimizing Duplicates

Beyond specific mechanisms, adopting a holistic approach significantly reduces the likelihood of duplicate consumption.

* **Design for Idempotency from the Start:** Whenever possible, make your message processing logic idempotent. This means the side effects of processing a message, regardless of how many times it's processed, should yield the same correct outcome. This is the most robust defense against duplicates.
    * **Example:** Instead of an "increment balance" operation, use an "set balance to X" operation if the target state can be derived from the message. Or, if incrementing, track the transaction ID to ensure each increment happens only once.
* **Leverage Kafka's Built-in Features:**
    * **Idempotent Producers (`enable.idempotence=true`):** Always enable this for producers unless you have a very specific reason not to.
    * **Transactional Producers:** Use for consume-transform-produce patterns where strong "exactly-once" guarantees are needed across multiple Kafka topics or when combining Kafka operations with external system interactions.
    * **`read_committed` Isolation Level:** For consumers that need to see only committed transactional messages.
* **Monitor Consumer Lag and Rebalances:** High consumer lag and frequent rebalances are strong indicators of potential duplicate processing issues. Use tools like Kafka's consumer group commands or monitoring platforms to track these metrics.
* **Tune Consumer Parameters:**
    * `max.poll.records`: Number of records returned in a single `poll()` call. Adjust based on processing capacity.
    * `max.poll.interval.ms`: Maximum time between `poll()` calls before the consumer is considered dead and a rebalance is triggered. Increase if processing a batch takes a long time.
    * `session.timeout.ms`: Time after which a consumer is considered dead if no heartbeats are received.
    * `heartbeat.interval.ms`: Frequency of heartbeats sent to the group coordinator. Should be less than `session.timeout.ms`.
* **Consider Data Model for Deduplication:** If implementing consumer-side deduplication, design your message schema to include a natural business key or a universally unique identifier (UUID) that can serve as the unique message ID.
* **Testing for Duplicates:** Thoroughly test your Kafka applications under failure scenarios (e.g., consumer crashes, network partitions, broker restarts) to observe and quantify duplicate behavior.

## Showcases and Practical Examples

### Financial Transaction Processing (Exactly-Once Critical)

**Scenario:** A system processes financial transactions. Each transaction involves debiting one account and crediting another. Duplicate processing would lead to incorrect balances.

**Solution:** Use Kafka's transactional API.

{% mermaid graph TD %}
    Producer["Payment Service (Transactional Producer)"] --> KafkaInputTopic[Kafka Topic: Payment Events]
    KafkaInputTopic --> StreamApp["Financial Processor (Kafka Streams / Consumer + Transactional Producer)"]
    StreamApp --> KafkaDebitTopic[Kafka Topic: Account Debits]
    StreamApp --> KafkaCreditTopic[Kafka Topic: Account Credits]
    StreamApp --> KafkaOffsetTopic[Kafka Internal Topic: __consumer_offsets]

    subgraph "Transactional Unit (Financial Processor)"
        A[Consume Payment Event] --> B{Begin Transaction};
        B --> C[Process Debit Logic];
        C --> D[Produce Debit Event to KafkaDebitTopic];
        D --> E[Process Credit Logic];
        E --> F[Produce Credit Event to KafkaCreditTopic];
        F --> G[Send Consumer Offsets to Transaction];
        G --> H{Commit Transaction};
        H -- Success --> I[Committed to KafkaDebit/Credit/Offsets];
        H -- Failure --> J["Abort Transaction (Rollback all)"];
    end

    KafkaDebitTopic --> DebitConsumer["Debit Service (read_committed)"]
    KafkaCreditTopic --> CreditConsumer["Credit Service (read_committed)"]
{% endmermaid %}

**Explanation:**

1.  **Payment Service (Producer):** Uses a transactional producer to ensure that if a payment event is sent, it's sent exactly once.
2.  **Financial Processor (Stream App):** This is the core. It consumes payment events from `Payment Events`. For each event, it:
    * Starts a Kafka transaction.
    * Processes the debit and credit logic.
    * Produces corresponding debit and credit events to `Account Debits` and `Account Credits` topics.
    * Crucially, it **sends its consumed offsets to the transaction**.
    * Commits the transaction.
3.  **Atomicity:** If any step within the transaction (processing, producing, offset committing) fails, the entire transaction is aborted. This means:
    * No debit/credit events are visible to downstream consumers.
    * The consumer offset is not committed, so the payment event will be re-processed on restart.
    * This ensures that the "consume-transform-produce" flow is exactly-once.
4.  **Downstream Consumers:** `Debit Service` and `Credit Service` are configured with `isolation.level=read_committed`, ensuring they only process events that are part of a successfully committed transaction, thus preventing duplicates.

### Event Sourcing (Idempotent Consumer for Snapshotting)

**Scenario:** An application stores all state changes as a sequence of events in Kafka. A separate service builds read-models or snapshots from these events. If the snapshotting service processes an event multiple times, the snapshot state could become inconsistent.

**Solution:** Implement an idempotent consumer for the snapshotting service.

{% mermaid graph TD %}
    EventSource["Application (Producer)"] --> KafkaEventLog[Kafka Topic: Event Log]
    KafkaEventLog --> SnapshotService["Snapshot Service (Idempotent Consumer)"]
    SnapshotService --> StateStore["Database / Key-Value Store (Processed Events)"]
    StateStore --> ReadModel[Materialized Read Model / Snapshot]

    subgraph Idempotent Consumer Logic
        A[Consume Event] --> B[Extract Event ID / Checksum];
        B --> C{Is Event ID in StateStore?};
        C -- Yes --> D[Skip Event];
        D --> A;
        C -- No --> E["Process Event (Update Read Model)"];
        E --> F[Store Event ID in StateStore];
        F --> G[Commit Kafka Offset];
        G --> A;
        E -.-> H[Failure during processing];
        H --> I[Event ID not stored, Kafka offset not committed];
        I --> J[Re-process Event on restart];
        J --> A;
    end
{% endmermaid %}

**Explanation:**

1.  **Event Source:** Produces events to the `Event Log` topic (ideally with idempotent producers).
2.  **Snapshot Service (Idempotent Consumer):**
    * Consumes events.
    * For each event, it extracts a unique identifier (e.g., `eventId` from the event payload, or `topic-partition-offset` if no inherent ID).
    * Before applying the event to the `Read Model`, it checks if the `eventId` is already present in a dedicated `StateStore` (e.g., a simple table `processed_events(event_id PRIMARY KEY)`).
    * If the `eventId` is found, the event is a duplicate, and it's skipped.
    * If not found, the event is processed (e.g., updating user balance in the `Read Model`), and then the `eventId` is *atomically* recorded in the `StateStore` along with the Kafka offset.
    * Only after the event is processed and its ID recorded in the `StateStore` does the Kafka consumer commit its offset.
3.  **Atomicity:** The critical part here is to make the "process event + record ID + commit offset" an atomic operation. This can often be achieved using a database transaction that encompasses both the read model update and the processed ID storage, followed by the Kafka offset commit. If the database transaction fails, the Kafka offset is not committed, ensuring the event is re-processed.

## Interview Question Insights Throughout the Document

* **"Explain the different delivery semantics in Kafka (at-most-once, at-least-once, exactly-once) and where duplicate consumption fits in."** (Section 1)
* **"How do consumer group rebalances contribute to duplicate consumption?"** (Section 1.2)
* **"What is the role of `enable.idempotence` and `acks=all` in Kafka producers?"** (Section 2.1)
* **"When would you use transactional producers over idempotent producers?"** (Section 2.2)
* **"Describe how you would implement an idempotent consumer. What are the challenges?"** (Section 2.3)
* **"When should you use `commitSync()` vs `commitAsync()`? What are the implications for duplicate consumption?"** (Section 2.4)
* **"Discuss a scenario where exactly-once processing is critical and how you would achieve it with Kafka."** (Section 4.1)
* **"How would you handle duplicate messages if your downstream system doesn't support transactions?"** (Section 4.2 - points to idempotent consumer)

By understanding these concepts, applying the best practices, and considering the trade-offs, you can effectively manage and mitigate duplicate consumption in your Kafka-based applications, leading to more robust and reliable data pipelines.