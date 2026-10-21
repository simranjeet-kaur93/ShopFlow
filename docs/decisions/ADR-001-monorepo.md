# ADR-001: Monorepo Architecture with NestJS and npm Workspaces

## Status
**Accepted**

---

## Context
ShopFlow is designed as a distributed e-commerce system comprising an API Gateway, seven backend microservices, and a React frontend. The development team needs to manage code sharing (such as Protocol Buffer definitions, shared DTOs, logging utilities, and RabbitMQ client abstractions) while maintaining independent deployability and service boundary isolation.

Two primary code organization strategies were evaluated:
1. **Polyrepo**: A dedicated Git repository for each microservice, the frontend, and shared libraries published via an internal npm registry.
2. **Monorepo**: A single unified Git repository containing all services and shared libraries.

---

## Decision
We will use a **Monorepo** structure powered by the **NestJS Monorepo CLI (`nest-cli.json`)** combined with **npm workspaces**.

The repository is structured as follows:
* `apps/`: Contains independently runnable applications (`api-gateway`, `user-service`, `product-service`, `order-service`, `payment-service`, `inventory-service`, `shipping-service`, `notification-service`, `frontend`).
* `libs/`: Contains shared, domain-agnostic cross-cutting libraries (`@shopflow/proto`, `@shopflow/common-dto`, `@shopflow/rabbitmq`, `@shopflow/logger`).

---

## Assumptions
* **Single Language Ecosystem**: All backend applications and shared contracts use TypeScript on Node.js, making a single language-native monorepo workspace highly effective without polyglot build orchestration overhead.
* **Shared Contract Ownership**: Contract updates (e.g., Protobuf definitions or event schemas) are managed collaboratively in the same codebase rather than published through an external private artifact registry.
* **Independent Scalability via Containers**: Applications within the monorepo compile to standalone distribution artifacts (`dist/apps/<service-name>`) and will be packaged into independent Docker containers without bundling other applications.

---

## Trade-offs and Consequences

### Positive
* **Atomic Changes**: Contract changes (such as updating a Protobuf definition or adding an event payload schema) can be committed alongside both the producer and consumer in a single pull request.
* **Streamlined Tooling**: Eliminates the operational overhead of versioning, publishing, and distributing internal npm packages across multiple repositories.
* **Unified Developer Experience**: A developer can clone one repository, install dependencies once, and run any combination of services locally.
* **Code Reusability**: Shared infrastructure code (NestJS interceptors, structured loggers, error filters) is easily shared without code duplication.

### Negative / Trade-offs
* **Risk of Boundary Erosion**: Developers might inadvertently import internal business logic or entities directly from one service into another (e.g., `apps/order-service` importing from `apps/payment-service`).
* **Repo Size Growth**: Monorepos accumulate Git history faster than polyrepos.
* **CI Build Complexity**: CI pipelines must distinguish which services changed to avoid rebuilding all containers on every commit.

### Mitigations
* Strict ESLint `no-restricted-imports` rules will prevent any application in `apps/*` from importing directly from sibling applications in `apps/*`. Imports are only permitted from declared shared libraries in `libs/*`.
* Each application in `apps/` retains its own independent build entrypoint (`main.ts`) and independent Dockerfile to ensure standalone containerization and deployment.

---

## Alternatives Considered
* **Polyrepo**: Rejected due to high overhead in local orchestration, version drift across internal packages, and slow iterative velocity during early educational phases.
* **Nx / Turborepo**: Evaluated for advanced dependency caching, but rejected for Step 1 to keep tooling simple and aligned with standard NestJS CLI capabilities, avoiding extra configuration layers until scale requires them.
