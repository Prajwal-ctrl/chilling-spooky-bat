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

---

## My understanding (simple English)

MVCC means **Multi-Version Concurrency Control**. PostgreSQL keeps information about different versions of a row so that a normal reader can see a committed version while another transaction is changing that row.

The important point I learned is that a normal SELECT and a SELECT FOR UPDATE behave differently.

### Example

Alice's committed balance is ₹10,000.

1. T1 starts and locks Alice's row using SELECT ... FOR UPDATE.
2. T1 changes the balance to ₹6,000, but has not committed yet.
3. T2 runs a normal SELECT. It normally does not wait for T1's row lock. It can read the previously committed balance: ₹10,000.
4. T1 commits the ₹6,000 balance.
5. T2 runs a **new SELECT statement**. Under READ COMMITTED, this new statement can now see ₹6,000.

T2 cannot see T1's uncommitted ₹6,000 change.

### Important difference

- Normal SELECT: reads a visible committed version and normally does not wait for this row lock.
- SELECT ... FOR UPDATE: asks for a row lock, so it normally waits for a conflicting lock.
- A conflicting UPDATE: normally waits for the other transaction's row lock.

### Easy way to remember

**MVCC helps readers see a safe committed version of data without having to wait for every writer.** Under READ COMMITTED, a later statement can see a change after it has been committed.

The correct name is MVCC — Multi-Version Concurrency Control.
