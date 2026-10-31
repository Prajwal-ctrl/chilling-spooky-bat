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

---

## My understanding (simple English)

A deadlock happens when two or more transactions are waiting for each other in a circle. None of them can continue unless the database breaks the situation, usually by aborting one transaction.

### Example: opposite-direction transfers

Suppose:

- T1 transfers money from Alice to Bob.
- T2 transfers money from Bob to Alice.

Imagine T1 locks Alice's account first. At the same time, T2 locks Bob's account first.

Now:

1. T1 needs Bob's account, but T2 is holding its lock.
2. T2 needs Alice's account, but T1 is holding its lock.
3. T1 waits for T2, and T2 waits for T1.

This is a deadlock.

### How can we reduce this problem?

We can always lock accounts in the same order, for example, by sorting account IDs before acquiring locks.

Then both T1 and T2 try to lock the lower account ID first, followed by the higher one. This removes this particular opposite-order locking pattern.

It reduces avoidable deadlocks, but it does not guarantee that deadlocks can never happen anywhere in the system. PostgreSQL can detect a deadlock and abort one transaction.

### Easy way to remember

**Deadlock = I am waiting for you, and you are waiting for me.**

Consistent lock ordering helps prevent this circular waiting.
