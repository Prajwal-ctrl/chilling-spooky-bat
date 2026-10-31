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
