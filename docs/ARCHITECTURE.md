# ShopFlow — Master Architecture Blueprint

This document defines the system topology, service boundaries, communication protocols, and reliability patterns for ShopFlow.

**Implementation Status**: All backend services and infrastructure are currently **PLANNED**. Architecture specifications and design contracts are **VERIFIED** (Phase 1).

---

## 1. System Topology

```mermaid
flowchart TB
    subgraph Client["Client Layer"]
        SPA["Frontend SPA\n(React + TypeScript + Vite + TanStack Query)"]
    end

    subgraph Edge["Edge / Ingress Layer"]
        GW["API Gateway\n(NestJS REST API / Auth Guard / Rate Limiter)"]
    end

    subgraph Services["Core Microservices (NestJS + TypeScript)"]
        USR["User Service\n(Auth & Identity)"]
        PRD["Product Service\n(Catalog & Categories)"]
        ORD["Order Service\n(Checkout & Saga Orchestrator)"]
        PAY["Payment Service\n(Simulated Transactions)"]
        INV["Inventory Service\n(Stock & Reservations)"]
        SHP["Shipping Service\n(Fulfillment & Tracking)"]
        NTF["Notification Service\n(Event Dispatcher)"]
    end

    subgraph Persistence["Persistence Layer (PostgreSQL via Prisma)"]
        DB_USR[("users_db")]
        DB_PRD[("products_db")]
        DB_ORD[("orders_db")]
        DB_PAY[("payments_db")]
        DB_INV[("inventory_db")]
        DB_SHP[("shipping_db")]
        DB_NTF[("notifications_db")]
    end

    subgraph Storage["Cache & Object Storage"]
        REDIS[("Redis\n(Rate Limiting, Catalog Cache, Locks)")]
        MINIO[("S3 Object Storage\n(MinIO / AWS S3 - Media)")]
    end

    subgraph Broker["Message Broker"]
        RMQ{{"RabbitMQ\n(Topic Exchanges, Queues, DLX)"}}
    end

    %% Client Ingress
    SPA -->|"HTTP / REST"| GW

    %% Internal Synchronous Ingress (gRPC)
    GW -->|"gRPC"| USR
    GW -->|"gRPC"| PRD
    GW -->|"gRPC"| ORD
    GW -->|"gRPC"| SHP

    %% Database 1:1 Attachments
    USR --- DB_USR
    PRD --- DB_PRD
    ORD --- DB_ORD
    PAY --- DB_PAY
    INV --- DB_INV
    SHP --- DB_SHP
    NTF --- DB_NTF

    %% Cache & Storage Access
    GW -.-> REDIS
    PRD -.-> REDIS
    PRD -.-> MINIO
    PAY -.-> REDIS

    %% Asynchronous Messaging
    ORD -->|"Saga Commands / Events"| RMQ
    PAY -->|"Payment Events"| RMQ
    INV -->|"Stock Events"| RMQ
    SHP -->|"Shipping Events"| RMQ
    USR -->|"User Events"| RMQ

    RMQ --> ORD
    RMQ --> PAY
    RMQ --> INV
    RMQ --> SHP
    RMQ --> NTF
```

---

## 2. Service Responsibilities & Database Ownership

ShopFlow strictly enforces the **Database-per-Service pattern**. Each microservice exclusively owns its database. Direct cross-database SQL queries, foreign keys, or shared schemas are prohibited.

| Application | Primary Responsibility | Persistent Datastore | Cache / Storage | External / Sync APIs | Async Queue Interfaces | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **API Gateway** | Reverse proxy, JWT auth validation, rate limiting, request correlation (`X-Correlation-Id`), error normalization (RFC 7807). No business logic. | *None (Stateless)* | Redis (Rate limits, denylist) | Inbound: Public REST<br>Outbound: gRPC to User, Product, Order, Shipping | None | `PLANNED` |
| **User Service** | User registration, credential storage, Argon2id password hashing, JWT access and refresh token issuance. | `users_db` (Postgres) | Redis (Refresh tokens) | Inbound: gRPC (`GetUserById`, `ValidateToken`) | Outbound: `user.registered` | `PLANNED` |
| **Product Service** | Product catalog, categories, pricing, descriptions, and media metadata. | `products_db` (Postgres) | Redis (Catalog cache)<br>MinIO/S3 (Images) | Inbound: gRPC (`GetProductById`, `CheckPricing`) | Outbound: `product.created`, `product.updated` | `PLANNED` |
| **Order Service** | Order lifecycle state machine, Checkout **Saga Orchestrator**, Transactional Outbox table. | `orders_db` (Postgres) | *None* | Inbound: gRPC (`GetOrderById`) | Outbound: Saga Commands<br>Inbound: Domain Events | `PLANNED` |
| **Payment Service** | Simulated payment authorization, capture, refunds, and idempotency key enforcement. Zero real card storage. | `payments_db` (Postgres) | Redis (Idempotency locks) | *None (Async-First)* | Inbound: `payment.process`<br>Outbound: `payment.succeeded`, `payment.failed` | `PLANNED` |
| **Inventory Service** | Warehouse stock tracking, temporary reservations during checkout, restock updates. | `inventory_db` (Postgres) | *None* | Inbound: gRPC (`GetStockLevels`) | Inbound: `inventory.reserve_stock`, `inventory.release_stock` | `PLANNED` |
| **Shipping Service** | Delivery address validation, carrier label generation simulation, parcel tracking stages. | `shipping_db` (Postgres) | *None* | Inbound: gRPC (`GetShipmentTracking`) | Inbound: `shipping.create`<br>Outbound: `shipping.dispatched` | `PLANNED` |
| **Notification Service** | Asynchronous transactional notification dispatcher (emails, alerts). | `notifications_db` (Postgres) | *None* | *None (Async-First)* | Inbound: Subscribes to all platform domain events | `PLANNED` |

---

## 3. Communication Protocols: REST vs. gRPC vs. RabbitMQ

| Protocol | Scope | What It Does | Why It Exists | When to Use It |
| :--- | :--- | :--- | :--- | :--- |
| **REST (HTTP/JSON)** | Public Client Ingress | Client-to-Gateway communication. | Standard browser compatibility, cacheability, and RFC 7807 error responses. | Web and mobile clients accessing the platform API. |
| **gRPC (HTTP/2 + Protobuf)** | Internal Synchronous | Direct service-to-service point-to-point queries. | Binary serialization, strongly-typed contracts (`.proto`), connection multiplexing, and low latency. | When a caller cannot proceed without an immediate response (e.g., Gateway checking user profile or product price). |
| **RabbitMQ (AMQP 0-9-1)** | Internal Asynchronous | Transactional workflows, domain events, and Saga commands. | Decouples services, buffers load, guarantees at-least-once delivery, and isolates downstream failures. | Multi-service business processes (e.g., checkout, order fulfillment, notifications). |

---

## 4. Order Workflow, Saga Orchestration, and Reliability

Distributed transactions across independent microservices cannot use two-phase commit (2PC). ShopFlow uses the **Orchestrated Saga Pattern**, where the **Order Service** manages the checkout state machine.

### 4.1 Happy Path Checkout Flow

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Customer / Browser
    participant GW as API Gateway
    participant ORD as Order Service (Orchestrator)
    participant INV as Inventory Service
    participant PAY as Payment Service
    participant SHP as Shipping Service
    participant NTF as Notification Service
    participant RMQ as RabbitMQ

    Customer->>GW: POST /api/v1/orders/checkout
    GW->>ORD: gRPC CreateOrderDraft()
    ORD->>ORD: Persist Order (Status: PENDING_PAYMENT)
    ORD-->>GW: Return Order ID (HTTP 202 Accepted)
    GW-->>Customer: Order Initiated (Tracking ID)

    Note over ORD,RMQ: Step 1: Reserve Stock
    ORD->>RMQ: Command: inventory.reserve_stock
    RMQ->>INV: Consume: inventory.reserve_stock
    INV->>INV: Hold stock in inventory_db
    INV->>RMQ: Event: inventory.reserved
    RMQ->>ORD: Consume: inventory.reserved

    Note over ORD,RMQ: Step 2: Process Payment
    ORD->>RMQ: Command: payment.process
    RMQ->>PAY: Consume: payment.process
    PAY->>PAY: Execute simulated charge (idempotent)
    PAY->>RMQ: Event: payment.succeeded
    RMQ->>ORD: Consume: payment.succeeded

    Note over ORD,RMQ: Step 3: Fulfill Order & Dispatch
    ORD->>ORD: Update Order (Status: CONFIRMED)
    ORD->>RMQ: Command: shipping.create
    RMQ->>SHP: Consume: shipping.create -> Create shipment
    ORD->>RMQ: Event: order.confirmed
    RMQ->>NTF: Consume: order.confirmed -> Send confirmation email
```

---

### 4.2 Failure Handling & Compensation (Payment Declined)

If payment fails, the Saga Orchestrator triggers **compensating transactions** to reverse previously completed steps:

```mermaid
sequenceDiagram
    autonumber
    participant ORD as Order Service (Orchestrator)
    participant INV as Inventory Service
    participant PAY as Payment Service
    participant NTF as Notification Service
    participant RMQ as RabbitMQ

    Note over ORD,INV: Step 1: Stock was successfully reserved earlier
    INV->>RMQ: Event: inventory.reserved
    RMQ->>ORD: Consume: inventory.reserved

    Note over ORD,PAY: Step 2: Payment fails
    ORD->>RMQ: Command: payment.process
    RMQ->>PAY: Consume: payment.process
    PAY->>PAY: Card declined / Insufficient funds
    PAY->>RMQ: Event: payment.failed (Reason: DECLINED)
    RMQ->>ORD: Consume: payment.failed

    Note over ORD,INV: Saga Compensation: Rollback Reserved Stock
    ORD->>ORD: Transition Order to CANCELLING
    ORD->>RMQ: Command: inventory.release_stock (OrderId, Items)
    RMQ->>INV: Consume: inventory.release_stock
    INV->>INV: Release reserved stock quantities
    INV->>RMQ: Event: inventory.released
    RMQ->>ORD: Consume: inventory.released

    Note over ORD,NTF: Finalize Cancellation
    ORD->>ORD: Transition Order to CANCELLED
    ORD->>RMQ: Event: order.cancelled
    RMQ->>NTF: Consume: order.cancelled -> Send alert to user
```

---

### 4.3 Key Reliability Patterns

1. **Transactional Outbox Pattern**:
   * *Problem*: Dual-writing (saving to PostgreSQL and then publishing to RabbitMQ) risks data loss or phantom messages if a network error occurs between the database commit and the broker call.
   * *Solution*: The business record (`Order`) and the event (`outbox_messages`) are written in the **same local database transaction**. A background worker polls the outbox table and publishes events to RabbitMQ with persistent delivery.
2. **Idempotency**:
   * Commands must be safely retryable. Clients pass an `Idempotency-Key` header.
   * Payment Service stores this key with a unique constraint in `payments_db` and locks it in Redis during processing. Duplicate requests return the original result without re-executing charges.
3. **Eventual Consistency**:
   * Checkout returns `202 Accepted` immediately. The frontend polls or listens to server-sent events (SSE) while the Saga completes asynchronously in the background.

---

## 5. Security, Observability, and Storage Invariants

1. **Edge Authentication**: The API Gateway validates incoming JWT signatures, checks the Redis revocation denylist, and forwards verified identity claims via trusted internal headers (`X-User-Id`, `X-User-Roles`). Internal services do not repeat cryptographic JWT verification ([ADR-004](decisions/ADR-004-authentication.md)).
2. **Network Containment**: Internal microservices, databases, and message queues have no public internet routing. Only the API Gateway is exposed.
3. **Simulated Payment Security**: The platform strictly uses synthetic tokens (`tok_simulated_*`). Real credit card numbers and live payment credentials are never handled or stored.
4. **Binary Object Storage**: Media files (product images) are stored in S3-compatible storage (MinIO for local dev; AWS S3 for production). Databases store only object URLs and metadata.
5. **Request Correlation & Tracing**: Every request carries an `X-Correlation-Id` propagated across REST headers, gRPC metadata, RabbitMQ message headers, and structured JSON logs ([ADR-005](decisions/ADR-005-observability.md)).
6. **Health Checking**: Every service exposes `@nestjs/terminus` endpoints:
   * `/health/liveness`: Checks if the Node.js event loop is responsive.
   * `/health/readiness`: Probes database, RabbitMQ, and Redis connectivity before routing traffic.
