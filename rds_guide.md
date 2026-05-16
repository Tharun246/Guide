# AWS RDS + Spring Boot Complete Setup Guide

# Overview

This guide explains:

1. Creating an AWS RDS database
2. Creating an initial database/schema
3. Connecting Spring Boot to RDS
4. Understanding security groups and networking
5. Running Spring Boot locally
6. Deploying Spring Boot to EC2
7. Connecting EC2 to RDS

---

# Final Architecture

```text
Laptop / Browser
        |
        | HTTP Request
        v
EC2 Instance (Spring Boot App)
        |
        | JDBC Connection
        v
AWS RDS (MySQL)
```

---

# PART 1 — Create AWS RDS Database

## Step 1 — Open RDS Console

Go to AWS Console:

* Search for: `RDS`
* Open the RDS Dashboard

---

# Step 2 — Create Database

Click:

```text
Create Database
```

Choose:

```text
Standard Create
```

---

# Step 3 — Choose Database Engine

Select:

```text
MySQL
```

Version:

* Keep default version

---

# Step 4 — Choose Template

Select:

```text
Free Tier
```

---

# Step 5 — Configure Database Settings

Example:

```text
DB Instance Identifier:
springboot-rds-db1

Master Username:
admin

Password:
StrongPassword123
```

Save:

* username
* password
* endpoint

These are required later.

---

# Step 6 — Create Initial Database

In:

```text
Additional Configuration
```

Find:

```text
Initial Database Name
```

Example:

```text
SpringDb1
```

AWS automatically creates:

```sql
CREATE DATABASE SpringDb1;
```

---

# Step 7 — Connectivity Configuration

## VPC

Keep default VPC.

---

## Public Access

For learning:

```text
Publicly Accessible = YES
```

Meaning:

```text
Your laptop can connect directly to RDS.
```

Production:

```text
Publicly Accessible = NO
```

---

# Step 8 — Security Group

Create a new security group OR use existing.

Add inbound rule:

| Type         | Port | Source |
| ------------ | ---- | ------ |
| MYSQL/Aurora | 3306 | My IP  |

Meaning:

```text
Only my laptop can connect to RDS.
```

---

# Step 9 — Create Database

Click:

```text
Create Database
```

Wait until status becomes:

```text
Available
```

---

# Step 10 — Copy Endpoint

Go to:

```text
RDS -> Databases -> Your DB
```

Find:

```text
Endpoint
```

Example:

```text
springboot-rds-db1.abcdefg.ap-south-1.rds.amazonaws.com
```

This acts like the database hostname.

---


# PART 2 — Create Spring Boot Project

## Dependencies

Use:

* Spring Web
* Spring Data JPA
* MySQL Driver

---

# Configure application.properties

```properties
spring.datasource.url=jdbc:mysql://springboot-rds-db1.abcdefg.ap-south-1.rds.amazonaws.com:3306/SpringDb1
spring.datasource.username=admin
spring.datasource.password=StrongPassword123
```

---

# Important JDBC URL Understanding

```text
jdbc:mysql://HOST:PORT/DATABASE
```

Example:

```text
jdbc:mysql://springboot-rds-db1.abcdefg.ap-south-1.rds.amazonaws.com:3306/SpringDb1
```

Where:

| Part                  | Meaning       |
| --------------------- | ------------- |
| springboot-rds-db1... | RDS Endpoint  |
| 3306                  | MySQL Port    |
| SpringDb1             | Database Name |

---

# PART 4 — Create Spring boot project

Create a GET and POST endpoint to add or get the entity information

---

# PART 5 — Run Spring Boot Application

Run:

```bash
mvn spring-boot:run
```
---

# PART 6 — Common Connection Errors
Make POST and GET requests and check if we are able to fetch the records .

# Error

```text
Communications link failure
```

Usually means:

```text
Application cannot reach RDS over network.
```

---

# Common Causes

| Problem                | Fix                           |
| ---------------------- | ----------------------------- |
| RDS SG blocks laptop   | Add My IP rule                |
| Public access disabled | Set Publicly Accessible = YES |
| Wrong endpoint         | Verify endpoint               |
| Wrong port             | Use 3306                      |
| DB not available       | Wait for Available status     |

---

# IMPORTANT Security Group Understanding

## Case 1 — App Running On Laptop

Need:

| Source |
| ------ |
| My IP  |

---

## Case 2 — App Running On EC2

Need:

| Source             |
| ------------------ |
| EC2 Security Group |

---

# PART 7 — Launch EC2 Instance

## Cretae a EC2 instance with desired inbound and outbound rules 

# Modify RDS Security Group

Add inbound rule:

| Type         | Source             |
| ------------ | ------------------ |
| MYSQL/Aurora | EC2 Security Group |

Meaning:

```text
Only EC2 can access database.
```
---

# PART 9 — SSH Into EC2

## Connect Using SSH

```bash
ssh -i spring-key.pem ec2-user@PUBLIC_IP
```

---

# Install Java

```bash
sudo dnf install java-17-amazon-corretto -y
```

Verify:

```bash
java -version
```

---

# Upload JAR

From local machine:

```bash
scp -i spring-key.pem target/app.jar ec2-user@PUBLIC_IP:/home/ec2-user
```

---

# Run Spring Boot App

Inside EC2:

```bash
java -jar app.jar
```

---

# PART 10 — Access Application Publicly

Open browser:

```text
http://EC2_PUBLIC_IP:8080/employees
```
---

