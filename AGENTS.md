# ShopFlow — AI Development Instructions

## 1. Project Goal

Build ShopFlow, a production-minded e-commerce microservices platform that demonstrates real-world architecture, secure APIs, distributed transactions, asynchronous messaging, testing, and deployment.

The developer is learning system design and microservices. Explain important decisions clearly without unnecessary theory.

---

## 2. Core Rules

- Inspect existing code and repository structure before making changes.
- Implement one small, logical step at a time.
- Do not scaffold the entire platform in one operation.
- Reuse existing code and dependencies where appropriate.
- Do not introduce unnecessary abstractions, packages, services, or documentation.
- Never overwrite or delete existing work without explicit permission.
- Never commit, push, reset, or change Git history without permission.
- Never expose secrets, tokens, credentials, or sensitive environment values.
- Ask before making destructive changes or decisions that materially alter the architecture.

---

## 3. Documentation

Keep documentation concise, readable, and useful.

- `README.md`: project introduction, compact tech stack, current status, and links.
- `docs/PROJECT-ROADMAP.md`: phases, progress, verification criteria, and next action.
- `docs/ARCHITECTURE.md`: service responsibilities, data ownership, and communication flows.
- `docs/DEVELOPMENT.md`: setup, commands, testing, and troubleshooting.
- `docs/DECISIONS.md`: index of important architectural decisions.
- `AGENTS.md`: persistent instructions for AI coding assistants.

Avoid duplicating detailed technical explanations across documents.

Keep detailed ADRs when they provide meaningful reasoning. Do not create documentation merely to increase file count.

---

## 4. Implementation Workflow

For every task:

1. Inspect relevant files and current implementation.
2. Explain the objective and proposed changes briefly.
3. Identify affected services, dependencies, and risks.
4. Implement only the agreed scope.
5. Run applicable tests, linting, type checks, or builds.
6. Verify service connections and relevant failure scenarios.
7. Update documentation and the roadmap when justified.
8. Report changed files, commands executed, actual results, and remaining issues.

Do not claim success when a check has not been executed or its result is unknown.

---

## 5. Architecture Principles

- Keep service responsibilities and ownership explicit.
- Each microservice owns its data; other services must not directly modify its database.
- Use REST for public APIs.
- Use synchronous internal communication only when a response is required.
- Use gRPC where its contract and performance benefits justify the complexity.
- Use RabbitMQ for asynchronous events and commands where appropriate.
- Design consumers to handle duplicate delivery safely.
- Use timeouts, bounded retries, dead-letter handling, and idempotency where appropriate.
- Use Saga orchestration and the Outbox Pattern when required by distributed transaction workflows.
- Avoid unnecessary synchronous dependencies between services.
- Document important trade-offs instead of treating every production pattern as mandatory.

---

## 6. Security and Reliability

- Validate external input and enforce server-side authorization.
- Never store secrets in source control.
- Use safe configuration defaults and least-privilege database access.
- Protect payment integrations and verify webhook signatures.
- Avoid logging credentials, tokens, payment details, or unnecessary personal data.
- Apply appropriate timeouts, rate limits, and error handling.
- Include health checks, structured logs, metrics, and tracing as the platform evolves.
- Test critical failure paths, not just successful requests.
- Do not claim production readiness without appropriate security, reliability, and recovery verification.

---

## 7. Testing and Git

- Follow the repository's existing package manager and scripts.
- Prefer focused tests for changed functionality.
- Run relevant tests before marking a task complete.
- Inspect `git status` and the diff before suggesting a commit.
- Never discard unrelated user changes.
- Do not stage, commit, or push unless explicitly authorized.

---

## 8. Progress Tracking

Classify features as:

- **PLANNED** — documented but not implemented.
- **IMPLEMENTED** — code or configuration exists.
- **VERIFIED** — relevant checks confirm the expected behavior.
- **BLOCKED** — progress depends on an unresolved issue.

Update progress only when supported by repository evidence.

---

## 9. Response Style

Keep explanations concise and practical.

For each completed task, report:

- **What changed**
- **Why it matters**
- **How it was verified**
- **What remains**
- **Next recommended step**

Explain unfamiliar architectural concepts when they affect a decision. Avoid long lectures, repeated summaries, and unnecessary generated files.

---

## 10. Current Working Rule

Always inspect the actual repository and current roadmap before selecting the next task.

Complete the smallest useful step, verify it, and wait for the developer's direction before proceeding to a materially different phase.
