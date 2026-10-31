# Transaction Isolation

## Purpose

Transaction isolation defines how concurrent transactions observe one another's effects and which interleavings the database permits.

## Levels in scope

### READ COMMITTED

In PostgreSQL, each statement sees a snapshot of data committed before that statement began. A later statement in the same transaction can see commits that occurred since the earlier statement.

### REPEATABLE READ

A transaction uses a stable snapshot for its reads. PostgreSQL may reject a transaction attempting to modify data changed since its snapshot, requiring the application to retry where appropriate.

## Important

Do not assume that isolation level names behave identically across every database. Our explanations and experiments must be PostgreSQL-specific.

## Questions to answer

1. Why can two SELECT statements in one READ COMMITTED transaction return different values?
2. What does a stable snapshot mean?
3. What errors or retry requirements can arise under stronger isolation?
4. Which invariant must still be enforced by the payment application?

---

## My understanding (simple English)

Transaction isolation controls what one transaction can see while other transactions are reading or changing the same data. It defines the visibility rules for concurrent transactions.

In PostgreSQL, READ COMMITTED and REPEATABLE READ behave differently.

### Example: READ COMMITTED

1. T1 changes Alice's balance from ₹10,000 to ₹6,000 but has not committed.
2. T2 runs a normal SELECT and sees the old committed value, ₹10,000.
3. T1 commits.
4. T2 runs another SELECT. Because READ COMMITTED takes a new snapshot for each statement, the second SELECT can see ₹6,000.

So two SELECT statements inside the same READ COMMITTED transaction can return different values if another transaction commits between them.

### Example: REPEATABLE READ

Under REPEATABLE READ, T2 keeps a stable snapshot for its reads during the transaction. So its later ordinary reads generally continue to see the version visible to its original snapshot, rather than automatically switching to every newly committed version.

If T2 tries to update a row that another transaction changed after T2's snapshot, PostgreSQL can raise a serialization failure. The application may need to retry the whole transaction.

### Easy way to remember

- **READ COMMITTED:** each statement gets a fresh snapshot.
- **REPEATABLE READ:** the transaction keeps a stable snapshot for its reads.

These descriptions are PostgreSQL-specific. Other databases can implement isolation-level behavior differently.
