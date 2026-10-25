# Security and Access Control — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Target design — no implementation exists |
| Version | 0.2 — four-role model reconciled 2026-10-25 |
| Scope owner | Roles, permissions, authorisation model, separation of duties, overrides |
| Requirement source | [`../PRD.md`](../PRD.md) §4, §6, §7, §12 |
| Data model context | [`./DATA_MODEL.md`](./DATA_MODEL.md) §2 |
| Decision record | [`../decisions/ADR-005-authorisation-model.md`](../decisions/ADR-005-authorisation-model.md) |

---

## 1. Principles

| # | Principle | Consequence |
|---|---|---|
| S1 | **Deny by default.** | New routes are inaccessible until a permission is attached |
| S2 | **Authorise actions on resources, not roles in the UI.** | Hiding a button is not a control (NFR-SEC-002) |
| S3 | **Enforce server-side at two layers** — application policy plus row-scoped data access. | A missed policy check still cannot leak another technician's data |
| S4 | **Least privilege, scoped by assignment.** | A technician's default reach is their own work orders |
| S5 | **Every privileged action is attributable.** | No shared accounts; no anonymous administrative actions |
| S6 | **Separation of duties proportional to team size.** | Two-person control where practical; documented compensating control where not |
| S7 | **Conflicting permissions are permitted but controlled.** | Small-team reality is acknowledged, not pretended away |
| S8 | **Secrets never enter the repository, logs, or responses.** | NFR-SEC-007, NFR-SEC-009 |

S6 and S7 are in tension by design. With ~3 office employees, a strict "no one may approve their own purchase" rule would stall the business in its first week. The resolution is not to abandon the control but to make the conflict **visible and logged**.

**One requirement is exempt from that compromise:** `PEMILIK` purchase approval (BR-PROC-005). It is not an authorisation conflict to be overridden — it is the client's core financial control.

---

## 2. Roles

Exactly **four**. Roles are permission bundles; job titles may map to roles but are not identical. Multiple roles may be assigned to one user (NFR-SEC-003).

| Role | Purpose | Typical holder |
|---|---|---|
| `SUPER_ADMIN` | Full application access plus system administration: users and role assignment, system settings, numbering configuration, SMTP, company profile and bank accounts | System administrator |
| `PEMILIK` | Business authority: dashboards, projects, documents, financial reports, operational progress and exceptions. **Mandatory approver for every purchase request** | Business owner |
| `ADMIN` | Full operational access: customers, projects, suppliers, work orders, project materials, purchasing **preparation**, documents, permitted financial administration | Office administration |
| `TEKNISI` | Own assignments only: work sessions, daily reports, uploads, own material requests, own completion submissions, WhatsApp report preparation | ~15 field technicians |

### 2.1 Hard prohibitions per role

| Role | Cannot |
|---|---|
| `SUPER_ADMIN` | Edit an issued document; reuse or renumber a number; delete or modify an audit record; **create an approved PO without a `PEMILIK` approval**; lower a numbering counter below its high-water mark |
| `PEMILIK` | Perform designated system-destructive actions: no user/role administration, no SMTP or numbering configuration, no audit deletion, no issued-document mutation |
| `ADMIN` | **Approve purchases**; bypass the approval workflow; manage users, roles, or system settings |
| `TEKNISI` | Access unrelated projects, financial records, supplier payments, settings, or other technicians' restricted data; verify own completion; alter session timestamps |

**[Confirmed]** `SUPER_ADMIN` is bound by the same business-integrity invariants as everyone else. Administrative power is not a licence to break invariants — it is a licence to configure the system that enforces them.

### 2.2 Removed roles

| Legacy | Disposition |
|---|---|
| `FINANCE` | **Removed.** Capabilities are now granular permissions grantable to `ADMIN` and/or `SUPER_ADMIN` (`invoice.issue`, `payment.record`, `payment.confirm`, `payable.record`, `report.finance`) |
| `TEAM_LEADER` | **Removed.** Now `project_members.is_team_leader` plus two narrowly scoped permissions (§4.1) |

No permission row, workflow, or acceptance criterion retains either as a role.

---

## 3. Permission model

Permissions are `resource.action` codes stored as data (`permissions`), grouped into roles (`role_permissions`), granted per user (`user_roles`). Code calls a policy — `Gate::allows('invoice.issue', $invoice)` — never `if (auth()->user()->role === 'admin')`.

### 3.1 Permission catalogue

| Permission | Resource | Action | Notes |
|---|---|---|---|
| `customer.read` / `.write` / `.deactivate` | customer | read / create-update / deactivate | |
| `supplier.read` / `.write` | supplier | read / create-update | |
| `item.read` / `.write` | item | read / create-update | |
| `project.read.all` / `.read.assigned` | project | read company-wide / read own assignments | |
| `project.write` / `.assign` / `.transition` / `.close` / `.cancel` | project | create-update / assign + designate leader / change status / close / cancel | |
| `workorder.read.all` / `.read.own` / `.read.team` | work order | company-wide / own / own project's team | `.read.team` is team-leader only |
| `workorder.assign` / `.report.submit` | work order | assign-reassign / submit daily report | |
| `workorder.session.start` / `.end` / `.correct` | work session | start own / end own / correct a timestamp | Correction is audited, reason mandatory |
| `workorder.teamsummary.submit` | work order | submit team-level daily summary | Team-leader only |
| `workorder.completion.submit` | work order | submit completion report | |
| `workorder.verify` | work order | verify completion | **High-risk** |
| `workorder.verify.own` | work order | verify own work — **conflict permission** | Requires override |
| `material.require` / `.request.additional` / `.verify` / `.reserve` | material | create lines / raise an additional request / record verification / reserve | `.request.additional` is technician-scoped |
| `material.issue` / `.consume` / `.return` / `.writeoff` / `.reconcile` | material | issue / consume / return / write off / reconcile | |
| `purchase.request` | purchase | create-submit request | |
| `purchase.approve` | purchase | approve-reject | **`PEMILIK` only, non-delegable, non-overridable** |
| `purchase.po.prepare` / `.issue` | purchase order | create-edit a draft / issue | Issue requires an approved request |
| `purchase.receive` | goods receipt | post receipt | |
| `payable.record` | payable | record supplier invoice | |
| `payment.record` / `.confirm` / `.reverse` | payment | record / confirm / reverse | Confirm is conflict-sensitive |
| `invoice.read.all` / `.write` / `.issue` / `.void` / `.credit` | invoice | read / draft / issue (allocate number) / void / credit note | Issue and void are **high-risk** |
| `quotation.write` / `.issue` | quotation | create-edit / issue | |
| `expense.record` | expense | record | |
| `report.finance` / `.operations` | report | view financial / operational reports | |
| `document.attach` / `.read` / `.read.own` | attachment | upload / download permitted / download own | |
| `whatsapp.prepare` | report | prepare WhatsApp text | Technician-scoped |
| `audit.view` | audit | view audit log | **Restricted** |
| `settings.mail.manage` / `.company.manage` / `.numbering.manage` | settings | SMTP / company profile and banks / numbering | `SUPER_ADMIN` only |
| `user.manage` / `role.manage` | user / role | create-edit-deactivate / assign roles | `SUPER_ADMIN` only |
| `override.execute` | override | execute a logged override | `SUPER_ADMIN`, `PEMILIK` |

### 3.2 Action × resource matrix

`✓` granted · `—` denied · `◐` granted with a scope limit (assignment, own record, or project team).

| Permission group | `SUPER_ADMIN` | `PEMILIK` | `ADMIN` | `TEKNISI` |
|---|---|---|---|---|
| Customer read | ✓ | ✓ | ✓ | — |
| Customer write / deactivate | ✓ | — | ✓ | — |
| Supplier read | ✓ | ✓ | ✓ | — |
| Supplier write | ✓ | — | ✓ | — |
| Item read | ✓ | ✓ | ✓ | ◐ own issued items |
| Item write | ✓ | — | ✓ | — |
| Project read | ✓ all | ✓ all | ✓ all | ◐ assigned only |
| Project write | ✓ | — | ✓ | — |
| Project assign (incl. team leader) | ✓ | ✓ | ✓ | — |
| Project transition | ✓ | ✓ | ✓ | ◐ session-driven only (§5) |
| Project close / cancel | ✓ | ✓ | ✓ | — |
| Work order read | ✓ all | ✓ all | ✓ all | ◐ own / ◐ project team if leader |
| Work order assign | ✓ | — | ✓ | — |
| Daily report submit | ✓ | — | — | ✓ own |
| Team-level daily summary | ✓ | — | — | ◐ designated leader only |
| Work session start / end | ✓ | — | — | ✓ own |
| Work session correction | ✓ | — | ✓ | — |
| Completion submission | ✓ | — | — | ✓ own |
| Work order verify | ✓ | ✓ | ✓ | — |
| Verify own work | ◐ override | ◐ override | ◐ override | — |
| Material require | ✓ | — | ✓ | — |
| Additional material request | ✓ | — | ✓ | ◐ own project |
| Material verify / reserve / issue / write-off / reconcile | ✓ | — | ✓ | — |
| Material consume / return | ✓ | — | ✓ | ◐ own issued |
| Purchase request | ✓ | — | ✓ | — |
| **Purchase approve** | **—** | **✓ only** | **—** | **—** |
| PO prepare (draft) | ✓ | — | ✓ | — |
| PO issue | ✓ | ✓ | ✓ | — |
| Goods receipt | ✓ | — | ✓ | — |
| Payable record | ✓ | — | ✓ | — |
| Payment record | ✓ | ✓ | ✓ | — |
| Payment confirm / reverse | ✓ | ✓ | ◐ override | — |
| Invoice read | ✓ all | ✓ all | ✓ all | — |
| Invoice write / issue | ✓ | — | ✓ | — |
| Invoice void / credit | ✓ | ✓ | ◐ override | — |
| Quotation write / issue | ✓ | ✓ | ✓ | — |
| Expense record | ✓ | — | ✓ | — |
| Financial reports | ✓ | ✓ | ✓ | — |
| Operational reports | ✓ | ✓ | ✓ | ◐ own |
| Attachment upload | ✓ | — | ✓ | ◐ own reports |
| Attachment read | ✓ | ✓ | ✓ | ◐ own |
| WhatsApp report prepare | ✓ | — | — | ✓ own |
| Audit view | ✓ | ✓ | — | — |
| SMTP / company profile / numbering configuration | **✓ only** | — | — | — |
| User manage / role manage | **✓ only** | — | — | — |
| Override execute | ✓ | ✓ | — | — |

**Rationale for the notable rows:**

- **`purchase.approve` is `PEMILIK` only.** No other role holds it, `SUPER_ADMIN` included. This is the client's stated control (FR-PROC-011) and is not overridable (BR-PROC-005).
- **`ADMIN` may issue invoices but not approve purchases.** Invoicing is an operational document act; approval is a spending decision belonging to the owner. The previous model denied `ADMIN` invoicing entirely because a separate `FINANCE` role existed — with that role removed, the permission moves to `ADMIN` and remains fully audited.
- **`ADMIN` holds `payment.confirm` only via override**, because confirming a payment one recorded is a genuine conflict (C2). `PEMILIK` holds it outright — the owner is the party the control protects.
- **`SUPER_ADMIN` cannot approve purchases**, the single most important cell in this table: administrative privilege does not confer financial authority.
- **`TEKNISI` holds no `invoice.*`, `payment.*`, `purchase.*`, `supplier.*`, or settings permission at all.** A structural absence, not a scoped grant (FR-TECH-009, NFR-SEC-006).

---

## 4. Scoping model

Permissions answer "may this user perform this action at all?" Scoping answers "on which rows?"

| Scope | Applies to | Mechanism |
|---|---|---|
| Company-wide | `SUPER_ADMIN`, `PEMILIK`, `ADMIN` | No row filter beyond active status |
| Assignment-scoped | `TEKNISI` | Query layer joins `project_members` / `work_orders` on the current user's technician record |
| Project-team-scoped | Team-leader designation | Join on projects where `is_team_leader = 1 AND unassigned_at IS NULL` |
| Own-records | All roles for their own submissions | `created_by = current user` or equivalent |

**Structural requirement:** scoped repositories do not accept an arbitrary filter that could widen scope. A technician-facing query for work orders is a dedicated function that cannot return an unassigned row — not a general query with an optional `technicianId` parameter. Parameterised scope is where access-control bugs live.

### 4.1 Team-leader designation, precisely

`project_members.is_team_leader` grants exactly two extra capabilities, on that project only:

1. `workorder.read.team` — see teammates' daily reports and work orders for that project.
2. `workorder.teamsummary.submit` — submit one team-level daily summary.

It grants **no** financial access, no material-issue authority, no approval authority, no company-wide read, and no ability to verify a teammate's completion. A designation with a scoped policy, not a role. **[Open Question] Q-ROLE-01** records whether it should survive at all.

### 4.2 Technician scope boundaries

| May | May not |
|---|---|
| See own assigned projects and work orders | See other technicians' assignments |
| Start and end own work sessions | Alter any session timestamp |
| Submit own daily reports and uploads | See any invoice, payment, supplier, or settings data |
| See materials issued to them | See company-wide reports |
| Record consumption/return of own issued material | Approve purchases or issue documents |
| Raise an additional material request on own project and view its status | Change project status |
| Submit own completion report | Verify own completion |
| Prepare a WhatsApp report for own project | See other technicians' personal data |

---

## 5. Status-change authority

| Transition | Who may perform | Notes |
|---|---|---|
| `DRAFT` → `PLANNED` | `ADMIN`, `PEMILIK`, `SUPER_ADMIN` | |
| `PLANNED` → `SCHEDULED` | Same | Requires ≥1 assignment |
| `SCHEDULED` → `IN_PROGRESS` | Same | A technician may start a **session** on their work order, not the project |
| `IN_PROGRESS` → `ON_HOLD` | Same, with reason | Blocker must be recorded |
| `ON_HOLD` → `IN_PROGRESS` | Same | |
| `IN_PROGRESS` → `PENDING_COMPLETION_VERIFICATION` | System-derived when all work orders are terminal | Not a manual action |
| `PENDING_COMPLETION_VERIFICATION` → `COMPLETED` | Same (verification) | BR-TECH-002 applies |
| `COMPLETED` → `CLOSED` | Same | Closure conditions (BR-PROJ-004) |
| Any pre-verification → `CANCELLED` | `PEMILIK`, `ADMIN`, with reason | Records retained (BR-PROJ-005) |

**Work-session transitions** are the technician's own: start and end. `workorder.session.correct` belongs to `ADMIN`/`SUPER_ADMIN` and always produces a correction record (BR-TECH-004). Technicians act on **work orders and sessions**, not on project status — project state remains a management fact.

---

## 6. Separation of duties and compensating controls

### 6.1 Conflicts that matter

| # | Conflict | Risk | Control |
|---|---|---|---|
| C1 | Same person approves a purchase and issues the PO | Unapproved spending presented as approved | **Structurally impossible for `ADMIN`** (no approval permission). `PEMILIK` approving and `ADMIN` issuing is the normal, separated path |
| C2 | Same person records and confirms a payment | Fabricated or unverified payment | `payment.confirm` distinct; `ADMIN` holding both requires override + `PEMILIK` visibility |
| C3 | Same person issues an invoice and voids it | Concealment of billing | `invoice.void` requires override when the voider is the issuer; both audited |
| C4 | Technician verifies own completed work | Unverified quality presented as verified | `workorder.verify.own` requires override with reason |
| C5 | Same person records a supplier invoice and pays it | Fictitious payable paid out | Distinct permissions; override if both held by `ADMIN` |
| C6 | Owner unavailable | Approvals stall, because approval is not delegable | Accepted for MVP; documented as an operational risk (BR-PROC-005) |

### 6.2 The override mechanism

1. The system blocks the action with a message naming the conflict.
2. The user may proceed if they hold `override.execute` — in practice `SUPER_ADMIN` or `PEMILIK`.
3. The override requires a **reason** (minimum length, free text) and records actor, action, target, reason, timestamp.
4. An `overrides` row is written and an audit event created with `is_override = true`.
5. The override appears in a dedicated `PEMILIK`-facing report. Unreviewed overrides older than a configured period are surfaced.

**This is the honest resolution.** A three-person office cannot have four-eyes control on every action. What it can have is a complete, visible, attributable record of every bypass. The control shifts from prevention to detection, and detection requires the owner to look — which is why the override report is a first-class deliverable.

**Hard limits — restated because this is the boundary most likely to erode:**

| Override may permit | Override may **never** permit |
|---|---|
| Verifying one's own completed work | Issuing a duplicate document number |
| Confirming a payment one recorded | Reserving more stock than is verified |
| Voiding an invoice one issued | Editing an issued document or its snapshot |
| Starting a project before materials are ready | Deleting or modifying an audit record |
| Closing a project with acknowledged exceptions | Exceeding an issue line's arithmetic |
| — | **Issuing a PO without `PEMILIK` approval (BR-PROC-005)** |
| — | **Backdating a session timestamp (BR-TECH-004)** |

An override bypasses an **authorisation conflict**, never a **data-integrity invariant**, and never the client's purchase-approval requirement.

---

## 7. Authentication

| Control | Requirement |
|---|---|
| Password hashing | Argon2id (preferred) or bcrypt with a documented cost; never MD5/SHA-1/plain |
| Password policy | Minimum 10 characters; reject known-breached passwords where a list is available; no forced rotation |
| Login throttling | Progressive delay and lockout after repeated failures; time-based and logged |
| Session | Server-side, idle expiry (default 8 hours, configurable), absolute expiry, revoked on password change and deactivation |
| Session fixation | New session identifier issued at login |
| Cookies | `HttpOnly`, `Secure`, `SameSite=Lax`; no business data in cookies |
| Logout | Server-side invalidation, not just cookie clearing |
| CSRF | Token on every state-changing request; refreshed after Turbo cache restoration (ARCHITECTURE §3.2) |
| MFA | **[Out of Scope]** for MVP; the session model must not preclude it |
| Technician login | Username + password where email is unavailable; password reset via office-administered reset, not email-only |

---

## 8. Secrets and configuration

- Secrets (database credentials, storage keys, signing keys, **SMTP passwords**) come from environment variables or the encrypted settings store. **Never committed** (NFR-SEC-007).
- SMTP credentials are **encrypted at rest** with a key derived from `APP_KEY`, are **write-only** in the UI, and never appear in page source, client JavaScript, logs, error output, or any response body (NFR-SEC-009).
- No secret is logged, returned, or included in an error message. Logs redact known-sensitive fields by pattern (password, token, key, authorization header, `mail.password`).
- Database backups are encrypted at rest and in transit.
- Production configuration access is limited to `SUPER_ADMIN` and the implementer; changes are audited (FR-SYS-001…004).

### 8.1 SMTP recovery when invalid settings block delivery

Invalid SMTP configuration can lock out every outbound email, including password resets. Documented recovery, in order:

1. **Environment fallback** — if the stored configuration is marked invalid or fails validation, the application falls back to the environment-configured transport so the system stays usable.
2. **Console override** — a command clears or resets the stored mail configuration for an operator with server access; the action is audited.
3. **Break-glass** — documented manual intervention on the settings row by a database operator, recorded in the operational log.

No path bypasses authentication, and none exposes the stored password.

---

## 9. Audit of security-relevant events

Login success/failure, lockout, logout · password change and administrative reset · role assignment and revocation · user creation, deactivation, reactivation · permission changes · every override · every denied authorisation attempt on a sensitive action (invoice issue, payment confirm, purchase approve, settings change, audit view) · every audit-log view (NFR-AUD-006) · **every settings change: SMTP, company profile, bank accounts, numbering** — recorded without the secret values.

---

## 10. Acceptance obligations

| Obligation | AC |
|---|---|
| No anonymous data access | AC-SEC-001 |
| Direct requests by unauthorised roles rejected | AC-SEC-002 |
| Multi-role union computed correctly | AC-SEC-003 |
| Session expiry and revocation | AC-SEC-004 |
| Password hashing and policy | AC-SEC-005 |
| Technician cannot reach finance, settings, or company-wide endpoints | AC-SEC-006 |
| No secrets in repo or logs | AC-SEC-007 |
| TLS enforced in production | AC-SEC-008 |
| SMTP credentials encrypted, write-only, never exposed | AC-SEC-009 |
| Private files unreachable without authorisation; streamed with `Content-Disposition` | AC-DOC-010 |
| Override requires reason and is visible to `PEMILIK` | AC-AUD-004 |
| `ADMIN` cannot approve a purchase by any route | AC-PROC-011 |
| Issued documents cannot be mutated by any role | AC-FIN-003 |

---

## 11. Known limitations

1. **`SUPER_ADMIN` can do almost anything operational.** The protections are audit, the invariant boundary, and the absence of approval authority. Inherent to an owner-operated business with an administrator.
2. **Override makes conflicts detectable, not impossible.** A determined insider can act and then explain. The system provides evidence, not prevention.
3. **No delegation of purchase approval.** If the owner is unavailable, purchases wait. A deliberate, client-stated consequence of BR-PROC-005.
4. **No IP allow-listing or device binding.** Field technicians work from arbitrary mobile networks.
5. **No field-level encryption** beyond the settings store. **[Open Question]** whether supplier bank details warrant stronger protection — the system stores no bank credentials, only account numbers for display on documents.
6. **`SUPER_ADMIN` and `ADMIN` may be the same person.** The model supports it; the override and audit trail are what make it survivable.

---

## 12. Cross-references

- Roles and workflows: [`../PRD.md`](../PRD.md) §4
- Entities: [`./DATA_MODEL.md`](./DATA_MODEL.md) §2
- Audit mechanics: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)
- Architectural enforcement: [`./ARCHITECTURE.md`](./ARCHITECTURE.md) §4
- Decision: [`../decisions/ADR-005-authorisation-model.md`](../decisions/ADR-005-authorisation-model.md)
