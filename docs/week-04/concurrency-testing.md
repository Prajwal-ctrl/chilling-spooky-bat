# Concurrency Testing

## Objective

A concurrency test should verify financial invariants under overlapping execution, not merely prove that multiple threads can run.

## Planned approach

1. Start from known account balances.
2. Launch two or more transfer operations on separate threads.
3. Use synchronization barriers/latches to control critical interleavings in the naive implementation.
4. Demonstrate the failing outcome before the fix.
5. Add pessimistic locking and repeat the test.
6. Assert minimum-balance and conservation invariants, and that debit/credit/payment recording is atomic.

## Avoid flaky tests

Do not rely on sleeps to produce a particular interleaving. Use explicit synchronization such as CountDownLatch or CyclicBarrier where deterministic coordination is required.

## Important assertion

Concurrency does not guarantee which request acquires a lock first. Unless execution order is explicitly controlled, assert invariants and valid outcomes rather than assuming a specific request must win.

## Experiments

Record initial state, interleaving, observed result, expected result, root cause, fix, and test evidence in an experiment report.

---

## My understanding (simple English)

A concurrency test checks whether our application stays correct when multiple requests happen at the same time.

It is not enough to check that two Java threads can run. We need to check that the database still contains correct balances after those threads finish.

### Example

Alice has ₹10,000. We start two transfer requests using two separate Java threads:

- T1 transfers ₹4,000 from Alice to Bob.
- T2 transfers ₹3,000 from Alice to Bob.

A useful test should check that:

1. Both requests are processed safely, or one is rejected for a valid business reason.
2. Alice's balance never goes below the minimum balance.
3. A debit is not lost because one request overwrote another.
4. If a transfer fails, its debit and credit do not get partially saved.
5. The money in the accounts is conserved, assuming there are no fees.

### Why do we use CountDownLatch or CyclicBarrier?

Sometimes we need to make two threads reach the same point before either continues. A barrier or latch helps us control that timing.

For example, in the intentionally broken implementation, we can make both threads read Alice's ₹10,000 balance before either writes its new balance. This helps us reproduce the lost-update bug reliably.

Using Thread.sleep() is not a reliable way to control the order. The test may behave differently on another machine.

### Easy way to remember

A good concurrency test should prove that our **financial rules remain correct under overlapping requests**, not just that our threads execute.
