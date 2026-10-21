# ShopFlow Production Readiness Checklist

This document tracks the operational, architectural, and security readiness criteria for the ShopFlow platform across all development stages.

**Current Stage**: Step 1 — Architecture and Repository Foundation  
**Legend**:
* `[x] VERIFIED`: Documented, audited, and verified in active repository artifacts.
* `[ ] PLANNED`: Designed and scheduled for future implementation stages.
* `[!] NOT APPLICABLE YET`: Deferred to cloud/production operational stages.

---

## 1. Architecture, Modularity, and Boundaries

| Requirement | Stage | Status | Verification Reference |
| :--- | :---: | :---: | :--- |
| **Strict Service Boundaries** | Step 1 | `[x] VERIFIED` | Documented in [`service-boundaries.md`](architecture/service-boundaries.md). 8 distinct applications with isolated business roles. |
| **Database-per-Service Isolation** | Step 1 | `[x] VERIFIED` | Documented in [`ADR-002`](decisions/ADR-002-service-data-ownership.md). No cross-database joins, foreign keys, or shared schemas. |
| **Zero Business Logic in Gateway** | Step 1 | `[x] VERIFIED` | Mandated in [`system-overview.md`](architecture/system-overview.md). Gateway handles only routing, rate limiting, and auth termination. |
| **Monorepo Structure with npm Workspaces** | Step 2 | `[ ] PLANNED` | Architecture approved in [`ADR-001`](decisions/ADR-001-monorepo.md). Implementation scheduled for Step 2. |
| **Independent Multi-Stage Dockerfiles** | Step 6 | `[ ] PLANNED` | Container definitions per service allowing independent horizontal scaling. |

---

## 2. Inter-Service Communication and Reliability

| Requirement | Stage | Status | Verification Reference |
| :--- | :---: | :---: | :--- |
| **Public REST API Standard (RFC 7807)** | Step 1 | `[x] VERIFIED` | Protocol justified in [`ADR-003`](decisions/ADR-003-service-communication.md). Problem Details error format defined. |
| **Internal Synchronous gRPC Interfaces** | Step 3 | `[ ] PLANNED` | Protocol Buffers (`.proto`) and gRPC client/server bindings scheduled for Step 3. |
| **RabbitMQ Asynchronous Messaging & DLX** | Step 4 | `[ ] PLANNED` | Topology documented in [`communication-flows.md`](architecture/communication-flows.md). Topic exchange and dead-letter queues. |
| **Order Saga Orchestrator & Compensation** | Step 4 | `[ ] PLANNED` | State machine and rollback workflows documented in [`communication-flows.md`](architecture/communication-flows.md). |
| **Transactional Outbox Pattern** | Step 4 | `[ ] PLANNED` | Outbox table and background dispatcher pattern designed in [`communication-flows.md`](architecture/communication-flows.md). |
| **Idempotent Mutations** | Step 4 | `[ ] PLANNED` | Idempotency keys for payment and stock reservations specified in [`communication-flows.md`](architecture/communication-flows.md). |

---

## 3. Security, Authentication, and Data Protection

| Requirement | Stage | Status | Verification Reference |
| :--- | :---: | :---: | :--- |
| **Centralized Ingress Auth Termination** | Step 1 | `[x] VERIFIED` | Architectural policy approved in [`ADR-004`](decisions/ADR-004-authentication.md). Gateway validates JWTs and injects `X-User-*`. |
| **Password Hashing (Argon2id / Bcrypt)** | Step 3 | `[ ] PLANNED` | User Service credential security specification documented in [`service-boundaries.md`](architecture/service-boundaries.md). |
| **Refresh Token Rotation & Revocation Denylist** | Step 3 | `[ ] PLANNED` | Redis-backed token denylist and hash-stored refresh tokens in `users_db`. |
| **Database Network Protection** | Step 2 | `[ ] PLANNED` | Local development uses container bridge networks; production isolates DBs in private VPC subnets with zero public IP exposure. |
| **Secret Management Hygiene** | Step 1 | `[x] VERIFIED` | `.gitignore` verified to block all `.env` files; `.env.example` contains only harmless placeholders. |
| **Simulated Payment Security (PCI Compliance)** | Step 1 | `[x] VERIFIED` | Mandated in [`service-boundaries.md`](architecture/service-boundaries.md). Zero real credit card numbers or live payment keys permitted. |
| **Object File Storage (Images / Assets)** | Step 3 | `[ ] PLANNED` | Product Service stores image metadata in `products_db`; binary assets stored in S3-compatible object storage (MinIO locally, AWS S3 in production). |
| **Immutable Audit Trails** | Step 4 | `[ ] PLANNED` | Payment transactions and Order state changes recorded in append-only database tables with actor and correlation ID logging. |

---

## 4. Observability, Telemetry, and Diagnostics

| Requirement | Stage | Status | Verification Reference |
| :--- | :---: | :---: | :--- |
| **Distributed Correlation ID (`X-Correlation-Id`)** | Step 1 | `[x] VERIFIED` | Propagation standard defined in [`ADR-005`](decisions/ADR-005-observability.md) and [`system-overview.md`](architecture/system-overview.md). |
| **Structured JSON Logging to stdout** | Step 2 | `[ ] PLANNED` | Pino/Winston logging library (`libs/logger`) scheduled for Step 2 monorepo scaffolding. |
| **Health Checks (Liveness vs. Readiness Probes)** | Step 1 | `[x] VERIFIED` | Specifications for `@nestjs/terminus` endpoints documented in [`system-overview.md`](architecture/system-overview.md) and [`ADR-005`](decisions/ADR-005-observability.md). |
| **OpenTelemetry Distributed Tracing** | Step 6 | `[ ] PLANNED` | W3C trace context propagation designed; collector integration scheduled for production phase. |
| **Metrics Collection (Prometheus / Grafana)** | Step 6 | `[ ] PLANNED` | Prometheus endpoint scraping and Grafana dashboard templates scheduled for Step 6. |

---

## 5. Local Development and Deployment Lifecycle

| Requirement | Stage | Status | Verification Reference |
| :--- | :---: | :---: | :--- |
| **Docker Compose Backing Infrastructure** | Step 2 | `[ ] PLANNED` | Local Docker Compose configuration for PostgreSQL, RabbitMQ, and Redis. |
| **Automated Multi-Database Provisioning** | Step 2 | `[ ] PLANNED` | PostgreSQL init script to create 7 isolated databases automatically upon container startup. |
| **Prisma Schema Migrations per Service** | Step 3 | `[ ] PLANNED` | Independent Prisma schemas and migrations per microservice. |
| **Automated Testing Suite (Unit, Integration, E2E)** | Step 3–5| `[ ] PLANNED` | Jest for unit tests, Supertest for Gateway E2E, Testcontainers for integration tests. |
| **Kubernetes Helm Charts & Ingress Configuration** | Step 6 | `[ ] PLANNED` | Cloud-native deployment manifests scheduled for final stage. |
| **CI/CD Automation via GitHub Actions** | Step 6 | `[ ] PLANNED` | Automated linting, test execution, container builds, and security scans on pull request. |
