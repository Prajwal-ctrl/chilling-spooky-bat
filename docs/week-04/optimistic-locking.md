# Optimistic Locking

## Idea

Optimistic locking allows work to proceed without holding a row lock for the entire read-modify-write interval. At update time, the system checks that the row has not changed since it was read.

## Version-column pattern

```sql
UPDATE account
SET balance = :new_balance,
    version = version + 1
WHERE account_id = :account_id
  AND version = :expected_version;
```

If zero rows are updated, the version did not match and the operation must be treated as a conflict.

## Trade-off

Optimistic locking can work well when conflicts are relatively uncommon. Under heavy contention on a small number of accounts, retries may become expensive. We will compare it conceptually with pessimistic locking; PAY-006 initially focuses on pessimistic locking.

## Questions to answer

1. What does the version column represent?
2. Why does an update affecting zero rows signal a conflict?
3. When might optimistic locking be preferable?
4. What retry policy is safe for a financial transfer?
