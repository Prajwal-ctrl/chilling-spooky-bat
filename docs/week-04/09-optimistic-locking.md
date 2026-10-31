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

---

## My understanding (simple English)

Optimistic locking assumes that conflicts are not very common. Instead of locking the row before doing all the work, we read the row and remember its version number. When we save the change, we check whether the version is still the same.

If someone else has already changed the row, our update does not happen. We detect the conflict and decide whether to retry.

### Example

Alice has ₹10,000 and the account version is 5.

- T1 reads balance ₹10,000, version 5.
- T2 also reads balance ₹10,000, version 5.
- T1 updates the balance to ₹6,000 and changes the version to 6.
- T2 tries to update using the old version 5.
- T2 updates zero rows because the current version is now 6.

T2 knows that the data it read is stale. It must read the latest balance again and check whether its own payment is still allowed.

### When is it useful?

It can work well when two requests rarely update the same account. If many requests keep changing the same account, repeated conflicts and retries may become expensive.

### Easy way to remember

**Optimistic locking lets requests try their update, but checks the version before accepting it.** If the version changed, the request must handle the conflict instead of overwriting newer data.
