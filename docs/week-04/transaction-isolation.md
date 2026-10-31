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
