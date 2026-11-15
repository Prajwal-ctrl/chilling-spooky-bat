# READ COMMITTED vs REPEATABLE READ — Snapshot Behaviour

## Why this topic matters

When two payment requests operate on the same account, we need to know exactly which committed balance each transaction can see.

PostgreSQL transaction isolation controls the visibility of concurrent changes. In particular, READ COMMITTED and REPEATABLE READ differ in **when a snapshot is taken and how long it remains stable**.

This note records the scenarios we reasoned through, including what happens when a transaction tries to update a row that another transaction changed after its snapshot.

> These explanations are specifically about PostgreSQL. Isolation-level behaviour can differ across database products.

## 1. READ COMMITTED: a fresh snapshot for every statement

READ COMMITTED is PostgreSQL's default isolation level.

Each SQL statement sees rows committed before that statement began. Therefore, two SELECT statements in the same transaction can return different values if another transaction commits an update between them.

### Timeline

Initial committed balance: Alice = ₹20,000.

```text
T2 (READ COMMITTED)                 T1
──────────────────────────────────────────────────
BEGIN

SELECT balance
→ ₹20,000

                                    UPDATE balance = 15000
                                    COMMIT

SELECT balance
→ ₹15,000
(COMMAND GETS A NEW SNAPSHOT)

COMMIT
```

### What this means

- T2's first SELECT sees ₹20,000.
- T1 commits the new balance of ₹15,000.
- T2's second SELECT gets a fresh statement snapshot and sees ₹15,000.
- The result of T2's first SELECT does not change retroactively. Only the later statement sees the newer committed value.

### Simple-English explanation

**READ COMMITTED means every new SQL statement gets a fresh view of committed data. If another transaction commits an update between my two SELECT statements, the second SELECT can see that update.**

This behaviour is an example of a **non-repeatable read**: a transaction reads a row, another transaction commits a change, and the first transaction reads the same row again but gets a different value.

## 2. REPEATABLE READ: a stable snapshot for the transaction

Under PostgreSQL REPEATABLE READ, the transaction uses a stable snapshot for its reads.

The snapshot is established when the transaction executes its first non-transaction-control statement—for example, its first SELECT. A BEGIN statement alone does not establish this snapshot.

### Timeline: T1 commits after T2's first SELECT

Initial committed balance: Alice = ₹20,000.

```text
T2 (REPEATABLE READ)                T1
──────────────────────────────────────────────────
BEGIN

SELECT balance
→ ₹20,000
(snapshot established)

                                    UPDATE balance = 15000
                                    COMMIT

SELECT balance
→ ₹20,000
(SAME SNAPSHOT IS REUSED)

COMMIT
```

### What this means

- T2's first SELECT establishes its snapshot and sees ₹20,000.
- T1 later commits ₹15,000.
- T2's next ordinary SELECT continues to use its original snapshot, so it still sees ₹20,000.
- T2 does not automatically switch to every newer value committed by another transaction.

### Simple-English explanation

**REPEATABLE READ means my transaction keeps the same snapshot for its reads. If another transaction commits a change after my snapshot was established, my later SELECT statements do not see that new change.**

## 3. Important difference: reading an old snapshot does not mean we can overwrite a newer row

We tested this situation:

- T2 uses REPEATABLE READ and reads Alice's balance as ₹20,000.
- T1 updates Alice's balance to ₹15,000 and commits.
- T2 attempts to write an absolute value of ₹18,000.

T2 might still read ₹20,000 from its snapshot, but PostgreSQL does not allow it to silently overwrite the row changed by T1 after T2's snapshot.

### Timeline

```text
T2 (REPEATABLE READ)                T1
──────────────────────────────────────────────────
BEGIN

SELECT balance
→ ₹20,000
(snapshot established)

                                    UPDATE balance = 15000
                                    COMMIT

UPDATE account
SET balance = 18000
WHERE account_id = 'ALICE';

→ SERIALIZATION FAILURE
  SQLSTATE 40001

ROLLBACK
```

PostgreSQL raises an error similar to:

```text
ERROR: could not serialize access due to concurrent update
SQLSTATE: 40001
```

The update is rejected because T2 is attempting to modify a row that another transaction changed after T2's snapshot.

### What should the application do?

The application must abort and retry the **entire transaction from the beginning**, not just repeat the failed UPDATE.

On retry:

1. T2 starts a new transaction.
2. The new transaction gets a fresh snapshot.
3. It reads the committed balance of ₹15,000.
4. It recalculates and validates the payment against the current balance.
5. It attempts the update again.

The business validation must run again because the balance may have changed.

### Simple-English explanation

**REPEATABLE READ protects my transaction's snapshot, but it does not let me overwrite a row that another transaction changed after my snapshot. PostgreSQL reports a serialization failure. I must roll back and retry the whole transaction using a new snapshot.**

## 4. Snapshot timing: why BEGIN alone is not enough

We then changed the order of events.

Suppose T1 commits ₹15,000 **before T2 runs its first SELECT**. T2 uses REPEATABLE READ.

```text
T1                                  T2 (REPEATABLE READ)
──────────────────────────────────────────────────
UPDATE balance = 15000
COMMIT

                                    BEGIN

                                    SELECT balance
                                    → ₹15,000
                                    (snapshot established)

                                    UPDATE balance = 18000
                                    → SUCCESS, assuming no
                                      other conflicting change

                                    COMMIT
```

T2 sees ₹15,000 because its snapshot is established by its first non-transaction-control statement, which runs after T1 commits.

**Important:** A transaction can execute BEGIN before T1 commits and still see T1's committed update if T2 has not yet executed its first snapshot-establishing statement.

### Simple-English explanation

**REPEATABLE READ does not take its snapshot just because I execute BEGIN. The snapshot is established when I run the first real SQL statement. If T1 commits before that statement, my transaction can see T1's committed value.**

## 5. Comparison table

| Situation | READ COMMITTED | REPEATABLE READ |
|---|---|---|
| T1 commits before T2's first SELECT | T2 sees the new committed value | T2 sees the new committed value |
| T1 commits after T2's first SELECT | T2's next SELECT can see the new value | T2's next SELECT keeps seeing the original snapshot value |
| T2 tries to update a row changed by T1 after T2's snapshot | PostgreSQL may wait for the conflicting updater and then apply READ COMMITTED update rules | PostgreSQL raises a serialization failure if T1 committed the conflicting row change after T2's snapshot |
| Dirty reads | Not allowed | Not allowed |
| Non-repeatable reads | Possible | Prevented |

The last two rows are important: REPEATABLE READ is stronger for snapshot stability, but it is not the same as SERIALIZABLE and does not automatically enforce every business invariant.

## 6. Terms to remember

- **Snapshot:** the database view used by a statement or transaction to decide which row versions are visible.
- **Non-repeatable read:** a transaction reads a row again and sees a different committed value.
- **Serialization failure:** PostgreSQL rejects a transaction because its attempted update conflicts with a row change made after its REPEATABLE READ snapshot.
- **Retry the whole transaction:** start a new transaction and repeat all reads, validations, and writes using a new snapshot.

## 7. My learning checkpoint

I was able to predict the result of the four scenarios, with one correction needed:

1. READ COMMITTED: T2's second SELECT sees ₹15,000 after T1 commits.
2. REPEATABLE READ: T2's second SELECT still sees ₹20,000 if T1 commits after T2's snapshot is established.
3. If T2 then tries to update that same row, PostgreSQL raises a serialization failure instead of silently accepting the stale write.
4. If T1 commits before T2's first snapshot-establishing statement, T2 sees ₹15,000 and can update it, assuming no later conflicting change.

### Recall in one sentence

**READ COMMITTED takes a fresh snapshot for every statement; REPEATABLE READ keeps a stable snapshot from the first non-transaction-control statement, and PostgreSQL rejects a conflicting update to a row changed after that snapshot.**

## 8. Questions for independent revision

1. Why can two SELECT statements return different values under READ COMMITTED?
2. Does BEGIN alone establish a REPEATABLE READ snapshot?
3. Why does a conflicting update fail under REPEATABLE READ instead of overwriting the newer row?
4. Why must the whole transaction be retried after SQLSTATE 40001?
5. Does REPEATABLE READ by itself guarantee every financial business invariant?

## Official reference

- [PostgreSQL 18 — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL 18 — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)
