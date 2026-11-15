# Phantom Reads — PostgreSQL READ COMMITTED vs REPEATABLE READ

## Why this topic matters

A transaction does not only read individual rows. It can also run a query that returns a set of rows matching a condition.

For example, a payment service might query all active accounts, all pending payments, or all transfers above a threshold. We need to understand whether that set can change between two executions of the same query inside one transaction.

This is the concept of a **phantom read**.

## 1. What is a phantom read?

A phantom read happens when a transaction executes the same query twice and gets a different set of rows because another transaction committed an insert, delete, or update that changes which rows match the query condition.

Example query:

\`\`\`sql
SELECT account_id, name
FROM account
WHERE status = 'ACTIVE';
\`\`\`

The first execution returns Alice and Bob. Another transaction inserts David as an ACTIVE account and commits. If the first transaction's second execution now returns Alice, Bob, and David, David is a new matching row—a phantom row.

### Simple-English explanation

**A phantom read means I run the same search again and a new row appears in the result, or a previous row disappears, because another transaction changed the data that matches my search condition.**

## 2. Phantom read under PostgreSQL READ COMMITTED

Initial data:

| Account ID | Name | Status |
|---|---|---|
| 101 | Alice | ACTIVE |
| 102 | Bob | ACTIVE |
| 103 | Charlie | INACTIVE |

T2 starts under READ COMMITTED and runs:

\`\`\`sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

SELECT COUNT(*)
FROM account
WHERE status = 'ACTIVE';
\`\`\`

Result: **2**

While T2 remains open, T1 inserts David and commits:

\`\`\`sql
INSERT INTO account(account_id, name, status)
VALUES (104, 'David', 'ACTIVE');

COMMIT;
\`\`\`

T2 runs the same SELECT again:

\`\`\`sql
SELECT COUNT(*)
FROM account
WHERE status = 'ACTIVE';
\`\`\`

Result: **3**

Why? READ COMMITTED gives each statement a fresh snapshot. T2's second SELECT can see David's committed row.

### Timeline

\`\`\`text
T2 (READ COMMITTED)                  T1
──────────────────────────────────────────────────
BEGIN

SELECT active accounts
→ Alice, Bob
→ COUNT = 2

                                      INSERT David (ACTIVE)
                                      COMMIT

SELECT active accounts
→ Alice, Bob, David
→ COUNT = 3

COMMIT
\`\`\`

This is a phantom read: the set of rows matching the same condition changed between T2's two queries.

## 3. What happens under PostgreSQL REPEATABLE READ?

Now run the same scenario with T2 using REPEATABLE READ.

\`\`\`sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*)
FROM account
WHERE status = 'ACTIVE';
\`\`\`

T2 gets **2**. This first non-transaction-control statement establishes T2's snapshot.

T1 inserts David as ACTIVE and commits.

T2 runs the same SELECT again. It still gets **2**, not 3.

### Timeline

\`\`\`text
T2 (REPEATABLE READ)                 T1
──────────────────────────────────────────────────
BEGIN

SELECT active accounts
→ Alice, Bob
→ COUNT = 2
(snapshot established)

                                      INSERT David (ACTIVE)
                                      COMMIT

SELECT active accounts
→ Alice, Bob
→ COUNT = 2
(same snapshot)

COMMIT
\`\`\`

David exists in the database after T1 commits, but David's row is not visible in T2's already-established snapshot.

### Important PostgreSQL-specific detail

The SQL standard's minimum definition of REPEATABLE READ permits phantom reads. PostgreSQL provides a stronger implementation: its REPEATABLE READ level prevents phantom reads by keeping a stable MVCC snapshot.

Do not assume every database implements REPEATABLE READ the same way.

### Simple-English explanation

**Under PostgreSQL REPEATABLE READ, T2 keeps using its original snapshot. Even if T1 inserts and commits David, T2's second SELECT does not include David. T2 still gets 2.**

## 4. Phantom read vs non-repeatable read

These two phenomena are related, but they describe different changes.

| Phenomenon | What changes? | Example |
|---|---|---|
| Non-repeatable read | The values of a row already read change | Alice's balance is ₹20,000, then another transaction commits ₹15,000 |
| Phantom read | The set of rows matching a query condition changes | A query returns two ACTIVE accounts, then a newly inserted ACTIVE account makes it return three |

Memory aid:

- **Non-repeatable read:** the row I read has changed.
- **Phantom read:** the collection of rows matching my query has changed.

## 5. What about changes made by my own transaction?

A stable snapshot does not hide a transaction's own writes from itself.

PostgreSQL REPEATABLE READ queries see the effects of earlier INSERT, UPDATE, or DELETE operations performed by the same transaction, even if those changes are not committed yet.

For example, if T2 itself inserts another ACTIVE account, a subsequent query in T2 can include the row that T2 inserted. The stable snapshot prevents visibility of concurrent transactions' later commits; it does not hide T2's own work from T2.

## 6. Does preventing phantom reads mean REPEATABLE READ is fully serializable?

No. This is a critical distinction.

PostgreSQL REPEATABLE READ prevents dirty reads, non-repeatable reads, and phantom reads. However, **serialization anomalies can still occur**.

That means two transactions can each see a stable snapshot, both complete successfully, and still produce a result that cannot be explained by running the transactions one at a time in some order.

A classic example is **write skew**. We will study it separately before deciding whether REPEATABLE READ is sufficient for a business rule or whether SERIALIZABLE isolation or explicit coordination is required.

## 7. My learning checkpoint

The question in our guided exercise was:

- T2 starts with REPEATABLE READ.
- T2 queries ACTIVE accounts and gets a count of 2.
- T1 inserts David as ACTIVE and commits.
- T2 runs the same query again.

The correct result for T2 is **2**, not 3.

The explanation is that T2 keeps using the original snapshot. It does not receive a new snapshot for its second SELECT.

Under READ COMMITTED, the second query would get a fresh snapshot and return 3.

### My original answer

> Okay so It sees david's newly committed account because T2 continues to see the new snapshot of the tell me if this is correct or not...

### Correction

I mixed up the two isolation levels. "New snapshot for every statement" is the READ COMMITTED rule. Under PostgreSQL REPEATABLE READ, later ordinary SELECT statements reuse the transaction's original snapshot.

### Recall in one sentence

**A phantom read changes the set of rows returned by the same query; PostgreSQL READ COMMITTED can show the new matching row on a later statement, while PostgreSQL REPEATABLE READ keeps the original snapshot and does not show that concurrent insert.**

## 8. Questions for independent revision

1. What is the difference between a phantom read and a non-repeatable read?
2. Under READ COMMITTED, why can David appear in T2's second SELECT?
3. Under PostgreSQL REPEATABLE READ, why does T2 still get a count of 2?
4. Does REPEATABLE READ hide rows inserted by the same transaction?
5. Does preventing phantom reads mean PostgreSQL REPEATABLE READ prevents every serialization anomaly?

## Official references

- [PostgreSQL 18 — Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html)
- [PostgreSQL 18 — Transaction Isolation, Repeatable Read subsection](https://www.postgresql.org/docs/18/transaction-iso.html#XACT-REPEATABLE-READ)
