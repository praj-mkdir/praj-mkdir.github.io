---
title: "Integrating Apache Kafka with Spring Boot"
date: 2020-09-15T11:30:03+00:00
# weight: 1
aliases: ["/first"]
tags: ["kafka"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
---

# Integrating Apache Kafka with Spring Boot

Apache Kafka is a robust distributed event-streaming platform that enables high-throughput, real-time data pipelines and event-driven applications. Integrating Kafka with Spring Boot provides developers with a seamless way to produce and consume messages, making it ideal for building scalable and resilient systems.

This article will guide you through setting up Kafka integration with Spring Boot using Spring Kafka's auto-configuration. By the end, you'll have a functional Spring Boot application with a producer, consumer, and controller to test the setup.

## Prerequisites

1. **Java Development Kit (JDK)**: Install JDK 11 or higher.

2. **Apache Kafka and Zookeeper (via Docker)**: Use Docker to set up Kafka and Zookeeper. Create a `docker-compose.yml` file with the following content:

```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

Start the containers:
```bash
docker-compose up -d
```

3. **Maven**: Ensure Maven is installed for dependency management.

4. **Spring Boot**: Familiarity with Spring Boot basics.

## 1. Spring Boot Project Setup

### Add Dependencies

In your `pom.xml`, add the following dependency for Spring Kafka:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### Configure Kafka Properties

Spring Boot's auto-configuration simplifies Kafka setup. Define Kafka-related properties in the `application.properties` file:

```properties
# Application Name
spring.application.name=kafkalearning

# Kafka Broker Connection
spring.kafka.bootstrap-servers=localhost:9092

# Producer Serialization Configurations
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

# Consumer Deserialization Configurations
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer

# Disable Auto-Topic Creation
spring.kafka.admin.auto-create=false
```

## 2. Implementing the Producer, Consumer, and Controller

### Producer Implementation

The producer component is responsible for sending messages to a Kafka topic. Here's the implementation:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class testProducer {
    private static final Logger log = LoggerFactory.getLogger(testProducer.class);
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public testProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void sendMessage(String topic, Object message) {
        kafkaTemplate.send(topic, message);
        log.info("Sent Message: " + message);
    }
}
```

**Key Points:**
- The `KafkaTemplate` is auto-configured by Spring Boot and simplifies message sending.
- The `sendMessage` method takes the topic name and message as parameters.

### Consumer Implementation

The consumer listens to a specific topic and processes incoming messages.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class testConsumer {
    private static final Logger log = LoggerFactory.getLogger(testConsumer.class);
    
    @KafkaListener(topics = "test-events", groupId = "groupA")
    public void consumerGroupA(String message) {
        log.info("Message received from producer: " + message);
    }
}
```

**Key Points:**
- The `@KafkaListener` annotation binds the method to the `test-events` topic.
- The `groupId` ensures that multiple instances of the same consumer process messages cooperatively.

### Controller Implementation

The controller acts as the entry point for testing the producer by exposing an API endpoint.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class testController {
    private static final Logger log = LoggerFactory.getLogger(testController.class);
    private final testProducer testProducer;
    
    @Autowired
    public testController(testProducer testProducer) {
        this.testProducer = testProducer;
    }
    
    @PostMapping("/testPublish")
    public String publishMessage(@RequestParam String topic, @RequestParam String message) {
        log.info("Inside the controller");
        testProducer.sendMessage(topic, message);
        return "Message sent to topic " + topic;
    }
}
```

**Key Points:**
- The `@PostMapping` endpoint `/testPublish` accepts topic and message as query parameters.
- The `testProducer` is injected into the controller to publish messages to Kafka.

## 3. Defining Kafka Topics in Configuration

Instead of manually creating topics, you can define them in a configuration class. This ensures topics are created when the application starts:

```java
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaTopicConfig {
    @Bean
    public NewTopic testTopic() {
        return TopicBuilder.name("test-events")
                .build();
    }
}
```

**Key Points:**
- Defining topics in configuration centralizes topic management.
- The `@Bean` ensures the topic is created at application startup.

## 4. Testing the Application

### Start the Kafka Environment

Ensure that the Docker containers for Kafka and Zookeeper are running:

```bash
docker-compose up -d
```

### Run the Application

Start the Spring Boot application.

### Publish a Message

Use a tool like cURL, Postman, or your browser to test the `/testPublish` endpoint:

```bash
curl -X POST "http://localhost:8080/testPublish?topic=test-events&message=HelloKafka"
```

### Verify the Output

**Producer Logs:**
```
Sent Message: HelloKafka
```

**Consumer Logs:**
```
Message received from producer: HelloKafka
```

## 5. Advantages of Auto-Configuration

Using Spring Boot's auto-configuration simplifies the Kafka integration by:

- Eliminating boilerplate code.
- Managing Kafka producer and consumer beans automatically.
- Providing flexibility for topic configurations.

However, if you need advanced configurations or custom behavior (e.g., transactional producers), you can define your own `ProducerFactory` and `KafkaTemplate` beans.

---

And that's it! Finished! I hope you found this article useful. Integrating Kafka with Spring Boot allows for efficient event-driven system development. With Spring Kafka's auto-configuration, the complexities of message production and consumption are greatly reduced.

Happy coding with Kafka and Spring Boot!