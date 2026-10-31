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

---

## My understanding (simple English)

Concurrency means **more than one task is in progress during the same period of time**. The tasks do not always need to run at the exact same instant. The system may switch between them, or different CPU cores may run them in parallel.

Think about a payment application. Alice starts sending money to Bob, while Charlie starts sending money to David. Both transfers can be processed around the same time. That is normal concurrency.

Concurrency itself is not a problem. The problem starts when two operations use or change the same data and the result becomes incorrect because of their timing.

### Example

Alice has ₹10,000. Two payment requests try to debit her account:

- T1 wants to debit ₹4,000.
- T2 wants to debit ₹3,000.

Both requests may run at the same time. Our system must make sure that the final balance is correct and that Alice does not go below her minimum balance.

### Easy way to remember

- **Concurrency:** multiple tasks are in progress during overlapping time.
- **Parallelism:** multiple tasks are actually executing at the same instant.
- Concurrency is expected in a payment system. We need to design it safely.
