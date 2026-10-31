# My Understanding — Week 4

This file captures my own explanations, mental models, corrections, and questions while learning. The aim is to preserve how I understood a concept at the time, not just store textbook definitions.

These notes are intentionally written in my own voice so I can revise them quickly later. Where useful, a technically precise recap is added below the original reflection.

---

## MVCC — Why a normal SELECT does not wait for an uncommitted update

**Date:** 1 November 2026

### My explanation

> Okay, so actually I forgot that it was a normal transaction that T2 is reading, not a transaction that it will commit any data. So yes, normally it will read the balance without any interruption, even if though that even is proceeding or committing any transactions, because at that time T2 will read the older data from the VCCS version control. Yeah, so that's all.

### My understanding, clarified

I initially mixed up an ordinary read with a locking read.

When T1 has updated Alice's balance but has not committed yet, T2 can execute a normal `SELECT` without waiting for T1's row lock. PostgreSQL MVCC allows T2 to read the previously committed version of the row; T2 cannot see T1's uncommitted balance.

After T1 commits, a **new SELECT statement** issued by T2 under PostgreSQL's default READ COMMITTED isolation level gets a new statement snapshot and can see the updated balance.

The important distinction is:

- Normal `SELECT`: reads a visible committed version and normally does not wait for the conflicting row lock.
- `SELECT ... FOR UPDATE`: requests a row lock and normally waits if another transaction holds a conflicting lock.
- Conflicting `UPDATE`: normally waits for the other transaction's conflicting row lock.

The correct term is **MVCC — Multi-Version Concurrency Control** (not "VCCS").

### Recall in one sentence

**MVCC lets an ordinary reader see the last committed row version while another transaction is still working; a new READ COMMITTED statement can see the writer's change after it commits.**

### Follow-up question

How does this behavior change under REPEATABLE READ, where a transaction keeps a stable snapshot?
