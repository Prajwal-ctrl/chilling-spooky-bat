# Snapshot Isolation, Write Skew, and PostgreSQL SERIALIZABLE

## Why this topic matters

PostgreSQL REPEATABLE READ gives a transaction a stable snapshot. PostgreSQL implements this level using a multiversion concurrency-control model known as Snapshot Isolation.

Snapshot Isolation prevents several read anomalies, including dirty reads, non-repeatable reads, and phantom reads in PostgreSQL. However, it does not prevent every serialization anomaly.

A particularly important anomaly is **write skew**: two transactions read overlapping data, make decisions from that data, and then update different rows. Both transactions can commit, but their combined changes violate a business rule.

This matters whenever a business rule depends on more than one row. The implementation details in this note are PostgreSQL-specific.

## 1. What is Snapshot Isolation?

Snapshot Isolation is a concurrency-control model in which each transaction reads from a consistent snapshot of committed data. A transaction also sees its own writes.

In PostgreSQL:

- REPEATABLE READ uses a stable snapshot established by the first non-transaction-control statement.
- A transaction does not see commits made by concurrent transactions after its snapshot was established.
- If two REPEATABLE READ transactions try to update the same row and one has already committed a change after the other's snapshot, PostgreSQL raises a serialization failure.
- Transactions that update different rows can still both commit, even when their decisions depended on overlapping reads.

The last point is the important limitation.

### Simple-English explanation

**Snapshot Isolation means a transaction works from a consistent picture of the database. It keeps that picture stable while it runs, but this alone does not guarantee that every combination of concurrent transactions will preserve our business rules.**

PostgreSQL's REPEATABLE READ level is implemented using Snapshot Isolation. Other database products may use different implementations or expose Snapshot Isolation as a separate isolation level.

## 2. What is write skew?

Write skew occurs when:

1. Two concurrent transactions read overlapping data.
2. Both make decisions based on what they read.
3. They update different rows.
4. Both updates commit.
5. Their combined changes violate a business invariant.

The transactions do not overwrite the same row, so a same-row write-conflict check does not catch this anomaly under Snapshot Isolation.

### Example: hospital on-call rule

Business rule:

**At least one doctor must always remain on call.**

Initial database:

| Doctor ID | Name | On call |
|---|---|---|
| 101 | Alice | TRUE |
| 102 | Bob | TRUE |

Each doctor may take leave only if at least one other doctor remains on call.

Two requests arrive concurrently:

- T1 wants to mark Alice as not on call.
- T2 wants to mark Bob as not on call.

Both requests check the number of on-call doctors. Both see 2, so each independently concludes that the other doctor will remain available.

## 3. Why both transactions can commit under REPEATABLE READ

### T1 — Alice's leave request

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Returns 2

UPDATE on_call_doctor_demo
SET on_call = FALSE
WHERE doctor_id = 101;

-- Keep this transaction open for now.
```

### T2 — Bob's leave request

Run this in a second database session:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Also returns 2

UPDATE on_call_doctor_demo
SET on_call = FALSE
WHERE doctor_id = 102;

COMMIT;
```

Now return to T1's session:

```sql
COMMIT;
```

Both transactions can commit because:

- T1 updates Alice's row.
- T2 updates Bob's row.
- They do not update the same row.
- Both use a snapshot in which Alice and Bob are on call.

Final state:

| Doctor | On call |
|---|---|
| Alice | FALSE |
| Bob | FALSE |

The count is now **0**, which violates the business rule.

### Timeline

```text
T1 (REPEATABLE READ)             T2 (REPEATABLE READ)
────────────────────────────────────────────────────────
BEGIN                            BEGIN

Read Alice=TRUE, Bob=TRUE        Read Alice=TRUE, Bob=TRUE
Count = 2                        Count = 2

Set Alice=FALSE                  Set Bob=FALSE

COMMIT                           COMMIT

Final state: Alice=FALSE, Bob=FALSE
On-call count = 0
```

### Read set versus write set

This is the most useful way to recognize write skew.

| Transaction | Read set | Write set |
|---|---|---|
| T1 | Alice and Bob's on-call state | Alice's row |
| T2 | Alice and Bob's on-call state | Bob's row |

The **read sets overlap**, but the **write sets are different**.

The database can see that the transactions updated different rows. It does not automatically know that the application-level rule connects those rows.

### Why is this not a lost update?

| Lost update | Write skew |
|---|---|
| Concurrent transactions update the same row. | Concurrent transactions update different rows. |
| One write can overwrite another transaction's effect. | Both writes can survive. |
| The problem is a conflicting replacement of the same data. | The problem is decisions based on overlapping reads. |

### Simple-English explanation

**Both transactions read the same business condition, but each changes a different row. Since neither overwrites the other's row, both can commit under REPEATABLE READ. The combined result is invalid even though each transaction's decision looked valid using its own snapshot.**

## 4. Why the result is a serialization anomaly

A serial execution means one complete transaction runs before the other.

### If T1 executes before T2

1. T1 sees two doctors on call.
2. T1 marks Alice off call and commits.
3. T2 then checks the current state.
4. T2 sees only Bob on call.
5. T2 must reject Bob's leave request because no doctor would remain on call.

### If T2 executes before T1

The reverse order has the same result: T2 may succeed, but T1 must reject Alice's leave request.

Therefore, a state in which both doctors are off call cannot be explained by either serial ordering.

A **serialization anomaly** is a result from successfully committed concurrent transactions that cannot be explained by any possible one-at-a-time serial execution.

## 5. REPEATABLE READ versus SERIALIZABLE

PostgreSQL SERIALIZABLE provides the strongest standard transaction isolation level.

It guarantees that successfully committed serializable transactions have an effect equivalent to some serial execution.

**SERIALIZABLE does not mean PostgreSQL literally executes every transaction one at a time.** It allows concurrency but detects dangerous dependency patterns and aborts transactions when necessary to preserve serializable outcomes.

| Property | REPEATABLE READ | SERIALIZABLE |
|---|---|---|
| Stable snapshot | Yes | Yes |
| Dirty reads | Prevented | Prevented |
| Non-repeatable reads | Prevented | Prevented |
| Phantom reads in PostgreSQL | Prevented | Prevented |
| Write skew / serialization anomalies | Can occur | Prevented for successfully committed transactions |
| Concurrent work | Allowed | Allowed |
| Serialization failures and retries | Required for certain conflicting writes | Required when a serialization anomaly is detected |

## 6. How PostgreSQL SERIALIZABLE works: SSI

PostgreSQL implements SERIALIZABLE using **Serializable Snapshot Isolation (SSI)**.

SSI builds on Snapshot Isolation and monitors read/write dependencies between concurrent transactions.

At a high level:

1. Transactions continue to read from snapshots.
2. PostgreSQL tracks data read by serializable transactions using predicate-lock information.
3. It detects dangerous patterns of read/write dependencies that could create a serialization anomaly.
4. It aborts a transaction when necessary to prevent an invalid result from being committed.

### Predicate locks are not ordinary blocking locks

PostgreSQL uses predicate-lock information to track what a transaction read and whether another transaction writes data that could have affected that read.

Important distinctions:

- These predicate locks are used for dependency tracking.
- They do not normally make a writer wait for the reader.
- They are different from row locks acquired with SELECT FOR UPDATE.
- They can appear in pg_locks with mode SIReadLock.

SSI therefore provides serializable protection without turning every read into a blocking row lock.

### Serialization failure

An application may receive an error similar to:

```text
ERROR: could not serialize access due to read/write dependencies among transactions
SQLSTATE: 40001
```

PostgreSQL may report the failure during a statement or at transaction commit. Do not assume a particular transaction will always be the one aborted.

## 7. What happens to our hospital example under SERIALIZABLE?

Run the same business operation with both transactions using SERIALIZABLE.

The application logic is:

1. Count doctors currently on call.
2. Allow the requested doctor to leave only if another doctor will remain on call.
3. Update that doctor's row.
4. Commit.

PostgreSQL monitors the dependencies between the reads and writes. It will not allow both conflicting transactions to commit if that would produce the invalid zero-doctor state.

One transaction can commit. The other receives a serialization failure and must be rolled back.

When the failed transaction is retried from the beginning, it sees only one doctor on call. Its business validation then rejects the second leave request.

### Simple-English explanation

**SERIALIZABLE makes the database protect the result as though transactions had run one at a time. If two concurrent transactions together create a result that no safe serial order could create, PostgreSQL aborts one transaction. The application retries the whole transaction and validates again.**

## 8. Hands-on PostgreSQL lab

Use two separate PostgreSQL sessions connected to the same database. Session A is T1; Session B is T2.

### Step 0 — Create demo data

Run once, before either experiment:

```sql
DROP TABLE IF EXISTS on_call_doctor_demo;

CREATE TABLE on_call_doctor_demo (
    doctor_id INTEGER PRIMARY KEY,
    name      TEXT NOT NULL,
    on_call   BOOLEAN NOT NULL
);

INSERT INTO on_call_doctor_demo (doctor_id, name, on_call)
VALUES
    (101, 'Alice', TRUE),
    (102, 'Bob', TRUE);
```

### Experiment A — Demonstrate write skew under REPEATABLE READ

**Session A (T1):**

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Expected: 2

UPDATE on_call_doctor_demo
SET on_call = FALSE
WHERE doctor_id = 101;

-- Do not commit yet.
```

**Session B (T2):**

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Expected: 2

UPDATE on_call_doctor_demo
SET on_call = FALSE
WHERE doctor_id = 102;

COMMIT;
```

**Return to Session A:**

```sql
COMMIT;
```

Both transactions should be able to commit.

Verify the business rule:

```sql
SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Result: 0 (business rule violated)
```

### Reset between experiments

```sql
BEGIN;

UPDATE on_call_doctor_demo
SET on_call = TRUE;

COMMIT;
```

Verify that the on-call count is back to 2.

### Experiment B — Prevent write skew using SERIALIZABLE

**Session A (T1):**

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Expected: 2

UPDATE on_call_doctor_demo
SET on_call = FALSE
WHERE doctor_id = 101;

-- Keep the transaction open.
```

**Session B (T2):**

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Expected: 2

UPDATE on_call_doctor_demo
SET on_call = FALSE
WHERE doctor_id = 102;

COMMIT;
```

**Return to Session A and attempt to commit:**

```sql
COMMIT;
```

One of the two transactions must be aborted with a serialization failure. The exact transaction that loses can depend on execution timing; do not assume a particular session will always fail.

If a session receives an error, its transaction is aborted. Execute:

```sql
ROLLBACK;
```

Then verify from either session:

```sql
SELECT doctor_id, name, on_call
FROM on_call_doctor_demo
ORDER BY doctor_id;

SELECT COUNT(*)
FROM on_call_doctor_demo
WHERE on_call = TRUE;
-- Expected: 1
```

The exact successful doctor does not matter. The invariant matters: **at least one doctor must remain on call**.

### Lab safety and execution notes

- Keep both sessions open and overlapping. Running T1 completely before T2 would not reproduce write skew.
- Use the dedicated on_call_doctor_demo table, not a real application table.
- If a COMMIT fails with SQLSTATE 40001, roll back that failed transaction.
- Retry the complete business transaction, including the count query and validation—not just the UPDATE.
- When finished, remove the demo table if it is no longer needed:

```sql
DROP TABLE on_call_doctor_demo;
```

## 9. When should an application use SERIALIZABLE?

SERIALIZABLE can be useful when a business invariant depends on a predicate or multiple rows and it is difficult to protect that invariant with a simpler constraint or explicit locking strategy.

However, it has trade-offs:

- Serialization failures are expected possibilities and must be handled.
- The application must retry the entire transaction.
- More dependency tracking can create overhead and serialization failures.
- Transactions should be short and should not contain unnecessary remote calls or unrelated work.
- The business rule still needs to be expressed correctly inside the transaction.

An alternative is explicit coordination, such as locking a shared guard row before checking and modifying the related data. We will study row-level locking in the next major topic and compare the approaches there.

For simple invariants, also consider whether a database constraint can enforce the rule directly. Constraints are often simpler and stronger than application-only checks when the rule can be represented declaratively.

## 10. Terms to remember

- **Snapshot Isolation:** a transaction reads from a consistent snapshot and sees its own changes; concurrent transactions can still produce write skew.
- **Read set:** the data a transaction reads to make its decision.
- **Write set:** the data a transaction modifies.
- **Write skew:** overlapping reads and disjoint writes combine to violate a business invariant.
- **Serialization anomaly:** a committed concurrent result that cannot be explained by any serial ordering.
- **SERIALIZABLE:** isolation that guarantees committed transactions are equivalent to a serial execution.
- **SSI:** PostgreSQL's Serializable Snapshot Isolation mechanism, which tracks dependencies and aborts transactions when required.
- **Predicate lock / SIReadLock:** non-blocking tracking information used by SSI to reason about read/write dependencies.
- **Serialization failure (40001):** a transaction must be rolled back and retried from the beginning.

## 11. My learning checkpoint

My answer to the guided question was:

> so In this situation both independent row updates the data... whereas as per buisness logic only one is allowed

### Technical clarification

The answer correctly identified that the two transactions update different rows while the business rule permits only one leave request.

The deeper mechanism is:

- Both transactions read the same initial condition: Alice and Bob are on call.
- T1's write set contains Alice's row.
- T2's write set contains Bob's row.
- The read sets overlap, but the write sets are different.
- REPEATABLE READ does not detect this as a same-row concurrent-update conflict.
- Both can commit, leaving zero doctors on call.

### Recall in one sentence

**Write skew occurs when concurrent transactions make decisions from overlapping reads but update different rows, allowing both to commit under Snapshot Isolation even though their combined result violates a business rule; PostgreSQL SERIALIZABLE uses SSI to prevent such non-serializable outcomes.**

## 12. Questions for independent revision

1. What is Snapshot Isolation, and how does it relate to PostgreSQL REPEATABLE READ?
2. What is the difference between a transaction's read set and write set?
3. Why can both doctor leave requests commit under REPEATABLE READ?
4. Why is this write skew rather than a lost update?
5. What does serializable execution guarantee?
6. Does SERIALIZABLE mean PostgreSQL literally runs one transaction at a time?
7. What is SSI, and how is it different from ordinary blocking row locks?
8. Why must an application retry the entire transaction after SQLSTATE 40001?
9. What simpler database constraint or explicit-locking approach might protect a particular business rule?

## Official references

- [PostgreSQL 18 — Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html)
- [PostgreSQL 18 — Serializable Isolation and Predicate Locking](https://www.postgresql.org/docs/18/transaction-iso.html#XACT-SERIALIZABLE)
- [PostgreSQL — Serializable Snapshot Isolation wiki](https://wiki.postgresql.org/wiki/SSI)


## 13. SSI teach-back — read-write dependencies

### My answers to the checkpoint

- Question 1: **B** — PostgreSQL tracks what a SERIALIZABLE transaction reads so that dangerous concurrent read-write dependencies can be detected.
- Question 2: **A** — SSI predicate-lock information is not the same as a blocking SELECT FOR UPDATE row lock; it normally does not make writers wait.

### My explanation

> Okay so according to business logic both rows were depend on each other ....so before writing T1 ...T1 reads T2 and before writing T2 it reads T1

### Technical clarification

The intuition that the transactions depend on each other's data was correct, but a transaction does not literally read another transaction. Each transaction reads database rows; the other transaction later writes a row that was read.

- T1 reads Alice=TRUE and Bob=TRUE. T2 later writes Bob=FALSE. This creates a read-write anti-dependency **T1 → T2**.
- T2 reads Alice=TRUE and Bob=TRUE. T1 later writes Alice=FALSE. This creates a read-write anti-dependency **T2 → T1**.

Together these edges form a dependency cycle:

    T1 ──rw──> T2
     ^          |
     |          |
     └───rw─────┘

A useful way to remember the direction is: **reader → concurrent writer whose change would have altered that earlier read**.

PostgreSQL SSI monitors read-write dependencies and detects dangerous structures that could lead to a serialization anomaly. Its implementation uses predicate-lock information (SIReadLock) for tracking; these are not ordinary blocking row locks. The detailed SSI algorithm looks for dangerous structures involving consecutive read-write anti-dependencies, rather than simply waiting for a completed cycle and then checking it.

### Learning status

- [x] Identify why SSI tracks reads and concurrent writes.
- [x] Distinguish predicate-lock tracking from blocking row locks.
- [x] Explain that the transactions read rows, not each other.
- [x] Identify T1 → T2 and T2 → T1 read-write anti-dependencies in the doctor example.
- [ ] Explain the SSI dangerous-structure rule in more detail.
- [ ] Complete the two-session SQL lab under REPEATABLE READ and SERIALIZABLE.
