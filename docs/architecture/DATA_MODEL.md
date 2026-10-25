# Data Model — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed — no schema exists, nothing has been generated |
| Version | 0.1 |
| Scope owner | This document owns entities, relationships, constraints, and immutability strategy |
| Requirement source | [`../PRD.md`](../PRD.md) |
| Architectural context | [`./ARCHITECTURE.md`](./ARCHITECTURE.md) |

---

## 1. Modelling principles

These principles resolve most design questions without further debate.

| # | Principle | Requirement |
|---|---|---|
| P1 | **Every entity has a surrogate primary key** (`uuid`). Human-readable document numbers are separate attributes with their own uniqueness. A document number is never a primary key and never a foreign key. | FR-DOC-004 |
| P2 | **Master data is soft-deactivated, never deleted**, once referenced. Deletion of a referenced record is refused by the database, not just the UI. | FR-CUST-001, FR-DATA-005 |
| P3 | **Quantities are append-only movements.** No table stores a user-writable quantity that can be edited into inconsistency. | FR-MATL-012, BR-MATL-003 |
| P4 | **Issued documents are snapshots.** An issued document renders from its own frozen copy of party and line data, never from live master data. | FR-FIN-003, BR-DOC-001 |
| P5 | **Status columns are enums with constrained values**, and transitions are validated in the domain layer with a matching database `CHECK` where practical. | FR-PROJ-011 |
| P6 | **Money is stored as integer minor units** (IDR has no practical minor unit in this business, but the column type must not be floating point). Never `float`. | — |
| P7 | **Timestamps are `timestamptz`**, stored UTC, rendered in `Asia/Jakarta`. Business dates (invoice date, work date) are `date`, not timestamps, because a date is a business fact independent of clock. | — |
| P8 | **Derived values are derived.** Material readiness, overdue status, and outstanding balance are computed, not stored as independently editable fields. Where cached, they carry a reconciliation path. | §9.3, FR-FIN-005 |
| P9 | **Every mutable business table carries `created_at`, `created_by`, `updated_at`, `updated_by`.** | FR-AUD-001 |
| P10 | **Optional means optional in the schema too.** Quotation, customer PO, and material lines are nullable relationships, not rows with sentinel values. | BR-PROJ-001 |

### 1.1 Naming conventions

- Tables: `snake_case`, singular concept, pluralised where it reads naturally (`projects`, `material_movements`).
- Foreign keys: `<entity>_id`.
- Enums: stored as text with a `CHECK` constraint, not native DB enums, so adding a value does not require a type migration that locks a table.
- Every table in the finance and document domains carries an explicit `status` plus `status_changed_at`.

---

## 2. Identity and access

### `users`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `email` | citext unique, nullable | Login identifier; unique when present |
| `username` | citext unique, nullable | Alternative login for technicians without email |
| `password_hash` | text | Argon2id/bcrypt; never selected by default in queries |
| `display_name` | text not null | |
| `is_active` | boolean not null default true | Deactivation, not deletion |
| `must_change_password` | boolean not null default false | |
| `last_login_at` | timestamptz | |
| audit columns | | |

Constraint: exactly one of `email` or `username` must be present (login must be possible).

### `roles`

`id`, `code` (`OWNER`, `OPS_ADMIN`, `FINANCE`, `TECHNICIAN`, `TEAM_LEADER`), `name`, `description`, `is_system` (system roles cannot be deleted).

### `permissions`

`id`, `code` (e.g. `invoice.issue`, `purchase.approve`, `audit.view`), `resource`, `action`, `description`.

### `role_permissions`

`role_id`, `permission_id` — composite PK. Permissions are data, not code branches.

### `user_roles`

`user_id`, `role_id`, `granted_by`, `granted_at` — composite PK. **Multiple roles per user is a first-class capability** (NFR-SEC-003).

### `sessions`

`id`, `user_id`, `created_at`, `last_seen_at`, `expires_at`, `revoked_at`, `ip`, `user_agent`. Revocation on password change is a query, not a convention.

### `overrides`

Records an authorised bypass of a blocked workflow or a separation-of-duties conflict.

| Column | Notes |
|---|---|
| `id` | uuid PK |
| `user_id` | who overrode |
| `action` | what was permitted (e.g. `workorder.verify_own_work`) |
| `entity_type`, `entity_id` | target |
| `reason` | mandatory, non-empty, minimum length enforced |
| `created_at` | |
| `audit_event_id` | links to the audit record (FR-AUD-004) |

This table is not a convenience escape hatch; it is a control. Every row is surfaced to the owner (AC-AUD-004).

---

## 3. Customer and catalogue

### `customers`

`id`, `code` (optional internal reference), `display_name` not null, `customer_type` (`INDIVIDUAL`/`COMPANY`), `primary_phone`, `email`, `address_line`, `city`, `province`, `postal_code`, `notes`, `is_active`, audit columns.

Index: trigram or normalised-name index to support duplicate warning (FR-CUST-006).

### `customer_contacts`

`id`, `customer_id` FK, `name`, `role_title`, `phone`, `email`, `is_primary` boolean.

Constraint: at most one primary contact per customer — **partial unique index** `UNIQUE (customer_id) WHERE is_primary`. This is a database guarantee, not an application hope.

### `service_categories`

`id`, `code`, `name`, `description`, `is_active`. **[Confirmed]** the business serves CCTV-related and other service types, so this is configurable data, never a hard-coded enum (FR-PROJ-002).

### `customer_service_categories`

`customer_id`, `service_category_id` — composite PK. Supports FR-CUST-003.

### `items`

Material/product catalogue.

`id`, `sku` (optional), `name`, `category`, `base_unit`, `specification`, `is_active`, audit columns.

**[Assumption]** The business has no reliable item master today; items will be created during migration and refined during pilot. Duplicate item names are likely — the model allows `sku` to be null rather than forcing fabricated codes.

### `suppliers`

`id`, `name`, `contact_name`, `phone`, `email`, `address`, `payment_terms_days`, `notes`, `is_active`, audit columns.

### `technicians`

`id`, `user_id` FK unique, `full_name`, `phone`, `skills` (free text or join table if needed later), `is_active`, audit columns.

Constraint: one technician record per user, one user per technician.

---

## 4. Project domain

### `projects`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `project_code` | text unique | Human reference, **not** a financial document number; internal only |
| `customer_id` | FK, not null | |
| `service_category_id` | FK, not null | Drives no CCTV assumptions (FR-PROJ-002) |
| `title` | text not null | |
| `description` | text | |
| `site_address`, `site_city` | text | Location |
| `planned_start_date` | date | |
| `target_completion_date` | date | Revisable, revision recorded separately |
| `actual_start_date` | date | Set on transition to `IN_PROGRESS` |
| `actual_completion_date` | date | Set on verification |
| `status` | text | §9.1 of PRD |
| `budget_revenue` | bigint | Optional (FR-PROJ-008) |
| `budget_material_cost` | bigint | Optional |
| `quotation_id` | FK nullable | Optional (FR-PROJ-003) |
| `customer_po_id` | FK nullable | Optional (FR-PROJ-004) |
| `closed_at`, `closed_by` | | |
| audit columns | | |

Constraint: `target_completion_date >= planned_start_date` where both present.

### `project_schedule_revisions`

`id`, `project_id`, `previous_start`, `previous_target`, `new_start`, `new_target`, `reason`, `changed_by`, `changed_at`.

Append-only. A revised target date overwrites nothing in history (FR-PROJ-005).

### `project_assignments`

`id`, `project_id`, `technician_id`, `role_on_project` (`TECHNICIAN`/`TEAM_LEADER`), `assigned_by`, `assigned_at`, `unassigned_at`, `unassign_reason`.

Constraint: **at most one active `TEAM_LEADER` per project** — partial unique index on `(project_id) WHERE role_on_project = 'TEAM_LEADER' AND unassigned_at IS NULL`. Enforces FR-PROJ-007 at the database level.

### `project_status_transitions`

Append-only record of every status change: `id`, `project_id`, `from_status`, `to_status`, `reason_code`, `reason_text`, `actor_id`, `occurred_at`, `override_id` nullable.

This gives project history without relying on the audit log for business reporting. Both exist; they serve different purposes (see §10).

### `project_categories_attributes` (optional, deferred)

If category-specific fields become necessary, they belong in a JSONB `attributes` column or a typed extension table — **not** in new nullable columns per category, which would make every service category leak into every project.

---

## 5. Material domain

This is the most correctness-sensitive area of the model.

### 5.1 `material_requirements`

One row per material line per project.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `project_id` | FK not null | |
| `item_id` | FK not null | |
| `required_quantity` | numeric not null, > 0 | |
| `unit` | text not null | Snapshot of unit at requirement time |
| `specification_note` | text | |
| **`stock_status`** | text not null default `UNKNOWN` | `UNKNOWN` / `VERIFIED` (FR-MATL-002) |
| `verified_on_hand_quantity` | numeric nullable | Set only by verification |
| `verified_by`, `verified_at` | | |
| **`fulfilment_status`** | text not null default `DRAFT` | PRD §9.4 |
| `is_optional` | boolean default false | Optional extras that do not block readiness |
| audit columns | | |

**Critical constraint:** `stock_status = 'VERIFIED'` requires `verified_on_hand_quantity IS NOT NULL` and `verified_at IS NOT NULL`. Enforced by `CHECK`. An "unverified but available" row is not representable.

### 5.2 `stock_verifications`

Append-only evidence of a physical count.

`id`, `material_requirement_id`, `observed_quantity`, `location_note`, `verified_by`, `verified_at`, `note`.

A re-verification creates a new row; the requirement's `verified_on_hand_quantity` reflects the latest, and the history remains intact.

### 5.3 `reservations`

| Column | Notes |
|---|---|
| `id` | uuid PK |
| `material_requirement_id` | FK |
| `item_id` | denormalised for locking and querying |
| `reserved_quantity` | numeric > 0 |
| `status` | `ACTIVE`, `RELEASED`, `CONSUMED` |
| `reserved_by`, `reserved_at` | |
| `released_at`, `release_reason` | |
| `consumed_at` | |

**Invariant enforced in the transaction:** for any `item_id`, `SUM(reserved_quantity) WHERE status='ACTIVE'` must not exceed verified available quantity. Enforced by locking the item's balance row (`SELECT … FOR UPDATE`) and re-checking inside the transaction — not by a pre-check read (FR-MATL-006, BR-MATL-002).

Constraint: a reservation may only reference a requirement with `stock_status = 'VERIFIED'`. Enforced by application check plus a trigger or equivalent, because a plain FK cannot express it (FR-MATL-005).

### 5.4 `material_movements` — the ledger

**The only table through which quantity changes.**

| Column | Notes |
|---|---|
| `id` | uuid PK |
| `item_id` | FK |
| `project_id` | FK nullable (null for later centralised stock adjustments) |
| `material_requirement_id` | FK nullable |
| `movement_type` | see below |
| `quantity` | numeric, signed convention documented below |
| `unit` | text |
| `reference_type`, `reference_id` | polymorphic link to the causing record (goods receipt line, issue, return, etc.) |
| `occurred_at` | business time |
| `recorded_at` | system time |
| `recorded_by` | |
| `note` | |
| `reversal_of_movement_id` | nullable, for compensating entries |

**Movement types:** `RECEIPT_IN`, `RESERVE`, `RELEASE`, `ISSUE_OUT`, `CONSUME`, `RETURN_IN`, `WRITE_OFF`, `ADJUSTMENT`.

**Sign convention (documented once, enforced everywhere):** inbound increases available quantity; outbound decreases it. Reservation is tracked in `reservations`, not as a quantity movement, so that "reserved" and "physically moved" are never confused. This distinction is deliberate: a reservation is a claim, not a physical event.

**Prohibitions:**
- No `UPDATE` or `DELETE` on this table. Corrections are new rows with `reversal_of_movement_id`.
- No application path may write a balance directly.

### 5.5 `material_issues`

Records physical handover to a technician.

`id`, `project_id`, `work_order_id` nullable, `technician_id`, `issued_by`, `issued_at`, `status` (`OPEN`/`PARTIALLY_RETURNED`/`CLOSED`), `note`.

### `material_issue_lines`

`id`, `material_issue_id`, `item_id`, `material_requirement_id` nullable, `issued_quantity`, `consumed_quantity`, `returned_quantity`, `written_off_quantity`, `unit`.

Constraint: `consumed + returned + written_off <= issued`. Enforced by `CHECK`. This is what makes "quantity cannot vanish" (BR-MATL-004) a database fact rather than a promise.

### 5.6 `stock_balances` (derived, reconciled)

`item_id`, `location_id` (nullable in MVP), `available_quantity`, `reserved_quantity`, `last_reconciled_at`.

This is a **cache**, not a source of truth. It exists because summing a large ledger on every reservation is wasteful. A reconciliation job recomputes it from `material_movements` and `reservations` and reports variance (FR-MATL-013). If the cache and the ledger disagree, **the ledger wins** and the discrepancy is an incident.

### 5.7 `material_reconciliations`

`id`, `scope` (`PROJECT`/`ITEM`), `project_id` nullable, `item_id` nullable, `expected_quantity`, `recorded_quantity`, `variance`, `run_by`, `run_at`, `resolution_note`.

---

## 6. Procurement domain

### `purchase_requests`

`id`, `request_code` (internal reference, not a financial document number), `project_id` nullable (a request may span projects), `requested_by`, `requested_at`, `needed_by_date`, `status` (PRD §9.5), `notes`, audit columns.

### `purchase_request_lines`

`id`, `purchase_request_id`, `material_requirement_id` nullable, `item_id`, `description`, `quantity`, `unit`, `estimated_unit_price`, `preferred_supplier_id` nullable.

### `purchase_approvals`

`id`, `purchase_request_id`, `decision` (`APPROVED`/`REJECTED`), `decided_by`, `decided_at`, `reason`, `policy_snapshot` (JSONB: the threshold/rule that applied at decision time).

Storing the policy snapshot matters: if thresholds change later, the historical decision must still explain itself.

Constraint: a purchase request may have at most one **current** decision; superseding decisions create new rows and mark the prior one superseded. This supports audit without allowing silent flipping.

### `purchase_orders`

| Column | Notes |
|---|---|
| `id` | uuid PK |
| `purchase_request_id` | FK, unique — one PO per approved request |
| `supplier_id` | FK |
| **`document_number`** | nullable until issued; unique when present |
| `status` | PRD §9.6 |
| `issued_at`, `issued_by` | |
| `expected_delivery_date` | |
| `currency` | `IDR` |
| `subtotal`, `tax_amount`, `total_amount` | bigint |
| `snapshot` | JSONB frozen supplier/line copy at issuance |
| `cancelled_at`, `cancel_reason` | |
| audit columns | |

Constraint: `document_number IS NOT NULL` requires `status <> 'DRAFT'` (a numbered PO is issued by definition).

### `purchase_order_lines`

`id`, `purchase_order_id`, `item_id`, `description`, `ordered_quantity`, `received_quantity` (derived/cached), `unit`, `unit_price`, `line_total`.

Constraint: `received_quantity <= ordered_quantity` unless an over-receipt approval reference is present (BR-PROC-003).

### `goods_receipts`

`id`, `purchase_order_id`, `received_by`, `received_at`, `delivery_note_reference`, `status` (`DRAFT`/`POSTED`), `note`.

### `goods_receipt_lines`

`id`, `goods_receipt_id`, `purchase_order_line_id`, `received_quantity`, `discrepancy_type` nullable (`SHORT`/`DAMAGED`/`WRONG_ITEM`/`PRICE_VARIANCE`), `discrepancy_note`, `resolved_at`, `resolved_by`.

Posting a receipt creates `RECEIPT_IN` movements atomically (BR-MATL-005). An unposted receipt has no stock effect.

---

## 7. Finance domain

### 7.1 Three layers kept structurally separate

| Layer | Tables | MVP |
|---|---|---|
| L1 Operational | `payments`, `expenses`, `project_cost_entries` | In scope |
| L2 AR/AP | `invoices`, `payables`, `payment_allocations` | In scope |
| L3 Accounting | *(none — deliberately absent)* | **[Out of Scope]** |

No journal, no chart of accounts, no ledger table. Adding them without an accountant's requirement would create the illusion of accounting without its correctness (Q-FIN-01).

### `quotations`

`id`, `customer_id`, `project_id` nullable, `document_number` nullable, `status` (PRD §9.10), `valid_until`, `subtotal`, `tax_amount`, `total_amount`, `snapshot`, `issued_at`, audit columns.

### `customer_purchase_orders`

`id`, `customer_id`, `project_id` nullable, `po_reference` (customer's own number, **not** ours), `po_date`, `attachment_id` nullable, `amount` nullable, audit columns.

Important distinction: this stores the customer's number as data. It never participates in our numbering series and must never be validated against it.

### `invoices`

| Column | Notes |
|---|---|
| `id` | uuid PK |
| `customer_id` | FK |
| `project_id` | FK nullable |
| `document_number` | **nullable until issued**; unique when present |
| `status` | PRD §9.8 |
| `invoice_date` | date |
| `due_date` | date |
| `subtotal`, `discount_amount`, `tax_amount`, `total_amount` | bigint |
| `amount_paid` | derived/cached |
| `snapshot` | JSONB immutable copy (FR-FIN-003) |
| `pdf_object_key`, `pdf_sha256` | immutable artefact |
| `issued_at`, `issued_by` | |
| `voided_at`, `void_reason` | |
| `credit_note_for_invoice_id` | nullable self-reference |
| `replaced_by_invoice_id` | nullable self-reference |
| audit columns | |

Constraints:
- `document_number IS NOT NULL` ⇒ `status <> 'DRAFT'`.
- Unique index on `(document_number)` where not null.
- `total_amount = subtotal - discount_amount + tax_amount` (CHECK).
- `due_date >= invoice_date`.

### `invoice_lines`

`id`, `invoice_id`, `line_no`, `description`, `quantity`, `unit`, `unit_price`, `line_total`, `item_id` nullable, `service_category_id` nullable.

**No `project_id` on the line**: invoice lines describe what is billed, which may not map one-to-one to projects or items. Forcing that mapping would break non-CCTV service billing (FR-PROJ-002).

### `credit_notes`

`id`, `document_number`, `invoice_id` (original), `reason`, `amount`, `issued_at`, `status`, `snapshot`.

### `payables`

`id`, `supplier_id`, `purchase_order_id` nullable, `supplier_invoice_reference`, `document_number` nullable (internal payable reference, if the business wants one), `status` (PRD §9.7), `invoice_date`, `due_date`, `amount`, `amount_paid`, `attachment_id`, `dispute_reason`, audit columns.

### `payments`

| Column | Notes |
|---|---|
| `id` | uuid PK |
| `direction` | `INBOUND` (from customer) / `OUTBOUND` (to supplier) |
| `party_type`, `party_id` | customer or supplier |
| `amount` | bigint |
| `payment_date` | date |
| `method` | `CASH`/`BANK_TRANSFER`/`OTHER` |
| `reference` | transfer reference, cheque number, etc. |
| `status` | `RECORDED` / `CONFIRMED` / `REVERSED` |
| `recorded_by`, `recorded_at` | |
| `confirmed_by`, `confirmed_at` | **distinct from recording** (BR-PROC-002) |
| `reversal_of_payment_id` | nullable |
| `evidence_attachment_id` | nullable |
| `note` | |

**No bank integration exists.** The `method` and `reference` fields record what a human observed. The system never claims to have executed a transfer (BR-PROC-004).

### `payment_allocations`

`id`, `payment_id`, `invoice_id` nullable, `payable_id` nullable, `allocated_amount`.

Constraint: exactly one of `invoice_id` or `payable_id` is present (XOR), enforced by CHECK.

**Invariant:** for each invoice, `SUM(allocated_amount)` over non-reversed payments ≤ `total_amount`. Checked under lock inside the transaction (BR-FIN-003). Overpayment becomes an unapplied credit, represented by an allocation to a credit record rather than a negative balance.

### `expenses`

`id`, `category_id`, `project_id` nullable, `amount`, `expense_date`, `description`, `paid_by_user_id`, `payment_id` nullable, `attachment_id` nullable`, audit columns.

### `project_cost_entries`

Materialised view of recognised project cost, one row per cost event, with `source_type` (`GOODS_RECEIPT`/`SUPPLIER_INVOICE`/`PAYMENT`/`EXPENSE`/`MATERIAL_ISSUE`) and `source_id`.

**The double-counting control (BR-FIN-005):** the table has a unique constraint on `(project_id, source_type, source_id)`, so the same source cannot be counted twice. Recognition rules decide *which* source type is authoritative per configuration, and the others are excluded rather than added.

**[Open Question]** Which source is authoritative for material cost — receipt (accrual at delivery) or supplier invoice (accrual at billing)? **Proposed default:** recognise at **goods receipt**, because the business cares about cost as soon as material exists, and reconcile to the supplier invoice when it arrives, recording any variance as a separate adjustment entry rather than silently changing history.

---

## 8. Document and attachment domain

### `document_series`

See [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) §4.

`id`, `series_key` unique, `document_type`, `company_prefix`, `format_pattern`, `reset_period` (`YEAR`/`MONTH`/`NONE`), `padding`, `is_active`.

### `document_sequences`

`series_key`, `period_key`, `last_value` — composite PK on `(series_key, period_key)`.

This is the row locked and incremented atomically at issuance. **Never read-then-write.**

### `issued_documents`

Registry of every issued human-readable document across types.

`id`, `document_type`, `series_key`, `document_number` (unique per series), `entity_type`, `entity_id`, `issued_at`, `issued_by`, `snapshot` JSONB, `pdf_object_key`, `pdf_sha256`, `status` (`ISSUED`/`VOID`/`SUPERSEDED`/`CREDITED`), `void_reason`, `supersedes_document_id`, `is_historical_import` boolean.

**`is_historical_import`** marks migrated numbers so they are excluded from sequence allocation and from "new number" reports (FR-DATA-004). Historical numbers are preserved verbatim (BR-DATA-001).

### `attachments`

`id`, `owner_type`, `owner_id`, `object_key`, `original_filename`, `content_type`, `byte_size`, `sha256`, `uploaded_by`, `uploaded_at`, `is_deleted` (soft), `deleted_at`.

Constraint: an attachment owned by an issued document cannot be soft-deleted (enforced in application layer with an audit record).

### `document_number_allocations`

`id`, `series_key`, `period_key`, `allocated_value`, `allocated_number`, `allocated_at`, `allocated_by`, `issuing_entity_type`, `issuing_entity_id`, `status` (`ALLOCATED`/`VOIDED`/`SUPERSEDED`).

This is the audit-grade record of *which number went to which document and when*. `issued_documents` holds the document; this table holds the allocation event. Keeping them separate means a voided document still has a permanent allocation trail (FR-FIN-011, NFR-AUD-003).

---

## 9. Audit domain

### `audit_events`

| Column | Notes |
|---|---|
| `id` | uuid PK |
| `occurred_at` | timestamptz |
| `actor_user_id` | nullable for system actions |
| `actor_role_codes` | text[] snapshot of effective roles at the time |
| `action` | e.g. `invoice.issue` |
| `entity_type`, `entity_id` | |
| `before_state` | JSONB, changed fields only |
| `after_state` | JSONB, changed fields only |
| `reason` | for overrides and cancellations |
| `correlation_id` | request trace |
| `ip_address` | where available |
| `override_id` | nullable |
| `prev_hash`, `record_hash` | hash chain |

`UPDATE`/`DELETE` revoked from the application role (NFR-AUD-001). Detail in [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md).

---

## 10. Why both `*_status_transitions` and `audit_events` exist

They are not duplicates:

| | `project_status_transitions` (and equivalents) | `audit_events` |
|---|---|---|
| Purpose | Business history for reporting and UI timelines | Forensic record of who did what, including non-status changes |
| Content | Status, reason, actor, time | Any field change, permission change, override, view |
| Consumer | Business users and reports | Owner, auditors, incident investigation |
| Mutability | Append-only | Append-only |
| Query pattern | Fast, indexed by entity | Filtered by actor/action/date |

Storing business history only in a generic audit table would make reporting slow and semantically awkward; storing forensics only in business tables would lose field-level change data. Both are required.

---

## 11. Referential integrity and deletion policy

| Relationship | On delete | Enforcement |
|---|---|---|
| Project → customer | `RESTRICT` | Cannot delete a customer with projects |
| Invoice → customer | `RESTRICT` | Cannot delete a customer with invoices |
| Requirement line → item | `RESTRICT` | Deactivate the item instead |
| Movement → item | `RESTRICT` | Ledger rows are permanent |
| Allocation → invoice | `RESTRICT` | Financial records are permanent |
| Attachment → owner | `RESTRICT` | Files of issued documents are permanent |
| Assignment → technician | `RESTRICT` | Deactivate the technician |

**No `CASCADE` delete exists on any business table.** Cascading deletes in a financial system are a data-loss mechanism wearing a convenience costume. Where a child genuinely has no independent meaning (e.g. draft lines), the parent's deletion is itself prohibited once the parent is non-draft, making cascade unnecessary.

---

## 12. Derived values and their sources

| Derived value | Computed from | Cached? | Reconciliation |
|---|---|---|---|
| Invoice outstanding | `total_amount - SUM(allocations)` | `amount_paid` cached | Recompute on read for detail views; nightly check |
| Invoice overdue | `due_date < today` AND status in (ISSUED, PARTIALLY_PAID) | No | Pure function |
| Payable outstanding | `amount - SUM(allocations)` | `amount_paid` cached | Nightly check |
| Material readiness | Requirement `stock_status`, `fulfilment_status`, receipts, issues | No | Pure function over lines |
| Available stock | Ledger sum − active reservations | `stock_balances` cache | `material_reconciliations` (FR-MATL-013) |
| Project cost | `project_cost_entries` | No | Unique constraint prevents double count |
| Work order progress % | Reports + completion events | No | Estimate only (FR-TECH-010) |

Rule: a cached derived value must always have a named reconciliation job. A cache without reconciliation is a future data-quality incident.

---

## 13. Open modelling questions

| ID | Question | Proposed default |
|---|---|---|
| Q-MODEL-01 | Should material requirements support substitute items (if the exact item is unavailable)? | No in MVP; note it as a line note. Adding substitutions introduces a whole comparison workflow |
| Q-MODEL-02 | Does the business need per-supplier price history? | Record prices on PO lines only; no separate price list in MVP |
| Q-MODEL-03 | Should technicians be modelled as employees with rates for cost allocation? | No. Labour cost is not a project cost source in MVP — **[Open Question]** whether labour cost must appear in project profitability |
| Q-MODEL-04 | Should invoices support multiple projects per invoice? | Model allows it (`project_id` nullable at header, no project on lines); UI default is one project per invoice |
| Q-MODEL-05 | Is a location dimension needed for stock in MVP? | Single implicit location (office/warehouse); `location_id` reserved but unused |

---

## 14. Cross-references

- Requirements: [`../PRD.md`](../PRD.md)
- Numbering tables in context: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Material workflow using this model: [`../workflows/PROCUREMENT_AND_MATERIALS.md`](../workflows/PROCUREMENT_AND_MATERIALS.md)
- Permission implications: [`./SECURITY_AND_ACCESS_CONTROL.md`](./SECURITY_AND_ACCESS_CONTROL.md)
