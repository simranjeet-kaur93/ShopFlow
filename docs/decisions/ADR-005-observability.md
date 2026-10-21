# ADR-005: Observability Strategy: Structured Logging, Correlation IDs, and Distributed Tracing

## Status
**Accepted**

---

## Context
In a distributed microservices architecture like ShopFlow, a single user transaction (such as placing an order) spans multiple applications, synchronous gRPC hops, and asynchronous RabbitMQ event workflows. Debugging issues, measuring end-to-end latency, and diagnosing distributed failures requires a unified observability framework across all 8 applications.

Key challenges include:
1. Tracing an asynchronous transaction from the API Gateway through Order, Inventory, Payment, Shipping, and Notification services.
2. Ensuring logs can be parsed and aggregated by standard log shipping tools (e.g., Grafana Loki, AWS CloudWatch, Datadog).
3. Differentiating between process health (liveness) and dependency availability (readiness) for container orchestrators.
4. Avoiding vendor lock-in with proprietary monitoring agents.

---

## Decision
We adopt a unified observability architecture based on open standards:

### 1. Request Correlation via `X-Correlation-Id`
* **Perimeter Generation**: The API Gateway inspects incoming HTTP requests for an `X-Correlation-Id` header. If absent or invalid, a new UUIDv4 is generated.
* **Propagation Across Boundaries**:
  * **HTTP**: Returned in response headers and forwarded in downstream HTTP requests.
  * **gRPC**: Injected into gRPC request `Metadata` (`x-correlation-id`).
  * **RabbitMQ**: Attached to message metadata headers (`headers['x-correlation-id']`).
* **Saga Context**: When the Order Service processes an asynchronous Saga step, the correlation ID from the initial checkout request is preserved in all published commands and events.

### 2. Structured JSON Logging
* All microservices use a centralized logging configuration (based on Pino/Winston via `@shopflow/logger`).
* All logs are emitted to `stdout` in standard JSON format containing mandatory metadata fields:
  * `timestamp`: ISO-8601 UTC string.
  * `level`: Log level (`DEBUG`, `INFO`, `WARN`, `ERROR`).
  * `service`: Application identifier (`api-gateway`, `order-service`, etc.).
  * `correlationId`: Distributed correlation identifier (or `null` if background job).
  * `context`: Class, controller, or handler name.
  * `message`: Human-readable log narrative.
  * `error`: Serialized error stack trace (only for `WARN`/`ERROR`).

### 3. Health Checks: Liveness vs. Readiness (`@nestjs/terminus`)
Each service implements two distinct health check endpoints under `/health`:
* **Liveness (`/health/liveness`)**:
  * Verifies only that the Node.js event loop is responsive and memory usage is within bounds.
  * Does not check external dependencies. Failure signals that the process is deadlocked and requires a container restart.
* **Readiness (`/health/readiness`)**:
  * Probes active connections to the service's dedicated PostgreSQL database (via Prisma `$queryRaw` ping), connection to RabbitMQ, and Redis ping.
  * Failure indicates temporary unavailability, causing the ingress router/orchestrator to cease sending traffic without killing the process.

### 4. Distributed Tracing Standards (OpenTelemetry)
* The architecture prepares for OpenTelemetry (OTel) instrumentation by standardizing on W3C Trace Context headers (`traceparent`, `tracestate`).
* In Stage 6, an OpenTelemetry NodeSDK wrapper will export spans to an OpenTelemetry Collector, feeding Prometheus (metrics) and Jaeger/Tempo (traces).

---

## Assumptions
* **Stdout Stream Standard**: Container environments (Docker, Kubernetes, AWS ECS) handle log aggregation by collecting container `stdout`/`stderr` streams. Services should not write directly to local log files in production.
* **Low Logging Overhead**: High-performance JSON loggers (Pino) have negligible throughput impact in Node.js compared to database I/O and network serialization.
* **Open Source Tooling**: Observability tools must be open-source and local-friendly (Prometheus, Grafana, Loki, Jaeger) before migrating to cloud-managed services.

---

## Consequences

### Positive
* **Full Distributed Traceability**: Any order, payment, or stock mutation can be traced end-to-end across multiple services using a single correlation ID.
* **No Vendor Lock-In**: Adherence to OpenTelemetry standards allows seamless switching between Grafana Tempo, Jaeger, AWS X-Ray, or Datadog without code refactoring.
* **Reliable Zero-Downtime Deployments**: Clear separation of Liveness and Readiness probes enables Kubernetes or Docker Compose to perform safe rolling restarts without dropped traffic.

### Negative / Trade-offs
* **Discipline Required**: Every asynchronous message producer, gRPC interceptor, and background task must explicitly pass the correlation context. Omitting this breaks the trace chain.
* **Storage Footprint**: Structured JSON logs generate higher storage volume than raw strings, requiring log retention and rotation policies.

### Mitigations
* Reusable NestJS interceptors (`LoggingInterceptor`, `CorrelationInterceptor`) and RabbitMQ consumer decorators in `libs/` will automate context extraction and injection so individual service handlers do not write manual boilerplate.

---

## Alternatives Considered
* **Ad-hoc Console Logs**: Rejected because unstructured strings are unsearchable across 8 microservices and cannot be correlated in centralized log dashboards.
* **Proprietary APM Agents**: Rejected to maintain portability and avoid cloud vendor lock-in during foundational and local development stages.
