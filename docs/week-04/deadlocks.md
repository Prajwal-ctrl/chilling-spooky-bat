# Deadlocks and Lock Ordering

## Definition

A deadlock occurs when transactions wait on one another in a cycle, so none can proceed without the database aborting a transaction or some external intervention.

## Opposite-direction transfer example

- T1 transfers Alice → Bob and locks Alice first.
- T2 transfers Bob → Alice and locks Bob first.
- T1 then waits for Bob.
- T2 then waits for Alice.

The wait-for relationships form a cycle.

## Prevention strategy

Acquire all required account locks in a deterministic order, such as ascending account ID, regardless of transfer direction. This reduces avoidable deadlocks but does not guarantee that every deadlock in a larger system is impossible.

PostgreSQL detects deadlocks and aborts one participant.

## Questions to answer

1. What is a circular wait?
2. How can two opposite-direction transfers deadlock?
3. Why does deterministic ordering reduce the risk?
4. What should the application do after a deadlock victim transaction is aborted?
