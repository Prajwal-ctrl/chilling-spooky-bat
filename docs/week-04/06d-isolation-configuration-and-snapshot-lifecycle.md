# Isolation Configuration and Snapshot Lifecycle in PostgreSQL

## Why this topic matters

We have learned what READ COMMITTED and REPEATABLE READ do. This note explains how PostgreSQL knows which level to use, how applications configure it, and exactly when a snapshot is created and stops being used.

The key distinction is:

- **MVCC** is PostgreSQL's row-version and visibility mechanism.
- **Transaction isolation** defines the visibility rules that MVCC applies to a transaction or statement.

PostgreSQL uses MVCC for both READ COMMITTED and REPEATABLE READ. It does not automatically switch between these isolation levels based on the type of query.

## 1. Does PostgreSQL automatically choose READ COMMITTED or REPEATABLE READ?

No. PostgreSQL uses the effective isolation level configured for the transaction.

PostgreSQL's default transaction isolation is **READ COMMITTED**.

Check the configured default:

```sql
SHOW default_transaction_isolation;
```

Check the effective isolation level for the current transaction:

```sql
SHOW transaction_isolation;

SELECT current_setting('transaction_isolation');
```

If an application does not explicitly choose another isolation level and the database/session configuration has not been changed, PostgreSQL uses READ COMMITTED.

## 2. Where can the isolation level be configured?

### A. For one transaction

READ COMMITTED:

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;

SELECT * FROM account;

COMMIT;
```

REPEATABLE READ:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT * FROM account;

COMMIT;
```

This is usually the clearest way to demonstrate or deliberately configure an isolation level for a specific unit of work.

The isolation level must be selected before the transaction executes its first query or data-modification statement.

### B. As the default for subsequent transactions in one session

```sql
SET SESSION CHARACTERISTICS AS TRANSACTION
ISOLATION LEVEL REPEATABLE READ;
```

This changes the default for subsequent transactions in that session. It does not retroactively change an already running transaction.

### C. As a database or role default

A database administrator can configure a default for new sessions connected to a database:

```sql
ALTER DATABASE payment_db
SET default_transaction_isolation = 'read committed';
```

A role-specific default can also be configured:

```sql
ALTER ROLE payment_app
SET default_transaction_isolation = 'read committed';
```

These are defaults, not a mechanism that automatically chooses a level based on each query. The transaction or session can have an explicit setting that determines its effective level.

## 3. How Spring Boot normally gets the isolation level

Consider a normal Spring service:

```java
@Service
public class AccountService {

    @Transactional
    public void transferMoney() {
        // Debit source account.
        // Credit destination account.
        // Save payment record.
    }
}
```

Spring's default @Transactional isolation setting is Isolation.DEFAULT. This means the transaction manager uses the underlying database/connection's default isolation.

With an ordinary PostgreSQL configuration, that means READ COMMITTED.

To request REPEATABLE READ deliberately:

```java
@Service
public class ReportService {

    @Transactional(
        isolation = Isolation.REPEATABLE_READ,
        readOnly = true
    )
    public AccountReport generateReport() {
        // Query 1.
        // Query 2.
        // Build the report.
        return report;
    }
}
```

Important Spring detail: transaction isolation settings apply when Spring starts a new transaction. If a method joins an existing transaction through the usual REQUIRED propagation, it normally participates in the existing transaction and inherits that transaction's isolation settings. It does not silently replace the already-running transaction's isolation.

Therefore, not writing READ COMMITTED in every service method does not mean PostgreSQL is choosing REPEATABLE READ. It normally means the service is using the database's default READ COMMITTED level.

## 4. Exact snapshot lifecycle

### READ COMMITTED

Each SQL statement gets its own snapshot.

1. A statement starts.
2. PostgreSQL determines which transactions' committed changes are visible at that statement's snapshot.
3. The statement executes using that snapshot.
4. The statement finishes.
5. A subsequent statement gets a new snapshot.

A single SELECT does not change its snapshot halfway through execution.

### REPEATABLE READ

One snapshot is reused for the transaction's reads.

1. BEGIN starts the transaction.
2. The first non-transaction-control statement executes.
3. PostgreSQL establishes the transaction snapshot.
4. Later statements reuse that snapshot.
5. The snapshot remains in effect until the transaction ends.
6. COMMIT or ROLLBACK ends the transaction.

BEGIN alone does not establish the REPEATABLE READ snapshot.

### Comparison

| Event | READ COMMITTED | REPEATABLE READ |
|---|---|---|
| BEGIN | Starts the transaction | Starts the transaction; snapshot not yet established |
| First SELECT starts | Statement snapshot S1 | Transaction snapshot S1 is established |
| Second SELECT starts | New statement snapshot S2 | Reuses S1 |
| Another transaction commits between the SELECTs | Second SELECT can see the commit | Second SELECT cannot see the concurrent commit |
| One SELECT is still running while another transaction commits | Running SELECT keeps its original statement snapshot | Running SELECT keeps the transaction snapshot |
| Transaction ends | Transaction ends | Transaction snapshot is no longer used |

## 5. Can Query 1 and Query 2 see different snapshots?

Yes, under READ COMMITTED. This is not a mid-query snapshot change. It happens because each statement starts with a new snapshot.

Suppose the database initially contains:

| Account | Balance | Status |
|---|---:|---|
| Alice | ₹20,000 | ACTIVE |
| Bob | ₹10,000 | ACTIVE |

T2 runs a report with two queries.

### Timeline under READ COMMITTED

**T2 — reporting transaction**

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;

SELECT COUNT(*)
FROM account
WHERE status = 'ACTIVE';
-- Result: 2
-- This statement uses snapshot S1.
```

T1 withdraws ₹5,000 from Alice and commits:

```sql
BEGIN;

UPDATE account
SET balance = balance - 5000
WHERE account_id = 'ALICE';

COMMIT;
```

T2 now executes its second query:

```sql
SELECT SUM(balance)
FROM account
WHERE status = 'ACTIVE';
-- Result: 25000
-- This statement uses new snapshot S2.

COMMIT;
```

T2 has observed:

- Account count = 2 using S1.
- Total balance = ₹25,000 using S2.

Each query is individually correct, but the results came from different database states.

### What if T1 commits while Query 1 is still executing?

Suppose Query 1 starts before T1 commits, but Query 1 takes several seconds to finish.

Under READ COMMITTED, Query 1 continues using the snapshot established when it started. It does not switch to the newer committed version halfway through its execution.

Only a later statement, such as Query 2, can get the new snapshot and see T1's committed change.

This is how PostgreSQL gives each statement a consistent view, even while other transactions commit concurrently.

## 6. The same report under REPEATABLE READ

Use the same starting balances: Alice ₹20,000 and Bob ₹10,000.

T2 starts:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY;

SELECT COUNT(*)
FROM account
WHERE status = 'ACTIVE';
-- Result: 2
-- Transaction snapshot S1 is established.
```

T1 withdraws ₹5,000 from Alice and commits.

T2 executes:

```sql
SELECT SUM(balance)
FROM account
WHERE status = 'ACTIVE';
-- Result: 30000
-- The transaction still uses S1.

COMMIT;
```

T2 sees one consistent database snapshot across both statements:

- Account count = 2.
- Total balance = ₹30,000.

This is useful for a multi-query report that must describe one consistent view of the database. It is not necessarily the newest possible view by the time the report finishes.

PostgreSQL also supports SERIALIZABLE READ ONLY DEFERRABLE for certain long-running read-only reports and backups when a snapshot guaranteed to be safe from serialization anomalies is valuable.

## 7. One connection versus multiple connections

### One connection and one transaction

Commands in one ordinary database transaction are issued sequentially on its connection:

```text
Connection 1 / T2
  BEGIN
    Query 1
    Query 2
    Query 3
  COMMIT
```

Under READ COMMITTED, these statements can use S1, S2, and S3 respectively.

Under REPEATABLE READ, they reuse the same transaction snapshot.

### Separate application requests

A typical Spring Boot service uses a connection pool. Separate requests can be assigned separate database connections and run separate transactions concurrently.

```text
HTTP Request A -> Connection 1 -> Transaction T1
HTTP Request B -> Connection 2 -> Transaction T2
```

Each transaction follows its own effective isolation level and snapshot rules.

Do not assume that two concurrent HTTP requests share one database transaction or one snapshot.

## 8. Which approach should we choose?

| Requirement | Suitable starting point |
|---|---|
| Ordinary CRUD and many payment operations | READ COMMITTED |
| Fresh committed view for each statement | READ COMMITTED |
| Multi-statement report requiring one consistent snapshot | REPEATABLE READ READ ONLY |
| Prevent serialization anomalies for complex multi-row invariants | SERIALIZABLE, with retry handling |
| Coordinate changes to specific shared account rows | Explicit row locking or carefully designed atomic SQL |
| Prevent simple invalid values or duplicates | Database constraints where the invariant can be expressed as a constraint |

No isolation level is a substitute for understanding and enforcing the application's business invariants.

For our payment system, we will generally start from PostgreSQL READ COMMITTED and deliberately add row locks, atomic conditional updates, or other coordination where the operation requires them.

## 9. Terms to remember

- **MVCC:** PostgreSQL's mechanism for maintaining row versions and deciding visibility.
- **Isolation level:** the rule that determines which committed versions a transaction or statement may see.
- **Statement snapshot:** the view used by one SQL statement under READ COMMITTED.
- **Transaction snapshot:** the stable view used by REPEATABLE READ and SERIALIZABLE.
- **Snapshot boundary under READ COMMITTED:** the beginning of each statement.
- **Snapshot boundary under REPEATABLE READ:** the first non-transaction-control statement.
- **Mixed-snapshot report:** a multi-query result where different statements saw different committed database states.
- **Transaction isolation default:** PostgreSQL normally defaults to READ COMMITTED.
- **Spring Isolation.DEFAULT:** delegate to the underlying transaction system's default isolation.

## 10. Questions for independent revision

1. Does PostgreSQL use MVCC only for REPEATABLE READ?
2. What is PostgreSQL's default isolation level?
3. What does Spring's @Transactional use when isolation is not specified?
4. At what exact point does READ COMMITTED establish a snapshot?
5. At what exact point does REPEATABLE READ establish a snapshot?
6. Can one SELECT switch to a newer snapshot while it is still executing?
7. How can two SELECT statements in one READ COMMITTED transaction see different committed states?
8. Why might a multi-query financial report use REPEATABLE READ READ ONLY?
9. What is the difference between two commands on one transaction connection and two concurrent HTTP requests using separate connections?

## Official references

- [PostgreSQL 18 — Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html)
- [PostgreSQL 18 — SET TRANSACTION](https://www.postgresql.org/docs/18/sql-set-transaction.html)
- [PostgreSQL 18 — Client Connection Defaults](https://www.postgresql.org/docs/18/runtime-config-client.html)
- [PostgreSQL 18 — Data Consistency Checks at the Application Level](https://www.postgresql.org/docs/18/applevel-consistency.html)
- [Spring Framework — Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
