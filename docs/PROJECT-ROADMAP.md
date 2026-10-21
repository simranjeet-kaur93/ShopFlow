# ShopFlow — Project Implementation Roadmap

This document outlines the incremental development phases for ShopFlow. Each phase must be verified before proceeding to the next.

**Legend**:
* `[x] COMPLETED`: Built and verified with automated tests or repository inspection.
* `[>] CURRENT / NEXT`: Active phase ready for implementation.
* `[ ] PLANNED`: Designed and scheduled for future phases.

---

## Roadmap Phases

### Phase 1: Repository Foundation & Architecture
* **Status**: `[x] COMPLETED`
* **Deliverables**:
  * Architecture specifications, service boundaries, and data ownership rules.
  * Architectural Decision Records ([ADR-001](decisions/ADR-001-monorepo.md) to [ADR-005](decisions/ADR-005-observability.md)).
  * Configuration templates ([`.env.example`](../.env.example)) and Git exclusion rules ([`.gitignore`](../.gitignore)).
* **Verification Criteria**:
  * `[x]` Clean Git working tree with no committed secrets.
  * `[x]` Consistent boundary, communication, and failure compensation specifications.
* **Next Action**: Proceed to Phase 2.

---

### Phase 2: Local Backing Infrastructure & Monorepo Scaffolding
* **Status**: `[>] CURRENT / NEXT ACTION`
* **Deliverables**:
  * Root `package.json` with npm workspaces and NestJS monorepo configuration (`nest-cli.json`, `tsconfig.json`).
  * `docker-compose.yml` for PostgreSQL, RabbitMQ, Redis, and MinIO.
  * PostgreSQL initialization script creating 7 logical databases (`shopflow_user`, `shopflow_product`, etc.).
  * Shared core library skeleton (`libs/common`, `libs/proto`, `libs/logger`).
* **Verification Criteria**:
  * Containers start cleanly via `docker compose up -d` with healthy status.
  * 7 distinct PostgreSQL databases are queryable.
  * RabbitMQ management console (port 15672) and Redis (port 6379) respond.
  * Root TypeScript build (`npm run build`) completes with zero errors.

---

### Phase 3: Core Domain Services & Synchronous gRPC
* **Status**: `[ ] PLANNED`
* **Deliverables**:
  * User Service: Registration, Argon2id password hashing, JWT token issuance, and `users_db` Prisma migrations.
  * Product Service: Catalog management, categories, Redis read caching, and `products_db` Prisma migrations.
  * API Gateway: Ingress routing, JWT signature verification, header injection (`X-User-Id`), and gRPC client bindings to User and Product services.
* **Verification Criteria**:
  * Unit tests pass with `npm run test`.
  * Gateway routes `POST /api/v1/auth/register` and `POST /api/v1/auth/login` successfully.
  * Authenticated Gateway requests forward verified identity headers over gRPC.

---

### Phase 4: Asynchronous Messaging & Order Saga Orchestration
* **Status**: `[ ] PLANNED`
* **Deliverables**:
  * RabbitMQ client abstraction (`libs/rabbitmq`) supporting topic exchanges, DLX, and ack/nack handling.
  * Order Service: Checkout initiation, Saga Orchestrator state machine, and Transactional Outbox worker.
  * Inventory Service: Stock allocation, temporary reservation hold, and release compensation.
  * Payment Service: Simulated idempotent transaction processing (`payments_db`).
  * Shipping Service: Parcel tracking creation and address validation.
  * Notification Service: Event subscriber logging transactional notifications.
* **Verification Criteria**:
  * Successful checkout completes the full Saga: Reserve -> Charge -> Ship -> Confirm.
  * Simulated payment decline triggers compensation: Order cancels and reserved inventory is released back.
  * Outbox messages publish reliably without duplicate state execution (idempotency key verified).

---

### Phase 5: Frontend Storefront Application
* **Status**: `[ ] PLANNED`
* **Deliverables**:
  * React + TypeScript SPA built with Vite.
  * Server-state management via TanStack Query.
  * Storefront pages: Product catalog, product detail, shopping cart, user auth, and checkout tracking.
  * Design system using pure Vanilla CSS.
* **Verification Criteria**:
  * Storefront builds cleanly via `npm run build`.
  * User can register, browse products, add to cart, and complete asynchronous checkout.

---

### Phase 6: Observability, Containerization & Production Readiness
* **Status**: `[ ] PLANNED`
* **Deliverables**:
  * Multi-stage Dockerfiles for each microservice and the API Gateway.
  * OpenTelemetry distributed tracing and metrics export.
  * Prometheus scraping and Grafana dashboard templates.
  * Kubernetes Helm charts with Ingress NGINX and Horizontal Pod Autoscaling (HPA).
  * GitHub Actions CI/CD workflows for linting, testing, and container packaging.
* **Verification Criteria**:
  * Full end-to-end trace viewable in Jaeger/Tempo using a single `X-Correlation-Id`.
  * All containers deploy and pass Terminus `/health/liveness` and `/health/readiness` probes.
