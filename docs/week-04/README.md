# Week 4 — Concurrency, MVCC, Locking & Deadlocks

## Purpose

This week focuses on a critical financial-systems requirement: multiple concurrent payment requests must never corrupt account balances or violate account invariants.

We will use a two-week learning cycle for this major topic:

- **Concept week:** build deep conceptual understanding, reason through scenarios, and understand trade-offs.
- **Implementation week:** build the feature, reproduce failures, fix them, write automated tests, and document the results.

The goal is understanding and engineering judgment, not rushing through topics.

## Learning objectives

By the end of the conceptual week, be able to explain and reason about:

1. Concurrency and parallelism.
2. Race conditions and lost updates.
3. Database transactions and ACID properties.
4. PostgreSQL MVCC and row-version visibility.
5. Transaction isolation levels, especially READ COMMITTED and REPEATABLE READ.
6. Pessimistic and optimistic locking.
7. Row-level locks, lock acquisition, lock release, and lock waiting.
8. Deadlocks and deterministic lock ordering.
9. Spring's `@Transactional` boundary and rollback behavior.
10. Concurrency testing and financial invariants.

## Implementation week

We will then apply the concepts to **PAY-006 — Prevent concurrent payments from corrupting account balances**.

Planned sequence:

1. Create the account schema using Flyway and seed sample accounts.
2. Implement a deliberately naive transfer operation.
3. Reproduce a lost update deterministically using Java threads and synchronization barriers.
4. Record the failing interleaving and root cause.
5. Implement pessimistic row locking.
6. Acquire multiple account locks in a deterministic order to reduce avoidable deadlocks.
7. Test minimum-balance validation, atomic debit/credit, rollback, concurrent transfers, and unrelated-account independence.
8. Inspect PostgreSQL lock activity where useful.
9. Document the experiment, results, and design decisions.

## PAY-006 — Acceptance criteria

- Concurrent payments against the same account must not overwrite one another's balance changes.
- The minimum-balance invariant must remain true.
- Source debit, destination credit, and payment record must be atomic.
- Operations on unrelated accounts should not unnecessarily block each other.
- Transfers involving two accounts should avoid preventable deadlocks.
- Automated concurrency tests must verify correctness.

## Financial invariants

For every successful transfer:

- Amount must be positive.
- Source and destination accounts must be different.
- Source account must satisfy its minimum-balance requirement after debit.
- The debit and credit must both commit, or neither must commit.
- The total value transferred between the two accounts must be conserved, ignoring fees or other explicitly modeled operations.

## Study method

For each concept:

1. Define the problem it solves.
2. Explain the internal mechanism.
3. Work through timelines and competing transaction scenarios.
4. Compare alternatives and trade-offs.
5. Answer prediction questions without notes.
6. Record remaining questions and misconceptions.

During implementation:

1. Design before coding.
2. Build the simplest version.
3. Reproduce the failure.
4. Explain the root cause.
5. Implement the fix.
6. Prove the fix with tests and experiments.
7. Record an ADR or experiment report when a meaningful design decision or finding is made.

## Recommended study order

Read the concept files in numerical order. The order follows the problem-solving story: first understand why concurrent work exists, then see how it creates correctness bugs, understand what the database guarantees, and finally study the available ways to coordinate writes.

| Order | File | Why it comes here |
|---:|---|---|
| 1 | [Concurrency fundamentals](01-concurrency-fundamentals.md) | Start with overlapping operations and why concurrency is normal. |
| 2 | [Race conditions](02-race-conditions.md) | Learn when timing and execution order can make a result incorrect. |
| 3 | [Lost updates](03-lost-updates.md) | Study a concrete payment-balance bug caused by a race condition. |
| 4 | [Spring transactions](04-spring-transactions.md) | Understand the all-or-nothing boundary for debit, credit, and payment records—and why a transaction alone does not coordinate every concurrent update. |
| 5 | [PostgreSQL MVCC](05-postgresql-mvcc.md) | Understand how readers can see committed row versions while another transaction is writing. |
| 6 | [Transaction isolation](06-transaction-isolation.md) | Learn how snapshots and isolation levels affect what each transaction can observe. |
| 7 | [Pessimistic locking](07-pessimistic-locking.md) | Study the first explicit write-coordination solution: lock the account row before changing it. |
| 8 | [Deadlocks and lock ordering](08-deadlocks-and-lock-ordering.md) | Understand a new problem that can arise when transactions hold multiple locks, and how consistent ordering reduces avoidable deadlocks. |
| 9 | [Optimistic locking](09-optimistic-locking.md) | Compare a different strategy based on version checks. It is not automatically a better form of pessimistic locking; it can be preferable when conflicts are relatively uncommon. |
| 10 | [Concurrency testing](10-concurrency-testing.md) | Reproduce the bug, test the fix, and prove that financial invariants hold. |

**Mental model:** problem → concrete failure → transaction and database behavior → explicit locking solution → lock-management complication → alternative strategy → tests.

### Other learning notes

- [My understanding and learning journal](my-understanding.md) — original explanations, corrections, and personal revision notes.

## Status

- [ ] Conceptual learning completed
- [ ] Concept review questions passed
- [ ] Account schema implemented
- [ ] Naive lost-update reproduced
- [ ] Pessimistic locking implemented
- [ ] Concurrency and invariant tests passing
- [ ] Experiments and decisions documented

> Keep the scope deliberate. Do not move on just because the calendar says the week is over. Move on when the concepts can be explained, predicted, and applied.
