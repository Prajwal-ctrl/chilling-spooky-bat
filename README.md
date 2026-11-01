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

## Day-by-day study schedule

The weekly topics above define **what** we need to learn. This section defines **when** to cover each part. Each week is divided into seven study days. Days 1–6 move through the week's material; Day 7 is reserved for recall, review, a small exercise, and catching up. Treat this as a sequence, not a deadline: repeat a day when you cannot yet explain or apply its concepts.

**Suggested daily allocation (about 2.5 hours):** 60 minutes for the core concept or implementation, 45 minutes for examples and hands-on work, 30 minutes for notes/diagrams, and 15 minutes of closed-book recall. On Day 7, use more time for review and less for new material.

**How the cadence works:** Weeks 4, 6, 8, 10, 12, 14, 16, 18, 20, and 22 are primarily concept weeks. Weeks 5, 7, 9, 11, 13, 15, 17, 19, 21, and 23 apply the preceding ideas. Weeks 1–3 are foundation weeks and remain marked as reconstructed outlines. Week 24 brings the complete system together.
### Week 1 — Project foundations and financial requirements

| Day | Study plan |
|---|---|
| **Day 1** | Understand the purpose of a payment system, the actors involved, and the business problem the project is solving. |
| **Day 2** | Define the core domain vocabulary: customer, account, payment, transfer, source account, and destination account. |
| **Day 3** | Trace a basic account-to-account transfer from client request through validation, processing, and final outcome. |
| **Day 4** | Separate functional requirements from non-functional requirements; define the first-version API and system scope. |
| **Day 5** | Study reliability, correctness, security, availability, latency, and scalability as payment-system non-functional requirements. |
| **Day 6** | Write business invariants and acceptance criteria, including positive amounts, distinct accounts, and no partial transfer. |
| **Day 7** | Draw and review the first transfer sequence diagram; consolidate requirements and list unresolved questions. |

### Week 2 — Java domain modeling and money representation

| Day | Study plan |
|---|---|
| **Day 1** | Review classes, objects, encapsulation, composition, and the role of domain behavior in an object-oriented design. |
| **Day 2** | Distinguish entities from value objects; compare identity, equality, lifecycle, and immutability. |
| **Day 3** | Model the Account entity: identity, ownership, balance, minimum balance, currency, and permitted operations. |
| **Day 4** | Model Money using BigDecimal; study decimal precision, scale, rounding, and why binary floating-point is unsuitable for money. |
| **Day 5** | Define amount and currency validation, including positive amounts, supported currency, and consistent scale rules. |
| **Day 6** | Model Payment and Transfer; decide which rules belong inside domain objects and which belong in application services. |
| **Day 7** | Review the domain model, write focused unit tests for its invariants, and explain the design without referring to framework code. |

### Week 3 — Spring Boot, REST, PostgreSQL, and persistence foundations

| Day | Study plan |
|---|---|
| **Day 1** | Study Spring Boot application startup, project structure, configuration, and the responsibilities of the application layers. |
| **Day 2** | Understand dependency injection and constructor injection; trace a request through controller, service, and repository. |
| **Day 3** | Design REST request/response DTOs, validation, status codes, and a consistent exception-handling approach. |
| **Day 4** | Review relational database fundamentals: tables, primary keys, foreign keys, unique constraints, and check constraints. |
| **Day 5** | Study JPA/Hibernate entities, persistence context, dirty checking, and how repository operations reach SQL. |
| **Day 6** | Understand Flyway migrations and ACID transaction fundamentals; distinguish application transaction boundaries from individual SQL statements. |
| **Day 7** | Sketch the first API-to-database flow and initial schema; review what each layer owns before starting the concurrency feature. |

### Week 4 — Concurrency concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study concurrency versus parallelism, threads, overlapping operations, and why concurrent requests are normal in a backend service. |
| **Day 2** | Study race conditions: identify shared state, timing-dependent outcomes, and interleavings using a simple account example. |
| **Day 3** | Work through the lost-update timeline with two debits; calculate the expected balance and show how a stale read can overwrite another update. |
| **Day 4** | Study Spring transactions and ACID behavior: transaction boundaries, commit, rollback, and why atomicity alone does not prevent every concurrent-write bug. |
| **Day 5** | Study PostgreSQL MVCC and row-version visibility; compare a normal SELECT with a locking read while another transaction is writing. |
| **Day 6** | Compare READ COMMITTED and REPEATABLE READ, then study row-level locks, SELECT FOR UPDATE, lock waiting, lock lifetime, and lock timeout. |
| **Day 7** | Compare pessimistic and optimistic locking; study deadlocks and deterministic lock ordering, then review concurrency-test fundamentals and predict outcomes for sample transactions. |

### Week 5 — Implement and prove concurrency control — PAY-006

| Day | Study plan |
|---|---|
| **Day 1** | Design the account schema and transfer transaction boundary; create the Flyway migration and development seed data. |
| **Day 2** | Implement the simplest transfer operation first, without explicit row locking; keep the code small enough to reason about. |
| **Day 3** | Create a deterministic Java concurrency test using threads and synchronization barriers so both transactions can read before either writes. |
| **Day 4** | Run the test, reproduce the lost update, capture the observed balances and interleaving, and write the experiment's root cause. |
| **Day 5** | Implement pessimistic account-row locking with SELECT FOR UPDATE; verify that conflicting transfers wait and then use the current balance. |
| **Day 6** | For transfers touching two accounts, acquire locks in deterministic account-ID order; test minimum balance, atomic debit/credit, and rollback. |
| **Day 7** | Test concurrent transfers and unrelated-account independence; inspect pg_stat_activity and pg_locks where useful, then document the evidence and PAY-006 result. |

### Week 6 — Idempotency concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study at-most-once, at-least-once, and effectively-once processing; understand why distributed systems cannot rely on a client sending only once. |
| **Day 2** | Trace HTTP retry scenarios: client timeout, lost response, server success, and a repeated payment request. |
| **Day 3** | Define an idempotency key and request fingerprint; decide which parts of a request must match for it to count as the same operation. |
| **Day 4** | Study the idempotency record lifecycle: initial request, in-progress state, completed response, failure handling, and retention. |
| **Day 5** | Analyze two simultaneous requests with the same key; understand unique constraints and atomic duplicate detection. |
| **Day 6** | Distinguish request deduplication from business uniqueness; define behavior for same key/same payload and same key/different payload. |
| **Day 7** | Write the payment API's idempotency contract and test scenarios on paper; review crash and retry timelines before implementation. |

### Week 7 — Implement idempotent payment creation

| Day | Study plan |
|---|---|
| **Day 1** | Design the idempotency table and API contract; define key scope, request fingerprint, status, and reference to the resulting payment. |
| **Day 2** | Add schema constraints and decide how the idempotency record and payment creation share one atomic transaction. |
| **Day 3** | Implement first-request processing: claim the key, validate the request, create the payment, and persist the result. |
| **Day 4** | Implement same-key/same-payload behavior so a valid retry returns the original result rather than creating another payment. |
| **Day 5** | Implement same-key/different-payload handling and return a clear conflict response. |
| **Day 6** | Test simultaneous requests with the same key, client timeout/retry, and failure during payment creation. |
| **Day 7** | Review concurrency and rollback behavior; document retention decisions, tests, and the final idempotency contract. |

### Week 8 — Ledger and double-entry accounting concepts

| Day | Study plan |
|---|---|
| **Day 1** | Understand the difference between a current account balance and a complete, auditable transaction history. |
| **Day 2** | Study journal entries and double-entry accounting; understand that each posted transaction has balanced debit and credit effects. |
| **Day 3** | Work through a transfer example and define debit/credit from the system's chosen accounting perspective. |
| **Day 4** | Study ledger-entry immutability and why a posted financial entry should be corrected with a reversal rather than silently edited. |
| **Day 5** | Review monetary precision, currency handling, and the rules required to balance entries for a transfer. |
| **Day 6** | Study ledger consistency, balance derivation or projection, and reconciliation between account balances and ledger activity. |
| **Day 7** | Draw the ledger model and write its invariants; explain how one transfer is represented and how correctness can be verified. |

### Week 9 — Implement ledger-backed transfers

| Day | Study plan |
|---|---|
| **Day 1** | Design ledger account and ledger entry tables; define keys, references, amount representation, and constraints. |
| **Day 2** | Implement posting of balanced debit and credit entries for a transfer. |
| **Day 3** | Make account updates, payment state changes, and ledger posting atomic within the chosen transaction boundary. |
| **Day 4** | Protect posted entries from mutation through application design and database constraints where appropriate. |
| **Day 5** | Implement reversal entries for a cancelled or corrected operation; do not delete or rewrite the original posted history. |
| **Day 6** | Add tests for balanced entries, account/ledger consistency, and rollback when any step fails. |
| **Day 7** | Review the ledger invariants and document the transfer posting flow, test evidence, and any balance-reconciliation approach. |

### Week 10 — Payment lifecycle and state-machine concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study state machines and why a payment's lifecycle should be represented by explicit states rather than scattered Boolean flags. |
| **Day 2** | Define the payment states used by the project, including pending, processing, succeeded, failed, and reversed. |
| **Day 3** | Map allowed transitions and identify invalid transitions; reason through normal, failed, and retry paths. |
| **Day 4** | Distinguish terminal states from retryable or intermediate states; decide which transitions must be blocked. |
| **Day 5** | Separate the internal payment status from an external provider's status and understand why they may not change at the same time. |
| **Day 6** | Study transition validation and transition history; identify what needs to be auditable. |
| **Day 7** | Draw the payment state-transition diagram and test yourself with valid/invalid transition scenarios. |

### Week 11 — Implement payment lifecycle management

| Day | Study plan |
|---|---|
| **Day 1** | Translate the state diagram into code-level states and an explicit allowed-transition table. |
| **Day 2** | Implement a central transition operation that validates the current state and requested next state. |
| **Day 3** | Persist payment transition history with enough context to explain who or what caused a change. |
| **Day 4** | Handle duplicate transition requests and invalid attempts without corrupting the payment state. |
| **Day 5** | Protect terminal-state rules and test concurrent attempts to change the same payment. |
| **Day 6** | Connect payment-state changes with ledger-posting rules so financial posting happens only at the intended lifecycle point. |
| **Day 7** | Write tests for valid, invalid, duplicate, and concurrent transitions; review the state machine against the documented requirements. |

### Week 12 — External provider integration and resilience concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study the provider abstraction and adapter pattern; identify what should be independent of a specific payment provider. |
| **Day 2** | Define provider request/response mapping and how internal payment identifiers relate to provider references. |
| **Day 3** | Analyze network timeouts and uncertain outcomes, especially when the provider may have processed a request but the response was lost. |
| **Day 4** | Study retry policies, bounded retries, and exponential backoff; identify which failures are safe to retry. |
| **Day 5** | Compare retryable and non-retryable errors; understand circuit-breaker and bulkhead concepts and the problem each addresses. |
| **Day 6** | Study provider-side idempotency support and reconciliation for unknown outcomes; explain why a retry must not cause a second charge. |
| **Day 7** | Create a provider failure matrix covering success, rejection, timeout, duplicate response, and unknown outcome; review the proposed integration. |

### Week 13 — Implement resilient provider integration

| Day | Study plan |
|---|---|
| **Day 1** | Define the provider interface and implement a fake provider that can return controlled success and failure outcomes. |
| **Day 2** | Implement request/response mapping and persist provider references and relevant outcome details. |
| **Day 3** | Add timeout handling and map provider errors into internal retryable or non-retryable categories. |
| **Day 4** | Implement bounded retry with backoff only for failures that are safe to retry. |
| **Day 5** | Add circuit-breaker behavior where justified and propagate idempotency keys to the provider when supported. |
| **Day 6** | Test timeout after provider success, duplicate callbacks/responses, unknown outcomes, and retry exhaustion. |
| **Day 7** | Review the failure matrix against the implementation; document how operations recover when the provider outcome is uncertain. |

### Week 14 — Messaging and transactional outbox concepts

| Day | Study plan |
|---|---|
| **Day 1** | Compare synchronous and asynchronous communication; identify why a payment system may publish business events. |
| **Day 2** | Study Kafka fundamentals: topics, partitions, producers, consumers, and the role of a message broker. |
| **Day 3** | Understand message keys and partition ordering; decide which payment events need ordering and within what boundary. |
| **Day 4** | Study at-most-once and at-least-once delivery, duplicate messages, and idempotent consumer behavior. |
| **Day 5** | Analyze the dual-write problem: why a database commit and Kafka publish are not automatically one atomic operation. |
| **Day 6** | Study the transactional outbox pattern, consumer offsets, and retry considerations. |
| **Day 7** | Define a payment event contract and draw the database-to-outbox-to-publisher-to-consumer flow. |

### Week 15 — Implement outbox and event publishing

| Day | Study plan |
|---|---|
| **Day 1** | Design and create the outbox table; define event identity, payload, creation time, and publication status. |
| **Day 2** | Write the payment/ledger change and outbox record in the same database transaction. |
| **Day 3** | Implement a publisher that reads pending outbox records and publishes them to the appropriate Kafka topic. |
| **Day 4** | Implement safe publication tracking and reason about the crash window between publishing and marking a record as published. |
| **Day 5** | Handle publisher restarts and possible duplicate publication; make consumers idempotent. |
| **Day 6** | Test failure after database commit but before publication, publisher restart, and consumer retry behavior. |
| **Day 7** | Review delivery semantics and document the outbox flow, failure windows, and test evidence. |

### Week 16 — Distributed workflows and consistency concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study the distributed transaction problem and why one local database transaction cannot atomically cover multiple independent services. |
| **Day 2** | Understand eventual consistency and define which service owns each piece of payment/workflow data. |
| **Day 3** | Study the Saga pattern and compare orchestration with choreography at a conceptual level. |
| **Day 4** | Understand compensating actions; distinguish compensation from a true rollback of an already committed remote action. |
| **Day 5** | Model workflow state, step outcomes, timeouts, and recovery when a process stops halfway through. |
| **Day 6** | Analyze duplicate, delayed, and out-of-order events, poison messages, and dead-letter queues. |
| **Day 7** | Draw a multi-step payment workflow with success, timeout, compensation, and manual-review paths. |

### Week 17 — Implement workflow coordination and reconciliation

| Day | Study plan |
|---|---|
| **Day 1** | Model workflow state, individual steps, outcomes, and the data needed to resume after a restart. |
| **Day 2** | Implement a small, clearly scoped orchestrated workflow or Saga coordinator. |
| **Day 3** | Implement compensating actions for the steps that can be safely compensated. |
| **Day 4** | Handle duplicate, delayed, and out-of-order events using explicit state checks and idempotent processing. |
| **Day 5** | Implement a reconciliation job for pending or unknown payment/provider outcomes. |
| **Day 6** | Add retry limits, timeout handling, and manual-review states for cases that cannot be resolved automatically. |
| **Day 7** | Test process restarts and partial workflow failures; document recovery behavior and reconciliation evidence. |

### Week 18 — Security and audit concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study authentication versus authorization and the role of Spring Security in protecting backend APIs. |
| **Day 2** | Compare role-based permissions with resource-level authorization; understand why a valid user must not automatically access every account. |
| **Day 3** | Study account-ownership checks, input validation, and common API threat scenarios relevant to payment operations. |
| **Day 4** | Review secure configuration and secret management; identify credentials and sensitive values that must not be committed. |
| **Day 5** | Study sensitive-data minimization and safe logging; decide which payment/customer fields should be masked or excluded. |
| **Day 6** | Distinguish an audit trail from application logs; define which sensitive actions and state changes need durable records. |
| **Day 7** | Create a threat model for account and payment operations; record security requirements and abuse cases. |

### Week 19 — Implement security and audit controls

| Day | Study plan |
|---|---|
| **Day 1** | Add the project's chosen authentication mechanism and define the authenticated principal available to the application. |
| **Day 2** | Enforce authorization for payment and account endpoints. |
| **Day 3** | Validate account ownership for every account-scoped operation; test cross-account access attempts. |
| **Day 4** | Review validation and error handling so security failures do not expose unnecessary internal details. |
| **Day 5** | Move secrets into environment/configuration mechanisms and verify that no credentials are committed or logged. |
| **Day 6** | Add audit records for sensitive actions and payment-state changes; sanitize application logs. |
| **Day 7** | Test unauthenticated, unauthorized, forbidden, and cross-account scenarios; document the security controls and remaining risks. |

### Week 20 — Observability and production diagnostics concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study structured application logs and why consistent fields make production investigation easier. |
| **Day 2** | Understand correlation IDs and trace context; follow one payment request across application boundaries. |
| **Day 3** | Compare metrics, logs, and distributed traces; identify the question each signal is best suited to answer. |
| **Day 4** | Study RED/USE-style service and resource signals, latency percentiles, throughput, and error rates. |
| **Day 5** | Understand SLIs, SLOs, and alerting; define what a healthy payment service should mean. |
| **Day 6** | Study database connection-pool metrics and lock-wait visibility; connect symptoms to likely bottlenecks. |
| **Day 7** | Create a payment-service observability checklist and walk through a sample failure investigation. |

### Week 21 — Implement observability

| Day | Study plan |
|---|---|
| **Day 1** | Add structured logs with consistent fields and correlation IDs for incoming requests. |
| **Day 2** | Instrument HTTP endpoints and important business operations such as payment creation and transfer execution. |
| **Day 3** | Expose application and database metrics, including request counts, error rates, latency, and connection-pool usage. |
| **Day 4** | Add distributed tracing across service boundaries where available and verify trace context propagation. |
| **Day 5** | Define payment-specific metrics such as success/failure counts, processing latency, and pending/unknown outcomes. |
| **Day 6** | Build useful dashboards and actionable alerts; include connection-pool pressure and lock-wait signals where available. |
| **Day 7** | Run a controlled failure scenario and verify that logs, metrics, and traces let you follow the request and identify the failure. |

### Week 22 — Database and application performance concepts

| Day | Study plan |
|---|---|
| **Day 1** | Study EXPLAIN and EXPLAIN ANALYZE, including BUFFERS; learn to read scans, joins, estimates, and actual execution time. |
| **Day 2** | Study index design, selectivity, and the trade-off between faster reads and write/storage overhead. |
| **Day 3** | Understand the N+1 query problem and compare JPA fetch strategies; identify how to detect excess database round trips. |
| **Day 4** | Study HikariCP connection pooling, pool sizing, queueing, and the difference between query time and waiting for a connection or lock. |
| **Day 5** | Review pagination and bounded queries; understand why unbounded data retrieval is dangerous as data grows. |
| **Day 6** | Study caching trade-offs, cache invalidation, and partitioning considerations; use them only when justified by the workload. |
| **Day 7** | Connect throughput, latency, and saturation; write a performance investigation plan with measurable hypotheses. |

### Week 23 — Performance testing and optimization

| Day | Study plan |
|---|---|
| **Day 1** | Define realistic workloads, data volumes, concurrency levels, and measurable performance targets. |
| **Day 2** | Create k6 or Gatling scenarios for the main payment and account operations; capture a baseline. |
| **Day 3** | Measure throughput, latency percentiles, and error rates; identify the slowest or most constrained operations. |
| **Day 4** | Inspect expensive query plans and optimize SQL or indexes based on evidence rather than guesswork. |
| **Day 5** | Examine connection-pool sizing, lock contention, and transaction duration under load. |
| **Day 6** | Evaluate caching only where measurements support it; rerun the workload after each meaningful change. |
| **Day 7** | Compare before/after measurements and write a performance report explaining the bottleneck, change, and evidence. |

### Week 24 — Production readiness and final architecture review

| Day | Study plan |
|---|---|
| **Day 1** | Package the service with Docker and make the local development environment reproducible; review configuration and secret handling. |
| **Day 2** | Create or finalize CI checks for compilation, tests, and required quality gates. |
| **Day 3** | Review Flyway migration order, deployment sequencing, backward compatibility, and rollback considerations. |
| **Day 4** | Add or verify health checks, graceful shutdown, and behavior during application termination. |
| **Day 5** | Review failure modes, recovery procedures, operational runbooks, and known limitations. |
| **Day 6** | Complete end-to-end and resilience test review; update architecture diagrams and important Architecture Decision Records. |
| **Day 7** | Prepare a final system-design walkthrough: explain the architecture, invariants, trade-offs, failure handling, measurements, and what you would improve next. |

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
        ├── 01-concurrency-fundamentals.md
        ├── 02-race-conditions.md
        ├── 03-lost-updates.md
        ├── 04-spring-transactions.md
        ├── 05-postgresql-mvcc.md
        ├── 06-transaction-isolation.md
        ├── 07-pessimistic-locking.md
        ├── 08-deadlocks-and-lock-ordering.md
        ├── 09-optimistic-locking.md
        ├── 10-concurrency-testing.md
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
