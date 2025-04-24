# Data Infrastructure Blueprint for Autonomous Fleet System

## 1. System Overview and Scope

The Autonomous Fleet System is a distributed software system that dynamically manages electric and self-driving public transit vehicles within a suburban city environment. It helps optimize fleet allocation, route planning, and scheduling based on real-time data such as demand, traffic, and vehicle status. The primary users include commuters, transit operators, and autonomous vehicle nodes.

**In Scope:**
- Dynamic scheduling and routing
- Real-time telemetry ingestion and command dispatch
- Public-facing apps (passenger tracking)
- Operator dashboard (monitor & control)

**Out of Scope:**
- In-vehicle low-level navigation stack
- Payments and fare collection

## 2. Functional and Non-Functional Requirements

### Functional:
- Monitor vehicle telemetry and location
- Adjust vehicle schedule dynamically
- Send optimized commands to autonomous vehicles
- Visual dashboards for users and operators

### Non-Functional:
- Scalability: handle 1000+ vehicles and millions of events/day
- Security: end-to-end TLS, encrypted storage
- Latency: sub-second response for critical services
- Reliability: 99.99% uptime for control services

## 3. High-Level Architecture

- Components: Web app, Backend APIs, Kafka Stream, Redis, PostgreSQL, ML model runner
- Services communicate via REST, MQTT, gRPC, and Kafka topics

```mermaid
graph TD
  subgraph Network Devices
    GW[API Gateway]
    LB[Load Balancer]
  end

  subgraph Services
    WebApp[Web/Mobile Frontend]
    API[Backend API Service]
    Scheduler[Scheduler Service]
    Allocator[Vehicle Allocator]
    Stream[Flink Stream Processor]
    CmdService[Vehicle Command Service]
  end

  subgraph Databases
    Redis[Redis Cache]
    PG[PostgreSQL + PostGIS]
    Kafka[Kafka Broker]
    S3[S3 / Datalake]
  end

  GW --> LB
  LB --> WebApp
  WebApp --> API
  API --> Scheduler
  API --> Allocator
  API --> Redis
  Scheduler --> PG
  Allocator --> PG
  Allocator --> Redis
  CmdService --> Kafka
  Stream --> Kafka
  Stream --> PG
  Stream --> Redis
  Kafka --> S3

```

Data Infrastructure

```mermaid
graph LR
  Vehicle[Vehicle Telemetry/Sensors]
  MobileApp[Mobile App Events]
  Sergek[Sergek Cameras]

  MQTT[MQTT Broker]
  Gateway[API Gateway]
  Kafka[Kafka Broker]
  Flink[Apache Flink Stream Processing]
  Redis[Redis Cache]
  PG[PostgreSQL + PostGIS]
  Airflow[Airflow ETL Jobs]
  S3[S3 / Data Lake]
  ClickHouse[ClickHouse / BigQuery]
  MLflow[MLflow + MinIO]

  Vehicle --> MQTT --> Kafka
  MobileApp --> Gateway --> Kafka
  Sergek --> Gateway --> Kafka

  Kafka --> Flink
  Flink --> Redis
  Flink --> PG
  PG --> Airflow --> S3
  S3 --> ClickHouse
  S3 --> MLflow

```

## 4. Component Breakdown

| Component            | Responsibilities                                                | Tech Stack               |
|---------------------|------------------------------------------------------------------|--------------------------|
| Web & Mobile App     | Display buses, allow route queries                              | Vue.js, REST, HTTPS      |
| API Gateway          | Secure routing to services, rate-limiting                       | Kong, TLS, JWT OAuth     |
| Scheduler Service    | Route planning based on traffic, demand                         | Python, PostGIS, Redis   |
| Vehicle Command Svc  | Issues commands to buses                                        | Go, MQTT, Redis Streams  |
| Stream Processor     | Real-time data aggregation from sensors                         | Kafka, Flink             |
| Sergek Adapter       | Ingests video data from external systems                        | Kafka Connect, Webhooks  |

## 5. Data Storage and Database Schema

- **Redis**: Caching for live vehicle state and ETA
- **PostgreSQL + PostGIS**: Route graphs, schedules, event logs
- **Kafka**: Event streaming (telemetry, control feedback)
- **S3/ClickHouse**: Long-term telemetry analytics
- **Schema**: Vehicles(id, type, location, battery); Routes(id, path, stops, traffic)

## 6. Technology Stack and Justification

| Layer               | Tech                 | Reason                                        |
|--------------------|----------------------|-----------------------------------------------|
| Frontend            | Vue.js               | Lightweight and fast user-facing app          |
| Backend API         | FastAPI / Flask      | Async capabilities, quick dev cycle           |
| Stream              | Apache Kafka + Flink | Real-time, fault-tolerant streaming           |
| Database            | PostgreSQL + Redis   | Strong consistency, geospatial queries        |
| Cloud/Infra         | AWS + Kubernetes     | Scalable, managed, multi-region support       |
| CI/CD               | GitHub Actions       | Fast deploy pipeline                          |
| Monitoring          | Prometheus + Grafana | Real-time alerts and metrics dashboard        |

## 7. Scalability, Security, Reliability

### Scalability:
- Kafka + Flink support distributed stream processing
- Stateless services behind load balancers
- Redis & Postgres with sharding or clustering as needed

### Security:
- HTTPS everywhere (TLS 1.3)
- OAuth2.0/JWT for user & service auth
- MQTT with client certs for vehicle comms
- Secrets stored in Vault or AWS Secrets Manager

### Reliability:
- Kafka ensures event durability
- PostgreSQL high-availability clusters
- Retry queues and circuit breakers for command delivery
- Cloud-based auto-scaling

## 8. Trade-Offs and Limitations

| Decision                  | Trade-Off                                                       |
|--------------------------|------------------------------------------------------------------|
| Real-time edge compute   | Increased complexity but better latency                          |
| Event streaming          | More infra to manage vs simple REST calls                        |
| Relational DB over NoSQL | Better data integrity, but harder horizontal scaling             |
| External traffic ingest  | Relies on 3rd-party data uptime and integrity                    |
| MQTT for telemetry       | Efficient but limited to small payloads, not great for bulk data |

## 9. C4 Architecture Diagrams (Mermaid.js)

### Context Diagram
```mermaid
C4Context
    title Autonomous Fleet System - Context Diagram
    Person(cityUser, "City User", "Uses the transport app")
    Person(operator, "Operator", "Manages fleet")
    System(system, "Fleet Optimization System", "Optimizes routes and commands autonomous vehicles")
    System_Ext(sergek, "Sergek Analytics", "City traffic camera system")
    System_Ext(app, "Mobile App", "UI for passengers")
    Rel(cityUser, app, "Uses")
    Rel(operator, system, "Monitors via dashboard")
    Rel(app, system, "Tracks buses, sends requests")
    Rel(system, sergek, "Fetches traffic data")
```

### Container Diagram
```mermaid
C4Container
    title Fleet Optimization System - Container Diagram
    Container(webApp, "Web UI", "Vue.js", "Visualizes buses and routes")
    Container(api, "Backend API", "FastAPI", "Handles route planning and scheduling")
    Container(db, "Database", "PostgreSQL", "Stores schedules and vehicle data")
    Container(redis, "Cache", "Redis", "Real-time vehicle state")
    Container(kafka, "Kafka", "Apache Kafka", "Ingest telemetry data")
    Container(flink, "Stream Processor", "Apache Flink", "Real-time processing of vehicle feeds")
    Rel(webApp, api, "REST")
    Rel(api, db, "SQL")
    Rel(api, redis, "Cache lookup")
    Rel(flink, db, "Stores aggregated data")
    Rel(kafka, flink, "Processes streams")
```

### Deployment Diagram
```mermaid
C4Deployment
    title Autonomous Fleet - Deployment
    Node(cloud, "AWS Cloud") {
      Container(api, "Fleet API")
      Container(redis, "Redis Cluster")
      Container(db, "PostgreSQL HA")
    }
    Node(edge, "Vehicle Edge Node") {
      Container(mqttClient, "MQTT Client")
    }
    Node(kafkaZone, "Kafka Stream") {
      Container(kafka, "Apache Kafka")
      Container(flink, "Apache Flink")
    }
    Rel(mqttClient, kafka, "Telemetry Stream")
    Rel(kafka, flink, "Stream Pipeline")
    Rel(flink, db, "Stores Processed Data")
```
