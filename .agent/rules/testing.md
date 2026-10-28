# Testing Rules (always apply)

Source: `docs/technical-spec.md` section 9. The catalog in 9.3 (T-BR1-a ... T-E2E-a) is the required test list.

1. **Real PostgreSQL for business-rule tests.** No mocked Prisma, no in-memory fakes. Use `DATABASE_URL_TEST`. **Refuse to run** if the database name does not end with `_test`.
2. **Setup.** Vitest `globalSetup` runs `prisma migrate deploy` on the test DB. Before each test file, `TRUNCATE ... RESTART IDENTITY CASCADE` all tables, then recreate the two users via factories. TRUNCATE is allowed even with the append-only trigger.
3. **Serial files.** `fileParallelism: false`. Concurrency is created deliberately inside a test with `Promise.all`.
4. **Naming.** Every test title contains the rule or test ID, for example `BR-1: rejects receipt above remaining`, `T-BR11-a ...`.
5. **Invariants.** Provide `assertInventoryInvariants()` (technical-spec 9.4) and call it at the end of every receiving test and every concurrency test.
6. **Layers.** Unit tests for pure domain helpers (`computeDocumentStatus`, `computeRemaining`, `hashReceiptRequest`, `normalizeSku`, transition table). Integration tests through `createApp()` with Supertest for every endpoint (success, validation 400, 401, 403, 404, 409, 422). Contract test against `docs/openapi.yaml`. One Playwright E2E scenario.
7. **Database-level proofs.** Include tests that bypass the service with raw SQL to prove CHECKs and triggers fire (T-BR1-c, T-BR4-c, T-INV-a).
8. **Table-driven authorization.** One test iterates the full permission matrix (technical-spec 8.2) for anonymous, OPERATOR, SUPERVISOR.
9. **Never** skip (`it.skip`, `xit`), delete, comment out, or loosen an assertion to get green. If a test seems wrong, explain and ask.
10. **Report honestly.** After running tests, paste the real summary (passed/failed/skipped counts). If a command could not run, say so.
11. Keep a list of anything not tested or flaky for `docs/testing.md` (known limitations).
