# Distributed Payment System

A production-oriented backend engineering project for learning how financial transaction systems are designed, implemented, tested, and operated.

The project is intentionally developed incrementally. We will start with a correct account-to-account transfer and progressively introduce financial correctness, concurrency control, idempotency, ledger design, distributed workflows, reliability, observability, performance, and production deployment.

The objective is not simply to finish an application. It is to build the engineering judgment to explain **why** each design exists, reproduce the problems it solves, implement a solution, and prove the behavior with tests and experiments.

---

## Project goals

Build a financial transaction platform that demonstrates:

- Correct account and payment domain modeling.
- Safe money representation and balance invariants.
- Atomic account-to-account transfers.
- PostgreSQL transaction and concurrency control.
- Duplicate-request protection through idempotency.
- An auditable ledger and transaction history.
- Resilient integration with external payment providers.
- Reliable event publishing and distributed workflows.
- Security, observability, performance testing, and deployment.

## Core technology direction

The stack will be introduced only when the project needs it.

| Area | Planned technology |
|---|---|
| Language | Java 17+ |
| Backend | Spring Boot, Spring MVC, Spring Data JPA |
| Persistence | PostgreSQL, Hibernate |
| Schema migrations | Flyway |
| Testing | JUnit 5, Mockito, Spring Boot Test, Testcontainers |
| Messaging | Apache Kafka |
| Caching / coordination | Redis, when justified |
| Build and packaging | Maven, Docker |
| Performance testing | k6 or Gatling |
| Operations | Structured logs, metrics, distributed tracing |
| Deployment | CI/CD and container-based deployment |

Technology choices will be documented with trade-offs. We will not add infrastructure just to make the architecture look complicated.

---

## Learning methodology

We have adopted a **two-week cycle for each major topic**, starting with the current concurrency topic.

### Week A — Conceptual understanding

- Identify the real problem and why it matters.
- Understand the mechanism, not just the definition.
- Work through examples, timelines, and failure scenarios.
- Compare alternatives and trade-offs.
- Answer questions that require predicting system behavior.
- Record misunderstandings and corrected explanations.

### Week B — Practical engineering

- Design the solution before coding.
- Implement a simple or deliberately naive version.
- Reproduce the failure where possible.
- Implement the appropriate fix.
- Write automated tests and verify business invariants.
- Measure behavior where useful.
- Document experiments and architecture decisions.

We will not move forward merely because a calendar week has ended. We move forward when the concept can be explained, predicted, and applied.

For every important concept, documentation should include:
1. A technically accurate explanation.
2. **My understanding** in simple English.
3. A practical example.
4. Questions, misconceptions, and corrections.
5. Implementation experiments and evidence when available.

Your original explanations should be preserved separately from technical clarifications. A generated simple-English explanation must not be presented as a verbatim personal explanation.

---

# 24-week roadmap

> **Roadmap status:** Weeks 1–3 below are a reconstructed foundation outline based on the overall project direction; the exact historical topic-by-topic record for those weeks was not available when this README was prepared. From Week 4 onward, the roadmap follows our agreed concept-then-implementation cadence. Planned topics are not claims that implementation has already been completed.

## Foundation already covered / reconstructed

### Week 1 — Project foundations and financial requirements

**Focus:** Understand the business problem before designing the software.

Concepts:
- What a payment system does.
- Customer, account, payment, and transfer terminology.
- Transfer lifecycle and actors.
- Functional requirements versus non-functional requirements.
- Business invariants and acceptance criteria.
- Basic payment and transfer sequence diagrams.

Expected outcome:
- A clear scope for the first version of the system.
- Requirements and acceptance criteria written down.
- Initial understanding of what must never go wrong in a money transfer.

### Week 2 — Java domain modeling and money representation

**Focus:** Model the financial domain without mixing business rules into infrastructure code.

Concepts:
- Classes, objects, encapsulation, and composition.
- Entity versus value object.
- Account and payment domain models.
- Money representation using BigDecimal, not floating-point types.
- Currency and amount validation.
- Immutability and defensive design.
- Business invariants at the domain level.

Expected outcome:
- A domain model for accounts, money, and transfers.
- Clear rules for valid amounts and account operations.
- A documented distinction between identity-bearing entities and value objects.

### Week 3 — Spring Boot, REST, PostgreSQL, and persistence foundations

**Focus:** Understand how a backend request reaches the database and how data is represented.

Concepts:
- Spring Boot application structure.
- Controller, service, and repository responsibilities.
- Dependency injection and constructor injection.
- REST request/response models and DTOs.
- Input validation and exception handling.
- Relational tables, primary keys, foreign keys, and constraints.
- JPA/Hibernate persistence fundamentals.
- Flyway schema migrations.
- Database transactions and ACID fundamentals.

Expected outcome:
- A basic backend and database design that can support account and payment operations.
- A clear understanding of the request-to-database flow.
- Initial schema and API decisions documented.

---

## Concurrency and financial correctness

### Week 4 — Concurrency concepts

**Focus:** Understand how overlapping payment requests can corrupt account balances.

Concepts:
- Concurrency versus parallelism.
- Race conditions.
- Lost updates.
- PostgreSQL MVCC and row-version visibility.
- READ COMMITTED versus REPEATABLE READ.
- Pessimistic versus optimistic locking.
- Row-level locks and lock lifetime.
- Lock waiting and lock timeout.
- Deadlocks and deterministic lock ordering.
- Spring @Transactional behavior.
- Concurrency testing fundamentals.

Expected outcome:
- Be able to predict what competing transactions can read and modify.
- Explain why a normal SELECT behaves differently from SELECT ... FOR UPDATE.
- Explain how a lost update and a deadlock happen.

Study notes: [Week 4 documentation](docs/week-04/README.md).

### Week 5 — Implement and prove concurrency control

**Focus:** Complete PAY-006 — prevent concurrent payments from corrupting account balances.

Implementation:
- Create the account schema and seed development data.
- Implement a naive transfer operation.
- Reproduce a lost update using Java threads and synchronization barriers.
- Record the failing interleaving and root cause.
- Implement pessimistic row locking.
- Lock both transfer accounts in deterministic order.
- Keep transaction boundaries short.
- Test minimum-balance validation and atomic debit/credit.
- Test rollback, opposite-direction transfers, and unrelated account behavior.
- Inspect PostgreSQL lock activity using pg_stat_activity and pg_locks.

Expected outcome:
- A repeatable failure experiment.
- A correct locking implementation.
- Automated concurrency tests and a documented PAY-006 result.

---

## Idempotency and payment lifecycle

### Week 6 — Idempotency concepts

**Focus:** Understand what happens when a client retries a request or does not receive a response.

Concepts:
- At-most-once, at-least-once, and effectively-once processing semantics.
- Why HTTP retries can create duplicate business operations.
- Idempotent versus non-idempotent operations.
- Idempotency keys and request fingerprints.
- Duplicate request detection.
- Idempotency record lifecycle and retention.
- Concurrent requests using the same idempotency key.
- The difference between request deduplication and business uniqueness.

Expected outcome:
- A clear idempotency contract for payment creation.
- Defined behavior for first request, duplicate request, conflicting payload, and retry after timeout.

### Week 7 — Implement idempotent payment creation

**Focus:** Ensure that retrying a payment request does not create a second payment.

Implementation:
- Design an idempotency table and unique constraints.
- Store key, request fingerprint, status, and response reference as appropriate.
- Make idempotency handling atomic with payment creation.
- Handle simultaneous requests with the same key.
- Return the original result for a valid retry.
- Reject reuse of the same key with a different request payload.
- Test timeout/retry and concurrent duplicate requests.
- Document key expiry and retention decisions.

Expected outcome:
- A tested idempotent payment API.
- Experiment notes covering duplicate and concurrent requests.

### Week 8 — Ledger and double-entry accounting concepts

**Focus:** Understand why a balance alone is not a sufficient financial history.

Concepts:
- Account balance versus transaction history.
- Ledger entries and journal entries.
- Double-entry accounting fundamentals.
- Debit and credit from the system's defined perspective.
- Transfer representation as balanced entries.
- Immutable financial records.
- Reversals instead of silently editing posted entries.
- Ledger consistency and reconciliation.
- Monetary precision and currency handling.

Expected outcome:
- A ledger model and rules for a balanced transfer.
- Clarity on how account balances relate to ledger entries.

### Week 9 — Implement ledger-backed transfers

**Focus:** Record transfers as auditable financial entries.

Implementation:
- Design ledger account and ledger entry tables.
- Represent a transfer with balanced debit/credit entries.
- Make ledger posting atomic with payment state changes.
- Prevent mutation of posted entries through application design.
- Implement reversal entries for a cancelled or corrected operation.
- Derive or reconcile account balances against ledger activity.
- Add tests for balanced entries and rollback.
- Document ledger invariants.

Expected outcome:
- Transfers produce an auditable ledger trail.
- Tests prove that entries balance and partial posting cannot occur.

---

## Payment state and external integrations

### Week 10 — Payment lifecycle and state-machine concepts

**Focus:** Make payment status transitions explicit and valid.

Concepts:
- Payment lifecycle and state transitions.
- Pending, processing, succeeded, failed, and reversed states.
- Valid versus invalid state transitions.
- State machine modeling.
- Terminal states and retryable states.
- Separation of payment status from provider status.
- State transition validation and audit history.

Expected outcome:
- A documented payment state-transition model.
- Rules that prevent invalid transitions.

### Week 11 — Implement payment lifecycle management

**Focus:** Enforce payment transitions in code and persistence.

Implementation:
- Define payment status values and allowed transitions.
- Centralize transition validation.
- Persist transition history.
- Prevent terminal states from being changed incorrectly.
- Add tests for valid, invalid, duplicate, and concurrent transitions.
- Connect payment states with ledger posting rules.

Expected outcome:
- A tested payment lifecycle with an auditable history.

### Week 12 — External provider integration and resilience concepts

**Focus:** Understand failure when the payment system depends on another service.

Concepts:
- Provider abstraction and adapter pattern.
- Request/response mapping.
- Network timeout and uncertain outcomes.
- Retry policies and exponential backoff.
- Retryable versus non-retryable errors.
- Circuit breaker and bulkhead concepts.
- Provider idempotency support.
- Reconciliation after an unknown provider outcome.
- Why retries must not create duplicate charges.

Expected outcome:
- A provider integration design with explicit failure and retry behavior.

### Week 13 — Implement resilient provider integration

**Focus:** Integrate a simulated external payment provider safely.

Implementation:
- Define a provider interface and adapter.
- Implement a fake provider for deterministic tests.
- Add timeout and error mapping.
- Add bounded retry with backoff for eligible failures.
- Add circuit breaker behavior where justified.
- Propagate idempotency keys to the provider when supported.
- Persist provider references and outcomes.
- Test timeout after provider success, duplicate callback, and retry exhaustion.

Expected outcome:
- A testable provider integration that handles uncertain outcomes deliberately.

---

## Events and distributed workflows

### Week 14 — Messaging and transactional outbox concepts

**Focus:** Understand how to publish events reliably after a database transaction.

Concepts:
- Synchronous versus asynchronous communication.
- Event-driven architecture.
- Kafka topics, partitions, producers, and consumers.
- Message keys and ordering boundaries.
- At-most-once and at-least-once delivery.
- Duplicate message handling.
- Transactional outbox pattern.
- Why a database commit and Kafka publish are not automatically one atomic operation.
- Consumer offset and retry considerations.

Expected outcome:
- An event contract and outbox design for successful payment events.

### Week 15 — Implement outbox and event publishing

**Focus:** Publish payment events without losing the relationship between database state and emitted events.

Implementation:
- Create an outbox table.
- Write the payment/ledger update and outbox record in one database transaction.
- Build a publisher that reads pending outbox records.
- Publish events to Kafka.
- Mark records as published using a safe approach.
- Handle publisher restarts and duplicate publication.
- Make consumers idempotent.
- Test failure between database commit and event publication.

Expected outcome:
- A reliable, restartable event publishing flow with documented delivery semantics.

### Week 16 — Distributed workflows and consistency concepts

**Focus:** Understand multi-step business operations that cross service boundaries.

Concepts:
- Distributed transaction problem.
- Eventual consistency.
- Saga pattern and compensating actions.
- Workflow state and timeout handling.
- Reconciliation jobs.
- Duplicate, delayed, and out-of-order events.
- Poison messages and dead-letter queues.
- Consistency boundaries and ownership of data.

Expected outcome:
- A design for a multi-step payment workflow with failure and compensation paths.

### Week 17 — Implement workflow coordination and reconciliation

**Focus:** Recover safely when a payment workflow is interrupted.

Implementation:
- Model workflow state and step outcomes.
- Implement a small orchestrated workflow or clearly scoped saga.
- Add compensating actions where appropriate.
- Handle delayed and duplicate events.
- Implement a reconciliation job for pending/unknown outcomes.
- Add retry limits and manual-review states.
- Test process restarts and partial workflow failures.

Expected outcome:
- A recoverable payment workflow and reconciliation evidence.

---

## Security, audit, and operations

### Week 18 — Security and audit concepts

**Focus:** Protect payment APIs, account operations, and sensitive data.

Concepts:
- Authentication versus authorization.
- Spring Security fundamentals.
- Role- and resource-based access control.
- Customer/account ownership checks.
- Secure secret and configuration management.
- Input validation and common API threats.
- Sensitive-data minimization and safe logging.
- Audit trail versus application logs.
- Threat modeling for payment operations.

Expected outcome:
- A threat model and security requirements for the payment API.

### Week 19 — Implement security and audit controls

**Focus:** Enforce access rules and preserve an audit trail.

Implementation:
- Add authentication appropriate for the project.
- Enforce authorization for account and payment operations.
- Validate that a caller can access the requested account.
- Secure secrets through environment/configuration mechanisms.
- Add audit records for sensitive actions and state changes.
- Avoid logging credentials, tokens, and sensitive payment data.
- Test unauthorized, forbidden, and cross-account access.

Expected outcome:
- Tested API access controls and an auditable security-sensitive operation trail.

### Week 20 — Observability and production diagnostics concepts

**Focus:** Understand how to know whether the system is healthy and diagnose failures.

Concepts:
- Structured application logs.
- Correlation IDs and trace context.
- Metrics, logs, and distributed traces.
- RED/USE-style service and resource signals.
- Latency percentiles and error rates.
- SLIs, SLOs, and alerting concepts.
- Database connection pool and lock-wait visibility.
- Payment-specific operational signals.

Expected outcome:
- An observability plan with key metrics, logs, traces, and diagnostic scenarios.

### Week 21 — Implement observability

**Focus:** Make important system behavior measurable.

Implementation:
- Add structured logs and correlation IDs.
- Instrument HTTP and business operations.
- Expose application and database metrics.
- Add distributed tracing across service boundaries where available.
- Measure payment success/failure rates and latency.
- Observe connection pool usage and lock waits.
- Create useful dashboards and actionable alerts.
- Test that failures can be traced across a request.

Expected outcome:
- Operational visibility into payment flow health and common failure modes.

---

## Performance, deployment, and final engineering review

### Week 22 — Database and application performance concepts

**Focus:** Understand where latency and resource bottlenecks come from.

Concepts:
- Query plans and EXPLAIN (ANALYZE, BUFFERS).
- Index design and selectivity.
- N+1 queries and fetch strategies.
- Connection pooling and HikariCP.
- Lock contention versus query execution time.
- Pagination and bounded queries.
- Caching trade-offs and cache invalidation.
- Partitioning and data growth considerations.
- Throughput, latency, and saturation.

Expected outcome:
- A performance investigation plan and hypotheses tied to measurable bottlenecks.

### Week 23 — Performance testing and optimization

**Focus:** Measure the system under realistic load and improve the actual bottlenecks.

Implementation:
- Define workload and performance targets.
- Add k6 or Gatling scenarios.
- Capture baseline throughput and latency percentiles.
- Profile expensive queries and inspect execution plans.
- Optimize indexes and queries based on evidence.
- Examine connection pool sizing and lock contention.
- Evaluate caching only where justified.
- Re-run tests and compare before/after measurements.

Expected outcome:
- Reproducible performance results and documented optimizations.

### Week 24 — Production readiness and final architecture review

**Focus:** Bring the project together as a coherent engineering system.

Concepts and implementation:
- Docker packaging and local environment reproducibility.
- Configuration and secret handling.
- CI pipeline for build, test, and quality checks.
- Database migration strategy.
- Deployment and rollback considerations.
- Health checks and graceful shutdown.
- Failure-mode review and recovery procedures.
- Architecture diagrams and decision records.
- End-to-end and resilience test review.
- Final system-design walkthrough.

Expected outcome:
- A documented, runnable project with automated checks, deployment instructions, architecture decisions, known limitations, and a final engineering review.

---

## Project invariants

The implementation and tests should continuously protect the following rules:

1. A transfer amount must be positive.
2. Source and destination accounts must be different.
3. The source account must satisfy its minimum-balance rule after debit.
4. Debit, credit, and the associated payment/ledger records must commit atomically within their defined transaction boundary.
5. A failed transfer must not leave a partial debit or credit.
6. Concurrent operations must not corrupt balances or ledger consistency.
7. A retry with the same idempotency key must not create a second business payment.
8. Ledger entries for a transfer must balance under the chosen accounting model.
9. Payment state transitions must follow the defined lifecycle.
10. Events and distributed workflows must tolerate retries, duplicate delivery, and recoverable failures.

These invariants will be refined as requirements become more precise.

## Documentation structure

The docs/ directory is the long-term engineering knowledge base.

    docs/
    ├── requirements/
    ├── architecture/
    ├── decisions/
    ├── experiments/
    ├── weekly-reviews/
    └── week-04/
        ├── README.md
        ├── concurrency.md
        ├── race-conditions.md
        ├── lost-updates.md
        ├── mvcc.md
        ├── transaction-isolation.md
        ├── pessimistic-locking.md
        ├── optimistic-locking.md
        ├── deadlocks.md
        ├── spring-transactions.md
        ├── concurrency-testing.md
        └── my-understanding.md

As each week begins, its folder and topic notes will be added. Implementation weeks should also include experiment reports, test evidence, and Architecture Decision Records (ADRs) where appropriate.

## Current progress

- [x] Week 4 conceptual notes created.
- [x] Week 4 simple-English understanding sections added to the concept files.
- [x] Personal learning journal started.
- [ ] Week 4 conceptual review completed.
- [ ] Week 5 concurrency implementation completed.
- [ ] Remaining roadmap weeks completed incrementally.

**Current working focus:** Week 4 — understand concurrency deeply, then implement and verify PAY-006 in Week 5.
