# Concurrency Fundamentals

## What is concurrency?

Concurrency is the ability of a system to make progress on multiple operations during overlapping periods. Operations may be interleaved on one CPU core or execute in parallel on multiple cores.

Concurrency is not automatically a bug. It is a normal operating condition in a payment service: many users can initiate transfers at roughly the same time.

## Why it matters in a payment system

Two requests can read or modify the same account concurrently. The implementation must preserve business invariants despite different possible execution orders.

## Key distinctions

- **Concurrency:** operations overlap in time.
- **Parallelism:** operations execute at the same instant on different execution resources.
- **Race condition:** correctness depends on the relative timing or ordering of operations.
- **Lost update:** one particular race in which one update overwrites another update's effect.
- **Deadlock:** transactions form a cycle of waiting and cannot proceed without intervention.

## Questions to answer

1. Can a system be concurrent without being parallel?
2. Why is concurrency normal in a banking system?
3. When does overlapping execution become a correctness problem?
4. How do race conditions differ from deadlocks?

## Payment-system connection

PAY-006 requires safe processing of simultaneous transfers while maintaining balance and minimum-balance invariants.

## Experiments

To be added during the implementation week.
