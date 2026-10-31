# PostgreSQL MVCC

## What is MVCC?

MVCC means Multi-Version Concurrency Control. PostgreSQL uses row versions and transaction visibility rules so a statement can read a version appropriate to its snapshot while concurrent transactions are changing data.

## Core ideas

- An uncommitted change is not visible to an ordinary reader in another transaction.
- Under READ COMMITTED, each SQL statement gets a new snapshot.
- A normal SELECT can generally read the previously committed row version while another transaction holds a row lock.
- After the writer commits, a subsequent statement under READ COMMITTED can see the newly committed version.
- A locking read such as SELECT ... FOR UPDATE, or a conflicting UPDATE, must wait for a conflicting row lock.

## Example

1. T1 begins and locks Alice's account with SELECT ... FOR UPDATE.
2. T1 updates balance from ₹10,000 to ₹6,000 but has not committed.
3. T2 performs an ordinary SELECT and normally reads the previously committed ₹10,000 version without waiting for the row lock.
4. T1 commits.
5. T2 performs a new SELECT under READ COMMITTED and sees ₹6,000.

## Important boundary

MVCC controls which row versions are visible to a transaction. It does not by itself prevent every concurrent-write anomaly or enforce application business rules.

## Questions to answer

1. Why can an ordinary SELECT proceed while a writer holds a row lock?
2. What does T2 see before T1 commits?
3. What does a new T2 statement see after T1 commits under READ COMMITTED?
4. How does REPEATABLE READ differ?
5. Why is MVCC not a substitute for a deliberate write-coordination strategy?
