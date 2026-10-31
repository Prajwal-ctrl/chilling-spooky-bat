# Spring Transactions

## Transaction boundary

In a Spring service, `@Transactional` commonly defines the transaction boundary around the business operation. Repository operations participating in that transaction are committed together or rolled back together.

## Important behavior

- A transfer's debit, credit, and payment record must be part of the same database transaction.
- `save()` or `saveAll()` by themselves do not guarantee that several operations are atomic; they must participate in the same transaction.
- Spring's default rollback behavior applies to unchecked exceptions and Error. Checked exceptions do not trigger rollback by default unless configured, for example with `rollbackFor`.
- Keep the transaction short, especially when holding account row locks.

## Questions to answer

1. Why should debit and credit share one transaction?
2. What happens if an unchecked exception occurs before commit?
3. What happens by default for a checked exception?
4. Why should network calls be kept outside a transaction holding row locks?

---

## My understanding (simple English)

In Spring Boot, @Transactional tells Spring that a business operation should run inside a database transaction.

For a money transfer, we should treat the debit from one account, the credit to the other account, and the payment record as one unit of work.

### Example

Alice sends ₹4,000 to Bob.

Inside one transactional service operation:

1. Debit ₹4,000 from Alice.
2. Credit ₹4,000 to Bob.
3. Save the payment record.

If an unchecked exception happens before the transaction commits, Spring rolls back the transaction by default. That means the debit, credit, and payment record should not be left partially saved, provided they all participate in the same transaction.

### Important points

- Just calling repository save() or saveAll() does not by itself make multiple operations atomic. They must participate in the same transaction.
- By default, Spring rolls back for unchecked exceptions and Error. Checked exceptions do not cause rollback by default unless we configure it, for example with rollbackFor.
- @Transactional gives us a transaction boundary, but it does not automatically solve every concurrency problem. We may still need locking or another concurrency-control strategy.
- Keep the transaction short, especially when it holds account locks.

### Easy way to remember

**A transfer should be all-or-nothing: either the debit, credit and payment record all commit, or none of them do.**
