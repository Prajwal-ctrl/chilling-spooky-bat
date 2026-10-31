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
