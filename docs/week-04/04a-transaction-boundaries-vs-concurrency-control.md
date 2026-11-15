# Why `@Transactional` Does Not Prevent Concurrent Transactions

> **Core idea:** `@Transactional` defines and manages a transaction boundary. It does not act as a global lock, stop other transactions from running, or automatically prevent every concurrency anomaly.

This is a key distinction for the payment system: **atomicity within one transaction is different from coordination between multiple concurrent transactions.**

## 1. What problem does `@Transactional` solve?

Imagine Alice transfers ₹4,000 to Bob.

Our service performs three operations:

1. Debit Alice.
2. Credit Bob.
3. Create the payment record.

Consider what happens if the application successfully debits Alice but encounters an exception before crediting Bob.

Without a shared database transaction, we could end up with:

**Incorrect financial state**

| Account | Change from this transfer |
|---|---:|
| Alice's account | **−₹4,000** |
| Bob's account | **₹0 received** |

**Partial transfer**

We cannot allow this.

This is the problem that `@Transactional` helps solve: the participating database operations can be committed together or rolled back together, instead of leaving a partial transfer behind.

### The transaction boundary

For example, one service operation can group these database changes:

```java
@Transactional
public void transfer(
        String sourceId,
        String destinationId,
        BigDecimal amount) {

    accountRepository.debit(sourceId, amount);
    accountRepository.credit(destinationId, amount);
    paymentRepository.save(createPayment(sourceId, destinationId, amount));
}
```

If all three operations participate in the same transaction managed by the same transaction manager, a successful commit makes the database changes permanent together. If the transaction is rolled back, the participating changes are undone together.

## 2. What does your understanding get right?

Your explanation was:

> so now why @Transational doesn't guarantee that 2 transaction won't overlap each other or can not read dirty or steal data it just guaranty that whatever transaction a particular thread has initialized it will be either completed successfully or completely rolled back it takes the guaranty of just 1 transaction not the whole ecosystem ....

The central intuition is right:

- T1 and T2 can each have their own transaction.
- T1's transaction can be atomic.
- T2's transaction can also be atomic.
- T1 and T2 can still overlap in time.
- Making each transaction atomic does not automatically coordinate their conflicting business operations.

A more precise version is:

**`@Transactional` manages the configured transaction boundary for a unit of work. It does not automatically serialize independent transactions or protect every shared row from concurrent modification.**

It is not a global lock for the application or database.

## 3. Two transactional methods can overlap

Imagine two requests invoke the same withdrawal service concurrently.

- T1 withdraws ₹5,000.
- T2 withdraws ₹2,000.
- Alice's starting balance is ₹20,000.
- Both methods use `@Transactional`.
- Both use PostgreSQL's default READ COMMITTED isolation, unless configured otherwise.

They are two separate transactions. A possible interleaving is:

| Step | Transaction T1 | Transaction T2 |
|---:|---|---|
| 1 | Begin transaction | |
| 2 | | Begin transaction |
| 3 | Read ₹20,000 | |
| 4 | | Read ₹20,000 |
| 5 | Calculate ₹15,000 in Java | |
| 6 | | Calculate ₹18,000 in Java |
| 7 | Write ₹15,000 | |
| 8 | Commit | |
| 9 | | Its conflicting UPDATE continues and writes the absolute value ₹18,000 |
| 10 | | Commit |

In this application-side read-modify-write example, both transactions can be individually transactional and yet the final balance can be ₹18,000 instead of the expected ₹13,000.

PostgreSQL coordinates conflicting row updates: T2 may wait for T1's update to finish. The lost update is still possible in this pattern because T2 sends the absolute value ₹18,000 that it calculated earlier in Java. Waiting does not recalculate that value.

## 4. What `@Transactional` guarantees — and what it does not

| Concern | What provides the behavior |
|---|---|
| Debit, credit, and payment record commit or roll back together | A shared database transaction boundary |
| Whether uncommitted changes are visible to another transaction | Database transaction isolation |
| Preventing or detecting conflicting balance modifications | Row locks, atomic conditional SQL, optimistic version checks, or stronger isolation with retries |
| Keeping a balance above its minimum | Business validation and database constraints/conditional updates |
| Rolling back an HTTP call, email, or Kafka publication | Not an ordinary local database transaction by itself |

### It does help with

- Grouping participating database operations into one transaction.
- Committing them together on success.
- Rolling them back together when a configured rollback condition applies.
- Applying the isolation, propagation, timeout, and read-only settings configured for the transaction.

### It does not automatically

- Stop T1 and T2 from executing concurrently.
- Lock every row read by the transaction.
- Make every ordinary SELECT wait for writers.
- Refresh a Java variable or managed entity after another transaction commits.
- Prevent every stale-read or lost-update scenario.
- Roll back external side effects such as an HTTP request to another service.

## 5. Dirty reads are different from stale reads

Your question also mentioned dirty reads. We must distinguish them from the lost update we have been studying.

| Term | Meaning |
|---|---|
| Dirty read | T2 reads a value written by T1 before T1 commits it. |
| Stale read/value | T2 uses an older value that was committed when read, but has since been superseded by another committed change. |
| Lost update | A later write based on an old value overwrites an earlier transaction's effect. |

PostgreSQL does not allow dirty reads. Its READ COMMITTED isolation level, which is the PostgreSQL default, prevents them. Spring's default `isolation = Isolation.DEFAULT` delegates to the underlying transaction system; it does not mean that the annotation itself independently implements dirty-read prevention.

However, READ COMMITTED can still permit the stale application-side read-modify-write pattern described above.

**Important:** `@Transactional` is not the reason PostgreSQL prevents dirty reads. That behavior comes from PostgreSQL's isolation semantics.

## 6. Important Spring defaults

The current Spring Framework reference documents these default `@Transactional` settings:

| Setting | Default |
|---|---|
| Propagation | `PROPAGATION_REQUIRED` |
| Isolation | `ISOLATION_DEFAULT` |
| Read-only | `false` |
| Timeout | Underlying transaction system's default, or none if unsupported |
| Rollback | By default, `RuntimeException` and `Error` trigger rollback; checked exceptions do not |

### What does PROPAGATION_REQUIRED mean?

It does not mean one new independent physical transaction is always created for every annotated method.

- If there is no transaction, Spring starts one.
- If a compatible transaction already exists, the method normally joins that existing transaction.
- Other propagation options, such as `REQUIRES_NEW`, change this behavior.

So the transaction boundary follows Spring's propagation rules and the configured transaction manager. It should not be simplified to "one thread always equals exactly one independent transaction."

### Thread-bound transaction context

For Spring's ordinary imperative transaction model, transaction context is generally bound to the current thread. Starting work on a new thread, such as through `@Async` or a manually created thread, does not automatically make that work part of the original transaction.

This is another reason to be precise about what work participates in a transaction.

## 7. How do we protect our payment balance?

We need to combine transaction atomicity with an appropriate concurrency-control strategy.

| Strategy | Purpose |
|---|---|
| `@Transactional` | Keep debit, credit, and payment persistence atomic |
| Atomic conditional SQL update | Apply a balance change against the database row and enforce a condition in one statement |
| Pessimistic locking | Lock the account row before reading and modifying it |
| Optimistic locking | Detect that the account changed since it was read |
| SERIALIZABLE isolation | Require successful concurrent transactions to behave like some serial ordering; retry serialization failures |

For our first PAY-006 implementation, we will keep the complete transfer in one transaction and use pessimistic account-row locking in deterministic account-ID order. We will later compare this with atomic SQL and optimistic locking.

## 8. Easy way to remember

- **Transaction atomicity:** one unit of work either commits together or rolls back together.
- **Transaction isolation:** controls what concurrent transactions can observe and which interleavings the database permits.
- **Concurrency control:** coordinates or detects conflicting operations.
- **Business invariants:** define what must remain true for account balances and transfers.

A transaction is not the same thing as a lock. `@Transactional` is not a global mutex.

## 9. Questions to answer

1. Why can two methods annotated with `@Transactional` execute at overlapping times?
2. What is the difference between atomicity and isolation?
3. Why does PostgreSQL READ COMMITTED prevent dirty reads but still allow the stale application-side read-modify-write pattern?
4. Does `PROPAGATION_REQUIRED` always create a new physical transaction?
5. What happens when a transactional method starts additional work on a new thread?
6. Which mechanism should we use to protect Alice's balance from conflicting withdrawals, and why?

## References

- [Spring Framework — Using `@Transactional`](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring Framework — Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
- [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL — SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html)
