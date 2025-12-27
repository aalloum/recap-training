# ⚡ Kafka Cheat Sheet

This document gathers essential Kafka concepts and commands for managing topics, producers, consumers, and more.

---

## 📋 Table of Contents

- [🏭 Producer](#-producer)
- [📥 Consumer](#-consumer)
- [🔗 Kafka Connect](#-kafka-connect)
- [🌊 Kafka Streams](kafka_streams_guide.md)

---

## 🏭 Producer

Producers are responsible for sending messages to Kafka topics. They are the data sources that push information into the Kafka cluster.

### Key Concepts
- **Topic**: Destination where messages are sent.
- **Partition**: Divides data across the cluster for parallel processing.
- **Key**: Optional identifier to determine which partition receives the message.
- **Value**: The actual message data.
- **Acknowledgment**: Confirmation that the message was successfully written (`acks=all`, `acks=1`, `acks=0`).

### Basic Configuration
```properties
bootstrap.servers=localhost:9092
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
acks=all
retries=3
linger.ms=10
```

### Common Methods
- `send()`: Asynchronously sends a message to Kafka.
- `flush()`: Ensures all messages are sent before proceeding.
- `close()`: Closes the producer and releases resources.

### Producer Example (Java)
```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import java.util.Properties;

Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>("my-topic", "key", "value");
producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        System.out.println("Message sent to partition " + metadata.partition());
    } else {
        exception.printStackTrace();
    }
});

producer.close();
```

---

## 📥 Consumer

Consumers are responsible for reading messages from Kafka topics. They form consumer groups to process data in parallel and ensure load balancing.

### Key Concepts
- **Consumer Group**: A group of consumers that together consume messages from a topic.
- **Offset**: The position of a message in a partition.
- **Polling**: The consumer continuously polls for new messages.
- **Rebalancing**: Automatic redistribution of partitions among consumers in a group.
- **Auto-commit**: Automatic tracking of offsets (can be disabled for manual control).

### Basic Configuration
```properties
bootstrap.servers=localhost:9092
group.id=my-consumer-group
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
auto.offset.reset=earliest
enable.auto.commit=true
auto.commit.interval.ms=1000
```

### Common Methods
- `subscribe()`: Subscribes to one or more topics.
- `poll()`: Fetches messages from subscribed topics.
- `commitSync()`: Synchronously commits the current offset.
- `commitAsync()`: Asynchronously commits the current offset.
- `close()`: Closes the consumer and releases resources.

### Consumer Example (Java)
```java
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import java.util.Arrays;
import java.util.Properties;

Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-consumer-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("auto.offset.reset", "earliest");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("my-topic"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    records.forEach(record -> {
        System.out.println("Key: " + record.key() + ", Value: " + record.value());
    });
}
```

---

## 🔗 Kafka Connect

Kafka Connect is a framework for building scalable and reliable data integrations between Kafka and external systems. It allows you to move data without writing custom producers or consumers.

### Key Concepts
- **Connector**: A plugin that implements data integration logic.
- **Source Connector**: Pulls data from external systems and writes to Kafka.
- **Sink Connector**: Reads data from Kafka topics and writes to external systems.
- **Task**: Individual units of work within a connector.
- **Worker**: JVM process running tasks.

### Types of Connectors
- **Source Connectors**: Database (JDBC), APIs, File systems, Message queues
- **Sink Connectors**: Databases, Data warehouses, Elasticsearch, Cloud storage (S3, GCS)

### Standalone vs Distributed Mode
- **Standalone**: Single worker process, suitable for development and testing.
- **Distributed**: Multiple worker processes, suitable for production with high availability and fault tolerance.

### Basic Configuration (Source Connector Example)
```properties
name=jdbc-source-connector
connector.class=io.confluent.connect.jdbc.JdbcSourceConnector
tasks.max=1
database.history.kafka.bootstrap.servers=localhost:9092
database.history.kafka.topic=schema-change-topic
connection.url=jdbc:mysql://localhost:3306/mydb
connection.user=root
connection.password=password
table.whitelist=users
mode=incrementing
incrementing.column.name=id
topic.prefix=mysql_
```

### REST API for Kafka Connect
```sh
# List all connectors
curl -X GET http://localhost:8083/connectors

# Create a new connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @connector-config.json

# Get connector status
curl -X GET http://localhost:8083/connectors/my-connector/status

# Delete a connector
curl -X DELETE http://localhost:8083/connectors/my-connector

# Pause a connector
curl -X PUT http://localhost:8083/connectors/my-connector/pause

# Resume a connector
curl -X PUT http://localhost:8083/connectors/my-connector/resume
```

---

## 🌊 Kafka Streams

Kafka Streams is a library for building streaming applications that process data in real-time. For detailed information, see the **[Kafka Streams Guide](kafka-streams.md)**.

### Quick Overview
- Real-time data processing
- Stateless and stateful transformations
- Fault tolerance and scalability
- Interactive queries on local state stores

### Basic Architecture
- **Source Topic**: Input data
- **Topology**: Computational logic
- **Sink Topic**: Output data

For comprehensive documentation on Kafka Streams, including examples, state management, windowing, and best practices, please refer to the **[Kafka Streams Documentation](kafka-streams.md)**.

---

## 🛠️ Kafka CLI Commands

### Topic Management
```sh
# Create a topic
kafka-topics --bootstrap-server localhost:9092 --create --topic my-topic --partitions 3 --replication-factor 1

# List all topics
kafka-topics --bootstrap-server localhost:9092 --list

# Describe a topic
kafka-topics --bootstrap-server localhost:9092 --describe --topic my-topic

# Delete a topic
kafka-topics --bootstrap-server localhost:9092 --delete --topic my-topic
```

### Producer Testing
```sh
# Start a console producer
kafka-console-producer --bootstrap-server localhost:9092 --topic my-topic

# Send messages with keys
kafka-console-producer --bootstrap-server localhost:9092 --topic my-topic --property "parse.key=true" --property "key.separator=:"
```

### Consumer Testing
```sh
# Start a console consumer from the beginning
kafka-console-consumer --bootstrap-server localhost:9092 --topic my-topic --from-beginning

# Start a consumer for a specific consumer group
kafka-console-consumer --bootstrap-server localhost:9092 --topic my-topic --group my-group

# Display consumer group information
kafka-consumer-groups --bootstrap-server localhost:9092 --group my-group --describe
```

---

## 📚 Additional Resources

- **[Kafka Streams Guide](kafka-streams.md)** - Comprehensive guide for building streaming applications
- Apache Kafka Documentation: https://kafka.apache.org/documentation/
- Confluent Kafka Platform: https://www.confluent.io/

