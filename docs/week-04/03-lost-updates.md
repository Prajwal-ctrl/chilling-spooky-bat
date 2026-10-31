# Lost Updates

## Definition

A lost update occurs when concurrent read-modify-write operations overwrite one another, causing one operation's effect to disappear.

## Example

Initial Alice balance: ₹10,000.

- T1 reads ₹10,000 and calculates ₹6,000 after a ₹4,000 debit.
- T2 reads ₹10,000 and calculates ₹7,000 after a ₹3,000 debit.
- T1 writes ₹6,000.
- T2 writes ₹7,000.

The final balance is ₹7,000, although both debits should result in ₹3,000.

This scenario assumes each transaction reads a balance and later writes a calculated absolute balance under an isolation/SQL pattern that allows the stale read-modify-write sequence. Actual behavior depends on the query shape and isolation level.

## Why this matters

The system has effectively lost one debit. A financial system must not permit this.

## Planned experiment

Use two Java threads and synchronization barriers so both transactions read the same initial balance before either performs its write. Capture the final database state and explain the exact interleaving.

## Questions to answer

1. Why is the final result ₹7,000?
2. Why doesn't wrapping each request in a transaction alone necessarily prevent this lost update?
3. How would row locking or an atomic conditional SQL update change the behavior?

---

## My understanding (simple English)

A lost update happens when two transactions read the same old value, calculate their own new values, and then one transaction overwrites the other's result.

In simple words, **one transaction's change gets lost**.

### Example

Alice has ₹10,000.

- T1 reads ₹10,000 and wants to debit ₹4,000. It calculates ₹6,000.
- T2 also reads ₹10,000 and wants to debit ₹3,000. It calculates ₹7,000.
- T1 writes ₹6,000.
- T2 writes ₹7,000 after T1.

The final balance becomes ₹7,000. But if both debits should have succeeded, the correct balance would be ₹3,000.

The second write overwrote the effect of the first debit.

This example assumes the application reads the balance and later writes a calculated absolute value in a way that allows both transactions to use the stale value. The exact behavior depends on the SQL and isolation level.

### How can we prevent it?

One option is to lock the account row before reading and changing the balance. Another option is to use a carefully designed atomic conditional update or optimistic version check.

### Easy way to remember

Both transactions started from the same old balance. They did not combine their changes, so one update disappeared.
