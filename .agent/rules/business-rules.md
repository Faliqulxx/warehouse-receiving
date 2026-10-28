# Business Rules (always apply)

Source of truth: `docs/prd.md` section 6 and `docs/technical-spec.md` sections 6 to 8. This file is a checklist. Do not weaken any rule.

## Rules

| ID | Rule |
|---|---|
| BR-1 | `received_quantity` never exceeds `expected_quantity`. A line with `quantity > remaining` is rejected (`422 OVER_RECEIPT`) and nothing is applied. |
| BR-2 | Receipts only on `OPEN` or `PARTIALLY_RECEIVED` (otherwise `409 DOCUMENT_NOT_RECEIVABLE`). |
| BR-3 | Quantities are integers from 1 to 1,000,000. Reject 0, negatives, fractions, strings. |
| BR-4 | After a receipt: all items full -> `COMPLETED`, otherwise `PARTIALLY_RECEIVED`. No manual complete. No endpoint sets status. |
| BR-5 | One product once per document (`422 DUPLICATE_PRODUCT_IN_DOCUMENT`). |
| BR-6 | Items and header editable only in `DRAFT` (`409 DOCUMENT_NOT_EDITABLE`). |
| BR-7 | `Idempotency-Key` required on receive. Same key + same payload + same document = replay (`200`, `Idempotent-Replay: true`, no changes). Different payload/document = `409 IDEMPOTENCY_KEY_REUSED`. |
| BR-8 | Cancel only `DRAFT`, or `OPEN` with zero receipts. Reason required. |
| BR-9 | `COMPLETED` and `CANCELLED` are terminal. |
| BR-10 | Per product: `on_hand = SUM(ledger.quantity)`, never negative; each `balance_after` equals the running total. |
| BR-11 | Concurrent receipts must never exceed expected. One wins, the other gets `OVER_RECEIPT`. |
| BR-12 | SKU unique, stored uppercase, immutable. |
| BR-13 | Cannot open a document without items. |
| BR-14 | A receipt is all-or-nothing across its lines. |

## Receive transaction checklist (technical-spec 6.2 / 6.3)

1. Compute payload hash (lines sorted by `itemId`).
2. Pre-check idempotency key (replay or `409`).
3. `BEGIN` (READ COMMITTED).
4. Lock document `FOR UPDATE`. `404` if missing.
5. Re-check idempotency key after the lock.
6. Check document status.
7. Lock items `WHERE id = ANY(...) ORDER BY id FOR UPDATE`. Verify membership and duplicates.
8. Validate ALL lines against `expected - received`. Collect every violation. Write nothing yet.
9. Lock inventory rows `ORDER BY product_id FOR UPDATE`.
10. Insert receipt (key + hash) and lines. Catch unique violation on `idempotency_key` -> rollback -> `409 IDEMPOTENCY_KEY_REUSED` (or replay if same).
11. Per line: `received_quantity = received_quantity + q`; `on_hand = on_hand + q RETURNING on_hand`; insert ledger row with `balance_after`.
12. Count items with remaining > 0. Update document status. **Update items before the document.**
13. `COMMIT`. Build the response from fresh reads.

**Lock order is global: document -> items (id ASC) -> inventory (product_id ASC).** Any other code that locks several of these must use the same order.

## Check order for any request

401 authentication -> 403 role -> 400 input validation -> 404 existence -> 409 state/idempotency -> 422 business rules.

## Database second line of defense (must exist in migrations)

- CHECKs: `0 <= received <= expected`, `expected BETWEEN 1 AND 1000000`, `on_hand >= 0`, `quantity` ranges, SKU format, cancel reason required iff `CANCELLED`.
- Unique: `sku`, `(document_id, product_id)`, `idempotency_key`, `(receipt_id, item_id)`, `receipt_line_id` on ledger.
- Trigger: ledger append-only. Trigger: `COMPLETED` only when no item has remaining quantity.
