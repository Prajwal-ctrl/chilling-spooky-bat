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
