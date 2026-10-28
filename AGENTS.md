# AGENTS.md: Warehouse Receiving System

You are implementing a small Warehouse Receiving System. This project is **specification-driven**: the documents in `docs/` are your input. Do not invent behavior. Do not work from memory of what a "typical" warehouse app does.

## 1. Read these first (in this order)

1. `docs/prd.md`: WHAT to build and the business rules (`BR-1`..`BR-14`, `FR-*`, `US-*`).
2. `docs/technical-spec.md`: HOW to build it (architecture, DDL, triggers, API behavior, receive algorithm, lock order, tests).
3. `docs/openapi.yaml`: exact wire format (paths, schemas, status codes).
4. `docs/erd.mmd`, `docs/data-flow.mmd`: diagrams.
5. `.agent/rules/business-rules.md` and `.agent/rules/testing.md`.

`docs/prd.id.md` is an Indonesian translation. Ignore it for implementation.

**Precedence:** OpenAPI wins for the wire format. The technical spec wins for behavior. The PRD wins for business intent. If two documents conflict, or something is unspecified, **stop and ask**. Do not guess. Do not silently change a document to match your code. If a document must change, propose the edit and wait.

## 2. Stack (fixed, do not substitute)

| Layer | Choice |
|---|---|
| Backend | Node.js + Express + TypeScript (strict), in `apps/api` |
| ORM / migrations | Prisma (`prisma migrate`). Raw SQL via `$queryRaw` only for `SELECT ... FOR UPDATE` |
| Validation | Zod (`.strict()` objects) |
| Auth | JWT (HS256) + bcrypt |
| Database | PostgreSQL 16 |
| Frontend | React 18 + Vite + TypeScript, TanStack Query, React Router, in `apps/web` |
| Tests | Vitest + Supertest (API), Playwright (one E2E scenario) |
| Deploy | Docker + Docker Compose: `db`, `migrate`, `api`, `web` |

Runtime: Node 22 in Docker. The developer's machine may run Node 20.18, so pick dependency versions compatible with **both** (for example Vite 6, not Vite 7). The developer works on **Windows with PowerShell**. Therefore:

- `package.json` scripts must be cross-platform (no `rm -rf`, no bash-only syntax; use `rimraf`, `cross-env` if needed).
- Do not rely on bash in instructions for the developer. Give PowerShell-compatible commands.
- Keep LF line endings (`.gitattributes` enforces this). Shell scripts used in containers must be LF.

## 3. Repository layout

Follow `docs/technical-spec.md` section 1.3 exactly. Summary:

```
apps/api/   src/{server.ts, app.ts, config/, middleware/, lib/, modules/<name>/}  prisma/  tests/
apps/web/   src/{api/, auth/, pages/, components/, hooks/}  e2e/
docs/       prd.md, technical-spec.md, openapi.yaml, erd.mmd, data-flow.mmd, testing.md, ai-workflow.md
docker-compose.yml  .env.example  README.md
```

Backend layering: **routes -> controller (thin) -> service (all business rules, owns the transaction) -> repository/Prisma**. Controllers never contain business rules. Services throw `AppError(code, httpStatus, message, details)`. One central error handler produces the error JSON.

## 4. Commands (define these npm scripts in `apps/api`)

| Script | Purpose |
|---|---|
| `npm run dev` | Run API with reload |
| `npm run build` / `npm start` | Compile and run |
| `npm run typecheck`, `npm run lint` | Static checks |
| `npm test` | All Vitest tests (needs `DATABASE_URL_TEST`) |
| `npm run db:migrate:dev` | `prisma migrate dev` (local only) |
| `npm run db:migrate:deploy` | `prisma migrate deploy` |
| `npm run db:seed` | Idempotent seed |

Same idea in `apps/web`: `dev`, `build`, `typecheck`, `lint`, `e2e`.

## 5. Hard rules (never violate)

1. **Backend enforces every business rule**, independent of the frontend. Frontend validation is UX only.
2. **Never allow receiving more than remaining** (BR-1), and never set `COMPLETED` unless every item is fully received (BR-4). No endpoint or request field may set a document status directly.
3. **Every receipt runs in ONE database transaction**, using the lock order **document -> items (ascending id) -> inventory rows (ascending product_id)**, relative updates (`x = x + $q`), and the idempotency algorithm in `technical-spec.md` 6.2 and 6.3. Do not simplify it.
4. **`inventory_transactions` is append-only.** No update or delete code path.
5. **Do not add features outside the PRD scope** (see PRD section 3.2). No extra endpoints, tables, roles, or libraries "for convenience".
6. **No secrets in the repo.** Only `.env.example` with placeholders. Never read, print, or commit `.env`.
7. **Never weaken, skip, delete, or loosen a test** to make it pass. If a test fails, fix the code, or explain why the test is wrong and ask.
8. **No mocking the database** in tests that cover business rules. Use the real PostgreSQL test database.
9. **No `prisma migrate reset` or `db push`** against any database except a disposable local one. Migrations are immutable once committed. Fix with a new migration.
10. **Do not leak internals in errors** (no stack traces, SQL, or Prisma messages in responses).
11. Keep `docs/openapi.yaml` in sync with the code. If you change an endpoint, update the contract in the same commit and say so.

## 6. How to work

- **Plan first.** Before coding a task, state a short plan that cites the spec sections you are following (for example "technical-spec 6.2 steps, PRD BR-7"). Then implement.
- **Small steps.** One feature or layer at a time. Do not generate the whole app in one pass.
- **Tests with code.** Each feature ships with its tests from the catalog in `technical-spec.md` section 9.3. Test names must contain the rule ID (for example `BR-1`).
- **Verify before claiming done.** Run `typecheck`, `lint`, and `npm test`. Report real output. If you could not run something, say so. Never claim tests pass without running them.
- **Report at the end of each task:** what changed, which spec sections were followed, which tests were added, anything ambiguous, and any deviation. The developer reviews your work and must be able to explain it.
- **Commits:** do not commit unless asked. Suggest a Conventional Commit message (`feat(api): ...`, `test(api): ...`, `docs: ...`, `chore: ...`, `fix: ...`).
- **Ambiguity:** ask one specific question and wait, rather than choosing silently.

## 7. Definition of done (per feature)

1. Implemented per spec, layered correctly.
2. Business rules enforced in the service, and in the database where the spec says so.
3. Tests from the catalog added and passing against real PostgreSQL.
4. `typecheck` and `lint` clean.
5. `openapi.yaml` still matches.
6. No unrelated changes. No secrets. No scope creep.
