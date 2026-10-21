# Service Boundaries and Data Ownership

## 1. Core Principle: Database-Per-Service

In ShopFlow, **each microservice strictly owns its domain logic and persistent data storage**. 
* Direct cross-database SQL queries, foreign-key relationships spanning different databases, and shared schema ownership are strictly prohibited.
* If a service requires information owned by another service, it must either:
  1. Query the owner service synchronously via its **gRPC contract**, or
  2. Maintain a local read projection populated by subscribing to **RabbitMQ domain events**.

```
+------------------+           +--------------------+
|   Order Service  |           |  Inventory Service |
+--------+---------+           +---------+----------+
         |                               |
         | (Prisma)                      | (Prisma)
         v                               v
+------------------+           +--------------------+
|  orders_db (SQL) |           | inventory_db (SQL) |
+------------------+           +--------------------+
         ^                               ^
         |                               |
         +------- NO CROSS-QUERIES ------+
```

---

## 2. Detailed Service Specifications

### 2.1 API Gateway
* **Type**: Ingress API / Reverse Proxy
* **Responsibility**: 
  * Acts as the single external HTTP/REST gateway for web clients and SPAs.
  * Handles rate limiting (token bucket via Redis), TLS termination, and CORS enforcement.
  * Centralizes JWT authentication verification and injects trusted internal headers (`X-User-Id`, `X-User-Roles`).
  * Generates and injects `X-Correlation-Id` for distributed tracing.
  * Translates external REST requests into internal gRPC calls.
  * Normalizes error responses according to RFC 7807 (Problem Details).
* **Persistent Storage**: None (Stateless).
* **Cache Usage**: Redis for rate-limiting counters and revoked JWT token denylists.
* **Inbound Interfaces**: Public HTTP/REST (`/api/v1/*`).
* **Outbound Interfaces**: Internal gRPC to User, Product, Order, and Shipping services.
* **Strict Boundary Rule**: Contains zero business calculation logic (e.g., tax calculation, discounts, payment validation).

---

### 2.2 User Service
* **Type**: Core Domain & Identity Service
* **Responsibility**:
  * Manages user registration, profiles, credentials, and password hashing using Argon2id.
  * Issues, validates, and rotates authentication tokens (JWT access tokens and refresh tokens).
  * Manages roles (`customer`, `admin`) and basic permissions.
* **Database**: `users_db` (PostgreSQL via Prisma)
* **Core Entities**:
  * `User`: `id`, `email`, `passwordHash`, `firstName`, `lastName`, `role`, `createdAt`, `updatedAt`
  * `RefreshToken`: `id`, `userId`, `tokenHash`, `expiresAt`, `revokedAt`
  * `UserProfile`: `id`, `userId`, `phoneNumber`, `defaultAddressId`
* **Security & Audit**:
  * Passwords hashed using Argon2id with unique cryptographic salts.
  * Refresh tokens stored as secure hashes, allowing instant single-session revocation.
  * Login attempts, password changes, and permission updates logged with timestamp and correlation ID.
* **Inbound Interfaces**:
  * gRPC: `GetUserById`, `ValidateToken`, `GetUserProfile`
  * REST (via Gateway): `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `GET /api/v1/users/me`
* **Outbound Events**:
  * `user.registered`: Published to RabbitMQ upon successful account registration.

---

### 2.3 Product Service
* **Type**: Core Domain Service
* **Responsibility**:
  * Manages product catalog, categories, pricing, descriptions, images, and search filtering.
  * Provides high-speed product catalog reading for the storefront UI.
* **Database**: `products_db` (PostgreSQL via Prisma)
* **Cache & Object Storage**:
  * **Redis**: Read-through cache for catalog listings, category trees, and high-traffic product detail pages.
  * **Binary File Storage**: Product images are stored in an S3-compatible object storage service (MinIO for local development; AWS S3 for production). `products_db` stores only asset URLs, dimensions, and metadata.
* **Core Entities**:
  * `Product`: `id`, `sku`, `name`, `slug`, `description`, `price`, `currency`, `categoryId`, `isActive`, `createdAt`, `updatedAt`
  * `Category`: `id`, `name`, `slug`, `parentId`, `createdAt`
  * `ProductImage`: `id`, `productId`, `storageKey`, `url`, `isPrimary`, `orderIndex`
* **Inbound Interfaces**:
  * gRPC: `GetProductById`, `GetProductsBatch`, `CheckProductPricing`
  * REST (via Gateway): `GET /api/v1/products`, `GET /api/v1/products/:id`, `GET /api/v1/categories`
* **Outbound Events**:
  * `product.created`, `product.updated`, `product.price_changed`

---

### 2.4 Order Service
* **Type**: Core Domain & Orchestration Service
* **Responsibility**:
  * Manages order lifecycles (Draft, Pending Payment, Confirmed, Shipped, Delivered, Cancelled).
  * Acts as the **Checkout Saga Orchestrator**, coordinating distributed transactions between Inventory, Payment, and Shipping services.
  * Records the Transactional Outbox table for atomic event publication.
* **Database**: `orders_db` (PostgreSQL via Prisma)
* **Core Entities**:
  * `Order`: `id`, `orderNumber`, `customerId`, `status`, `subtotal`, `taxTotal`, `shippingTotal`, `grandTotal`, `currency`, `createdAt`, `updatedAt`
  * `OrderItem`: `id`, `orderId`, `productId`, `productSku`, `unitPrice`, `quantity`, `subtotal`
  * `OrderSagaState`: `id`, `orderId`, `currentStep`, `sagaStatus`, `payload`, `failureReason`, `updatedAt`
  * `OutboxMessage`: `id`, `aggregateType`, `aggregateId`, `eventType`, `payload`, `status`, `createdAt`, `publishedAt`
* **Audit Trails**:
  * Append-only history of every order status transition (`PENDING_PAYMENT` -> `CONFIRMED` -> `SHIPPED`) recording timestamp, actor, and correlation ID.
* **Inbound Interfaces**:
  * gRPC: `GetOrderById`, `ListCustomerOrders`
  * REST (via Gateway): `POST /api/v1/orders/checkout`, `GET /api/v1/orders/:id`, `GET /api/v1/orders`
  * RabbitMQ Subscriptions: `payment.succeeded`, `payment.failed`, `inventory.reserved`, `inventory.insufficient`, `shipping.dispatched`
* **Outbound Commands & Events**:
  * Commands: `inventory.reserve_stock`, `inventory.release_stock`, `payment.process`, `shipping.create`
  * Events: `order.created`, `order.confirmed`, `order.cancelled`, `order.completed`

---

### 2.5 Payment Service
* **Type**: Core Domain Service
* **Responsibility**:
  * Handles payment authorization and capture in a fully simulated environment.
  * Enforces idempotency via `Idempotency-Key` tracking to prevent duplicate transactions.
  * Handles refund operations when triggered by Saga compensation workflows.
* **Payment Security & PCI Scope**:
  * **Strict Simulation Policy**: Never handles, stores, or transmits real credit card numbers or banking credentials.
  * Simulates PCI-compliant tokenized transactions using simulated payment tokens (`tok_simulated_*`).
  * Employs test payment methods with mock failure triggers (e.g., amount ending in `.99` triggers decline for testing).
* **Database**: `payments_db` (PostgreSQL via Prisma)
* **Cache Usage**: Redis for short-term distributed locks on idempotency keys during active processing.
* **Core Entities**:
  * `PaymentTransaction`: `id`, `orderId`, `customerId`, `amount`, `currency`, `status`, `paymentMethod`, `idempotencyKey`, `gatewayReference`, `createdAt`, `updatedAt`
  * `PaymentRefund`: `id`, `transactionId`, `amount`, `reason`, `status`, `createdAt`
* **Inbound Interfaces**:
  * RabbitMQ Subscriptions: `payment.process`, `payment.refund`
* **Outbound Events**:
  * `payment.succeeded`: Triggered upon successful authorization/capture.
  * `payment.failed`: Triggered upon card decline, timeout, or simulated error.
  * `payment.refunded`: Triggered upon successful compensation refund.

---

### 2.6 Inventory Service
* **Type**: Core Domain Service
* **Responsibility**:
  * Maintains stock counts and stock allocations per SKU.
  * Handles stock reservations during active checkout flows.
  * Releases reservations if checkout fails (compensation step) or commits reservations when the order is confirmed.
* **Database**: `inventory_db` (PostgreSQL via Prisma)
* **Core Entities**:
  * `InventoryItem`: `id`, `sku`, `productId`, `quantityOnHand`, `quantityReserved`, `reorderThreshold`, `updatedAt`
  * `StockReservation`: `id`, `orderId`, `sku`, `quantity`, `status` (`RESERVED`, `COMMITTED`, `RELEASED`), `expiresAt`, `createdAt`
* **Inbound Interfaces**:
  * gRPC: `GetStockLevels`
  * RabbitMQ Subscriptions: `inventory.reserve_stock`, `inventory.commit_stock`, `inventory.release_stock`
* **Outbound Events**:
  * `inventory.reserved`: Stock successfully held.
  * `inventory.insufficient`: Not enough stock available to satisfy the order.
  * `inventory.released`: Stock released back to available pool.

---

### 2.7 Shipping Service
* **Type**: Core Domain Service
* **Responsibility**:
  * Manages shipping options, destination address validation, and simulated parcel tracking.
  * Dispatches delivery orders once payment is confirmed.
* **Database**: `shipping_db` (PostgreSQL via Prisma)
* **Core Entities**:
  * `Shipment`: `id`, `orderId`, `carrier`, `trackingNumber`, `shippingAddress`, `status` (`PENDING`, `DISPATCHED`, `IN_TRANSIT`, `DELIVERED`), `dispatchedAt`, `createdAt`
* **Inbound Interfaces**:
  * gRPC: `GetShipmentTracking`
  * REST (via Gateway): `GET /api/v1/shipments/:trackingNumber`
  * RabbitMQ Subscriptions: `shipping.create`, `shipping.cancel`
* **Outbound Events**:
  * `shipping.created`, `shipping.dispatched`, `shipping.delivered`

---

### 2.8 Notification Service
* **Type**: Auxiliary Consumer Service
* **Responsibility**:
  * Subscribes to platform-wide domain events to trigger simulated transactional notifications (Welcome emails, Order confirmation emails, Payment failure alerts, Tracking notifications).
  * Implements rate limiting and duplicate prevention for outbound notifications.
* **Database**: `notifications_db` (PostgreSQL via Prisma)
* **Core Entities**:
  * `NotificationLog`: `id`, `recipientId`, `channel` (`EMAIL`, `SMS`), `templateId`, `payload`, `status` (`QUEUED`, `SENT`, `FAILED`), `createdAt`
* **Inbound Interfaces**:
  * RabbitMQ Subscriptions: `user.registered`, `order.confirmed`, `order.cancelled`, `payment.failed`, `shipping.dispatched`.
* **Outbound Events**: None (Leaf consumer).

---

## 3. Service Interface Summary Matrix

| Service | Primary Database | Fast Cache / Storage | Sync Protocol (In) | Async Events / Commands (In) | Async Events (Out) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **API Gateway** | None (Stateless) | Redis | HTTP/REST | None | None |
| **User Service** | `users_db` | Redis | gRPC | None | `user.registered` |
| **Product Service** | `products_db` | Redis + Object Storage | gRPC | None | `product.*` |
| **Order Service** | `orders_db` | None | gRPC / REST | `payment.*`, `inventory.*` | `order.*`, Saga Commands |
| **Payment Service** | `payments_db` | Redis | None (Async-First) | `payment.process`, `payment.refund` | `payment.succeeded`, `payment.failed` |
| **Inventory Service**| `inventory_db`| None | gRPC | `inventory.reserve_stock`, etc. | `inventory.reserved`, `inventory.insufficient` |
| **Shipping Service** | `shipping_db` | None | gRPC / REST | `shipping.create`, `shipping.cancel`| `shipping.dispatched`, `shipping.delivered` |
| **Notification Service**| `notifications_db`| None | None (Async-First) | Platform Domain Events | None |
