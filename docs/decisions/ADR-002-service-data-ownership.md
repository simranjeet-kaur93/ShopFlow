# ADR-002: Service Data Ownership and Database-Per-Service Pattern

## Status
**Accepted**

---

## Context
In a microservices architecture, data persistence can follow two distinct models:
1. **Shared Database**: All microservices connect to a single central database instance and share tables, views, or schemas.
2. **Database-Per-Service**: Each microservice owns its private persistent datastore, and other services may only access data through public APIs or domain events.

ShopFlow requires high availability, autonomous service deployment, clear business domain boundaries, and zero risk of hidden schema coupling.

---

## Decision
We adopt the **Database-Per-Service pattern**. Each microservice exclusively owns its database and schema:
* **User Service** -> `shopflow_user`
* **Product Service** -> `shopflow_product`
* **Order Service** -> `shopflow_order`
* **Payment Service** -> `shopflow_payment`
* **Inventory Service** -> `shopflow_inventory`
* **Shipping Service** -> `shopflow_shipping`
* **Notification Service** -> `shopflow_notification`

### Implementation Rules
1. **Technology**: **PostgreSQL** is the standard relational database engine, managed via **Prisma ORM** with dedicated `schema.prisma` files located inside each service's directory.
2. **Access Isolation**: Microservices connect only to their own database via credentials configured through environment variables.
3. **No Direct SQL Sharing**: No service may execute direct SQL queries, views, or joins against another service's tables.
4. **Data Aggregation**: When a service requires foreign data (e.g., Order Service needing Product details at checkout), it queries the owner service via gRPC or subscribes to domain events to maintain a denormalized local projection.

---

## Assumptions
* **Relational ACID Suitability**: Standard e-commerce transaction semantics (inventory decrementing, order status tracking, financial idempotency) require relational ACID guarantees rather than pure document/NoSQL stores.
* **Low Local Hardware Overhead**: Running a single PostgreSQL container with multiple logical databases locally provides identical isolation semantics to multiple physical instances without excessive RAM usage.
* **Eventual Consistency Acceptability**: Business stakeholders accept that cross-service consistency (e.g., product title updates appearing in customer order history) is eventual rather than instantaneous.

---

## Trade-offs and Consequences

### Positive
* **Decoupled Schema Evolution**: A service team can refactor its database schema, run migrations, or alter column types without risking downtime or breaking changes in other services.
* **Failure Isolation**: An outage, performance degradation, or lock contention in the Product database does not lock up the User or Payment databases.
* **Independent Scalability**: High-write databases (e.g., Inventory, Orders) can be partitioned, migrated, or provisioned with specialized read-replicas independently of low-write databases.
* **Security Segmentation**: Database credentials are partitioned; compromising one service database does not expose customer credentials in `users_db` or financial transactions in `payments_db`.

### Negative / Trade-offs
* **No Distributed Foreign Keys**: Relational constraints across services are impossible. Referential integrity must be managed at the application layer or through asynchronous compensation.
* **Complex Reporting and Joins**: Joining data across orders, products, and users requires API composition, event-driven read models, or an external analytics pipeline.
* **Distributed Transactions**: Atomic multi-table updates across services are replaced with the Saga pattern and eventual consistency.

### Mitigations
* Order items store snapshot data (e.g., `productSku`, `unitPrice`) at the time of purchase rather than relying on live joins with Product Service tables.
* The Transactional Outbox pattern guarantees that state changes and event notifications commit together atomically inside the service's database.

---

## Alternatives Considered
* **Shared Database with Separate Schemas**: Rejected because developers frequently bypass API boundaries by creating direct foreign keys or cross-schema views, leading to tight coupling and migration deadlocks.
* **NoSQL / Document DB for all services**: Evaluated (e.g., MongoDB), but rejected because relational transactions, ACID guarantees for financial transactions (Payments, Orders, Inventory), and structured schemas with Prisma offer superior safety for e-commerce workflows.
