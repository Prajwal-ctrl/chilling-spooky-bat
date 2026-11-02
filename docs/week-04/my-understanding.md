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

## 2. What I have been introduced to, but still need to explain in my own words

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

## 3. How to use this journal

For every new concept:

1. First write my explanation in my own words, even if it is incomplete or partly incorrect.
2. Preserve that original explanation.
3. Add a separate clarification after discussion.
4. Record one short recall sentence.
5. Keep unresolved questions for later revision.

The purpose is to track how my understanding improves, including the mistakes I corrected—not to make every first attempt sound perfect.
