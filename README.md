# Gym CRM Microservices

A microservices-based extension of the Gym CRM REST API built with Spring Boot.

This project extends the main Gym CRM application with a dedicated Trainer Workload Microservice. Whenever a training session is created or deleted, the main application sends the corresponding workload information to the Trainer Workload Service, which calculates and stores the trainer's monthly workload.

---

## Architecture

```text
                    ┌─────────────────────┐
                    │       Postman       │
                    └──────────┬──────────┘
                               │
                               │ HTTP + JWT
                               ▼
                    ┌─────────────────────┐
                    │   Gym Spring Boot   │
                    │      :8080          │
                    └──────────┬──────────┘
                               │
                    Eureka Discovery
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Trainer Workload    │
                    │     Service         │
                    │      :8081          │
                    └──────────┬──────────┘
                               │
                               ▼
                         H2 In-Memory DB


                    ┌─────────────────────┐
                    │   Eureka Server     │
                    │      :8761          │
                    └─────────────────────┘
```

### Services

| Service | Port | Responsibility |
|---|---:|---|
| Gym Spring Boot | `8080` | Main Gym CRM REST API |
| Trainer Workload Service | `8081` | Trainer monthly workload calculation |
| Eureka Discovery Service | `8761` | Service discovery and registration |

---

## Main Features

### Gym CRM

The main application provides:

- Trainee management
- Trainer management
- Training management
- Training type management
- Authentication with JWT
- User authorization
- H2 database
- REST API
- Swagger / OpenAPI
- Actuator and Prometheus metrics

### Trainer Workload Microservice

The workload service:

- Receives trainer workload information
- Processes `ADD` and `DELETE` operations
- Calculates monthly training duration
- Stores workload information in an in-memory H2 database
- Provides REST endpoints for workload operations
- Registers itself with Eureka
- Validates JWT Bearer tokens

---

# Microservice Integration

Whenever a training is created:

```text
POST /api/trainings
        │
        ▼
Gym Spring Boot
        │
        │ ActionType = ADD
        ▼
Trainer Workload Service
        │
        ▼
Monthly workload + training duration
```

When a training is deleted:

```text
DELETE /api/trainings/{trainingId}
        │
        ▼
Gym Spring Boot
        │
        │ ActionType = DELETE
        ▼
Trainer Workload Service
        │
        ▼
Monthly workload is decreased
```

---

# Trainer Workload Model

The workload is stored using the following structure:

```text
TrainerWorkload
│
├── trainerUsername
├── trainerFirstName
├── trainerLastName
├── active
│
└── years
      │
      ├── YearSummary
      │      └── year
      │
      └── months
             ├── month
             └── trainingSummaryDuration
```

Example:

```text
Mike.Smith
└── 2026
    └── August
        └── 180 minutes
```

---

# REST API

## Create Training

```http
POST /api/trainings
```

Creates a new training and sends an `ADD` workload event to the Trainer Workload Service.

### Authorization

```http
Authorization: Bearer <JWT_TOKEN>
```

---

## Delete Training

```http
DELETE /api/trainings/{trainingId}
```

Deletes a training and sends a `DELETE` workload event to the Trainer Workload Service.

---

## Get Trainer Trainings

```http
GET /api/trainings/trainers/{username}/trainings
```

---

## Get Trainee Trainings

```http
GET /api/trainings/trainees/{username}/trainings
```

---

## Get Training Types

```http
GET /api/trainings/types
```

---

## Trainer Workload

```http
POST /api/workloads
```

### Request

```json
{
  "trainerUsername": "Mike.Smith",
  "trainerFirstName": "Mike",
  "trainerLastName": "Smith",
  "isActive": true,
  "trainingDate": "2026-08-08",
  "trainingDuration": 60,
  "actionType": "ADD"
}
```

### Action Types

```text
ADD
DELETE
```

---

# Service Discovery

The system uses Netflix Eureka for service discovery.

The Gym application does not directly depend on a fixed workload-service address.

Instead of:

```text
http://localhost:8081
```

the service is accessed through:

```text
http://trainer-workload-service
```

Eureka resolves the service name to an available instance.

### Eureka Server

```text
http://localhost:8761
```

---

# Circuit Breaker

The communication between Gym and Trainer Workload Service uses the Circuit Breaker pattern.

The Circuit Breaker protects the main application when the workload service becomes unavailable.

Configuration includes:

- Sliding window
- Failure rate threshold
- Open-state duration
- Half-open calls
- Automatic transition
- TimeLimiter

Example configuration:

```properties
resilience4j.circuitbreaker.instances.trainerWorkloadService.sliding-window-size=5
resilience4j.circuitbreaker.instances.trainerWorkloadService.minimum-number-of-calls=3
resilience4j.circuitbreaker.instances.trainerWorkloadService.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.trainerWorkloadService.wait-duration-in-open-state=10s
resilience4j.circuitbreaker.instances.trainerWorkloadService.permitted-number-of-calls-in-half-open-state=2
resilience4j.circuitbreaker.instances.trainerWorkloadService.automatic-transition-from-open-to-half-open-enabled=true

resilience4j.timelimiter.instances.trainerWorkloadService.timeout-duration=10s
```

If the workload service is unavailable, the Circuit Breaker executes a fallback and prevents the failure from directly breaking the main training operation.

---

# Security

The system uses JWT-based authentication.

Requests requiring authentication must contain:

```http
Authorization: Bearer <JWT_TOKEN>
```

The JWT token is validated by the Gym application.

The same Bearer token is propagated to the Trainer Workload Service during microservice communication.

```text
Client
  │
  │ Bearer Token
  ▼
Gym
  │
  │ Bearer Token
  ▼
Trainer Workload Service
```

---

# Transaction Tracking & Logging

The application implements transaction-level and operation-level logging.

Each transaction receives a unique:

```text
transactionId
```

The transaction ID is stored in MDC and included in application logs.

Example:

```text
[d6ac94a9-8b0f-4e8a-ae5c-36638294f2ca]
```

The same transaction ID is propagated to downstream services using:

```http
X-Transaction-Id
```

Example flow:

```text
Gym

[d6ac94a9-8b0f-4e8a-ae5c-36638294f2ca]
POST /api/trainings

        │
        │ X-Transaction-Id
        ▼

Trainer Workload Service

[d6ac94a9-8b0f-4e8a-ae5c-36638294f2ca]
Processing trainer workload

[d6ac94a9-8b0f-4e8a-ae5c-36638294f2ca]
Trainer workload processed successfully
```

This makes it possible to follow a complete request across multiple microservices.

---

# REST Maturity

The REST API follows the second level of the Richardson Maturity Model.

The API uses:

- Resource-oriented URLs
- HTTP methods
- HTTP status codes

Examples:

```text
GET     /api/trainings/...
POST    /api/trainings
DELETE  /api/trainings/{id}
```

HATEOAS is not required because the implementation targets Level 2 rather than Level 3.

---

# Technologies

## Backend

- Java 21
- Spring Boot
- Spring Web
- Spring Data JPA
- Spring Security
- Spring Validation
- Spring Cloud
- Netflix Eureka
- Resilience4j
- JWT
- MapStruct
- Lombok

## Database

- H2 In-Memory Database

## Documentation & Monitoring

- Swagger / OpenAPI
- Spring Boot Actuator
- Micrometer
- Prometheus

## Build Tool

- Maven

---

# Running the Application

## 1. Start Eureka Server

Start the Eureka Discovery Service first.

```bash
./mvnw spring-boot:run
```

Eureka Dashboard:

```text
http://localhost:8761
```

---

## 2. Start Trainer Workload Service

From the Trainer Workload Service directory:

### Windows

```powershell
.\mvnw spring-boot:run
```

The service runs on:

```text
http://localhost:8081
```

---

## 3. Start Gym Spring Boot

From the Gym project directory:

### Windows

```powershell
.\mvnw spring-boot:run
```

The main application runs on:

```text
http://localhost:8080
```

---

# Testing

The API can be tested using Postman or Swagger UI.

Swagger:

```text
http://localhost:8080/swagger-ui/index.html
```

A typical integration test flow is:

```text
1. Start Eureka
       ↓
2. Start Trainer Workload Service
       ↓
3. Start Gym Spring Boot
       ↓
4. Login and obtain JWT
       ↓
5. Send POST /api/trainings
       ↓
6. Gym creates training
       ↓
7. Gym sends ADD workload
       ↓
8. Workload Service updates monthly duration
       ↓
9. Verify logs and transactionId
```

---

# Example Integration Log

Successful integration:

```text
Gym:
[d6ac94a9-8b0f-4e8a-ae5c-36638294f2ca]
Training created successfully. id=1

Trainer Workload Service:
[d6ac94a9-8b0f-4e8a-ae5c-36638294f2ca]
Processing trainer workload...

Trainer Workload Service:
[d6ac94a9-8b0f-4e8a-ae5c-36638294f2ca]
Trainer workload processed successfully.
```

The same transaction ID confirms that the request was successfully traced across services.

---

# Project Structure

```text
gym-springboot/
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
└── pom.xml
```

```text
trainer-workload-service/
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
└── pom.xml
```

```text
discovery-service/
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
└── pom.xml
```

---

# Key Design Patterns

The project demonstrates several backend and microservice patterns:

- Microservice Architecture
- Service Discovery
- Circuit Breaker
- JWT Authentication
- Transaction ID Propagation
- Layered Architecture
- DTO Pattern
- Mapper Pattern
- Repository Pattern
- Facade Pattern

---

# Status

The microservices integration includes:

- [x] Separate Trainer Workload Microservice
- [x] Trainer workload REST API
- [x] Trainer monthly workload calculation
- [x] In-memory H2 database
- [x] Gym → Workload integration
- [x] ADD / DELETE workload operations
- [x] Eureka Service Discovery
- [x] Circuit Breaker
- [x] JWT Bearer Authorization
- [x] Transaction ID propagation
- [x] Transaction-level logging
- [x] Operation-level logging
- [x] Richardson Maturity Model Level 2 REST API