# ShopFlow — Architectural Decision Records Index

This document summarizes the architectural decisions governing ShopFlow. Detailed technical justifications, assumptions, and alternatives are preserved in the individual ADR files linked below.

---

## Decision Index

| ADR | Title | Decision Summary | Key Trade-off / Rationale | Status |
| :--- | :--- | :--- | :--- | :---: |
| **[ADR-001](decisions/ADR-001-monorepo.md)** | Monorepo with NestJS & npm Workspaces | Manage all microservices and shared libraries in a single repository using npm workspaces and NestJS CLI. | Simplifies atomic contract updates and local orchestration; requires lint rules to prevent illegal inter-service imports. | `Accepted` |
| **[ADR-002](decisions/ADR-002-service-data-ownership.md)** | Database-per-Service Pattern | Each microservice strictly owns a dedicated PostgreSQL database managed via Prisma. No cross-database SQL queries or foreign keys. | Guarantees independent service scalability and decoupled schema evolution; requires eventual consistency and Saga rollbacks instead of ACID joins. | `Accepted` |
| **[ADR-003](decisions/ADR-003-service-communication.md)** | Hybrid gRPC & RabbitMQ Architecture | Public client uses REST (JSON); internal synchronous queries use gRPC (Protobuf); asynchronous transactions and Saga commands use RabbitMQ. | Optimizes latency for fast reads (gRPC) while ensuring fault isolation, buffering, and retry resilience for checkout flows (RabbitMQ). | `Accepted` |
| **[ADR-004](decisions/ADR-004-authentication.md)** | Centralized Ingress Authentication | API Gateway validates client JWTs and injects verified identity headers (`X-User-Id`, `X-User-Roles`) to internal services. | Offloads repetitive cryptographic verification from internal services; internal network must be private and trusted. | `Accepted` |
| **[ADR-005](decisions/ADR-005-observability.md)** | Structured Logging, Correlation IDs & Probes | Standardize structured JSON logging to `stdout`, propagate `X-Correlation-Id` across all hops, and split `/health` into Liveness and Readiness. | Delivers full distributed traceability and reliable container rolling restarts without vendor lock-in. | `Accepted` |

---

## Detailed ADR Records

For complete context, assumptions, trade-offs, and alternatives considered, inspect the individual ADR files:
* [`ADR-001: Monorepo Architecture with NestJS and npm Workspaces`](decisions/ADR-001-monorepo.md)
* [`ADR-002: Service Data Ownership and Database-Per-Service Pattern`](decisions/ADR-002-service-data-ownership.md)
* [`ADR-003: Service Communication Architecture: Hybrid gRPC and RabbitMQ`](decisions/ADR-003-service-communication.md)
* [`ADR-004: Centralized Ingress Authentication and Internal Identity Propagation`](decisions/ADR-004-authentication.md)
* [`ADR-005: Observability Strategy: Structured Logging, Correlation IDs, and Distributed Tracing`](decisions/ADR-005-observability.md)
