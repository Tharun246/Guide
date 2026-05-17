# Kafka + Spring Boot Docker Networking Scenarios

---

# Scenario 1
# Spring Boot Outside Docker

## Architecture

```text
Your Laptop
   ├── Spring Boot App
   └── Kafka Docker Container
```

In this setup:
- Kafka runs inside Docker
- Spring Boot runs directly on your machine
- Spring Boot connects using `localhost:9092`

---

# Producer application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

server:
  port: 8081
```

---

# Consumer application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092

    consumer:
      group-id: order-group
      auto-offset-reset: earliest

      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer

      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

      properties:
        spring:
          json:
            trusted:
              packages: "*"

server:
  port: 8082
```

---

# Docker Compose File

```yaml
version: '3.8'

services:

  kafka:
    image: bitnami/kafka:latest
    container_name: kafka

    ports:
      - "9092:9092"

    environment:

      - KAFKA_CFG_NODE_ID=1

      - KAFKA_CFG_PROCESS_ROLES=broker,controller

      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER

      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093

      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092

      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka:9093

      - ALLOW_PLAINTEXT_LISTENER=yes
```

---

# Communication Flow

```text
Spring Boot App
        ↓
localhost:9092
        ↓
Kafka Container
```

---

# Why INTERNAL and EXTERNAL Listeners Not Needed

Only ONE network path exists:

```text
Host Machine → Kafka
```

So a single PLAINTEXT listener is enough.

---

# Important Rule

Outside Docker:

```text
localhost = your laptop
```

---

# Scenario 2
# Spring Boot ALSO Inside Docker

## Architecture

```text
Docker Network
   ├── Kafka Container
   ├── Producer Container
   └── Consumer Container
```

In this setup:
- Kafka runs inside Docker
- Producer runs inside Docker
- Consumer runs inside Docker

All applications communicate internally through Docker network.

---

# Very Important Docker Rule

Inside a container:

```text
localhost = that SAME container only
```

Example:

Inside Producer container:

```text
localhost = Producer container itself
```

NOT Kafka container.

---

# Producer application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer

      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

server:
  port: 8081
```

---

# Consumer application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092

    consumer:
      group-id: order-group
      auto-offset-reset: earliest

      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer

      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

      properties:
        spring:
          json:
            trusted:
              packages: "*"

server:
  port: 8082
```

---

# Docker Compose File

```yaml
version: '3.8'

services:

  kafka:
    image: bitnami/kafka:latest
    container_name: kafka

    ports:
      - "9092:9092"
      - "29092:29092"

    environment:

      - KAFKA_CFG_NODE_ID=1

      - KAFKA_CFG_PROCESS_ROLES=broker,controller

      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER

      - KAFKA_CFG_LISTENERS=INTERNAL://:9092,EXTERNAL://:29092,CONTROLLER://:9093

      - KAFKA_CFG_ADVERTISED_LISTENERS=INTERNAL://kafka:9092,EXTERNAL://localhost:29092

      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT

      - KAFKA_CFG_INTER_BROKER_LISTENER_NAME=INTERNAL

      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka:9093

      - ALLOW_PLAINTEXT_LISTENER=yes

  producer-service:
    build: ./Producer
    container_name: producer-service

    depends_on:
      - kafka

    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092

  consumer-service:
    build: ./Consumer
    container_name: consumer-service

    depends_on:
      - kafka

    environment:
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

---

# Why localhost Fails Here

Suppose Producer container tries:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
```

This means:

```text
Try connecting to port 9092 INSIDE Producer container itself
```

But Kafka exists in another container.

So connection fails.

---

# Why INTERNAL and EXTERNAL Listeners Needed

Now TWO different network paths exist.

---

# Internal Docker Communication

Containers need:

```text
kafka:9092
```

---

# External Host Machine Communication

Your laptop needs:

```text
localhost:29092
```

These are DIFFERENT addresses.

---

# Multiple Listener Configuration

```yaml
KAFKA_CFG_LISTENERS=
INTERNAL://:9092,
EXTERNAL://:29092,
CONTROLLER://:9093
```

Meaning:

| Listener | Purpose |
|---|---|
| INTERNAL | Docker containers |
| EXTERNAL | Host machine / outside clients |
| CONTROLLER | Kafka internal KRaft coordination |

---

# Advertised Listeners

```yaml
KAFKA_CFG_ADVERTISED_LISTENERS=
INTERNAL://kafka:9092,
EXTERNAL://localhost:29092
```

Meaning:

| Client Type | Address Kafka Returns |
|---|---|
| Docker Containers | kafka:9092 |
| Host Machine | localhost:29092 |

---

# Internal Communication Flow

```text
Producer Container
      ↓
kafka:9092
      ↓
Kafka INTERNAL listener
```

---

# External Communication Flow

```text
Laptop
   ↓
localhost:29092
   ↓
Kafka EXTERNAL listener
```

---

# Final Mental Model

## Outside Docker

```text
localhost = your laptop
```

## Inside Docker Container

```text
localhost = that SAME container only
```

That single concept explains most Kafka Docker networking confusion.
