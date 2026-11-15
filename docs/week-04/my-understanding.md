# My Understanding — Week 4

This file captures my own explanations, mental models, corrections, and questions while learning. The aim is to preserve how I understood a concept at the time, not just store textbook definitions.

These notes are intentionally written in my own voice so I can revise them quickly later. Where useful, a technically precise recap is added below the original reflection. Original explanations are kept as written; clarifications are separate.

---

## 1. MVCC — Why a normal SELECT does not wait for an uncommitted update

**Date:** 1 November 2026

### My first attempt

> Okay, so I think the T2 will wait normally, and it can read the previous record of the T1 transactions, because unless or until the lock is not open for the T2 transaction, it is reading the older version of T1 transaction. But as soon as the transaction T1 commits, the T2 will read the updated transaction of T1. So yeah, it will wait in a queue, and before update it will read the old data or old row of T1 transaction. But as soon as T1 gets committed and it is successful, it will read the new updated balance of the T1 thread.

### What I realized

I had mixed up two different operations: an ordinary SELECT and a SELECT that asks PostgreSQL to acquire a row lock.

### My corrected explanation

> Okay, so actually I forgot that it was a normal transaction that T2 is reading, not a transaction that it will commit any data. So yes, normally it will read the balance without any interruption, even if though that even is proceeding or committing any transactions, because at that time T2 will read the older data from the VCCS version control. Yeah, so that's all.

### My understanding, clarified

When T1 has updated Alice's balance but has not committed yet, T2 can execute a normal `SELECT` without waiting for T1's row lock. PostgreSQL MVCC allows T2 to read the previously committed version of the row; T2 cannot see T1's uncommitted balance.

After T1 commits, a **new SELECT statement** issued by T2 under PostgreSQL's default READ COMMITTED isolation level gets a new statement snapshot and can see the updated balance. This describes a subsequent statement; it does not mean an already completed SELECT changes its result.

The important distinction is:

- Normal `SELECT`: reads a visible committed version and normally does not wait for the conflicting row lock.
- `SELECT ... FOR UPDATE`: requests a row lock and normally waits if another transaction holds a conflicting lock.
- Conflicting `UPDATE`: normally waits for the other transaction's conflicting row lock.

The correct term is **MVCC — Multi-Version Concurrency Control** (not "VCCS").

### Recall in one sentence

**MVCC lets an ordinary reader see the last committed row version while another transaction is still working; a new READ COMMITTED statement can see the writer's change after it commits.**

### Follow-up question

How does this behavior change under REPEATABLE READ, where a transaction keeps a stable snapshot?

---

## 2. Concurrency vs Parallelism vs Resource Contention

**Date:** 2 November 2026

### My initial explanation

> So concurrency means how multiple requests are processing at same time. So at this point the task can be switched. The task can be means one can execute, another is waiting, and while the another is waiting, another task can be executed. Parallelism is running the same task at same time on the same instance, or running the multiple tasks at the same instance, not at two different instances. So on a single core only we are executing two tasks, two different tasks at same time on the same instance. So that's the parallelism, and this is contention. Contention is on two different cores we are trying to execute same task, which are operating the same row or same IDs.

### What I corrected

I initially mixed up parallelism with executing inside one application instance, and I mixed up resource contention with two CPU cores accessing the same database row.

The important correction is that **the number of application instances does not define parallelism**. A single Java Spring Boot JVM can use multiple CPU cores when the machine/container and thread scheduling allow it.

### My applied understanding

When asked about one Spring Boot JVM running on a machine with four CPU cores and receiving 100 payment requests, I explained:

> The simple answer is yes, because it can run multiple requests as per the service load concurrently on four CPU cores.

> Yes, it can execute the request in parallel at the same time, at the same moment, on different CPU cores, the request can be processed.

> It is possible. It is possible. It is not completely reliable that it cannot happen. It is possible that two CPU cores try to update the same row in a table.

### My understanding, clarified

- **Concurrency:** Multiple tasks make progress during overlapping periods. A single CPU core can support concurrency by switching between tasks; the tasks do not have to execute at the exact same instant.
- **Parallelism:** Multiple tasks execute at the same instant on separate execution resources, such as different CPU cores. A single JVM/application instance can execute threads in parallel when multiple cores are available and the threads are runnable.
- **Resource contention:** Multiple tasks compete for access to a shared resource. Examples include two transactions requesting conflicting locks on the same account row, or multiple threads competing for a synchronized Java object.

These are different ideas:

- Parallelism is not defined by whether requests use the same or different account IDs.
- Contention does not require different CPU cores. Tasks can contend for a shared resource even when they are scheduled on one core.
- Two ordinary PostgreSQL SELECT statements reading the same row do not automatically contend for a row lock. Conflicting updates or locking reads can make transactions wait.
- Four CPU cores make parallel execution possible, but they do not mean every incoming request executes simultaneously. Thread pools, CPU quotas, scheduling, database connections, and other limits affect actual execution.

### Recall in one sentence

**Concurrency is multiple tasks making progress during overlapping periods; parallelism is multiple tasks executing at the same instant; contention is multiple tasks competing for a shared resource.**

### Status

- [x] Explain concurrency versus parallelism.
- [x] Explain why a single JVM can use multiple CPU cores.
- [x] Identify resource contention in the shared-account example.
- [ ] Continue to race conditions and how timing can violate correctness.

---

## 3. Race Conditions and Lost Updates — learning in progress

**Date:** 15 November 2026

### My first explanation of a race condition

> So what I think about race conditions is race condition occurs when two resources or two threads try to update the same shared record, but without the pessimistic lock. So Alice has 10,000 rupees in his account, and T1 is trying to update it to 14,000 dollar, and T2 is trying to cut it down by 4,000. So both of them are reading two different values, and that is when the race condition is occur.

### What we clarified

A race condition is not defined only by the absence of pessimistic locking. It happens when the correctness of a result depends on the timing or ordering of concurrent operations, and a possible execution order produces an incorrect result.

A lost update is one specific example of a race condition. Other concurrency-control strategies, such as optimistic version checks or atomic SQL updates, can also prevent particular unsafe outcomes.

### My first answer to the lost-update exercise

> Final balance: 3000
> My reasoning: this happens because we don't follow the pessemestic lock on row level so both of them read different values

### What I corrected

In the example, both T1 and T2 read **the same old value: ₹10,000**. They calculate different new values because their withdrawal amounts differ:

- T1 calculates ₹10,000 − ₹4,000 = ₹6,000.
- T2 calculates ₹10,000 − ₹3,000 = ₹7,000.

If T1 writes ₹6,000 and T2 later writes its previously calculated absolute value of ₹7,000, the final balance is ₹7,000. The expected balance, if both withdrawals succeed, is ₹3,000. The later write has overwritten the effect of T1.

### My question about PostgreSQL MVCC

I asked why MVCC does not prevent this race condition, because I expected T2 to wait until T1 commits rather than read an old record.

### My understanding, clarified

MVCC and write coordination solve related but different problems:

- MVCC controls which row versions a transaction can see. An ordinary SELECT under READ COMMITTED can read the previously committed version while another transaction has an uncommitted update.
- A conflicting UPDATE does coordinate through PostgreSQL row locking. T2 may wait for T1 to commit.
- But if Java already calculated an absolute replacement value such as ₹7,000 using the old ₹10,000 balance, waiting does not recalculate that value.

### What I understood about database-side arithmetic

I compared these two updates:

Application-calculated absolute value:

```sql
UPDATE account
SET balance = 7000
WHERE account_id = 'ALICE';
```

Database-side arithmetic:

```sql
UPDATE account
SET balance = balance - 3000
WHERE account_id = 'ALICE';
```

My explanation was:

> because here 1st one actual deduct the amount from updated value rather than doing calculation on java side and then update the balance directly

### My understanding, clarified

This is the important difference I understood:

- With `SET balance = 7000`, Java has already calculated the absolute balance using a value it read earlier. If the UPDATE waits for another transaction, the literal ₹7,000 does not get recalculated.
- With `SET balance = balance - 3000`, PostgreSQL performs the subtraction as part of the UPDATE. Under READ COMMITTED, after waiting for a concurrent updater, PostgreSQL applies the expression to the updated row version. If T1 has committed ₹6,000, T2 subtracts ₹3,000 from ₹6,000, resulting in ₹3,000.

For a withdrawal, we should also enforce the minimum-balance condition in the UPDATE and check the affected-row count. For a complete transfer, source debit and destination credit still need one transaction.

### Read-modify-write, stale reads, and the ₹20,000 exercise

We studied the common read-modify-write pattern:

1. **Read:** fetch the current account balance from PostgreSQL.
2. **Modify:** calculate the new balance in Java.
3. **Write:** persist the calculated value back to PostgreSQL.

My understanding of a stale value is that it was correct when the application read it, but another transaction committed a newer value before the application used its old copy.

For the ₹20,000 exercise, I answered:

> so Read Read Write update write update
> when T1 updates the actual balance or writes the balance and T2 is still in read mode or write mode at that time T2 value becomes stale
> because it maybe read from service layer of java instead of directly from Database and it doesn't read the read committed value from DB
> actual balance will be 18000 instead of 13000

### My understanding, clarified

The exercise was:

- Initial balance: ₹20,000.
- T1 withdraws ₹5,000 and calculates ₹15,000.
- T2 withdraws ₹2,000 and calculates ₹18,000.
- Both transactions read ₹20,000 before either writes.
- T1 writes ₹15,000 and commits.
- T2 writes its previously calculated absolute value of ₹18,000 afterward.

The expected balance, if both withdrawals succeed, is ₹13,000. The actual balance in this write order is ₹18,000 because T2 overwrites T1's update.

One correction to my explanation: T2 originally **did read from PostgreSQL**. After that query, it kept a local value in its Java variable or entity. PostgreSQL committing a new value does not automatically refresh that existing Java value. A new query or an explicit refresh/locking strategy is needed to obtain the current value.

A stale value and a lost update are related but different:

- **Stale value:** the outdated value held by the application.
- **Lost update:** the incorrect result caused when a later write based on that stale value overwrites an earlier update.

This is not a dirty read. T2 read a previously committed value, not T1's uncommitted change.

### Recall in one sentence

**Read-modify-write means read a value, calculate a replacement, and write it later; a stale Java value does not refresh automatically when another transaction commits, and using it for a later absolute-value write can cause a lost update.**

### Learning status

- [x] Identify that a lost update can be a race condition.
- [x] Understand that both transactions can read the same old balance.
- [x] Calculate expected and actual balances in the ₹20,000 exercise.
- [x] Explain that Java's local value does not refresh automatically after another transaction commits.
- [x] Distinguish a stale value from a dirty read.
- [ ] Explain the complete execution timeline independently, without hints.
- [ ] Understand and compare read-modify-write, atomic SQL updates, pessimistic locking, and optimistic locking in more depth.
- [ ] Apply these approaches in a deterministic PostgreSQL concurrency experiment.


---

## 4. @Transactional — transaction boundary vs concurrency control

**Date:** 15 November 2026

### My explanation

> so now why @Transational doesn't guarantee that 2 transaction won't overlap each other or can not read dirty or steal data it just guaranty that whatever transaction a particular thread has initialized it will be either completed successfully or completely rolled back it takes the guaranty of just 1 transaction not the whole ecosystem ....

### What I understood correctly

I identified that `@Transactional` does not prevent two independent transactions from overlapping. Each transaction can have its own transaction boundary and can commit or roll back independently. Making T1 atomic does not automatically coordinate T1 with T2.

### My understanding, clarified

- `@Transactional` tells Spring to apply transaction semantics to a method or class through the configured transaction manager.
- It can make participating database operations commit together or roll back together.
- It does not act as a global lock and does not automatically prevent two independent transactions from overlapping.
- It does not automatically lock every row read by the method or prevent every stale-read/lost-update pattern.
- The annotation's default isolation is `Isolation.DEFAULT`, which delegates to the underlying transaction system. In our PostgreSQL setup, the database default is READ COMMITTED.
- PostgreSQL READ COMMITTED prevents dirty reads. That protection comes from PostgreSQL's isolation behavior, not simply from adding the Spring annotation.
- A stale application-side value and a lost update can still happen under READ COMMITTED when Java reads a balance, calculates an absolute replacement value, and writes it later.
- Spring's default propagation is `REQUIRED`: a method joins an existing transaction when appropriate, or starts one if there is no existing transaction. Therefore, it is not always accurate to say one annotated method or one thread always equals one independent physical transaction.
- In the usual imperative Spring transaction model, the transaction context is generally bound to the current thread. Work started on another thread does not automatically join the original transaction.
- By default, Spring rolls back for `RuntimeException` and `Error`; checked exceptions do not trigger rollback unless rollback rules are configured.

### The financial example I asked to preserve

Alice transfers ₹4,000 to Bob. The service has three database operations:

1. Debit Alice.
2. Credit Bob.
3. Create the payment record.

Without one shared database transaction, an exception between these operations could leave Alice debited by ₹4,000 while Bob receives ₹0 and the transfer remains incomplete. A shared transaction protects the all-or-nothing persistence of these participating database operations.

### Recall in one sentence

**`@Transactional` manages the transaction boundary and atomicity of participating work; database isolation controls visibility, while locking or another concurrency-control strategy is needed to coordinate conflicting operations.**

### Status

- [x] Identify that two independent `@Transactional` methods can overlap.
- [x] Distinguish per-transaction atomicity from application-wide serialization.
- [x] Identify that PostgreSQL READ COMMITTED, not the annotation alone, prevents dirty reads.
- [ ] Explain propagation and transaction participation without notes.
- [ ] Explain how isolation differs from atomicity.
- [ ] Explain independently why `@Transactional` does not prevent the lost-update example.

---

## 5. What I have been introduced to, but still need to explain in my own words

The following concepts were discussed during this learning session. I have not yet given a full teach-back in my own words for the remaining concepts, so these are marked as pending rather than presented as my personal understanding.

- **Race conditions:** how timing-dependent execution can cause an incorrect result.
- **Lost updates:** how one read-modify-write operation can overwrite another operation's effect.
- **Transaction isolation:** how READ COMMITTED and REPEATABLE READ provide different visibility guarantees.
- **Pessimistic locking:** how `SELECT ... FOR UPDATE` coordinates competing modifications.
- **Optimistic locking:** how a version column can detect that a row changed before an update.
- **Lock lifetime and waiting:** when PostgreSQL releases row locks, what `lock_timeout` controls, and how lock waits can be inspected.
- **Deadlocks:** how opposite-direction transfers can create circular waiting, and why deterministic lock ordering reduces avoidable deadlocks.
- **Spring transactions:** how `@Transactional` defines a transaction boundary and how rollback behavior works.
- **Concurrency testing:** how to reproduce unsafe interleavings and test financial invariants without relying on arbitrary sleeps.

I should add my own explanation to each topic after I can explain it without simply repeating the concept notes.

---

## 6. How to use this journal

For every new concept:

1. First write my explanation in my own words, even if it is incomplete or partly incorrect.
2. Preserve that original explanation.
3. Add a separate clarification after discussion.
4. Record one short recall sentence.
5. Keep unresolved questions for later revision.

The purpose is to track how my understanding improves, including the mistakes I corrected—not to make every first attempt sound perfect.
