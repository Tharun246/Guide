# Kafka + Spring Boot Docker Networking Scenarios

---

# Scenario 1
## Spring Boot Outside Docker

Architecture:

```text
Laptop
  ├── Spring Boot App
  └── Kafka Docker Container
```

Bootstrap Server:

```yaml
localhost:9092
```

Reason:
- Kafka port exposed to host machine
- Spring Boot app runs directly on laptop

Kafka Listener:

```yaml
KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
```

Advertised Listener:

```yaml
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
```

Flow:

```text
Spring Boot App
      ↓
localhost:9092
      ↓
Kafka Container
```

---

# Scenario 2
## Spring Boot ALSO Inside Docker

Architecture:

```text
Docker Network
   ├── Kafka Container
   ├── Producer Container
   └── Consumer Container
```

Important Docker Rule:

```text
localhost inside container = same container only
```

Containers communicate using:
- service names
- container names

Correct Bootstrap Server:

```yaml
kafka:9092
```

NOT:

```yaml
localhost:9092
```

Kafka Listeners:

```yaml
KAFKA_CFG_LISTENERS=
INTERNAL://:9092,
EXTERNAL://:29092,
CONTROLLER://:9093
```

Advertised Listeners:

```yaml
KAFKA_CFG_ADVERTISED_LISTENERS=
INTERNAL://kafka:9092,
EXTERNAL://localhost:29092
```

Meaning:

| Listener | Used By |
|---|---|
| INTERNAL | Docker containers |
| EXTERNAL | Host machine |
| CONTROLLER | Kafka internal coordination |

Internal Flow:

```text
Producer Container
      ↓
kafka:9092
      ↓
Kafka INTERNAL listener
```

External Flow:

```text
Laptop
   ↓
localhost:29092
   ↓
Kafka EXTERNAL listener
```

---

# Final Mental Model

Outside Docker:

```text
localhost = your laptop
```

Inside Docker:

```text
localhost = that container itself
```

That is the core reason why INTERNAL and EXTERNAL listeners exist.
