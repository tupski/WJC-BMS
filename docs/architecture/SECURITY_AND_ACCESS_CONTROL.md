# Security and Access Control — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed — no implementation exists |
| Version | 0.1 |
| Scope owner | This document owns roles, permissions, authorisation model, separation of duties, overrides |
| Requirement source | [`../PRD.md`](../PRD.md) §6.1–6.9, §7, §12 |
| Data model context | [`./DATA_MODEL.md`](./DATA_MODEL.md) |

---

## 1. Principles

| # | Principle | Consequence |
|---|---|---|
| S1 | **Deny by default.** A permission not granted is denied. | New endpoints are inaccessible until a permission is explicitly attached |
| S2 | **Authorise actions on resources, not roles in the UI.** | Hiding a button is not a control (NFR-SEC-002) |
| S3 | **Enforce server-side at two layers.** Application-layer policy check plus row-scoped data access. | A missed policy check still cannot leak another technician's data |
| S4 | **Least privilege, scoped by assignment.** | A technician's default reach is their own work orders |
| S5 | **Every privileged action is attributable.** | No shared accounts; no anonymous administrative actions |
| S6 | **Separation of duties proportional to team size.** | Two-person control where practical; documented compensating control where not |
| S7 | **Conflicting permissions are permitted but controlled.** | Small-team reality is acknowledged, not pretended away |
| S8 | **Secrets never enter the repository, logs, or API responses.** | NFR-SEC-007 |

S6 and S7 are in tension by design. With ~3 office employees, a strict "no one may approve their own purchase" rule would stall the business the first week. The resolution is not to abandon the control but to make the conflict **visible and logged**.

---

## 2. Roles

Roles are permission bundles. Job titles may map to roles but are not identical. Multiple roles may be assigned to one user (NFR-SEC-003).

| Role code | Purpose | Typical holder |
|---|---|---|
| `OWNER` | Full business authority: approvals, permission administration, audit access, overrides | Business owner |
| `OPS_ADMIN` | Day-to-day operations: customers, projects, scheduling, material requirements, purchase requests, PO preparation | Office administration |
| `FINANCE` | Invoicing, payments, payables, expenses, financial reports | Finance/administrative finance |
| `TECHNICIAN` | Own assignments and reports only | ~15 field technicians |
| `TEAM_LEADER` | Technician scope plus review of that project's team reports | Optional per-project designation |

**[Assumption]** `FINANCE` may not exist as a separate person. In a 3-person office, the same individual may hold `OPS_ADMIN` + `FINANCE`. §6 defines the compensating controls.

**[Open Question] Q-ROLE-01** (PRD §14): whether `TEAM_LEADER` should be a role at all or merely a per-project designation. **Proposed default:** implement as a per-project designation (`project_assignments.role_on_project = 'TEAM_LEADER'`) that grants a narrow review permission scoped to that project, rather than a company-wide role. This avoids inventing an org chart that may not exist while still allowing one accountable person on site.

---

## 3. Permission model

Permissions are `resource.action` codes stored as data (`permissions` table), grouped into roles (`role_permissions`), and granted per user (`user_roles`). Code checks `can(user, 'invoice.issue', invoice)` — never `if user.role == 'owner'`.

### 3.1 Permission catalogue

| Permission | Resource | Action | Notes |
|---|---|---|---|
| `customer.read` | customer | read | |
| `customer.write` | customer | create/update | |
| `customer.deactivate` | customer | deactivate | |
| `supplier.read` | supplier | read | |
| `supplier.write` | supplier | create/update | |
| `item.read` | item | read | |
| `item.write` | item | create/update | |
| `project.read.all` | project | read (company-wide) | |
| `project.read.assigned` | project | read (own assignments) | Technician scope |
| `project.write` | project | create/update | |
| `project.assign` | project | assign technicians | |
| `project.transition` | project | change status | |
| `project.close` | project | close | |
| `project.cancel` | project | cancel | |
| `workorder.read.all` | work order | read company-wide | |
| `workorder.read.own` | work order | read own | |
| `workorder.assign` | work order | assign/reassign | |
| `workorder.report.submit` | work order | submit daily report | |
| `workorder.verify` | work order | verify completion | |
| `workorder.verify.own` | work order | verify own work — **conflict permission** | Requires override |
| `material.require` | material | create requirement lines | |
| `material.verify` | material | record stock verification | |
| `material.reserve` | material | create/release reservation | |
| `material.issue` | material | issue to technician | |
| `material.consume` | material | record consumption | |
| `material.return` | material | record return | |
| `material.writeoff` | material | record damage/write-off | |
| `material.reconcile` | material | run reconciliation | |
| `purchase.request` | purchase | create/submit request | |
| `purchase.approve` | purchase | approve/reject | **Conflict-sensitive** |
| `purchase.po.issue` | purchase order | issue PO | |
| `purchase.receive` | goods receipt | post receipt | |
| `payable.record` | payable | record supplier invoice | |
| `payment.record` | payment | record payment | |
| `payment.confirm` | payment | confirm payment | **Conflict-sensitive** |
| `payment.reverse` | payment | reverse payment | |
| `invoice.read.all` | invoice | read company-wide | |
| `invoice.write` | invoice | create/edit draft | |
| `invoice.issue` | invoice | issue (allocate number) | **High-risk** |
| `invoice.void` | invoice | void | **High-risk** |
| `invoice.credit` | invoice | issue credit note | |
| `quotation.write` | quotation | create/edit | |
| `quotation.issue` | quotation | issue | |
| `expense.record` | expense | record | |
| `report.finance` | report | view financial reports | |
| `report.operations` | report | view operational reports | |
| `document.attach` | attachment | upload | |
| `document.read` | attachment | download permitted files | |
| `audit.view` | audit | view audit log | **Restricted** |
| `user.manage` | user | create/edit users | |
| `role.manage` | role | assign roles / change permissions | **Owner only** |
| `override.execute` | override | execute a logged override | **Owner only** |

### 3.2 Permission matrix

`✓` = granted · `—` = denied · `◐` = granted with scope limit

| Permission group | OWNER | OPS_ADMIN | FINANCE | TECHNICIAN | TEAM_LEADER (on project) |
|---|---|---|---|---|---|
| Customer read | ✓ | ✓ | ✓ | — | — |
| Customer write/deactivate | ✓ | ✓ | — | — | — |
| Supplier read | ✓ | ✓ | ✓ | — | — |
| Supplier write | ✓ | ✓ | — | — | — |
| Item read | ✓ | ✓ | ✓ | ◐ own issued items | ◐ |
| Item write | ✓ | ✓ | — | — | — |
| Project read | ✓ all | ✓ all | ✓ all | ◐ assigned only | ◐ assigned only |
| Project write | ✓ | ✓ | — | — | — |
| Project assign | ✓ | ✓ | — | — | — |
| Project transition | ✓ | ✓ | — | ◐ limited (see §5) | ◐ limited |
| Project close | ✓ | ✓ | — | — | — |
| Project cancel | ✓ | ✓ | — | — | — |
| Work order read | ✓ all | ✓ all | — | ◐ own | ◐ project team |
| Work order assign | ✓ | ✓ | — | — | — |
| Daily report submit | ✓ | — | — | ✓ own | ✓ own |
| Work order verify | ✓ | ✓ | — | — | ◐ project team |
| Verify own work | ◐ override | ◐ override | — | — | — |
| Material require | ✓ | ✓ | — | — | — |
| Material verify | ✓ | ✓ | — | — | — |
| Material reserve | ✓ | ✓ | — | — | — |
| Material issue | ✓ | ✓ | — | — | — |
| Material consume | ✓ | ✓ | — | ◐ own issued | ◐ project team |
| Material return | ✓ | ✓ | — | ◐ own issued | ◐ project team |
| Material write-off | ✓ | ✓ | — | — | — |
| Material reconcile | ✓ | ✓ | — | — | — |
| Purchase request | ✓ | ✓ | — | — | — |
| Purchase approve | ✓ | ◐ override | — | — | — |
| PO issue | ✓ | ✓ | — | — | — |
| Goods receipt | ✓ | ✓ | — | — | — |
| Payable record | ✓ | — | ✓ | — | — |
| Payment record | ✓ | — | ✓ | — | — |
| Payment confirm | ✓ | — | ◐ override | — | — |
| Payment reverse | ✓ | — | ◐ override | — | — |
| Invoice read | ✓ all | ✓ all | ✓ all | — | — |
| Invoice write/issue | ✓ | — | ✓ | — | — |
| Invoice void / credit | ✓ | — | ◐ override | — | — |
| Quotation write/issue | ✓ | ✓ | ◐ | — | — |
| Expense record | ✓ | ◐ | ✓ | — | — |
| Financial reports | ✓ | — | ✓ | — | — |
| Operational reports | ✓ | ✓ | ✓ | ◐ own | ◐ project |
| Attachment upload | ✓ | ✓ | ✓ | ◐ own reports | ◐ project |
| Attachment read | ✓ | ✓ | ✓ | ◐ own | ◐ project |
| Audit view | ✓ | — | — | — | — |
| User manage | ✓ | — | — | — | — |
| Role manage | ✓ | — | — | — | — |
| Override execute | ✓ | — | — | — | — |

**Rationale for notable denials:**

- `OPS_ADMIN` cannot issue invoices. Invoicing is a financial act with legal and tax consequences; the office administrator prepares drafts and hands them to `FINANCE`/`OWNER`. This is the strongest available separation in a small team.
- `FINANCE` cannot approve purchases. Approval is a spending decision belonging to the owner (or the ops function, not the payer).
- `TECHNICIAN` cannot read any invoice, payment, supplier, or other technician's data (FR-TECH-009, NFR-SEC-006). This is a hard boundary, not a soft preference.
- `OPS_ADMIN` has `purchase.approve` only as an override, because the confirmed model is that the **owner** approves purchases.

---

## 4. Scoping model

Permissions answer "may this user perform this action at all?" Scoping answers "on which rows?"

| Scope | Applies to | Mechanism |
|---|---|---|
| Company-wide | Owner, Ops Admin, Finance | No row filter beyond active status |
| Assignment-scoped | Technician | Query layer joins `project_assignments` / `workorders` on `technician_id = current user's technician` |
| Project-team-scoped | Team Leader | Join on projects where the user is the active team leader |
| Own-records | All roles for their own submissions | `created_by = current user` or equivalent |

**Structural requirement:** scoped repositories do not accept an arbitrary filter that could widen scope. A technician-facing query for work orders is a dedicated function that cannot return an unassigned row, rather than a general query with an optional `technicianId` parameter. Parameterised scope is where access-control bugs live.

---

## 5. Status-change authority

Not every role that can read a project may move it. Transition authority is a separate permission dimension:

| Transition | Who may perform | Notes |
|---|---|---|
| `DRAFT` → `PLANNED` | `OPS_ADMIN`, `OWNER` | |
| `PLANNED` → `SCHEDULED` | `OPS_ADMIN`, `OWNER` | Requires ≥1 assignment |
| `SCHEDULED` → `IN_PROGRESS` | `OPS_ADMIN`, `OWNER`, assigned `TEAM_LEADER` | Technician may start own work order but not the project |
| `IN_PROGRESS` → `ON_HOLD` | `OPS_ADMIN`, `OWNER`, assigned `TEAM_LEADER` (with reason) | Blocker must be recorded |
| `ON_HOLD` → `IN_PROGRESS` | `OPS_ADMIN`, `OWNER` | |
| `IN_PROGRESS` → `PENDING_COMPLETION_VERIFICATION` | System-derived when all work orders reach a terminal state | Not a manual action |
| `PENDING_COMPLETION_VERIFICATION` → `COMPLETED` | `OPS_ADMIN`, `OWNER` (verification) | BR-TECH-002 applies |
| `COMPLETED` → `CLOSED` | `OPS_ADMIN`, `OWNER` | Closure conditions (BR-PROJ-004) |
| Any → `CANCELLED` | `OWNER`, `OPS_ADMIN` with reason | Records retained (BR-PROJ-005) |

Technicians act on **work orders**, not on project status. This keeps project state a management fact rather than a field-level opinion.

---

## 6. Separation of duties and compensating controls

### 6.1 Conflicts that matter

| # | Conflict | Risk | Control |
|---|---|---|---|
| C1 | Same person approves a purchase and issues the PO | Unapproved spending presented as approved | Approval decision and PO issuance are separate permissions; `OPS_ADMIN` cannot approve without override |
| C2 | Same person records and confirms a payment | Fabricated or unverified payment | `payment.confirm` is a distinct permission; if held by the recorder, override + owner visibility required |
| C3 | Same person issues an invoice and voids it | Concealment of billing | `invoice.void` requires override when the voider is the issuer; both actions audited |
| C4 | Technician verifies own completed work | Unverified quality presented as verified | `workorder.verify.own` requires override with reason |
| C5 | Same person records a supplier invoice and pays it | Fictitious payable paid out | Distinct permissions; override if combined |
| C6 | Owner is the only approver and is unavailable | Business stalls | Delegation is **[Out of Scope]** for MVP; documented as an operational risk rather than solved by weakening approval |

### 6.2 The override mechanism

Where a user holds both sides of a conflict (small-team reality), the action is permitted only through an explicit override:

1. The system blocks the action with a clear message naming the conflict.
2. The user may request an override if they hold `override.execute` — in practice, `OWNER` only.
3. The override requires a **reason** (minimum length, free text) and records: actor, action, target, reason, timestamp.
4. An `overrides` row is written and an audit event is created with `is_override = true`.
5. The override appears in a dedicated owner-facing report. Unreviewed overrides older than a configured period are surfaced.

**This is the honest resolution.** A three-person office cannot have four-eyes control on every action. What it can have is a complete, visible, attributable record of every time the control was bypassed. The control shifts from prevention to detection, and detection requires the owner to actually look — which is why the override report is a first-class deliverable, not a log entry buried in an audit table.

**What override must never do:** bypass a data-integrity rule (e.g. issue a duplicate number, exceed reserved stock, edit an issued invoice). Override bypasses **authorisation conflicts**, never **invariants**. This distinction is critical and is restated as a hard invariant in [`../../AGENTS.md`](../../AGENTS.md).

---

## 7. Authentication

| Control | Requirement |
|---|---|
| Password hashing | Argon2id (preferred) or bcrypt with documented cost; never MD5/SHA-1/plain | 
| Password policy | Minimum 10 characters; reject known-breached passwords where a list is available; no forced rotation (modern guidance — rotation drives weak passwords) |
| Login throttling | Progressive delay and lockout after repeated failures; lockout is time-based and logged |
| Session | Server-side session, idle expiry (default 8 hours, configurable), absolute expiry, revoked on password change and on deactivation |
| Session fixation | New session identifier issued at login |
| Cookies | `HttpOnly`, `Secure`, `SameSite=Lax`; no business data in cookies |
| Logout | Server-side invalidation, not just cookie clearing |
| MFA | **[Out of Scope]** for MVP; the session model must allow adding it without redesign |
| Technician login | Username + password acceptable where email is unavailable; password reset via office-administered reset, not email-only |

**[Assumption]** Technicians can log in with a username rather than an email address, since not all field staff have a work email. The model supports either (`users.email` or `users.username`).

---

## 8. Secrets and configuration

- Secrets (database credentials, storage keys, signing keys) come from environment variables or a managed secret store. **Never committed** (NFR-SEC-007).
- No secret is logged, returned in an API response, or included in an error message.
- Application logs redact known-sensitive fields by pattern (password, token, key, authorization header).
- Database backups are encrypted at rest and in transit.
- Access to production configuration is limited to the owner and the implementer, and any change is documented outside the application.

---

## 9. Audit of security-relevant events

Every event below produces an audit record (see [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)):

- Login success and failure, lockout, logout.
- Password change and administrative password reset.
- Role assignment and revocation.
- Permission changes to roles.
- User creation, deactivation, reactivation.
- Every override execution.
- Every denied authorisation attempt on a sensitive action (invoice issue, payment confirm, approval, audit view) — denied attempts are evidence, not noise.
- Every audit-log view (NFR-AUD-006).

---

## 10. Security acceptance obligations

Mapped to `AC-` entries in [`../implementation/ACCEPTANCE_CRITERIA.md`](../implementation/ACCEPTANCE_CRITERIA.md):

| Obligation | AC |
|---|---|
| No anonymous data access | AC-SEC-001 |
| Direct API calls by unauthorised roles rejected | AC-SEC-002 |
| Multi-role accounts compute union permissions correctly | AC-SEC-003 |
| Session expiry and revocation | AC-SEC-004 |
| Password hashing and policy | AC-SEC-005 |
| Technician cannot reach finance endpoints | AC-SEC-006 |
| No secrets in repo or logs | AC-SEC-007 |
| TLS enforced in production | AC-SEC-008 |
| Override requires reason and is visible to owner | AC-AUD-004 |
| Issued documents cannot be mutated by any role | AC-FIN-003 |

---

## 11. Known limitations of this model

Stated explicitly so nobody discovers them during an audit:

1. **The owner can do anything.** There is no technical control above `OWNER`. The only protections are audit logging and the fact that the owner is the party whose interests the controls protect. This is inherent to an owner-operated business.
2. **Override makes conflicts detectable, not impossible.** A determined insider with owner credentials can act and then explain. The system provides evidence, not prevention, at that level.
3. **No delegation.** If the owner is unavailable, approvals wait. Accepted for MVP; see C6.
4. **No IP allow-listing or device binding.** Field technicians work from arbitrary mobile networks, so network-based restrictions are impractical.
5. **No field-level encryption.** Sensitive fields are protected by access control and encryption at rest, not by per-field encryption. **[Open Question]** whether any field (e.g. supplier bank account) warrants stronger protection — not currently modelled because the system stores no bank credentials.

---

## 12. Cross-references

- Roles and workflows: [`../PRD.md`](../PRD.md) §4
- Entities: [`./DATA_MODEL.md`](./DATA_MODEL.md) §2
- Audit mechanics: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)
- Architectural enforcement: [`./ARCHITECTURE.md`](./ARCHITECTURE.md) §4
