# Local Development, Configuration, and Testing Guide

## 1. Local Development Philosophy

ShopFlow balances lightweight developer experience with production realism. 
* In early stages, **backing infrastructure** (PostgreSQL, RabbitMQ, Redis, MinIO) runs in lightweight containers via Docker Compose.
* **Microservices and the Frontend** run directly on the host machine using Node.js and TypeScript via npm workspaces. This preserves sub-second hot reloading, rapid debugger attachment, and low memory consumption without requiring heavy full-container rebuilds during code iteration.

---

## 2. Infrastructure Port Allocations

| Service / Infrastructure | Protocol | Host Port | Inbound Mode | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **API Gateway** | HTTP / REST | `3000` | Ingress | Public client ingress & Swagger docs |
| **User Service** | HTTP / gRPC | `3001` / `50051` | Dual | Auth & Profile endpoints |
| **Product Service** | HTTP / gRPC | `3002` / `50052` | Dual | Catalog queries & admin management |
| **Order Service** | HTTP / gRPC | `3003` / `50053` | Dual | Checkout & Saga orchestration |
| **Payment Service** | HTTP | `3004` | Async-First | Health checks & telemetry (Async worker) |
| **Inventory Service** | HTTP / gRPC | `3005` / `50055` | Dual | Stock management & reservations |
| **Shipping Service** | HTTP / gRPC | `3006` / `50056` | Dual | Shipment tracking simulation |
| **Notification Service** | HTTP | `3007` | Async-First | Health checks & dispatch logs (Leaf consumer) |
| **Frontend SPA (Vite)** | HTTP | `5173` | UI | React storefront UI |
| **PostgreSQL** | TCP | `5432` | Storage | Relational storage for all microservices |
| **RabbitMQ AMQP** | TCP | `5672` | Broker | Message broker communication |
| **RabbitMQ Management** | HTTP | `15672` | Admin | Broker web UI (`guest` / `guest`) |
| **Redis** | TCP | `6379` | Cache | In-memory cache & distributed locks |
| **MinIO (S3 API)** | HTTP | `9000` / `9001` | Storage | Local S3 object storage & console |

*Note on Inbound Mode*: Payment Service and Notification Service are **asynchronous-first**. They expose HTTP ports exclusively for `/health` (Terminus) and telemetry, processing all business workflows via RabbitMQ queues.

---

## 3. Database Isolation in Local Development

To maintain strict service boundary isolation without running 7 separate PostgreSQL container instances locally, a single PostgreSQL instance hosts **7 isolated logical databases**:

```sql
CREATE DATABASE shopflow_user;
CREATE DATABASE shopflow_product;
CREATE DATABASE shopflow_order;
CREATE DATABASE shopflow_payment;
CREATE DATABASE shopflow_inventory;
CREATE DATABASE shopflow_shipping;
CREATE DATABASE shopflow_notification;
```

Each service connects to its dedicated database using its own Prisma schema configuration and connection string (defined in `.env`). This ensures physical data separation while keeping local resource consumption minimal.

---

## 4. Configuration and Secrets Management

1. **Template**: All required configuration variables are declared in `.env.example`.
2. **Local Overrides**: Copy `.env.example` to `.env` for local execution. `.env` is ignored by Git.
3. **Fail-Fast Validation**: Each NestJS service runs `@nestjs/config` with Zod/Joi schemas at bootstrap. If a required environment variable is absent or violates type constraints, the process crashes immediately with a descriptive error.
4. **Secret Hygiene**: Real passwords, production API keys, or private certificates must never be placed in `.env.example` or committed to Git.

---

## 5. Multi-Tier Testing Strategy (Planned)

### 5.1 Unit Testing (Jest)
* Isolated tests for domain services, Saga state machines, value objects, and utility functions.
* Mocks external dependencies (Prisma client, RabbitMQ producer, Redis client).
* Executed locally via `npm run test`.

### 5.2 Inter-Service Contract Testing
* Validates Protobuf definitions against gRPC client implementations.
* Guarantees message format backward compatibility across versions.

### 5.3 Integration Testing (Testcontainers & Supertest)
* Spins up ephemeral Docker containers for PostgreSQL and RabbitMQ during test runs.
* Verifies real Prisma migrations, database queries, and RabbitMQ queue bindings without polluting local development state.

### 5.4 End-to-End (E2E) Checkout Saga Testing
* Triggers a checkout request via the API Gateway REST endpoint.
* Verifies asynchronous propagation across Order, Inventory, Payment, and Shipping services.
* Tests both successful checkout completion and compensation rollback upon simulated payment declines.

---

## 6. Progressive Evolution Roadmap

### Stage 1: Architecture, Baseline Documentation, and Contracts (Current)
* Status: `[x] COMPLETED`
* Deliverables: System topology, service boundaries, communication specifications, ADRs, and repository baseline.

### Stage 2: Backing Infrastructure and Monorepo Scaffolding (Next)
* Status: `[ ] PLANNED`
* Deliverables: Docker Compose file for PostgreSQL, RabbitMQ, Redis, and MinIO. Initialization scripts for 7 logical databases. NestJS monorepo initialization with npm workspaces.

### Stage 3: Core Domain Services and Inter-Service gRPC
* Status: `[ ] PLANNED`
* Deliverables: User Service and Product Service implementations. API Gateway authentication termination and gRPC routing. Prisma schemas and seed data generation.

### Stage 4: Asynchronous Messaging and Order Saga Orchestration
* Status: `[ ] PLANNED`
* Deliverables: RabbitMQ client abstraction, Order Saga Orchestrator, Inventory, Payment, and Shipping services. Transactional outbox workers and compensation workflows.

### Stage 5: Frontend Storefront and Full Integration
* Status: `[ ] PLANNED`
* Deliverables: React + Vite storefront using TanStack Query. End-to-end checkout flow testing and validation.

### Stage 6: Cloud Native Evolution (Production Grade)
* Status: `[ ] PLANNED`
* Deliverables: Multi-stage Dockerfiles per service. Kubernetes Helm charts with Ingress NGINX and Horizontal Pod Autoscaling (HPA). OpenTelemetry distributed tracing with Prometheus and Grafana. Target AWS deployment (ECS/EKS, RDS Aurora, ElastiCache, Amazon MQ).
