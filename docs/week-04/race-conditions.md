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
