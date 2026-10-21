# ADR-003: Service Communication Architecture: Hybrid gRPC and RabbitMQ

## Status
**Accepted**

---

## Context
A microservices architecture requires robust mechanisms for inter-service communication. In an e-commerce platform like ShopFlow, two distinct communication profiles exist:
1. **Low-latency, synchronous queries**: Operations where an immediate response is required by the caller (e.g., retrieving product details, validating user session credentials, querying order tracking).
2. **Resilient, asynchronous workflows**: Multi-service business processes spanning several aggregates where immediate temporal coupling creates fragility and cascading failures (e.g., placing an order, charging payment, reserving stock, dispatching shipping, sending notifications).

---

## Decision
We adopt a **Hybrid Communication Architecture**:

### 1. Ingress Layer: HTTP/REST (JSON)
* The **API Gateway** exposes conventional HTTP/REST endpoints for client applications (React SPA, mobile clients, external integrations).
* Standard HTTP verbs, status codes, and RFC 7807 Problem Details error formats are used.

### 2. Internal Synchronous Communication: gRPC
* Used exclusively for direct point-to-point queries between the API Gateway and internal services (User, Product, Order, Shipping), or between microservices when real-time data validation is required.
* Backed by **Protocol Buffers (`.proto`)** shared via `@shopflow/proto` in the monorepo.
* Delivers high performance via HTTP/2 multiplexing, compact binary serialization, and strong compile-time type safety.

### 3. Internal Asynchronous Communication: RabbitMQ
* Used for all distributed business workflows, event notifications, and transactional commands.
* Implements the **Topic Exchange** model (`shopflow.events`) and **Direct Exchange** model (`shopflow.commands`).
* **Saga Orchestration**: The Order Service acts as the Saga Orchestrator, dispatching commands to Inventory, Payment, and Shipping services over RabbitMQ, while maintaining checkout transaction state and executing compensating actions if any step fails.
* **Dead Letter Exchanges (DLX)**: Poison-pill messages are routed to a dead-letter queue after three failed retries with exponential backoff.
* **Transactional Outbox**: Outgoing events are written to an outbox table in the emitting service's PostgreSQL database within the aggregate transaction, ensuring at-least-once delivery.

---

## Assumptions
* **Binary Serialization Compatibility**: Node.js microservices running `@grpc/grpc-js` provide sufficient throughput and low CPU overhead for internal synchronous queries.
* **Message Broker Sizing**: RabbitMQ message volumes during MVP and medium scale can be comfortably handled by a single broker instance with durable disk queues.
* **Network Partition Tolerance**: Network transient disconnects between services and RabbitMQ will occur in production; services must withstand broker downtime via the Outbox pattern.

---

## Trade-offs and Consequences

### Positive
* **Optimal Protocol Selection**: Low latency and schema validation where speed matters (gRPC); strong decoupling and resilience where reliability matters (RabbitMQ).
* **Fault Isolation**: If the Payment or Notification service is temporarily down, checkout requests can still be queued safely without dropping customer transactions.
* **Auditability & Traceability**: The Saga Orchestrator in Order Service provides a single, inspectable state machine to trace every step and compensation of any checkout transaction.

### Negative / Trade-offs
* **Dual Protocol Complexity**: Developers must manage both Protobuf schemas (`.proto`) and RabbitMQ queue bindings.
* **Eventual Consistency**: Asynchronous checkout means order confirmation is eventual rather than instantaneous, requiring optimistic UI updates or polling/SSE in the frontend.
* **Message Ordering and Duplicate Deliveries**: Distributed queues guarantee at-least-once delivery, requiring all consumers to implement idempotency handling.

### Mitigations
* Reusable `@shopflow/rabbitmq` and `@shopflow/proto` shared libraries in `libs/` encapsulate boilerplate connection management, serialization, and retry logic.
* Idempotency keys and database unique constraints prevent duplicate processing of re-delivered messages.

---

## Alternatives Considered
* **HTTP/REST for all internal calls**: Rejected due to high JSON serialization overhead, lack of strict contract enforcement, and brittle cascading failure chains during synchronous multi-service calls.
* **Apache Kafka**: Evaluated, but rejected for this architecture. While Kafka excels at massive log ingestion and event streaming, RabbitMQ provides superior task queuing, direct routing keys, flexible exchange topologies, lower memory overhead, and simpler local development setup for transactional e-commerce workflows.
* **Pure Choreography (No Orchestrator)**: Rejected because distributed choreography scatters checkout state across 4 different services, making compensation tracking, timeouts, and business rule auditing difficult to maintain.
