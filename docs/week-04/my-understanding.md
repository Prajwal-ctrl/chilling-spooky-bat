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

## 2. What I have been introduced to, but still need to explain in my own words

The following concepts were discussed during this learning session. I have not yet given a full teach-back in my own words for each one, so these are marked as pending rather than presented as my personal understanding.

- **Concurrency and parallelism:** how multiple operations overlap and how that differs from executing at the same instant.
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
