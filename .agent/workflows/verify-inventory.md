---
description: Run the inventory integrity, idempotency and concurrency checks and produce evidence for docs/testing.md.
---

# /verify-inventory

Goal: prove that stock stays correct after receiving, and that duplicate or invalid operations do not corrupt it (assessment deliverable E).

1. Make sure the test database is up and migrated (`DATABASE_URL_TEST`, name ends with `_test`).
2. Run only the relevant suites and capture the real output:
   - tests named `BR-1`, `BR-4`, `BR-7`, `BR-10`, `BR-11`, `BR-14`
   - tests `T-BR1-c`, `T-BR4-c`, `T-INV-a` (database-level proofs)
3. Run `assertInventoryInvariants()` against the state left by the scenario script below. If no script exists yet, create `apps/api/tests/scenarios/inventory-evidence.test.ts` that executes these scenarios and prints a before/action/after table:

   | # | Scenario | Expected |
   |---|---|---|
   | 1 | Partial receipt 40 of 100 | item received 40, on_hand +40, status `PARTIALLY_RECEIVED`, 1 ledger row |
   | 2 | Remaining 60 | received 100, status `COMPLETED`, on_hand +100 total |
   | 3 | Over-receipt (remaining + 1) | `422`, nothing changed |
   | 4 | Two-line receipt, second line invalid | `422`, first line NOT applied |
   | 5 | Same idempotency key twice | second is `200` replay, stock changed once |
   | 6 | Same key, different payload | `409`, nothing changed |
   | 7 | 5 parallel receipts of 4 on expected 10, distinct keys | exactly 2 succeed, 3 `422`, received 8 |
   | 8 | 10 parallel requests with one key | one `201`, nine `200` replays, one receipt row |
   | 9 | 20 parallel receipts across 3 documents sharing 2 products | no deadlock, invariants hold |
   | 10 | Receipt on COMPLETED / CANCELLED / DRAFT | `409`, nothing changed |

4. Report a table of results with real numbers (before and after values), plus the test summary line. Do not invent numbers. If a scenario fails, show the failure and stop.
5. Append the evidence (command, output, table) to `docs/testing.md` under "Inventory integrity evidence". Do not edit other sections.
6. List known limitations discovered (see `technical-spec.md` section 15).
