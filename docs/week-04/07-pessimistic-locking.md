# Pessimistic Locking

## Purpose

Pessimistic locking coordinates transactions by acquiring a lock before modifying a row that may be concurrently modified by another transaction.

## PostgreSQL example

```sql
BEGIN;

SELECT balance
FROM account
WHERE account_id = 'ALICE'
FOR UPDATE;

-- Validate and update the account.

COMMIT;
```

A conflicting locking read or write must wait until the lock holder commits or rolls back.

## Lock lifetime

A row lock acquired this way is held until the transaction ends. The practical way to limit lock duration is to keep the transaction short.

- lock_timeout limits how long a statement waits to acquire a lock.
- It does not limit how long an acquired lock is held.
- Avoid network calls or unnecessary work inside a transaction holding account locks.

## Spring Data JPA direction

A repository method can use `@Lock(LockModeType.PESSIMISTIC_WRITE)`. For transfers involving two accounts, acquire both locks in deterministic account-ID order rather than always locking source first.

## Questions to answer

1. Why does T2 wait when it also executes SELECT ... FOR UPDATE?
2. When is the lock released?
3. Why is deterministic multi-row lock ordering important?
4. What is the throughput cost of holding locks too long?

---

## My understanding (simple English)

Pessimistic locking means that before changing an account, a transaction first locks that account row. If another transaction also wants a conflicting lock on the same row, it must wait.

We use this when we want to coordinate changes to important shared data, such as an account balance.

### Example

Alice has ₹10,000.

1. T1 executes SELECT ... FOR UPDATE for Alice's account and gets the lock.
2. T1 checks the balance and debits ₹4,000.
3. T2 also tries SELECT ... FOR UPDATE for Alice's account.
4. T2 waits because T1 is still holding the lock.
5. T1 commits. The balance is now ₹6,000 and the lock is released.
6. T2 gets the lock, reads the current balance, and checks whether its own payment is allowed.

The lock is held until T1's transaction commits or rolls back.

### Things to remember

- The lock is held until the transaction ends.
- Keep the transaction short so other requests do not wait unnecessarily.
- lock_timeout limits how long a statement waits to acquire a lock. It does not decide how long an acquired lock is held.
- For a transfer involving two accounts, acquire both locks in a consistent order to reduce avoidable deadlocks.
- A lock on Alice's row does not mean we should lock every account in the database.

### Easy way to remember

**Pessimistic locking = lock the row first, then safely read and change it. If another transaction needs the same lock, it waits.**
