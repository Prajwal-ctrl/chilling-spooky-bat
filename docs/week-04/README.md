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

## Concept notes

- [Concurrency fundamentals](concurrency.md)
- [Race conditions](race-conditions.md)
- [Lost updates](lost-updates.md)
- [PostgreSQL MVCC](mvcc.md)
- [Transaction isolation](transaction-isolation.md)
- [Pessimistic locking](pessimistic-locking.md)
- [Optimistic locking](optimistic-locking.md)
- [Deadlocks and lock ordering](deadlocks.md)
- [Spring transactions](spring-transactions.md)
- [Concurrency testing](concurrency-testing.md)

## Status

- [ ] Conceptual learning completed
- [ ] Concept review questions passed
- [ ] Account schema implemented
- [ ] Naive lost-update reproduced
- [ ] Pessimistic locking implemented
- [ ] Concurrency and invariant tests passing
- [ ] Experiments and decisions documented

> Keep the scope deliberate. Do not move on just because the calendar says the week is over. Move on when the concepts can be explained, predicted, and applied.
