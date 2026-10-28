---
description: Implement one feature strictly from the PRD and technical spec, with tests, then report.
---

# /implement-feature

Use: `/implement-feature <feature name or spec section>` (example: `products module`, `receive service`).

1. **Locate the spec.** Read the relevant parts of `docs/prd.md` (FR, BR, US), `docs/technical-spec.md` (sections for this feature), and `docs/openapi.yaml` (paths and schemas). List the IDs you will satisfy.
2. **Check for gaps.** If anything is ambiguous or conflicting, stop and ask one specific question.
3. **Plan.** Write a short plan: files to create or change, DB changes, endpoints, tests (by catalog ID). Cite spec sections. Wait for approval if the plan touches the database schema or the receive algorithm.
4. **Implement** in layers: schema/migration (if any), domain helpers, repository, service, controller/routes, Zod schemas. Follow `AGENTS.md` hard rules and `.agent/rules/business-rules.md`.
5. **Write tests** from the catalog in `technical-spec.md` 9.3 (names contain the rule ID). Add `assertInventoryInvariants()` where stock is touched.
6. **Run** `npm run typecheck`, `npm run lint`, `npm test` in the affected app. Fix failures in the code, not by weakening tests.
7. **Sync the contract.** If any request/response/status code changed, update `docs/openapi.yaml`.
8. **Report** in this format:
   - **Done:** what was implemented
   - **Spec followed:** section and rule IDs
   - **Tests:** names added plus the real result line
   - **Deviations / assumptions:** none, or list
   - **Open questions:** none, or list
   - **Suggested commit message:** Conventional Commit
9. **Do not commit.** The developer reviews the diff and commits.
