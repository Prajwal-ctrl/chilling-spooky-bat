# Lost Updates

## Definition

A lost update occurs when concurrent read-modify-write operations overwrite one another, causing one operation's effect to disappear.

## Worked example: two withdrawals from Alice's account

Initial Alice balance: **₹10,000**.

- T1 wants to withdraw ₹4,000.
- T2 wants to withdraw ₹3,000.
- Both transactions read the account balance before either transaction writes an update.

### Step 1 — Both transactions read the same value

| Transaction | Value read |
|---|---:|
| T1 | ₹10,000 |
| T2 | ₹10,000 |

The important detail is that both transactions read **the same old value**, ₹10,000. They do not read two different values.

### Step 2 — Each transaction calculates its own new balance

| Transaction | Calculation | Calculated balance |
|---|---|---:|
| T1 | ₹10,000 − ₹4,000 | ₹6,000 |
| T2 | ₹10,000 − ₹3,000 | ₹7,000 |

Each transaction calculates independently using the old balance it read.

### Step 3 — The writes happen one after another

| Order | Operation | Balance written |
|---:|---|---:|
| 1 | T1 writes its calculated balance | ₹6,000 |
| 2 | T2 writes its calculated balance afterward | ₹7,000 |

T2's write is an absolute balance calculated from the stale ₹10,000 value. It overwrites the effect of T1's withdrawal.

### Step 4 — Compare expected and actual results

**Expected balance, if both withdrawals succeed:**

₹10,000 − ₹4,000 − ₹3,000 = **₹3,000**

**Actual balance in the execution above:**

**₹7,000**

The T1 withdrawal of ₹4,000 has been lost from the final balance. If the write order were reversed in this same faulty pattern, the final balance could instead be ₹6,000.

## Visual execution structure

```text
Initial committed balance: ₹10,000
                  |
          +-------+-------+
          |               |
       Thread T1       Thread T2
          |               |
     Read ₹10,000     Read ₹10,000
          |               |
   Calculate ₹6,000  Calculate ₹7,000
          |               |
     Write ₹6,000        |
          |          Write ₹7,000
          |               |
          +-------+-------+
                  |
       Final balance: ₹7,000
       Expected balance: ₹3,000
       Lost effect: T1's ₹4,000 withdrawal
```

## Why MVCC alone does not prevent this example

MVCC controls which row versions a transaction can see. It prevents T2 from seeing T1's uncommitted value, but it does not automatically make a value already calculated in Java fresh again.

The example has two separate stages:

1. **Read stage:** T1 and T2 both run ordinary SELECT statements and can read the previously committed ₹10,000 version. Under READ COMMITTED, each statement gets a snapshot. MVCC allows these reads without forcing every reader to wait for the writer.
2. **Write stage:** conflicting UPDATE statements on the same row are coordinated by PostgreSQL row locking. T2 may wait for T1 to commit. But if T2 sends the absolute value `7000` that Java calculated earlier, waiting does not change that literal into a new calculation.

So PostgreSQL can serialize the conflicting writes and still store a stale application-calculated absolute value. MVCC visibility and application-level read-modify-write correctness are different concerns.

## Why database-side arithmetic behaves differently

### Application calculates an absolute value

For example, Java reads ₹10,000 and calculates ₹7,000 for T2:

```sql
UPDATE account
SET balance = 7000
WHERE account_id = 'ALICE';
```

If T2 waits for T1's conflicting update, the literal value `7000` is still what T2 asks PostgreSQL to store afterward.

### PostgreSQL calculates from the row it updates

Instead, perform the subtraction in the UPDATE statement:

```sql
UPDATE account
SET balance = balance - 3000
WHERE account_id = 'ALICE';
```

Under PostgreSQL READ COMMITTED, when this UPDATE waits for a concurrent updater, PostgreSQL can apply the operation to the updated row version after the other transaction commits. The expression `balance - 3000` is then evaluated using the current row balance, not the old value previously copied into Java.

Example:

1. T1 subtracts ₹4,000 and commits ₹6,000.
2. T2's UPDATE continues and subtracts ₹3,000 from ₹6,000.
3. The resulting balance is ₹3,000.

This is an atomic database-side arithmetic update and avoids this particular lost-update pattern.

### Include the minimum-balance condition

For a withdrawal, the update can also enforce the minimum-balance rule:

```sql
UPDATE account
SET balance = balance - :amount
WHERE account_id = :accountId
  AND balance - :amount >= minimum_balance;
```

Check the affected-row count:

- One row updated: the withdrawal passed the condition and was applied.
- Zero rows updated: the account was not found or the balance condition was not satisfied.

In production code, distinguish these outcomes where necessary, and keep the source debit and destination credit in the same database transaction. An atomic single-row update does not by itself make an entire two-account transfer atomic.

## Other ways to prevent or detect lost updates

1. **Pessimistic locking:** lock the account row before reading and calculating the new balance, for example with `SELECT ... FOR UPDATE`.
2. **Optimistic locking:** include an expected version in the UPDATE and treat zero affected rows as a conflict.
3. **Atomic conditional SQL:** perform the arithmetic and business condition in the database UPDATE, then inspect the affected-row count.

Choose the strategy based on the whole business operation, isolation level, contention pattern, and required invariants.

## Read-modify-write and stale values

Many application operations follow a **read-modify-write** pattern:

1. **Read:** fetch the current value from the database.
2. **Modify:** calculate a new value in application code, often in a Java variable or entity.
3. **Write:** send that calculated value back to the database.

Example in a Spring service:

```java
Account account = accountRepository.findById(accountId).orElseThrow();

BigDecimal newBalance = account.getBalance().subtract(amount);

account.setBalance(newBalance);
accountRepository.save(account);
```

The value returned by the initial database read is now held in the Java application. A later commit from another transaction does not automatically refresh that local variable or managed entity.

### What is a stale read/value?

A value is stale when it was correct at the time it was read but another transaction has changed the database value before the application uses the old value to make a later decision or write.

In our example:

- T2 reads ₹20,000 from PostgreSQL.
- T2 keeps ₹20,000 in its Java variable.
- T1 changes the database balance to ₹15,000 and commits.
- T2 still has ₹20,000 in its local variable and may calculate ₹18,000.

T2 did read from the database originally. The issue is not that it read only from Java. The issue is that the Java copy is not automatically refreshed when another transaction commits.

A stale read/value is not the same as a dirty read:

- **Dirty read:** reading another transaction's uncommitted change.
- **Stale read/value:** using an older value that has since been superseded by a committed change.

### Worked exercise — ₹20,000 starting balance

- T1 withdraws ₹5,000.
- T2 withdraws ₹2,000.
- Both read ₹20,000 before either writes.

| Step | T1 | T2 |
|---|---|---|
| 1. Read | ₹20,000 | ₹20,000 |
| 2. Calculate | ₹15,000 | ₹18,000 |
| 3. Write | Writes ₹15,000 | |
| 4. Write | | Writes its stale absolute value ₹18,000 |

**Expected balance if both withdrawals succeed:** ₹20,000 − ₹5,000 − ₹2,000 = **₹13,000**.

**Actual balance in the illustrated write order:** **₹18,000**.

The later absolute-value write overwrites the earlier withdrawal's effect.

### Key distinction

A **stale value** is the outdated input held by the application. A **lost update** is the incorrect outcome when a later write based on that stale value overwrites an earlier change.

## Planned experiment

Use two Java threads and synchronization barriers so both transactions read the same initial balance before either performs its write. Capture the final database state and explain the exact interleaving. Then compare the absolute-value update with the database-side arithmetic update.

## Questions to answer

1. Why is the final result ₹7,000 in the illustrated absolute-value write order?
2. Why did both transactions calculate different values even though they read the same balance?
3. Why doesn't MVCC automatically refresh a value already calculated in Java?
4. Why does `balance = balance - :amount` behave differently from `balance = :newBalance` under PostgreSQL READ COMMITTED?
5. Why does an atomic account-row update not, by itself, guarantee that a two-account transfer is atomic?
