# Race Conditions

## Definition

A race condition occurs when the correctness of a result depends on the timing or ordering of concurrent operations, and at least one possible execution order produces an incorrect result.

Concurrency by itself is not a race condition. It becomes a correctness problem when overlapping operations interact with shared state and a possible interleaving violates the expected business rules.

## What can cause a race condition?

A common example is an unsafe read-modify-write sequence:

1. A transaction reads a shared value.
2. The application calculates a replacement value.
3. The transaction writes that replacement value later.
4. Another transaction can read the same old value and calculate its own replacement before the first change is visible to it.

When the two transactions write their independently calculated absolute values, the later write can overwrite the earlier transaction's effect. This is a lost update, which is one specific type of race condition.

## Payment example

Alice has ₹10,000.

- T1 wants to withdraw ₹4,000.
- T2 wants to withdraw ₹3,000.
- Both transactions read ₹10,000 before either writes.
- T1 calculates ₹6,000.
- T2 calculates ₹7,000.
- If T1 writes ₹6,000 and T2 writes ₹7,000 afterward, the final balance is ₹7,000 instead of the expected ₹3,000.

See [Lost Updates](03-lost-updates.md) for the complete transaction timeline and SQL comparison.

## Important distinctions

- A race condition is **not defined only by the absence of pessimistic locking**. It is defined by a timing-dependent correctness failure.
- Two transactions accessing the same row do not automatically create a race condition. The SQL operation, isolation level, and coordination strategy matter.
- Pessimistic locking is one way to coordinate conflicting operations. Optimistic version checks and carefully designed atomic SQL updates are other possible approaches.
- PostgreSQL MVCC controls which row versions a transaction can see. It does not automatically enforce every application-level business invariant.
- An ordinary PostgreSQL `SELECT` and a locking read such as `SELECT ... FOR UPDATE` have different behavior.

## Questions to answer

1. What shared state is involved?
2. Which operations form the unsafe read-modify-write sequence?
3. Which interleaving causes the invariant to fail?
4. Why is the absence of pessimistic locking not the complete definition of a race condition?
5. What database or application strategies can prevent the unsafe outcome?

## Experiments

During implementation week, reproduce a race with two Java threads and explicit synchronization. Capture the initial state, each transaction's read and calculated value, the write order, and the final database state. Do not rely on arbitrary sleeps to create the interleaving.

---

## My understanding (simple English)

A race condition happens when the final result depends on which operation runs first, and some possible execution order can produce an incorrect result.

Multiple requests running together is not automatically a race condition. It becomes a problem when their timing can break our expected business rules.

### Example

Alice has ₹10,000.

- T1 wants to debit ₹4,000.
- T2 wants to debit ₹3,000.

Both requests can read the same ₹10,000 balance before either one updates it. Each can calculate a different new balance from that same old value. Depending on the order of the writes, one update can overwrite the other. This is a lost update, which is one example of a race condition.

### Difference between the terms

- **Concurrency:** requests overlap in time.
- **Race condition:** timing or ordering can make the result incorrect.
- **Lost update:** a specific race condition where one update overwrites another.
- **Deadlock:** transactions wait for one another in a cycle.

### How can we prevent a race?

We need to coordinate the shared-state operation or make it safe against concurrent changes. Depending on the use case, options include pessimistic locking, optimistic version checks, or an atomic conditional SQL update.

### Easy way to remember

A race condition is about **the order in which things happen affecting correctness**. A lost update is one particular way that can go wrong.

## Learning checkpoint

- [x] Identify concurrency as different from a race condition.
- [x] Identify a lost update as one example of a race condition.
- [ ] Explain the definition and a failure interleaving independently.
- [ ] Compare the prevention strategies and their trade-offs.
