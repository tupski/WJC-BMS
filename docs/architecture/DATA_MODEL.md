# Data Model — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Target design — no schema exists, nothing has been generated |
| Version | 0.2 — engine reconciled to MySQL/MariaDB 2026-10-25 |
| Scope owner | Entities, columns, constraints, immutability strategy |
| Requirement source | [`../PRD.md`](../PRD.md) |
| Architectural context | [`./ARCHITECTURE.md`](./ARCHITECTURE.md) §5 |
| Engine | MySQL 8.0.16+ or MariaDB 10.2.1+ (NFR-DATA-009) |

---

## 1. Modelling principles

| # | Principle | Requirement |
|---|---|---|
| P1 | **Every entity has a surrogate primary key** (UUID/ULID). Human-readable document numbers are separate attributes with their own uniqueness; a document number is never a PK and never an FK. | FR-DOC-004 |
| P2 | **Master data is soft-deactivated, never deleted**, once referenced. Refusal is enforced by the database, not just the UI. | FR-CUST-001, FR-DATA-005 |
| P3 | **Quantities are append-only movements.** No table stores a user-writable quantity. | FR-MATL-012, BR-MATL-003 |
| P4 | **Issued documents are snapshots** of party, line, **and company** data — never rendered from live master data. | FR-FIN-003, BR-DOC-001, BR-SYS-001 |
| P5 | **Status columns are constrained strings**, validated in the domain and backed by `CHECK`. Native `ENUM` is avoided so adding a value needs no table-locking type migration. | FR-PROJ-011 |
| P6 | **Money is integer minor units** (`BIGINT`). Never `FLOAT`/`DOUBLE`. | — |
| P7 | **All timestamps stored UTC** in `DATETIME(3)`, converted at the boundary, rendered in `Asia/Jakarta`. Business dates are `DATE`. | NFR-OPS-009 |
| P8 | **Derived values are derived.** Readiness, overdue, outstanding are computed; where cached, they carry a reconciliation path. | §9.3, FR-FIN-005 |
| P9 | **Every mutable business table carries `created_at`, `created_by`, `updated_at`, `updated_by`.** | FR-AUD-001 |
| P10 | **Optional means optional in the schema too** — nullable relationships, not sentinel values. | BR-PROJ-001 |

### 1.1 Naming and typing

Tables `snake_case` plural; FKs `<entity>_id`; engine **InnoDB** (required for FKs and transactions). Character set `utf8mb4` with `utf8mb4_0900_ai_ci` (MySQL 8) or `utf8mb4_general_ci` (MariaDB) — see §1.2 for why explicit normalised columns are still required. Status columns are `VARCHAR(32)` + `CHECK`. Finance and document tables carry `status` plus `status_changed_at`.

### 1.2 Case-insensitive uniqueness (replaces `citext`)

| Option | Behaviour | Verdict |
|---|---|---|
| `_ci` collation only | Implicitly case-insensitive; no schema artefact to review; behaviour varies by engine and collation version | Rejected as the primary control |
| `*_normalized` column + unique index | Explicit, indexable, portable, testable | **Chosen** |

Applied to `users.email`, `users.username`, `customers.display_name_key`, and any other case-insensitive identifier: store the entered value for display, plus `email_normalized` (`LOWER(TRIM(...))`) with `UNIQUE (email_normalized)`. Where supported, `email_normalized` is a **stored generated column**, removing the chance of the application forgetting to maintain it; the application-maintained variant is the MariaDB fallback.

### 1.3 Soft-delete-aware uniqueness (replaces partial unique indexes)

Partial unique indexes do not exist, so `UNIQUE (x) WHERE deleted_at IS NULL` is unavailable. Two patterns, documented once and reused:

**Pattern A — nullable active-key column (preferred).**

```sql
business_key       VARCHAR(191)
unique_active_key  VARCHAR(191) GENERATED ALWAYS AS
                   (IF(deleted_at IS NULL, business_key, NULL)) STORED
UNIQUE (unique_active_key)
```

Unique indexes ignore `NULL`, so any number of soft-deleted rows coexist.

**Pattern B — nullable composite discriminator.** Where a generated column cannot express the rule (e.g. "at most one active team leader per project"), store `active_flag TINYINT(1)` as `1` when active and `NULL` when not, then `UNIQUE (project_id, active_flag)`. Setting `active_flag = NULL` on unassignment releases the slot.

### 1.4 Isolation, locking, deadlock

| Concern | Decision |
|---|---|
| Default isolation | `REPEATABLE READ`. Acceptable for reads. |
| Allocation and reservation paths | `READ COMMITTED`, **or** a single-row conditional `UPDATE` needing no explicit lock. The latter is preferred — atomic without lock escalation. |
| Explicit row locks | `SELECT … FOR UPDATE` only where a check-then-write cannot be collapsed. Lock scope is wider than in PostgreSQL (gap locks), so always lock by primary key or a unique index. |
| Deadlock policy | Bounded retry (3 attempts, exponential backoff) **only** for proven-idempotent transactions; otherwise surface the conflict. Silent infinite retry hides a design error. |
| Forbidden | `SERIALIZABLE` reliance, deferrable-constraint reliance, long transactions holding locks while rendering a PDF |

### 1.5 Unavailable idioms and their replacements

Reasoning and alternatives for each substitution are in [`../decisions/ADR-003-single-relational-database.md`](../decisions/ADR-003-single-relational-database.md) D2–D9. This table is the working reference.

| PostgreSQL idiom | Status here | Replacement |
|---|---|---|
| `UPDATE … RETURNING` | **Not supported** (MariaDB has `RETURNING` for `INSERT`/`DELETE` in some versions; MySQL has none) | `LAST_INSERT_ID()` claim — §5.1 of DOCUMENT_NUMBERING |
| `ON CONFLICT DO NOTHING` | Not supported | `INSERT … ON DUPLICATE KEY UPDATE id = id`, or `INSERT IGNORE` (warning-swallowing caveat) |
| `ON CONFLICT DO UPDATE` | Not supported | `INSERT … ON DUPLICATE KEY UPDATE col = VALUES(col)` — **note** this advances `AUTO_INCREMENT` on conflicting inserts even when no row is written |
| `JSONB` + GIN | Not supported | `JSON` column; functional index or generated column for any queried path |
| `timestamptz` | Not supported | UTC in `DATETIME(3)` |
| Partial unique index | Not supported | §1.3 |
| Deferrable FK | Not supported | Ordered inserts, nullable FK + application sequencing |
| Transactional DDL | Not supported (implicit commit) | Expand → migrate → contract (NFR-DATA-010) |

---

## 2. Identity and access

**`users`** — `id` PK · `email` NULL · `email_normalized` NULL UNIQUE · `username` NULL · `username_normalized` NULL UNIQUE · `password_hash` (never selected by default) · `display_name` · `locale` DEFAULT `id` · `is_active` · `must_change_password` · `last_login_at` · audit columns. `CHECK (email IS NOT NULL OR username IS NOT NULL)`.

**`roles`** — `id`, `code` (`SUPER_ADMIN`, `PEMILIK`, `ADMIN`, `TEKNISI`), `name`, `description`, `is_system`. **Exactly four rows.** `FINANCE` and `TEAM_LEADER` are not roles and have no row.

**`permissions`** — `id`, `code` (e.g. `invoice.issue`, `purchase.approve`), `resource`, `action`, `description`.

**`role_permissions`** — `role_id`, `permission_id`, composite PK. Permissions are data, not code branches.

**`user_roles`** — `user_id`, `role_id`, `granted_by`, `granted_at`, composite PK. **Multiple roles per user is a first-class capability** (NFR-SEC-003).

**`sessions`** — `id`, `user_id`, `created_at`, `last_seen_at`, `expires_at`, `revoked_at`, `ip`, `user_agent`. Revocation on password change is a query, not a convention.

**`overrides`** — `id`, `user_id`, `action`, `entity_type`, `entity_id`, `reason` (mandatory, minimum length), `created_at`, `audit_event_id`. A control, not a convenience; every row surfaces to `PEMILIK` (AC-AUD-004). **It can never reference `purchase.approve`** — that requirement is not overridable (BR-PROC-005).

---

## 3. Customer, catalogue, supplier

**`customers`** — `id`, `code` NULL, `display_name`, `display_name_key` (normalised, indexed), `customer_type`, `primary_phone`, `phone_normalized`, `email`, `address_line`, `city`, `province`, `postal_code`, `notes`, `is_active`, `deleted_at` NULL, audit columns. Indexes on `display_name_key` and `phone_normalized` support the duplicate warning (FR-CUST-006); `FULLTEXT` is **[Proposed]** for name similarity, with prefix matching as fallback.

**`customer_contacts`** — `id`, `customer_id`, `name`, `role_title`, `phone`, `email`, `is_primary`, `active_flag` NULL. At most one primary per customer: Pattern B, `UNIQUE (customer_id, active_flag)`.

**`service_categories`** — `id`, `code`, `name`, `description`, `is_active`. **[Confirmed]** the business serves CCTV and other service types, so this is data, never a hard-coded enum (FR-PROJ-002). Linked via **`customer_service_categories`** (`customer_id`, `service_category_id`, composite PK).

**`items`** — `id`, `sku` NULL, `name`, `name_normalized`, `category`, `base_unit`, `specification`, `is_active`, audit columns. **[Assumption]** no reliable item master exists today; duplicates are likely, so `sku` stays nullable rather than forcing fabricated codes.

**`suppliers`** — `id`, `name`, `name_normalized`, `contact_name`, `phone`, `email`, `address`, `payment_terms_days`, `notes`, `is_active`, audit columns.

**`technicians`** — `id`, `user_id` UNIQUE, `full_name`, `phone`, `skills` (free text in MVP), `is_active`, audit columns. One technician per user.

---

## 4. Project domain

### `projects`

| Column | Type | Notes |
|---|---|---|
| `id` | PK | |
| `project_code` | `VARCHAR(32)` UNIQUE | Human reference, **not** a financial document number |
| `customer_id`, `service_category_id` | FK NOT NULL, RESTRICT | Category drives no CCTV assumptions (FR-PROJ-002) |
| `title` | `VARCHAR(200)` | |
| `description` | `TEXT` | |
| `site_address` | `VARCHAR(255)` | Human-readable address (FR-PROJ-013) |
| `site_city`, `site_province` | `VARCHAR(100)` | |
| `site_latitude`, `site_longitude` | `DECIMAL(10,7)` NULL | **Optional.** `DECIMAL`, never `FLOAT`, to avoid coordinate drift |
| `planned_start_date`, `target_completion_date` | `DATE` | Target revisable; revisions recorded separately |
| `actual_start_date`, `actual_completion_date` | `DATE` NULL | Set on transition and verification |
| `status` | `VARCHAR(32)` | PRD §9.1 |
| `budget_revenue`, `budget_material_cost` | `BIGINT` NULL | Optional (FR-PROJ-008) |
| `quotation_id`, `customer_po_id` | FK NULL | Optional (FR-PROJ-003/004) |
| `closed_at`, `closed_by`, audit columns | | |

Constraints: `CHECK (target_completion_date >= planned_start_date)`; latitude between −90 and 90, longitude between −180 and 180; coordinates stored together or not at all — `CHECK ((site_latitude IS NULL) = (site_longitude IS NULL))`.

**`project_schedule_revisions`** — `id`, `project_id`, `previous_start`, `previous_target`, `new_start`, `new_target`, `reason`, `changed_by`, `changed_at`. Append-only (FR-PROJ-005).

### `project_members`

Assignment plus team-leader designation in one table (replaces the former `role_on_project` enum).

| Column | Notes |
|---|---|
| `id`, `project_id`, `technician_id` | |
| `is_team_leader` | `TINYINT(1)` DEFAULT 0 — a designation, **not a role** |
| `leader_active_flag` | `TINYINT(1)` NULL — `1` when an active leader, `NULL` otherwise |
| `assigned_by`, `assigned_at`, `unassigned_at` NULL, `unassign_reason` | Append-only in effect |

`UNIQUE (project_id, leader_active_flag)` enforces **at most one active team leader per project** (FR-PROJ-007) at the database level.

**`project_status_transitions`** — append-only: `id`, `project_id`, `from_status`, `to_status`, `reason_code`, `reason_text`, `actor_id`, `occurred_at`, `override_id` NULL.

### `work_sessions`

**First-class entity (FR-TECH-011).** A technician's discrete period of work on a project.

| Column | Notes |
|---|---|
| `id`, `project_id`, `work_order_id`, `technician_id` | |
| `started_at` | `DATETIME(3)` NOT NULL — **server-generated, UTC, immutable** |
| `ended_at` | `DATETIME(3)` NULL — **server-generated, UTC, immutable once set** |
| `status` | PRD §9.14: `ACTIVE`, `COMPLETED`, `CORRECTED`, `ABANDONED` |
| `ended_by_user_id` | Technician or correcting admin |
| `start_source`, `end_source` | `TECHNICIAN` or `ADMIN_CORRECTION` |
| `correction_of_session_id` | Links a correction to the original |
| `note`, audit columns | |

`CHECK (ended_at IS NULL OR ended_at >= started_at)`. At most one `ACTIVE` session per technician — `UNIQUE (technician_id, active_flag)` (Pattern B). **Multiple sessions per technician per project are normal and must not be constrained.**

**Immutability rule:** `started_at` and `ended_at` are server-written and not editable through any user-facing path. A correction inserts a **new** row with `correction_of_session_id` set, status `CORRECTED`, the corrected timestamps, and the correcting user; the original is never modified (BR-TECH-004). This makes "the technician cannot backdate" a schema fact rather than a UI convention.

**`work_session_corrections`** — `id`, `work_session_id` (original), `corrected_started_at`, `corrected_ended_at`, `reason` NOT NULL, `corrected_by`, `corrected_at`, `audit_event_id`. Append-only; both original and correction remain retrievable and attributable.

**`project_categories_attributes`** (deferred) — category-specific fields belong in a `JSON` `attributes` column, not new nullable columns per category. `JSON` is for flexible metadata only; anything filtered or aggregated must be a real column.

---

## 5. Material domain

### 5.1 `material_requirements`

| Column | Type | Notes |
|---|---|---|
| `id`, `project_id`, `item_id` | | |
| `required_quantity` | `DECIMAL(14,3)` | `CHECK (> 0)` |
| `unit` | `VARCHAR(20)` | Snapshot of unit at requirement time |
| `specification_note` | `TEXT` NULL | |
| **`stock_status`** | `VARCHAR(16)` DEFAULT `UNKNOWN` | `UNKNOWN` / `VERIFIED` (FR-MATL-002) |
| `verified_on_hand_quantity` | `DECIMAL(14,3)` NULL | Set only by verification |
| `verified_by`, `verified_at` | | |
| **`fulfilment_status`** | `VARCHAR(24)` DEFAULT `DRAFT` | PRD §9.4 |
| `is_optional` | `TINYINT(1)` DEFAULT 0 | Optional extras do not block readiness |
| audit columns | | |

**Critical constraint:** `CHECK (stock_status <> 'VERIFIED' OR (verified_on_hand_quantity IS NOT NULL AND verified_at IS NOT NULL))`. An "unverified but available" row is not representable. Enforced only on the engine floor — an older engine would parse and ignore it (NFR-DATA-009).

**`stock_verifications`** — append-only physical count: `id`, `material_requirement_id`, `observed_quantity`, `location_note`, `verified_by`, `verified_at`, `note`. Re-verification appends; the requirement reflects the latest and history stays intact.

### 5.2 `reservations`

`id`, `material_requirement_id`, `item_id` (denormalised for locking), `reserved_quantity` (`CHECK > 0`), `status` (`ACTIVE`/`RELEASED`/`CONSUMED`), `reserved_by`, `reserved_at`, `released_at`, `release_reason`, `consumed_at`.

**Invariant enforced in the transaction:** for any `item_id`, `SUM(reserved_quantity) WHERE status='ACTIVE'` must not exceed verified available quantity. Enforced by a conditional single-row `UPDATE` on the item balance (or `SELECT … FOR UPDATE` by primary key) and re-checked inside the transaction — never by a pre-check read (FR-MATL-006, BR-MATL-002).

A reservation may only reference a `VERIFIED` requirement. A plain FK cannot express this, so it is an application check plus a trigger where the engine permits (`SIGNAL SQLSTATE` in both MySQL and MariaDB). Limits are stated in AUDIT_LOGGING §5.

### 5.3 `material_movements` — the ledger

**The only table through which quantity changes.** `id`, `item_id`, `project_id` NULL, `material_requirement_id` NULL, `movement_type`, `quantity` (`DECIMAL(14,3)` signed), `unit`, `reference_type`, `reference_id`, `occurred_at`, `recorded_at`, `recorded_by`, `note`, `reversal_of_movement_id` NULL.

**Types:** `RECEIPT_IN`, `RESERVE`, `RELEASE`, `ISSUE_OUT`, `CONSUME`, `RETURN_IN`, `WRITE_OFF`, `ADJUSTMENT`.

**Sign convention:** inbound increases availability; outbound decreases it. Reservation is tracked in `reservations`, never as a movement type — a claim is not a physical event (rationale: PROCUREMENT_AND_MATERIALS §4.4).

**Prohibitions:** no `UPDATE`/`DELETE` (revoked at grant level); corrections are new rows with `reversal_of_movement_id`; no path writes a balance directly.

### 5.4 Issues, balances, reconciliation

**`material_issues`** — `id`, `project_id`, `work_order_id` NULL, `technician_id`, `issued_by`, `issued_at`, `status` (`OPEN`/`PARTIALLY_RETURNED`/`CLOSED`), `note`.

**`material_issue_lines`** — `id`, `material_issue_id`, `item_id`, `material_requirement_id` NULL, `issued_quantity`, `consumed_quantity`, `returned_quantity`, `written_off_quantity`, `unit`. `CHECK (consumed_quantity + returned_quantity + written_off_quantity <= issued_quantity)` — this makes "quantity cannot vanish" (BR-MATL-004) a database fact.

**`stock_balances`** — `item_id`, `location_id` NULL, `available_quantity`, `reserved_quantity`, `last_reconciled_at`. A **cache**, not a source of truth. If cache and ledger disagree, **the ledger wins** and the discrepancy is an incident (FR-MATL-013).

**`material_reconciliations`** — `id`, `scope` (`PROJECT`/`ITEM`), `project_id` NULL, `item_id` NULL, `expected_quantity`, `recorded_quantity`, `variance`, `run_by`, `run_at`, `resolution_note`.

---

## 6. Procurement domain

**`purchase_requests`** — `id`, `request_code` (internal, not a document number), `project_id` NULL (may span projects), `requested_by`, `requested_at`, `needed_by_date`, `status` (PRD §9.5), `submitted_by_technician` (FR-MATL-014), `notes`, audit columns.

**`purchase_request_lines`** — `id`, `purchase_request_id`, `material_requirement_id` NULL, `item_id`, `description`, `quantity`, `unit`, `estimated_unit_price`, `preferred_supplier_id` NULL.

### `purchase_approvals`

| Column | Notes |
|---|---|
| `id`, `purchase_request_id` | |
| `decision` | `APPROVED` / `REJECTED` |
| `decided_by` | **Must hold the `PEMILIK` role** |
| `decided_by_role_code` | Snapshot: `PEMILIK`. Stored because roles change later |
| `decided_at` | UTC |
| `reason` | Mandatory for rejection; optional for approval |
| `superseded_at` | NULL unless replaced |

At most one **current** decision per request — `UNIQUE (purchase_request_id, current_flag)` (Pattern B). Superseding decisions create new rows and mark the prior superseded; silent flipping is impossible.

**There is no `policy_snapshot` column.** Approval is not policy-driven (FR-PROC-011); the recorded facts are who decided, in what role, when, and why.

### `purchase_orders`

`id`, `purchase_request_id` UNIQUE (one PO per approved request), `supplier_id`, `series_key`, `document_number` NULL until issued, `status` (PRD §9.6), `issued_at`, `issued_by`, `expected_delivery_date`, `currency` (`IDR`), `subtotal`/`tax_amount`/`total_amount` (`BIGINT`), `snapshot` `JSON` (frozen supplier, **company**, and line copy), `cancelled_at`, `cancel_reason`, audit columns.

Constraints: `CHECK (document_number IS NULL OR status <> 'DRAFT')`; `UNIQUE (series_key, document_number)`. Plus the transactional half of FR-PROC-011: `CHECK (status = 'DRAFT' OR approved_request_flag = 1)` where `approved_request_flag` is a generated column, or — where a cross-table generated column is impossible — a **trigger** refusing `INSERT`/`UPDATE` of a non-`DRAFT` PO whose request has no current `APPROVED` decision. The application state guard is the other half.

**`purchase_order_lines`** — `id`, `purchase_order_id`, `item_id`, `description`, `ordered_quantity`, `received_quantity` (cached), `unit`, `unit_price`, `line_total`. `CHECK (received_quantity <= ordered_quantity)` unless an over-receipt approval reference is present (BR-PROC-003).

**`goods_receipts`** — `id`, `purchase_order_id`, `received_by`, `received_at`, `delivery_note_reference`, `status` (`DRAFT`/`POSTED`), `note`. **`goods_receipt_lines`** — `id`, `goods_receipt_id`, `purchase_order_line_id`, `received_quantity`, `discrepancy_type` NULL (`SHORT`/`DAMAGED`/`WRONG_ITEM`/`PRICE_VARIANCE`), `discrepancy_note`, `resolved_at`, `resolved_by`.

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

**`quotations`** — `id`, `customer_id`, `project_id` NULL, `series_key`, `document_number` NULL, `status` (PRD §9.10), `valid_until`, `subtotal`, `tax_amount`, `total_amount`, `snapshot` `JSON`, `issued_at`, audit columns.

**`customer_purchase_orders`** — `id`, `customer_id`, `project_id` NULL, `po_reference` (the customer's own number, **not** ours), `po_date`, `attachment_id` NULL, `amount` NULL, audit columns. Never participates in our numbering series and is never validated against it.

### `invoices`

`id`, `customer_id`, `project_id` NULL, `series_key`, `document_number` NULL until issued, `status` (PRD §9.8), `invoice_date`, `due_date`, `subtotal`/`discount_amount`/`tax_amount`/`total_amount` (`BIGINT`), `amount_paid` (cached), `snapshot` `JSON` — immutable copy **including company name, address, and bank details** (FR-FIN-003, BR-SYS-001) — `pdf_disk`, `pdf_path`, `pdf_sha256` (**disk + relative path + hash, never an absolute URL**), `issued_at`, `issued_by`, `voided_at`, `void_reason`, `credit_note_for_invoice_id` NULL, `replaced_by_invoice_id` NULL, audit columns.

Constraints: `CHECK (document_number IS NULL OR status <> 'DRAFT')`; `UNIQUE (series_key, document_number)`; `CHECK (total_amount = subtotal - discount_amount + tax_amount)`; `CHECK (due_date >= invoice_date)`.

**`invoice_lines`** — `id`, `invoice_id`, `line_no`, `description`, `quantity`, `unit`, `unit_price`, `line_total`, `item_id` NULL, `service_category_id` NULL. **No `project_id` on the line:** invoice lines describe what is billed, which may not map one-to-one to projects or items; forcing that would break non-CCTV billing (FR-PROJ-002).

**`credit_notes`** — `id`, `series_key`, `document_number`, `invoice_id` (original), `reason`, `amount`, `issued_at`, `status`, `snapshot` `JSON`.

**`payables`** — `id`, `supplier_id`, `purchase_order_id` NULL, `supplier_invoice_reference`, `document_number` NULL (internal reference if wanted), `status` (PRD §9.7), `invoice_date`, `due_date`, `amount`, `amount_paid`, `attachment_id`, `dispute_reason`, audit columns.

### `payments`

`id`, `direction` (`INBOUND`/`OUTBOUND`), `party_type`, `party_id`, `amount` (`BIGINT`), `payment_date`, `method` (`CASH`/`BANK_TRANSFER`/`OTHER`), `reference`, `status` (`RECORDED`/`CONFIRMED`/`REVERSED`), `recorded_by`, `recorded_at`, `confirmed_by`, `confirmed_at` (**distinct from recording**, BR-PROC-002), `reversal_of_payment_id` NULL, `evidence_attachment_id` NULL, `note`.

**No bank integration exists.** `method` and `reference` record what a human observed; the system never claims to have executed a transfer (BR-PROC-004).

**`payment_allocations`** — `id`, `payment_id`, `invoice_id` NULL, `payable_id` NULL, `allocated_amount`. `CHECK ((invoice_id IS NULL) <> (payable_id IS NULL))` — exactly one target. **Invariant:** for each invoice, `SUM(allocated_amount)` over non-reversed payments ≤ `total_amount`, checked under lock inside the transaction (BR-FIN-003). Overpayment becomes an unapplied credit, not a negative balance.

**`expenses`** — `id`, `category_id`, `project_id` NULL, `amount`, `expense_date`, `description`, `paid_by_user_id`, `payment_id` NULL, `attachment_id` NULL, audit columns.

**`project_cost_entries`** — one row per recognised cost event, `source_type` (`GOODS_RECEIPT`/`SUPPLIER_INVOICE`/`PAYMENT`/`EXPENSE`/`MATERIAL_ISSUE`) and `source_id`. **Double-counting control (BR-FIN-005):** `UNIQUE (project_id, source_type, source_id)`. Recognition rules decide which source type is authoritative; the others are excluded rather than added.

**[Open Question] Q-COST-01** — which source is authoritative for material cost? **Proposed default:** recognise at **goods receipt**, because the business cares about cost as soon as material exists, and reconcile to the supplier invoice when it arrives, recording any variance as a separate adjustment entry rather than silently changing history.

---

## 8. Document, numbering, attachment domain

### `document_series` — configuration per document type (FR-DOC-006, FR-SYS-004)

All of the following are **data**, editable by `SUPER_ADMIN`: `id` · `series_key` UNIQUE (e.g. `WJC:INV`) · `document_type` (`INV`, `PO`, `QUO`, `CN`, `RCP`, `WO`, `PAY`) · `company_prefix` · `format_pattern` (e.g. `{PREFIX}/{TYPE}/{YYYY}/{MM}/{SEQ}`, rendered server-side only) · `separator` · `padding` (default 5) · `starting_value` (default 1) · `increment_step` (default 1 — **not** a gap permission, BR-FIN-006) · `reset_period` (`YEAR`/`MONTH`/`NONE`) · `new_period_rule` (`CONTINUE`/`RESET_TO_START`) · `effective_from` (never retroactive) · `is_active` (deactivating a series never invalidates issued numbers).

### `number_series` — the counter row, one per series per period

`series_key`, `period_key` (e.g. `2026-10`), `next_value` (the value the **next** allocation will claim), `high_water_mark` (highest value ever allocated — **the counter may never be lowered below this**). Composite PK `(series_key, period_key)`; `CHECK (next_value > high_water_mark - 1)`. This is the row updated atomically at issuance (ADR-009). **Never read-then-write.**

### `number_allocations`

`id`, `series_key`, `period_key`, `allocated_value`, `allocated_number`, `allocated_at`, `allocated_by`, `issuing_entity_type`, `issuing_entity_id`, `status` (`ALLOCATED`/`VOIDED`/`SUPERSEDED`), `void_reason` NULL. `UNIQUE (series_key, period_key, allocated_value)` prevents double allocation of one value. Kept separate from the document so a voided document still has a permanent allocation trail (FR-FIN-011).

### `issued_documents`

`id`, `document_type`, `series_key`, `document_number`, `entity_type`, `entity_id`, `issued_at`, `issued_by`, `snapshot` `JSON`, `pdf_disk`, `pdf_path`, `pdf_sha256`, `status` (`ISSUED`/`VOID`/`SUPERSEDED`/`CREDITED`), `void_reason`, `supersedes_document_id`, `is_historical_import`.

`UNIQUE (series_key, document_number)` — **the authoritative guarantee** (NFR-DATA-002). `is_historical_import` marks migrated numbers so they are excluded from allocation and from "new number" reports (FR-DATA-004); historical numbers are preserved verbatim (BR-DATA-001).

### `attachments`

`id`, `owner_type`, `owner_id`, `disk` (e.g. `private`), `path` (relative), `original_filename`, `content_type`, `byte_size`, `sha256`, `image_width`, `image_height` NULL, `uploaded_by`, `uploaded_at`, `is_deleted` (soft), `deleted_at`.

**No absolute URL and no absolute filesystem path is stored** (FR-DOC-001, FR-DOC-009). `disk` + `path` resolve through the configured disk — what makes the S3 migration a configuration change rather than a data rewrite. An attachment owned by an issued document cannot be soft-deleted (application layer plus audit record).

---

## 9. Settings domain

**`settings`** — `id`, `group` (`mail`/`company`/`numbering`), `key`, `UNIQUE (group, key)`, `value` `TEXT` NULL (non-secret), `encrypted_value` `TEXT` NULL (encrypted cast, key from `APP_KEY`), `is_secret`, `updated_by`, `updated_at`, audit columns. **Never** store a secret in `value`, and never select `encrypted_value` into a view model that reaches a Blade template or JSON response (NFR-SEC-009).

**`company_profiles`** — `id`, `company_name` NOT NULL, `address_line` NOT NULL, `city`, `province`, `postal_code`, `phone` NOT NULL, `email` NULL, `website` NULL, `logo_disk`, `logo_path`, `logo_sha256`, `updated_by`, `updated_at`. Required fields per Q-SYS-01 proposed default: name, address, phone. Logo optional, validated (MIME, dimensions, size), stored privately.

**`bank_accounts`** — `id`, `bank_name`, `account_number`, `account_holder`, `is_primary`, `active_flag` NULL, `is_active`. Single account in MVP (Q-SYS-01); the table supports more without schema change. **Snapshotted into issued documents** (BR-SYS-001), so a later change never rewrites an invoice already sent.

---

## 10. Audit domain

**`audit_events`** — `id`, `occurred_at` `DATETIME(3)` UTC, `actor_user_id` NULL for system actions, `actor_role_codes` (`JSON` — MySQL has no array type), `action`, `entity_type`, `entity_id`, `before_state` `JSON`, `after_state` `JSON`, `reason`, `correlation_id`, `ip_address`, `override_id` NULL, `prev_hash` `CHAR(64)`, `record_hash` `CHAR(64)`.

`UPDATE`/`DELETE` revoked from the application database user. Enforcement limits — no rule system with security-definer parity — are stated in [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md) §5.

---

## 11. Derived values

| Derived value | Computed from | Cached? | Reconciliation |
|---|---|---|---|
| Invoice outstanding | `total_amount - SUM(allocations)` | `amount_paid` cached | Recompute on read; nightly check |
| Invoice overdue | `due_date < today` AND status in (`ISSUED`, `PARTIALLY_PAID`) | No | Pure function |
| Payable outstanding | `amount - SUM(allocations)` | `amount_paid` cached | Nightly check |
| Material readiness | Requirement status, receipts, issues | No | Pure function over lines |
| Available stock | Ledger sum − active reservations | `stock_balances` cache | `material_reconciliations` |
| Project cost | `project_cost_entries` | No | Unique constraint prevents double count |
| Work order progress % | Reports + completion events | No | Estimate only (FR-TECH-010) |
| Session duration | `ended_at - started_at` | No | Pure function; `NULL` while `ACTIVE` |

A cached derived value must always have a named reconciliation job. A cache without reconciliation is a future data-quality incident.

---

## 12. Referential integrity and deletion

| Relationship | On delete | Enforcement |
|---|---|---|
| Project → customer | `RESTRICT` | Cannot delete a customer with projects |
| Invoice → customer | `RESTRICT` | Cannot delete a customer with invoices |
| Requirement line → item | `RESTRICT` | Deactivate the item instead |
| Movement → item | `RESTRICT` | Ledger rows are permanent |
| Allocation → invoice | `RESTRICT` | Financial records are permanent |
| Attachment → owner | `RESTRICT` | Files of issued documents are permanent |
| Work session → technician | `RESTRICT` | Deactivate the technician |
| PO → purchase request | `RESTRICT` | Requests are permanent |

**`ON DELETE RESTRICT` is the default on every business relationship. No `CASCADE` delete exists on any business table.** InnoDB enforces FKs but has **no deferrable constraints**: inserts must be ordered correctly, and where a genuine cycle exists (e.g. project ↔ quotation) a nullable FK plus application sequencing is used.

---

## 13. Open modelling questions

| ID | Question | Proposed default |
|---|---|---|
| Q-MODEL-01 | Substitute items when the exact item is unavailable? | No in MVP; record as a line note |
| Q-MODEL-02 | Per-supplier price history? | Record prices on PO lines only |
| Q-MODEL-03 | Technician rates for cost allocation? | No; labour is not a project cost source in MVP |
| Q-MODEL-04 | Multiple projects per invoice? | Model allows it; UI default is one project per invoice |
| Q-MODEL-05 | Location dimension for stock in MVP? | Single implicit location; `location_id` reserved, unused |
| Q-MODEL-06 | Geolocation captured at session start? | No — GPS tracking is out of scope (PRD §5.3) |

---

## 14. Cross-references

- Requirements: [`../PRD.md`](../PRD.md)
- Numbering tables in context: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Material workflow: [`../workflows/PROCUREMENT_AND_MATERIALS.md`](../workflows/PROCUREMENT_AND_MATERIALS.md)
- Session and verification workflow: [`../workflows/TECHNICIAN_OPERATIONS.md`](../workflows/TECHNICIAN_OPERATIONS.md)
- Permissions: [`./SECURITY_AND_ACCESS_CONTROL.md`](./SECURITY_AND_ACCESS_CONTROL.md)
- Engine decision: [`../decisions/ADR-009-mysql-document-number-allocation.md`](../decisions/ADR-009-mysql-document-number-allocation.md)
