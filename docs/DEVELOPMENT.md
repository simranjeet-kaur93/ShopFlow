# ShopFlow — Local Development and Operations Guide

This guide covers developer prerequisites, verified setup commands, port allocations, testing strategies, and Git workflows.

---

## 1. Prerequisites

Before running ShopFlow locally, ensure you have the following installed:

* **Node.js**: `>= 20.x` LTS
* **npm**: `>= 10.x`
* **Docker Desktop**: `>= 24.x` (with Docker Compose v2)
* **Git**: `>= 2.40`

---

## 2. Infrastructure Port Allocations

| Service / Tool | Protocol | Host Port | Mode | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **API Gateway** | HTTP / REST | `3000` | Ingress | Public client REST API & Swagger UI |
| **User Service** | HTTP / gRPC | `3001` / `50051` | Dual | Auth & user profiles |
| **Product Service** | HTTP / gRPC | `3002` / `50052` | Dual | Catalog & categories |
| **Order Service** | HTTP / gRPC | `3003` / `50053` | Dual | Checkout & Saga orchestration |
| **Payment Service** | HTTP | `3004` | Async-first | Health & telemetry (Queue consumer) |
| **Inventory Service** | HTTP / gRPC | `3005` / `50055` | Dual | Stock reservations |
| **Shipping Service** | HTTP / gRPC | `3006` / `50056` | Dual | Carrier tracking |
| **Notification Service** | HTTP | `3007` | Async-first | Health & telemetry (Event subscriber) |
| **Frontend SPA (Vite)** | HTTP | `5173` | UI | React storefront |
| **PostgreSQL** | TCP | `5432` | Storage | Shared database instance (7 logical DBs) |
| **RabbitMQ AMQP** | TCP | `5672` | Broker | Inter-service messaging |
| **RabbitMQ Console** | HTTP | `15672` | Admin | Web management UI (`guest` / `guest`) |
| **Redis** | TCP | `6379` | Cache | In-memory cache & distributed locks |
| **MinIO (S3 API)** | HTTP | `9000` / `9001` | Storage | Local S3 storage & console UI |

---

## 3. Verified Setup & Workflow

### 3.1 Step 1 (Current State)
Currently, ShopFlow is in the **Architecture & Foundation** phase. No containers or code services are running yet.

Verified commands:
```powershell
# 1. Review environment variables template
Get-Content .env.example

# 2. Verify git status
git status
```

### 3.2 Step 2 (Upcoming Infrastructure Preview)
Once Phase 2 is approved:
```powershell
# 1. Create your local environment file
Copy-Item .env.example .env

# 2. Start backing infrastructure containers
docker compose up -d postgres rabbitmq redis minio

# 3. Verify container health
docker compose ps
```

---

## 4. Multi-Database Provisioning (Local Development)

To avoid running 7 physical database containers, a single PostgreSQL instance hosts **7 isolated logical databases**:

```sql
CREATE DATABASE shopflow_user;
CREATE DATABASE shopflow_product;
CREATE DATABASE shopflow_order;
CREATE DATABASE shopflow_payment;
CREATE DATABASE shopflow_inventory;
CREATE DATABASE shopflow_shipping;
CREATE DATABASE shopflow_notification;
```
Each service connects exclusively to its assigned database URL configured in `.env`.

---

## 5. Testing and Debugging Strategy (Planned)

### 5.1 Unit Tests (Jest)
* Tests domain logic, Saga state machines, and calculations in isolation with mocked dependencies.
* Command: `npm run test`

### 5.2 Integration Tests (Testcontainers)
* Automatically spins up temporary PostgreSQL and RabbitMQ test containers to verify real queries and message publishing.
* Command: `npm run test:integration`

### 5.3 End-to-End Tests (Supertest)
* Simulates complete client checkout requests against the API Gateway, verifying that the asynchronous Saga flow commits and releases reservations as expected.
* Command: `npm run test:e2e`

### 5.4 Diagnostic Probes
* **Liveness Probe**: `curl http://localhost:<port>/health/liveness` (Confirms Node.js event loop is alive).
* **Readiness Probe**: `curl http://localhost:<port>/health/readiness` (Confirms database, broker, and cache connections are active).

---

## 6. Git & Branching Workflow

* **Branching Strategy**:
  * `main`: Stable, verified architecture and code.
  * `feat/<feature-name>`: New phase deliverables or services.
  * `fix/<issue-name>`: Bug fixes and documentation corrections.
* **Commit Guidelines**:
  * Use Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`).
  * Example: `docs: update master architecture blueprint and roadmap`
* **Hygiene Rules**:
  * Never commit `.env` files, credentials, or private keys.
  * Always run `git diff --staged` before committing to verify clean, focused changes.
  * Never push or commit without explicit review.
