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
