# Race Conditions

## Definition

A race condition occurs when the correctness of a result depends on the timing or ordering of concurrent operations, and at least one possible interleaving produces an incorrect result.

## Payment example

Two transfers may both read Alice's balance before either writes its debit. If each computes a new absolute balance from the same old value, one write can overwrite the other.

## Important distinction

Concurrency itself is not a race condition. A race condition exists when a timing-dependent interleaving can violate expected behavior.

## Questions to answer

1. What shared state is involved?
2. Which operations form the unsafe read-modify-write sequence?
3. Which interleaving causes the invariant to fail?
4. What synchronization or database mechanism prevents that interleaving?

## Experiments

Reproduce a race in the naive transfer implementation during the implementation week.

---

## My understanding (simple English)

A race condition happens when the final result depends on which operation runs first, and some possible execution order can produce an incorrect result.

Multiple requests running together is not automatically a race condition. It becomes a problem when their timing can break our expected business rules.

### Example

Alice has ₹10,000.

- T1 wants to debit ₹4,000.
- T2 wants to debit ₹3,000.

Both requests read the same ₹10,000 balance before either one updates it. Depending on how their reads and writes happen, one update might overwrite the other. This is a lost update, which is one example of a race condition.

### Difference between the terms

- **Concurrency:** requests overlap in time.
- **Race condition:** timing or ordering can make the result incorrect.
- **Lost update:** a specific race condition where one update overwrites another.
- **Deadlock:** transactions wait for one another in a cycle.

### Easy way to remember

A race condition is about **the order in which things happen affecting correctness**. A lost update is one particular way that can go wrong.
