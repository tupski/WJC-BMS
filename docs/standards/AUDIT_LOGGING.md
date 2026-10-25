# Audit Logging Standard — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Target design |
| Version | 0.2 — enforcement limits on MySQL/MariaDB stated 2026-10-25 |
| Scope owner | The **single authority** on what is audited, how records are written, and how they are protected |
| Requirement source | [`../PRD.md`](../PRD.md) §6.9, NFR-AUD-001…006, NFR-SEC-009, BR-AUD-001 |
| Data model | [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §10 |
| Decision record | [`../decisions/ADR-004-audit-logging-approach.md`](../decisions/ADR-004-audit-logging-approach.md) |

---

## 1. Purpose

The audit log answers questions the business currently cannot: who approved this purchase · who changed this target date, and why · who issued this invoice, and was it later voided by someone else · did anyone override a blocked workflow, and with what reason · was this payment recorded and confirmed by two different people · who changed the SMTP settings or the numbering configuration · who corrected a work session's timestamps, and why.

**[Assumption]** P7 in the PRD: today these questions are answered by scrolling WhatsApp and guessing.

### 1.1 What the audit log is not

| Not this | Why |
|---|---|
| Application debug logging | Debug logs diagnose faults; audit logs establish accountability. Different retention, access, format |
| A business history report | Business timelines are separate, queryable tables (DATA_MODEL §11) |
| A backup | Audit records are not a substitute for data backup |
| Tamper-proof | It is **tamper-evident**. A determined party with database access can alter data; hash chaining makes alteration detectable |

Claiming "tamper-proof" would be false. The realistic guarantee is that modification is *detectable*, which is what matters for dispute resolution.

---

## 2. What is audited

### 2.1 Mandatory events

| Category | Events |
|---|---|
| **Authentication** | Login success, failure, lockout, logout, password change, administrative reset, session revocation |
| **Authorisation** | Role assignment, revocation, permission change, user creation, deactivation, reactivation |
| **Denied access** | Authorisation denial on sensitive actions: invoice issue, invoice void, payment confirm, purchase approve, settings change, audit view, permission change |
| **Project** | Create, status transition, schedule revision, assignment, unassignment, team-leader designation and removal, closure, cancellation |
| **Work order** | Create, assign, reassign, status transition, completion submission, verification outcome, return for correction |
| **Work session** | Start, end, abandonment, **every timestamp correction with before/after values and reason** |
| **Daily report** | Submission, supersession, review, rejection |
| **Material** | Requirement creation/change, technician additional request, stock verification, invalidation of verification, reservation create/release/consume, issue, consumption, return, write-off, reconciliation run |
| **Procurement** | Request submit, **approval decision including the decider's role**, rejection, PO issuance attempt refused for lack of approval, PO issuance, PO cancellation, receipt posting, discrepancy resolution, over-receipt approval |
| **Finance** | Quotation issuance, invoice issuance, void, credit note, payable recording, payment recording, confirmation, reversal, expense recording, allocation change |
| **Documents** | Number allocation, number void, series configuration change, PDF generation, attachment upload, deletion, download of an issued document |
| **Settings** | **SMTP change (values redacted), connection test, test email, company profile change, bank account change, numbering configuration change** |
| **Overrides** | Every override execution, with reason |
| **Audit** | Every audit-log view (self-auditing) |
| **Data** | Import validation, commit, rollback, export |
| **System** | Backup completion, restore execution, migration applied, reconciliation variance detected, service-worker cache version released |

### 2.2 What is not audited

Read access to ordinary business records (volume would swamp the log; sensitive reads are audited) · UI navigation and page views · draft field edits before issuance · local draft keystrokes in the technician form (client-side only, never reaches the server) · cached value refreshes · health checks · **SMTP password values** — the *change* is audited, the *value* is not (NFR-SEC-009).

**The distinction that matters:** drafting an invoice is not audited; issuing it is. Editing a draft is ordinary work; issuing is a commitment.

---

## 3. Record structure

| Field | Required | Purpose |
|---|---|---|
| `id` | Yes | Unique identifier |
| `occurred_at` | Yes | `DATETIME(3)`, **UTC**; rendered in `Asia/Jakarta` |
| `actor_user_id` | Yes (nullable only for system actions) | Who |
| `actor_role_codes` | Yes | Effective roles **at the time** — roles change later |
| `action` | Yes | Namespaced verb (e.g. `invoice.issue`) |
| `entity_type`, `entity_id` | Yes | Target |
| `before_state`, `after_state` | Conditional | Changed fields only |
| `reason` | Conditional | Required for overrides, voids, cancellations, rejections, session corrections |
| `override_id` | Conditional | Link to the override record |
| `correlation_id` | Yes | Request trace identifier |
| `ip_address` | Where available | Source of the action |
| `user_agent` | Optional | Client context |
| `prev_hash`, `record_hash` | Yes | Hash chain |

**Why `actor_role_codes` is stored:** if a user is later demoted or promoted, "they were `PEMILIK` at the time" must remain answerable. Joining to current role tables would rewrite history retroactively — the exact failure the audit log exists to prevent. MySQL has no array type, so this is a `JSON` array.

**Why before/after are diff-only:** full snapshots multiply storage and make diffs unreadable. Changed fields only keeps records small and the change obvious; full state is reconstructable by replaying diffs.

**Action naming:** `<entity>.<verb>` (`invoice.issue`, `project.transition`, `payment.confirm`) or `<entity>.<verb>.<qualifier>` (`workorder.verify.override`, `material.verification.invalidate`, `worksession.correct`).

---

## 4. Transactional coupling

**An audit record is written in the same database transaction as the state change it describes** (FR-AUD-003, BR-AUD-001).

```
BEGIN;
  UPDATE invoices SET status='ISSUED', ... WHERE id = ?;
  INSERT INTO audit_events (...) VALUES (...);
COMMIT;
```

| Approach | Failure mode |
|---|---|
| Write after commit, in application code | Crash between commit and audit write → permanently unattributable change |
| Write to a queue, process later | Queue loss, ordering ambiguity, lag during incident investigation |
| Write in the same transaction | Atomic; slightly more lock contention; **correct** |

The trade-off is accepted deliberately: audit writes add one row insert per transaction, which is irrelevant at ~18 users. Correctness under crash is not.

**Enforcement layers**

| Layer | Mechanism |
|---|---|
| Application | A single audit service writes within the transaction; no ad-hoc inserts |
| Database | `BEFORE UPDATE`/`BEFORE DELETE` triggers on `audit_events` raise `SIGNAL SQLSTATE '45000'` |
| Database | `AFTER INSERT`/`AFTER UPDATE` triggers on high-risk tables assert an audit row exists and abort otherwise |
| Privileges | `UPDATE`/`DELETE` not granted to the application database user |
| Review | Every new state-changing action must include an audit write, enforced by checklist and a per-transition test |

**[Proposed]** Database triggers on the highest-risk tables: `invoices`, `payments`, `purchase_orders`, `purchase_approvals`, `issued_documents`, `number_allocations`, `work_sessions`, `settings`. Lower-risk tables rely on application-level writes plus test coverage.

---

## 5. Immutability — what MySQL/MariaDB can and cannot enforce

This section exists because the earlier design assumed capabilities the mandated engine does not have. It is stated plainly so nobody later believes a stronger guarantee exists than actually does.

| Control | Mechanism | Strength |
|---|---|---|
| No update / delete by the application | Privileges not granted to the application database user | **Strong** — enforced by the server |
| No update / delete even by a privileged session | `BEFORE UPDATE`/`BEFORE DELETE` triggers raising an error | **Strong against accidents and application bugs** |
| No insert bypassing the audit service | Application discipline plus `AFTER` triggers on high-risk tables | **Moderate** — a direct `INSERT` by a privileged user is possible |
| Retention archival | Moves rows to cold storage; never deletes | **Strong if the job is correct** |

### 5.1 What is weaker than a PostgreSQL design would be

| Limitation | Consequence |
|---|---|
| No rule system with PostgreSQL's `CREATE RULE … DO INSTEAD NOTHING` semantics | Cannot make a table reject `DELETE` declaratively without a trigger |
| Triggers are **not** `SECURITY DEFINER` in the PostgreSQL sense; they run with the invoking user's privileges | A sufficiently privileged user (`root`, a DBA account) can drop the trigger and then mutate rows |
| A DDL-capable user can `DROP TRIGGER` | Immutability is enforced against the **application path**, not a hostile DBA |
| `CHECK` constraints only enforced from MySQL 8.0.16 / MariaDB 10.2.1 | An older engine accepts the schema and ignores the checks (NFR-DATA-009) |

**What this means honestly.** The log is protected against application bugs, ORM misuse, accidental admin scripts, and casual tampering. It is **not** protected against a determined attacker with full database credentials — who can also rewrite the hash chain, because the hashing key lives in the application environment. Closing that gap requires **external anchoring**, which is **[Out of Scope]** for MVP and recorded as **[Open Question] Q-AUD-02**.

**Stated consequence:** no document in this set may claim the audit log is tamper-proof, immutable against a DBA, or legally conclusive. It is tamper-**evident** and attributable.

### 5.2 Verification test

An integration test attempts `UPDATE audit_events SET action='x'` and `DELETE FROM audit_events` **using the application database user** and asserts both fail. A second test attempts the same as a privileged user and asserts the trigger fires. Both run on every build, because a migration or a grant change could silently restore the privilege.

### 5.3 Hash chaining

Each record stores `prev_hash` and `record_hash` (hash of this record's canonical fields plus `prev_hash`).

| Property | Effect |
|---|---|
| Altering a record breaks its own hash | Detection at that record |
| Altering a record breaks the chain from that point | Detection of all subsequent records |
| Deleting a record breaks the chain | Detection at the gap |
| Appending is always possible | Normal operation unaffected |

**Verification job:** periodically recompute and report the first divergence. A divergence is a security incident, not a data-quality issue.

**Concurrency note:** a single global chain serialises inserts on the previous hash, risking deadlocks under concurrent writes. **[Proposed]** chain per entity class (documents, approvals, settings), which removes the contention while keeping tamper evidence per stream. The choice must be recorded when implemented.

---

## 6. Access to the audit log

| Role | Access |
|---|---|
| `PEMILIK` | Full read, filter, export |
| `SUPER_ADMIN` | Full read, filter, export |
| Explicit `audit.view` permission | Full read, grantable to a named individual |
| `ADMIN` | **No** |
| `TEKNISI` | **No** |

**Viewing is itself audited** (NFR-AUD-006). Access to the record of who did what is itself sensitive, and an unexplained audit-log export before a dispute is exactly the signal an owner needs.

**Filters** (FR-AUD-005): actor, entity type/ID, action, date range; optional correlation ID, override-only, denied-access-only, settings-only. **Export** to CSV is permitted for `PEMILIK` and `SUPER_ADMIN`; exports are audited and contain personal data.

---

## 7. Overrides in the audit trail

An override records: actor · action permitted (e.g. `workorder.verify.own`) · target · **reason** (mandatory, minimum length) · timestamp (UTC) · the linked audit event flagged as an override.

`PEMILIK` must be able to see, without querying the audit log: all overrides this period with actor, action, and reason · overrides by user (pattern detection) · unreviewed overrides · overrides on high-risk actions (invoice void, payment confirm, completion verification).

**Why a dedicated report:** an override buried in an audit log is an override nobody reviews. The control only works if the owner actually sees it.

### 7.1 What an override may never do

**An override bypasses authorisation conflicts, never data-integrity invariants, and never the client's purchase-approval requirement.**

| May permit | May **never** permit |
|---|---|
| Verifying one's own work | Issuing a duplicate document number |
| Confirming a payment one recorded | Reserving more stock than is verified |
| Voiding an invoice one issued | Editing an issued document or its snapshot |
| Starting a project before materials are ready | Deleting or modifying an audit record |
| Closing a project with acknowledged exceptions | Exceeding an issue line's arithmetic |
| — | **Issuing a PO without `PEMILIK` approval (BR-PROC-005)** |
| — | **Backdating a work-session timestamp (BR-TECH-004)** |

Restated as a hard invariant in [`../../AGENTS.md`](../../AGENTS.md).

---

## 8. Retention

| Aspect | Policy |
|---|---|
| Financial and document events | At least 10 years (subject to accountant confirmation — Q-OPS-01) |
| Operational events | At least the life of the related project plus the financial retention period |
| Authentication / denied-access events | 2 years **[Proposed]** |
| Settings events | Life of the system **[Proposed]** — configuration history explains historical numbering and mail behaviour |
| Archival | Older records move to cold storage; never deleted |
| Deletion | Only by an explicit, documented legal/retention decision — never a routine job |
| Integrity after archival | The chain remains verifiable across archived segments |

**Proposed default:** 10 years for financial events, driven by typical Indonesian tax audit windows. Storage is cheap; reconstructing deleted history is impossible.

---

## 9. Privacy and data minimisation

| Rule | Application |
|---|---|
| No secrets in audit records | Never log passwords, hashes, tokens, API keys, **or SMTP credentials** |
| Settings changes audited by key, not value | `mail.password` appears as a changed key with a redacted value |
| No full file contents | Log metadata and hash, not bytes |
| Personal data minimised | Log user IDs and names where needed; do not duplicate addresses or phone numbers |
| Sensitive field redaction | Fields marked sensitive are redacted in `before_state`/`after_state` |
| Technician privacy | Reports are visible to office staff by design; not to other technicians |
| Location data | Project coordinates are business-confidential; they appear only inside a project-change diff, never as a separate location stream |

Audit logging and data minimisation pull in opposite directions. The resolution is to log *actions and identifiers*, not *content*, except where content is the point (an override reason, a session-correction reason).

---

## 10. Failure behaviour

| Failure | Required behaviour |
|---|---|
| Audit write fails | The whole transaction rolls back; the business action does not occur |
| Hash computation fails | Transaction rolls back |
| Audit table unavailable | Action blocked with a clear Indonesian error; never silently proceed |
| Chain verification finds divergence | Alert to `PEMILIK`; incident procedure; no automatic "repair" |
| Audit volume grows unexpectedly | Investigate; never disable logging to relieve pressure |
| Trigger missing after a migration | Constraint test fails the build before release |

**The first row is the important one.** A system that proceeds when auditing fails has no audit trail during exactly the conditions — database problems, incidents — when the trail matters most.

---

## 11. Acceptance criteria references

| Behaviour | AC |
|---|---|
| A record exists for every transition, including sessions, verification, and settings | AC-AUD-001 |
| Records cannot be updated or deleted | AC-AUD-002 |
| Written in the same transaction (no orphan) | AC-AUD-003 |
| Override requires a reason and appears in the report | AC-AUD-004 |
| Log restricted and self-audited | AC-AUD-006 |
| Hash chain detects tampering | AC-AUD-003 |
| Retention job archives without loss | AC-AUD-005 |
| Settings changes audited without secret values | AC-SYS-001, AC-AUD-001 |

---

## 12. Cross-references

- Requirements: [`../PRD.md`](../PRD.md) §6.9, §7
- Data model: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §10
- Override mechanism: [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md) §6
- Numbering audit events: [`./DOCUMENT_NUMBERING.md`](./DOCUMENT_NUMBERING.md) §8
- Decision: [`../decisions/ADR-004-audit-logging-approach.md`](../decisions/ADR-004-audit-logging-approach.md)
