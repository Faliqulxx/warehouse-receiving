# Product Requirements Document: Warehouse Receiving System

| | |
|---|---|
| **Version** | 1.0 |
| **Status** | Approved for implementation |
| **Audience** | Developers and AI coding agents (Antigravity). Requirements are written to be implemented without guessing business behavior. |
| **Related docs** | `technical-spec.md`, `erd.mmd`, `openapi.yaml`, `data-flow.mmd`, `testing.md`, `ai-workflow.md` |

**Conventions.** The keywords MUST, MUST NOT, SHOULD, and MAY follow RFC 2119. Requirement IDs are stable and referenced by the technical spec and the tests: `FR-*` functional, `NFR-*` non-functional, `BR-*` business rule, `US-*` user story, `A-*` assumption, `Q-*` open question.

---

## 1. Problem Statement and Business Objectives

### 1.1 Problem

The warehouse records incoming inventory in spreadsheets. This causes:

- Over-receiving that goes unnoticed, because nothing blocks a typed quantity above what was ordered.
- No reliable record of who received what, and when.
- Stock figures that drift from reality because several people edit the same file.
- No way to tell which deliveries are open, partial, or finished without reading every row.

### 1.2 Objectives

| ID | Objective | Measure of success |
|---|---|---|
| O-1 | Make it impossible to receive more than was expected. | 0 receipts can exceed the remaining quantity, including under concurrent or duplicate requests. |
| O-2 | Keep stock quantities always correct after receiving. | For every product, `on_hand` equals the sum of its transaction quantities. |
| O-3 | Give a trustworthy audit trail. | Every stock change has a transaction row with user, time, and source document. |
| O-4 | Show delivery progress at a glance. | Each document shows status and per-item expected/received/remaining quantities. |
| O-5 | Be simple enough for operators to use during a delivery. | Recording a receipt takes one screen and no more than 3 interactions per item. |

---

## 2. Users and Roles

| Role | Description | Responsibilities |
|---|---|---|
| **OPERATOR** | Warehouse floor staff who unload and count deliveries. | Create draft receiving documents, edit drafts, record received quantities, view products, documents, inventory, and transactions. |
| **SUPERVISOR** | Warehouse lead who owns master data and approves work. | Everything an operator can do, plus: create products, release (open) documents, cancel documents. |

### 2.1 Permission Matrix

| Action | OPERATOR | SUPERVISOR |
|---|:---:|:---:|
| Log in / view own profile | Yes | Yes |
| List and view products | Yes | Yes |
| Create product | No | Yes |
| List and view receiving documents | Yes | Yes |
| Create receiving document (`DRAFT`) | Yes | Yes |
| Edit `DRAFT` document (header and items) | Yes | Yes |
| Open (release) document: `DRAFT` to `OPEN` | No | Yes |
| Cancel document | No | Yes |
| Record receipt | Yes | Yes |
| View inventory | Yes | Yes |
| View inventory transactions | Yes | Yes |

The backend MUST enforce this matrix on every request. The frontend hides forbidden actions as a convenience only.

---

## 3. Scope

### 3.1 In Scope

1. Authentication with two roles (JWT).
2. Product management: list, search, create. SKU codes.
3. Receiving documents: create, list, filter by status, view detail with items, edit while `DRAFT`.
4. Document lifecycle: `DRAFT`, `OPEN`, `PARTIALLY_RECEIVED`, `COMPLETED`, `CANCELLED`.
5. Recording partial and full receipts against document items.
6. Automatic inventory update and append-only transaction history.
7. Inventory list and transaction history list with filtering and pagination.
8. Backend enforcement of every business rule, plus database constraints as a second layer.
9. Idempotent receive operation.
10. Health and readiness endpoints, Docker deployment, migrations, seed users.

### 3.2 Out of Scope

| Item | Reason |
|---|---|
| Editing, reversing, or deleting receipts (corrections) | Keeps the ledger append-only. Listed as a known limitation. |
| Closing a short-shipped document before it is fully received | See Q-1. |
| Multiple warehouses, locations, bins | Single warehouse assumed (A-13). |
| Batch/lot numbers, expiry dates, serial numbers | Not required by the scenario. |
| Suppliers master data and purchase orders | Supplier is a free-text field on the document. |
| Outbound shipments, transfers, stock adjustments, returns | Receiving only. |
| Editing or deactivating products after creation | Not required. SKU must stay immutable anyway. |
| User management UI, registration, password reset | Users are created by seed script. |
| Barcode scanning, printing, email notifications | Not required. |
| Reporting and analytics, CSV import/export | Not required. |
| Multi-language UI | UI is English only. |

---

## 4. Glossary

| Term | Meaning |
|---|---|
| **SKU** | Stock Keeping Unit. Unique code identifying a product. |
| **Receiving document** | An inbound document listing products and expected quantities for one delivery. |
| **Document item (line)** | One product with an expected quantity inside a document. |
| **Receipt** | One recording event: a set of received quantities for items of one document, submitted in one request. |
| **Remaining quantity** | `expected_quantity - received_quantity` for a document item. |
| **On hand** | Current available stock for a product. |
| **Inventory transaction** | One immutable ledger row describing a change in stock. |
| **Idempotency key** | Client-generated unique value that makes a retried receive request safe. |

---

## 5. Functional Requirements

### 5.1 Authentication and Authorization

| ID | Requirement |
|---|---|
| FR-AUTH-1 | The system MUST authenticate users by username and password via `POST /api/v1/auth/login` and return a signed JWT and the user profile (`id`, `username`, `fullName`, `role`). |
| FR-AUTH-2 | All endpoints except login and health/readiness MUST require a valid `Authorization: Bearer <token>` header. |
| FR-AUTH-3 | Tokens MUST expire after a configurable duration (default 8 hours). Expired or malformed tokens return `401`. |
| FR-AUTH-4 | Passwords MUST be stored only as bcrypt hashes. |
| FR-AUTH-5 | The role MUST be read from the verified token and re-checked against the database user (user must exist and be active). |
| FR-AUTH-6 | A user calling an action not permitted for their role MUST receive `403`. |
| FR-AUTH-7 | Two users MUST be created by the seed script: one `OPERATOR` and one `SUPERVISOR`. Usernames and passwords come from environment variables. No credentials are committed. |
| FR-AUTH-8 | `GET /api/v1/auth/me` returns the current user. |

### 5.2 Products

| ID | Requirement |
|---|---|
| FR-PRD-1 | The system MUST list products with pagination and a search term matching SKU or name (case-insensitive, partial). |
| FR-PRD-2 | A supervisor MUST be able to create a product with: `sku` (required), `name` (required), `unit` (optional, default `PCS`), `description` (optional). |
| FR-PRD-3 | SKU MUST be unique (case-insensitive). The input is trimmed and stored uppercase. |
| FR-PRD-4 | SKU format: 2 to 32 characters, `A-Z`, `0-9`, and hyphen, starting with a letter or digit. |
| FR-PRD-5 | A SKU MUST NOT be changeable after creation. No product update endpoint exists. |
| FR-PRD-6 | Creating a product MUST also create its inventory balance row with `on_hand = 0`, in the same transaction. |
| FR-PRD-7 | Product detail MUST be retrievable by id. |

### 5.3 Receiving Documents

| ID | Requirement |
|---|---|
| FR-DOC-1 | Operators and supervisors MUST be able to create a receiving document with: `supplierName` (optional, max 120), `expectedDate` (optional, date), `notes` (optional, max 500), and 1 to 100 items, each `{productId, expectedQuantity}`. |
| FR-DOC-2 | The system MUST generate a unique, human-readable document number `RCV-` followed by a 6-digit zero-padded sequence (for example `RCV-000042`). Gaps are allowed. The client cannot set it. |
| FR-DOC-3 | A new document MUST start in status `DRAFT`. |
| FR-DOC-4 | The document list MUST support pagination, filtering by status, and search by document number or supplier name. Default sort: newest first. |
| FR-DOC-5 | Document detail MUST return header fields, status, creator, timestamps, and all items with `expectedQuantity`, `receivedQuantity`, `remainingQuantity`. It MUST also return the receipt history (see FR-REC-8). |
| FR-DOC-6 | While `DRAFT`, header fields and the full item list MAY be replaced via `PUT`. In every other status, editing is rejected. |
| FR-DOC-7 | A supervisor MUST be able to open a `DRAFT` document (`DRAFT` to `OPEN`). The document MUST have at least one item. |
| FR-DOC-8 | A supervisor MUST be able to cancel a document in `DRAFT`, or in `OPEN` with no receipts. A cancellation reason (1 to 255 chars) is required. |
| FR-DOC-9 | There is no endpoint to complete a document manually. `COMPLETED` is set only by the system (BR-4). |
| FR-DOC-10 | Documents MUST NOT be deleted. |

### 5.4 Receiving (Recording Receipts)

| ID | Requirement |
|---|---|
| FR-REC-1 | Operators and supervisors MUST be able to record a receipt on a document in status `OPEN` or `PARTIALLY_RECEIVED` via `POST /api/v1/receiving-documents/{id}/receipts`. |
| FR-REC-2 | A receipt request contains `lines` (1 to 100): each `{itemId, quantity}`, plus optional `note` (max 255). |
| FR-REC-3 | `quantity` MUST be an integer from 1 to the item's remaining quantity (BR-1). |
| FR-REC-4 | Each `itemId` MUST belong to the document and MUST appear at most once per request. |
| FR-REC-5 | The request MUST include an `Idempotency-Key` header (a UUID, 8 to 64 characters). Requests without it are rejected with `400`. See BR-7. |
| FR-REC-6 | The operation MUST be atomic: if any line is invalid, no line is applied and no stock changes. |
| FR-REC-7 | On success, within the same database transaction, the system MUST: (a) create the receipt and its lines, (b) increase each item's `received_quantity`, (c) increase each product's `on_hand`, (d) append one inventory transaction per line, (e) recompute and update the document status. |
| FR-REC-8 | The response MUST include the receipt, the updated items, the updated document status, and the new `on_hand` for each affected product. The document detail MUST list previous receipts (receipt number, user, time, lines). |

### 5.5 Inventory and Transactions

| ID | Requirement |
|---|---|
| FR-INV-1 | The system MUST list inventory: one row per product with `sku`, `name`, `unit`, `onHand`, `updatedAt`. Products with zero stock MUST be included. Supports search and pagination. |
| FR-INV-2 | The system MUST list inventory transactions, newest first, with pagination. |
| FR-INV-3 | Each transaction MUST show: id, timestamp, product (SKU and name), type, quantity change, balance after, source document number, receipt id, performing user. |
| FR-INV-4 | Transactions MUST be filterable by product, by document, and by date range. |
| FR-INV-5 | Transactions are append-only. No API or code path updates or deletes them. |
| FR-INV-6 | The only transaction type in this release is `RECEIPT`. The type is modeled as an enum so other types can be added later. |

### 5.6 System

| ID | Requirement |
|---|---|
| FR-SYS-1 | `GET /health` returns `200` when the process is alive (no database check). |
| FR-SYS-2 | `GET /ready` returns `200` only when the database is reachable and migrations are applied; otherwise `503`. |
| FR-SYS-3 | Migrations MUST run through a single documented command and be safe to run repeatedly. |
| FR-SYS-4 | A seed command MUST create the two users from environment variables, and MAY create sample products when explicitly enabled. |

### 5.7 Frontend

| ID | Requirement |
|---|---|
| FR-UI-1 | Login page. After login, the token is kept for the session and attached to API calls. Expired token redirects to login. |
| FR-UI-2 | Products page: searchable table; "New product" form visible to supervisors only. |
| FR-UI-3 | Receiving documents page: table with status badge, filter by status, search, "New document" action. |
| FR-UI-4 | Create/edit document page: supplier, expected date, notes, add/remove item rows using a product picker, client-side validation mirroring server rules. |
| FR-UI-5 | Document detail page: header, status badge, items table with expected/received/remaining and a progress indicator, receipt history, and role-appropriate action buttons (Open, Cancel, Record receipt). |
| FR-UI-6 | Receive form: one numeric input per item with remaining quantity as the maximum. Submission disables the button while in flight, and a client-generated idempotency key is reused on retry of the same submission. |
| FR-UI-7 | Inventory page and transactions page with filters and pagination. |
| FR-UI-8 | Server errors MUST be shown to the user in readable form using the API error `message` and field errors. The UI MUST NOT assume its own validation is sufficient. |

---

## 6. Business Rules and Validation

### 6.1 Business Rules

| ID | Rule |
|---|---|
| **BR-1** | **No over-receiving.** For every item, `received_quantity` MUST never exceed `expected_quantity`. A receipt line with `quantity > remaining` is rejected and nothing is applied. |
| **BR-2** | **Only active documents receive.** Receipts are accepted only when the status is `OPEN` or `PARTIALLY_RECEIVED`. |
| **BR-3** | **Quantities are positive integers.** Expected and received quantities are whole numbers of at least 1 (maximum 1,000,000 per line). Zero, negative, and fractional values are rejected. |
| **BR-4** | **Completion is automatic and strict.** After applying a receipt, if every item has `received_quantity = expected_quantity`, the status becomes `COMPLETED`. Otherwise it becomes `PARTIALLY_RECEIVED`. A document can never be `COMPLETED` while any item has remaining quantity above zero. |
| **BR-5** | **Unique product per document.** A product may appear only once in one document. |
| **BR-6** | **Immutability after release.** Items and expected quantities are editable only in `DRAFT`. After release, they are fixed. |
| **BR-7** | **Idempotent receiving.** A receipt request is identified by its `Idempotency-Key`. Same key and same payload returns the original response (HTTP `200`, header `Idempotent-Replay: true`) and changes nothing. Same key with a different payload or a different document returns `409 IDEMPOTENCY_KEY_REUSED`. Keys are globally unique. |
| **BR-8** | **Cancellation limits.** Only `DRAFT` and `OPEN` documents with zero receipts can be cancelled. `PARTIALLY_RECEIVED`, `COMPLETED`, and `CANCELLED` documents cannot. |
| **BR-9** | **Terminal states.** `COMPLETED` and `CANCELLED` allow no further changes. |
| **BR-10** | **Stock integrity.** For each product, `on_hand` equals the sum of its transaction quantities, and is never negative. Each transaction's `balance_after` equals the previous balance plus its quantity. |
| **BR-11** | **Concurrency safety.** Two simultaneous receipts for the same item must never exceed the expected quantity. One succeeds, and the other receives a business-rule error if the combined total would exceed it. |
| **BR-12** | **SKU uniqueness.** SKUs are unique case-insensitively and immutable. |
| **BR-13** | **Open requires items.** A document without items cannot be opened. |
| **BR-14** | **Atomicity.** A receipt is all-or-nothing across all its lines. |

### 6.2 Valid Status Transitions

| From | To | Trigger | Actor | Conditions |
|---|---|---|---|---|
| `DRAFT` | `OPEN` | Open action | Supervisor | At least 1 item (BR-13) |
| `DRAFT` | `CANCELLED` | Cancel action | Supervisor | Reason provided |
| `OPEN` | `PARTIALLY_RECEIVED` | Receipt applied | System | Some item still has remaining quantity |
| `OPEN` | `COMPLETED` | Receipt applied | System | All items fully received |
| `OPEN` | `CANCELLED` | Cancel action | Supervisor | Zero receipts, reason provided |
| `PARTIALLY_RECEIVED` | `PARTIALLY_RECEIVED` | Receipt applied | System | Some item still has remaining quantity |
| `PARTIALLY_RECEIVED` | `COMPLETED` | Receipt applied | System | All items fully received |

Every other transition is invalid and returns `409 INVALID_STATE_TRANSITION` (or `409 DOCUMENT_NOT_RECEIVABLE` for receipts).

### 6.3 Field Validation Summary

| Field | Rule |
|---|---|
| `username` | Required, 3 to 50 chars. |
| `password` | Required, 8 to 128 chars on login input. |
| `sku` | Required, trimmed, uppercased, regex `^[A-Z0-9][A-Z0-9-]{1,31}$`. |
| `product.name` | Required, trimmed, 1 to 120 chars. |
| `product.unit` | Optional, 1 to 16 chars, default `PCS`. |
| `product.description` | Optional, max 500 chars. |
| `supplierName` | Optional, max 120 chars. |
| `expectedDate` | Optional, ISO date `YYYY-MM-DD`. A past date is allowed. |
| `notes` | Optional, max 500 chars. |
| `items` | 1 to 100 entries, unique `productId`. |
| `expectedQuantity` | Integer, 1 to 1,000,000. |
| `quantity` (receipt line) | Integer, 1 to remaining quantity. |
| `Idempotency-Key` | Required on receive. 8 to 64 chars, `A-Za-z0-9-_`. |
| `cancel reason` | Required, 1 to 255 chars. |
| Pagination | `page` >= 1 (default 1), `pageSize` 1 to 100 (default 20). |

### 6.4 Error Handling Requirements

All errors use one JSON shape:

```json
{
  "error": {
    "code": "OVER_RECEIPT",
    "message": "Quantity 12 exceeds remaining quantity 10 for item 7.",
    "details": [{ "field": "lines[0].quantity", "issue": "exceeds_remaining", "remaining": 10 }],
    "requestId": "b1f1c7a2-..."
  }
}
```

| HTTP | Meaning | Example codes |
|---|---|---|
| 400 | Malformed or schema-invalid input | `VALIDATION_ERROR`, `IDEMPOTENCY_KEY_REQUIRED` |
| 401 | Missing, expired, or invalid token / bad credentials | `UNAUTHENTICATED`, `INVALID_CREDENTIALS` |
| 403 | Role not allowed | `FORBIDDEN` |
| 404 | Resource not found | `NOT_FOUND` |
| 409 | State or uniqueness conflict | `SKU_ALREADY_EXISTS`, `INVALID_STATE_TRANSITION`, `DOCUMENT_NOT_RECEIVABLE`, `DOCUMENT_NOT_EDITABLE`, `IDEMPOTENCY_KEY_REUSED` |
| 422 | Business rule violated | `OVER_RECEIPT`, `DUPLICATE_PRODUCT_IN_DOCUMENT`, `DOCUMENT_HAS_NO_ITEMS`, `DUPLICATE_ITEM_IN_RECEIPT` |
| 500 | Unexpected error (no internals leaked) | `INTERNAL_ERROR` |
| 503 | Not ready | `NOT_READY` |

The technical specification is the authority for the full code list.

---

## 7. User Stories and Acceptance Criteria

### US-01: Log in

*As a warehouse user, I want to log in so that I can use the system with the permissions of my role.*

- **Given** valid credentials, **when** I log in, **then** I receive a token and my role and land on the documents page.
- **Given** wrong credentials, **when** I log in, **then** I see "Invalid username or password" and the API returns `401 INVALID_CREDENTIALS` without revealing which part was wrong.
- **Given** an expired token, **when** I call any API, **then** I get `401` and the UI sends me to login.

### US-02: Manage products

*As a supervisor, I want to register products with SKUs so that they can be received.*

- **Given** I am a supervisor, **when** I submit SKU `abc-100` and name "Widget", **then** the product is saved with SKU `ABC-100`, and an inventory row with `on_hand = 0` exists.
- **Given** SKU `ABC-100` exists, **when** I create `abc-100`, **then** I get `409 SKU_ALREADY_EXISTS`.
- **Given** SKU `bad sku!`, **when** I submit, **then** I get `400 VALIDATION_ERROR` naming the `sku` field.
- **Given** I am an operator, **when** I call create product, **then** I get `403`, and the UI does not show the form.
- **Given** 30 products, **when** I search "widget", **then** only matching SKU or name rows are returned, paginated.

### US-03: Create a receiving document

*As an operator, I want to register an expected delivery with its products and quantities.*

- **Given** valid items, **when** I create a document, **then** it gets a number like `RCV-000001`, status `DRAFT`, and `receivedQuantity = 0` on all items.
- **Given** the same product twice, **when** I submit, **then** I get `422 DUPLICATE_PRODUCT_IN_DOCUMENT`, and nothing is saved.
- **Given** `expectedQuantity` of 0, -3, or 2.5, **when** I submit, **then** I get `400 VALIDATION_ERROR`.
- **Given** a non-existent `productId`, **when** I submit, **then** I get `404 NOT_FOUND` (or `422`, per the spec) and nothing is saved.
- **Given** an empty item list, **when** I submit, **then** I get `400 VALIDATION_ERROR`.

### US-04: Edit a draft

- **Given** a `DRAFT` document, **when** I replace its items and header, **then** the changes persist.
- **Given** a document in any other status, **when** I try to edit it, **then** I get `409 DOCUMENT_NOT_EDITABLE`.

### US-05: Release a document

*As a supervisor, I want to open a draft so operators can start receiving against it.*

- **Given** a `DRAFT` with items, **when** I open it, **then** the status is `OPEN`.
- **Given** I am an operator, **when** I open it, **then** I get `403`.
- **Given** a document that is not `DRAFT`, **when** I open it, **then** I get `409 INVALID_STATE_TRANSITION`.

### US-06: Cancel a document

- **Given** a `DRAFT` or an `OPEN` document with no receipts, **when** a supervisor cancels with a reason, **then** the status is `CANCELLED` and the reason is stored.
- **Given** a `PARTIALLY_RECEIVED` or `COMPLETED` document, **when** I cancel, **then** I get `409 INVALID_STATE_TRANSITION`.
- **Given** no reason, **when** I cancel, **then** I get `400 VALIDATION_ERROR`.

### US-07: Record a partial receipt

*As an operator, I want to record the quantity that actually arrived.*

- **Given** an `OPEN` document with item A expected 100, **when** I receive 40 of A, **then** item A shows received 40 and remaining 60, the status is `PARTIALLY_RECEIVED`, product A `on_hand` increases by 40, and one `RECEIPT` transaction of +40 is created with `balance_after` correct.
- **Given** that state, **when** I receive 60 more of A (and A is the only item), **then** the status becomes `COMPLETED`, and `on_hand` has increased by 100 in total.

### US-08: Prevent over-receiving

- **Given** item A with remaining 60, **when** I receive 61, **then** I get `422 OVER_RECEIPT`, and received quantity, `on_hand`, transactions, and status are unchanged.
- **Given** a receipt with two lines where line 1 is valid and line 2 exceeds remaining, **when** I submit, **then** neither line is applied (atomicity).
- **Given** quantity 0, -5, or 1.5, **when** I submit, **then** I get `400 VALIDATION_ERROR`.

### US-09: Completion rules

- **Given** a document with items A (expected 10) and B (expected 5), **when** only A is received fully, **then** the status is `PARTIALLY_RECEIVED`, not `COMPLETED`.
- **Given** the same document, **when** B is then received fully, **then** the status becomes `COMPLETED` automatically.
- **Given** a `COMPLETED` or `CANCELLED` or `DRAFT` document, **when** I record a receipt, **then** I get `409 DOCUMENT_NOT_RECEIVABLE` and no stock changes.
- **Given** any API call, **when** I try to set a status directly, **then** no such endpoint exists.

### US-10: Safe retries and duplicate submissions

- **Given** a receipt succeeded with key K, **when** the same request with key K is sent again (double click, network retry), **then** I get the original result with `Idempotent-Replay: true`, and stock is unchanged by the retry.
- **Given** key K was used with different lines, **when** I send it again, **then** I get `409 IDEMPOTENCY_KEY_REUSED`.
- **Given** no `Idempotency-Key`, **when** I submit, **then** I get `400 IDEMPOTENCY_KEY_REQUIRED`.
- **Given** two simultaneous requests with different keys that together exceed remaining quantity, **when** both are processed, **then** exactly one succeeds, the other gets `422 OVER_RECEIPT`, and the final received quantity does not exceed expected.
- **Given** two simultaneous requests with the same key, **when** both are processed, **then** only one receipt is created.

### US-11: View document detail

- **Given** a document, **when** I open its detail page, **then** I see header, status, items with expected/received/remaining, and the receipt history with user and time.

### US-12: View and filter the document list

- **Given** documents in various statuses, **when** I filter by `PARTIALLY_RECEIVED`, **then** only those are shown. Search by number or supplier works. Results are paginated, newest first.

### US-13: View inventory

- **Given** products with receipts, **when** I open inventory, **then** each product shows its current `on_hand`, including products at 0.
- **Given** the receipts in US-07, **when** I open inventory, **then** `on_hand` equals the sum of the received quantities for that product across all documents.

### US-14: View transaction history

- **Given** receipts have been recorded, **when** I open transactions, **then** I see each stock change with product, +quantity, balance after, document number, user, and time, newest first.
- **Given** a product filter, a document filter, or a date range, **when** I apply it, **then** only matching rows are shown.
- **Given** any user role, **when** I look for a way to edit or delete a transaction, **then** none exists.

### US-15: Role enforcement

- **Given** an operator token, **when** calling product creation, open, or cancel endpoints directly (bypassing the UI), **then** each returns `403`.
- **Given** no token, **when** calling any protected endpoint, **then** it returns `401`.

### US-16: Operability

- **Given** a clean machine with Docker, **when** I follow the README, **then** the stack starts, migrations apply, seed users exist, `/health` and `/ready` return `200`, and I can log in.

---

## 8. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-1 | Data integrity | All multi-row writes (receipt, document create/edit, product create) MUST run in a single database transaction. |
| NFR-2 | Data integrity | The database MUST enforce: `expected_quantity > 0`, `received_quantity >= 0`, `received_quantity <= expected_quantity`, `on_hand >= 0`, unique SKU (case-insensitive), unique `(document_id, product_id)`, unique idempotency key, and valid foreign keys. |
| NFR-3 | Concurrency | Receiving MUST lock the document and affected item rows (`SELECT ... FOR UPDATE`) in a consistent order to prevent lost updates and deadlocks. |
| NFR-4 | Security | Passwords hashed with bcrypt (cost 10 or higher). JWT secret from environment, minimum 32 characters. Inputs validated with schemas. Queries parameterized (ORM). Security headers via `helmet`. CORS limited to configured origins. Login rate limited. |
| NFR-5 | Security | No secrets in the repository. `.env.example` lists variables with placeholder values only. Error responses never leak stack traces or SQL. |
| NFR-6 | Performance | With 10,000 documents and 100,000 transactions, list endpoints respond in under 300 ms at p95 on a developer laptop. A receipt of 20 lines completes in under 500 ms. Lists MUST be paginated and backed by indexes. |
| NFR-7 | Reliability | The API starts only after migrations succeed. Graceful shutdown closes database connections. |
| NFR-8 | Observability | Structured request logs with a request ID returned in `X-Request-Id` and in error bodies. Receipts are logged with document id and user id. |
| NFR-9 | Portability | Entire stack starts with `docker compose up`. All configuration via environment variables. |
| NFR-10 | Maintainability | TypeScript strict mode. Layers separated: routes, services (business rules), repositories/Prisma. Business rules live only in services and are not duplicated inconsistently in handlers. |
| NFR-11 | Testability | Business rules covered by automated tests that run against a real PostgreSQL instance (not mocks) for the receiving workflow. |
| NFR-12 | Time | All timestamps stored in UTC (`timestamptz`) and returned in ISO 8601. |
| NFR-13 | Usability | Responsive layout usable on tablet and desktop. Loading, empty, and error states on every page. Keyboard-operable forms with labels. |
| NFR-14 | API | REST, JSON, versioned under `/api/v1`, documented in `openapi.yaml`, which MUST match the implementation. |
| NFR-15 | Compatibility | Evergreen browsers (latest Chrome, Edge, Firefox, Safari). |

---

## 9. Assumptions, Constraints, and Open Questions

### 9.1 Assumptions

| ID | Assumption | Rationale / impact |
|---|---|---|
| A-1 | Status set is `DRAFT`, `OPEN`, `PARTIALLY_RECEIVED`, `COMPLETED`, `CANCELLED`. | The brief mentions tracking statuses without naming them. A draft stage supports review before work starts. |
| A-2 | Operators and supervisors can both create documents. | Floor staff often know first what is arriving. |
| A-3 | Only supervisors open (release) documents. | Gives a lightweight approval step and separation of duties. |
| A-4 | Both roles can record receipts, only on `OPEN` or `PARTIALLY_RECEIVED` documents. | Operators do the physical counting. |
| A-5 | Completion is automatic, with no manual "complete" button. | Directly satisfies "must not be completed unless all items are fully received". |
| A-6 | Only supervisors cancel, and only when nothing has been received. | Cancelling after receipt would require stock reversal, which is out of scope. |
| A-7 | A product appears once per document. | Prevents ambiguous partial receiving across duplicate lines. |
| A-8 | Receipts cannot be edited or reversed. | Keeps the ledger append-only and simple to audit. Known limitation. |
| A-9 | Only supervisors create products. SKU is immutable and unique. | SKU is master data referenced by history. |
| A-10 | Idempotency keys are required on the receive endpoint, with row locking. | Satisfies the requirement that duplicate operations do not corrupt stock. |
| A-11 | Database CHECK constraints duplicate the key rules. | Defense in depth if application code has a bug. |
| A-12 | `inventory` holds the current balance. `inventory_transactions` is the append-only ledger. Both update in one transaction. | Fast reads plus a full audit trail. BR-10 is testable. |
| A-13 | One warehouse, one stock pool per product. | Locations are not mentioned in the brief. |
| A-14 | Quantities are whole units (integers). | Avoids decimal handling. Fractional units are out of scope. |
| A-15 | A receipt may span several items of one document in a single request. | Matches how a delivery is checked in one pass. |
| A-16 | Receiving beyond expected, even by a small tolerance, is never allowed. | The brief is explicit. No tolerance setting. |
| A-17 | Users are created by seed, not through the UI. | Authentication is "basic" per the brief. |
| A-18 | The supplier is free text, not a master record. | Keeps scope small. |

### 9.2 Constraints

- Relational database: PostgreSQL.
- Backend: Node.js, Express, TypeScript, Prisma, Zod. Frontend: React, Vite, TypeScript.
- Documented REST API (OpenAPI 3).
- Docker and Docker Compose for deployment. Configuration via environment variables only.
- Developed with a specification-driven AI agent workflow (Antigravity). Documents in `/docs` are the primary agent input.
- Target effort 8 to 12 hours. Core workflow correctness takes priority over visual polish.
- No unnecessary features beyond the scope above.

### 9.3 Open Questions (unresolved, to be raised with stakeholders)

| ID | Question | Current handling |
|---|---|---|
| Q-1 | What should happen when a supplier delivers less than expected and will not deliver the rest? | The document stays `PARTIALLY_RECEIVED`. A "close short" action with supervisor approval is the likely future answer. |
| Q-2 | How are receiving mistakes corrected (over-counted, wrong product)? | Out of scope. A future stock adjustment or receipt reversal feature is needed. |
| Q-3 | Should receipts be limited to the document's expected date? | No restriction. |
| Q-4 | Do products need units other than whole items, or multiple units of measure? | Single free-text `unit` label, informational only. |
| Q-5 | Is a physical location/bin required at receipt time? | Not modeled. |
| Q-6 | Is single sign-on or an audit export required? | Not required. |
| Q-7 | Should supervisors be able to see or revoke operators? | No user management in scope. |
| Q-8 | How long must idempotency keys be retained? | Kept with the receipt for its lifetime. |

---

## 10. Definition of Done

A requirement is done when:

1. It is implemented in the backend and, where applicable, in the UI.
2. Its business rules are enforced in the backend and verified by an automated test that references the rule ID.
3. `openapi.yaml` matches the implementation.
4. It works under `docker compose up` from a clean environment.
5. It is listed correctly in `testing.md`, and any shortcut is listed under known limitations.
