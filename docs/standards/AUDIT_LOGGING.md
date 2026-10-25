# Audit Logging Standard — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed |
| Version | 0.1 |
| Scope owner | This document is the **single authority** on what is audited, how audit records are written, and how they are protected |
| Requirement source | [`../PRD.md`](../PRD.md) §6.9, NFR-AUD-001…006, BR-AUD-001 |
| Data model | [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §9 |
| Decision record | [`../decisions/ADR-004-audit-logging-approach.md`](../decisions/ADR-004-audit-logging-approach.md) |

---

## 1. Purpose

The audit log answers questions the business currently cannot answer:

- Who approved this purchase, and when?
- Who changed this project's target date, and why?
- Who issued this invoice, and was it later voided by someone else?
- Did anyone override a blocked workflow, and what reason did they give?
- Was this payment recorded and confirmed by two different people?

**[Assumption]** P7 in the PRD: today these questions are answered by scrolling WhatsApp and guessing. That is the gap this standard closes.

### 1.1 What the audit log is not

| Not this | Why |
|---|---|
| Application debug logging | Debug logs are for diagnosing faults; audit logs are for accountability. Different retention, different access, different format |
| A business history report | Business timelines (project status history) are separate, queryable tables (DATA_MODEL §10) |
| A backup | Audit records are not a substitute for data backup |
| Tamper-proof | It is **tamper-evident**. A determined party with database-level access can alter data; hash chaining makes alteration detectable |

**Honesty about tamper-evidence:** claiming "tamper-proof" would be false. The realistic guarantee is that modification is *detectable*, which is what matters for dispute resolution.

---

## 2. What is audited

### 2.1 Mandatory events

Every event below **must** produce an audit record. The list is normative.

| Category | Events |
|---|---|
| **Authentication** | Login success, login failure, lockout, logout, password change, administrative password reset, session revocation |
| **Authorisation** | Role assignment, role revocation, permission change to a role, user creation, user deactivation, user reactivation |
| **Denied access** | Authorisation denial on sensitive actions: invoice issue, invoice void, payment confirm, purchase approve, audit view, permission change |
| **Project** | Create, status transition, schedule revision, assignment, unassignment, team-leader designation, closure, cancellation |
| **Work order** | Create, assign, reassign, status transition, completion submission, verification, rejection for rework |
| **Daily report** | Submission, supersession, review, rejection |
| **Material** | Requirement creation/change, stock verification, invalidation of verification, reservation create/release/consume, issue, consumption, return, write-off, reconciliation run |
| **Procurement** | Purchase request submit, approval decision, rejection, PO issuance, PO cancellation, goods receipt posting, receipt discrepancy resolution, over-receipt approval |
| **Finance** | Quotation issuance, invoice issuance, invoice void, credit note issuance, payable recording, payment recording, payment confirmation, payment reversal, expense recording, allocation change |
| **Documents** | Document number allocation, document number void, series configuration change, PDF generation, attachment upload, attachment deletion |
| **Overrides** | Every override execution, with reason |
| **Audit** | Every audit-log view (self-auditing) |
| **Data** | Import validation, import commit, import rollback, export |
| **System** | Backup completion, restore execution, migration applied, reconciliation variance detected |

### 2.2 What is not audited

| Not audited | Reason |
|---|---|
| Read access to ordinary business records | Volume would swamp the log; sensitive reads (audit log, financial exports) are audited |
| UI navigation, page views | Noise |
| Draft field edits before issuance | The draft is not yet a business fact; the issuance is |
| Cached value refreshes | Derived, not authoritative |
| Health checks | Noise |

**The distinction that matters:** drafting an invoice is not audited; issuing it is. Editing a draft is ordinary work; issuing is a commitment. Auditing drafts would produce a log nobody reads.

---

## 3. Record structure

| Field | Required | Purpose |
|---|---|---|
| `id` | Yes | Unique identifier (UUID) |
| `occurred_at` | Yes | Timestamp, UTC, with timezone |
| `actor_user_id` | Yes (nullable only for system actions) | Who |
| `actor_role_codes` | Yes | Effective roles **at the time** — critical, because roles change later |
| `action` | Yes | Namespaced verb (e.g. `invoice.issue`) |
| `entity_type` | Yes | Target type |
| `entity_id` | Yes | Target identifier |
| `before_state` | Conditional | Changed fields only, prior values |
| `after_state` | Conditional | Changed fields only, new values |
| `reason` | Conditional | Required for overrides, voids, cancellations, rejections |
| `override_id` | Conditional | Link to the override record |
| `correlation_id` | Yes | Request trace identifier, links related events |
| `ip_address` | Where available | Source of the action |
| `user_agent` | Optional | Client context |
| `prev_hash` | Yes | Previous record's hash |
| `record_hash` | Yes | Hash of this record's canonical content |

### 3.1 Why `actor_role_codes` is stored

If a user is later demoted or promoted, "they were an owner at the time" must remain answerable. Joining to current role tables would rewrite history retroactively — the exact failure the audit log exists to prevent.

### 3.2 Why before/after are diff-only

Storing full entity snapshots on every change multiplies storage and makes diffs unreadable. Storing changed fields only keeps records small and makes the change obvious. Full historical state is reconstructable by replaying diffs where needed.

### 3.3 Canonical action names

| Pattern | Example |
|---|---|
| `<entity>.<verb>` | `invoice.issue`, `project.transition`, `payment.confirm` |
| `<entity>.<verb>.<qualifier>` | `workorder.verify.override`, `material.verification.invalidate` |

Consistent naming makes filtering and reporting possible without a lookup table of free-text actions.

---

## 4. Transactional coupling

### 4.1 The rule

**An audit record is written in the same database transaction as the state change it describes** (FR-AUD-003, BR-AUD-001).

```
BEGIN;
  UPDATE invoices SET status='ISSUED', ... WHERE id=$1;
  INSERT INTO audit_events (...) VALUES (...);
COMMIT;
```

If either fails, both roll back. There is no committed state change without its audit record.

### 4.2 Why not asynchronous

| Approach | Failure mode |
|---|---|
| Write audit after commit, in application code | Crash between commit and audit write → unattributable change, permanently |
| Write audit to a queue, process later | Queue loss, ordering ambiguity, lag during incident investigation |
| Write audit in same transaction | Atomic; slightly more lock contention; correct |

**The trade-off is accepted deliberately.** Audit writes add a row insert to each transaction. For a system with ~18 users, that cost is irrelevant. Correctness under crash is not.

### 4.3 Enforcement

| Layer | Mechanism |
|---|---|
| Application | Use-case layer writes the audit record as part of the transaction |
| Database | A trigger on audited tables can assert an audit row exists for the transaction; a missing audit row aborts the transaction |
| Review | Any new state-changing use case must include an audit write; enforced by a checklist and a test that asserts audit presence per transition |

**[Proposed]** For the highest-risk tables (`invoices`, `payments`, `purchase_orders`, `purchase_approvals`, `issued_documents`, `document_number_allocations`), enforce with a database trigger rather than relying on developer discipline. For lower-risk tables, application-level writes with test coverage suffice.

---

## 5. Immutability

### 5.1 Database-level protection

| Control | Mechanism |
|---|---|
| No update | `UPDATE` privilege revoked from the application role |
| No delete | `DELETE` privilege revoked from the application role |
| Insert only | Application role has `INSERT` and `SELECT` only |
| Retention job | Archival moves records to cold storage; it does not delete |

**This must be a database privilege, not a code convention** (NFR-AUD-001). An ORM method that "doesn't expose update" is not a control; a revoked privilege is.

### 5.2 Verification test

An integration test attempts `UPDATE audit_events SET action='x'` and `DELETE FROM audit_events` using the application role, and asserts both fail. This test must run on every build, because a migration or a role change could silently restore the privilege.

### 5.3 Hash chaining

Each record stores `prev_hash` (the previous record's `record_hash`) and `record_hash` (hash of this record's canonical fields plus `prev_hash`).

| Property | Effect |
|---|---|
| Altering a record breaks its own hash | Detection at that record |
| Altering a record breaks the chain from that point | Detection of all subsequent records |
| Deleting a record breaks the chain | Detection at the gap |
| Appending is always possible | Normal operation unaffected |

**Verification job:** periodically recompute the chain and report the first divergence. A divergence is a security incident, not a data-quality issue.

**Scope of chaining:** applied to all audit records in MVP. **[Proposed]** If volume becomes a concern, chaining can be limited to high-risk events with the rest chained in daily batches — but not before volume is measured.

### 5.4 What hashing does not protect against

An attacker with full database access and the application's hashing key could rewrite the entire chain. Protection against that requires external anchoring (publishing daily hashes to an append-only external store), which is **[Out of Scope]** for MVP and should be revisited if the threat model changes.

---

## 6. Access to the audit log

| Role | Access |
|---|---|
| `OWNER` | Full read, filter, export |
| Explicit `audit.view` permission | Full read (grantable to a trusted administrator) |
| `OPS_ADMIN`, `FINANCE` | **No** |
| `TECHNICIAN` | **No** |

**Viewing the audit log is itself audited** (NFR-AUD-006). This is not paranoia: access to the record of who did what is itself sensitive, and an unexplained audit-log export before a dispute is exactly the signal an owner needs.

### 6.1 Filtering

Required filters (FR-AUD-005): actor, entity type, entity ID, action, date range. Optional: correlation ID, override-only, denied-access-only.

### 6.2 Export

Export to CSV is permitted for `OWNER`. Exports are audited. Exported files contain personal data and must be handled accordingly.

---

## 7. Overrides in the audit trail

Overrides deserve their own treatment because they are the mechanism by which controls are bypassed in a small team.

### 7.1 What an override records

| Field | Content |
|---|---|
| Actor | Who overrode |
| Action | What was permitted (e.g. `workorder.verify.own`) |
| Target | The entity affected |
| Reason | Mandatory free text, minimum length |
| Timestamp | When |
| Linked audit event | The audit record with `is_override = true` |

### 7.2 Override reporting

The owner must be able to see, without querying the audit log directly:

| Report | Content |
|---|---|
| Overrides this period | All overrides with actor, action, reason |
| Overrides by user | Pattern detection |
| Unreviewed overrides | Those without an owner acknowledgement |
| Overrides on high-risk actions | Invoice void, payment confirm, purchase approval |

**Why a dedicated report:** an override buried in an audit log is an override nobody reviews. The control only works if the owner actually sees it, which requires it to be surfaced, not merely recorded.

### 7.3 What an override may never do

**An override bypasses authorisation conflicts, never data-integrity invariants.**

| Override may permit | Override may never permit |
|---|---|
| A user verifying their own work | Issuing a duplicate document number |
| A user confirming a payment they recorded | Reserving more stock than is verified |
| A user approving a purchase they requested | Editing an issued invoice |
| Starting a project before materials are ready | Deleting an audit record |
| Closing a project with acknowledged exceptions | Exceeding an issue line's arithmetic |

This boundary is restated as a hard invariant in [`../../AGENTS.md`](../../AGENTS.md).

---

## 8. Retention

| Aspect | Policy |
|---|---|
| Minimum retention | Financial and document events: at least 10 years (subject to accountant confirmation — Q-OPS-01) |
| Operational events | At least the life of the related project plus the financial retention period |
| Authentication events | 2 years **[Proposed]** |
| Denied-access events | 2 years **[Proposed]** |
| Archival | Records older than the active window move to cold storage; they are never deleted |
| Deletion | Only by an explicit, documented legal/retention decision — never by routine job |
| Integrity after archival | Hash chain remains verifiable across archived segments |

**[Open Question] Q-OPS-01** — retention period is unconfirmed. **Proposed default:** 10 years for financial events, driven by typical Indonesian tax audit windows, pending accountant confirmation. **Rationale:** storage is cheap; reconstructing deleted history is impossible.

---

## 9. Privacy and data minimisation

| Rule | Application |
|---|---|
| No secrets in audit records | Never log passwords, hashes, tokens, API keys |
| No full file contents | Log file metadata and hash, not bytes |
| Personal data minimised | Log user IDs and names where needed for accountability; do not duplicate addresses or phone numbers into audit rows |
| Sensitive field redaction | Fields marked sensitive are redacted in `before_state`/`after_state` |
| Technician privacy | A technician's daily reports are visible to office staff by design; they are not visible to other technicians |

**Tension acknowledged:** audit logging and data minimisation pull in opposite directions. The resolution is to log *actions and identifiers*, not *content*, except where content is the point (e.g. an override reason).

---

## 10. Failure behaviour

| Failure | Required behaviour |
|---|---|
| Audit write fails | The whole transaction rolls back; the business action does not occur |
| Hash computation fails | Transaction rolls back |
| Audit table unavailable | Action blocked with a clear error; never silently proceed |
| Chain verification finds divergence | Alert to owner; incident procedure; no automatic "repair" |
| Audit volume grows unexpectedly | Investigate; never disable logging to relieve pressure |

**The first row is the important one.** A system that proceeds when auditing fails is a system with no audit trail during exactly the conditions (database problems, incidents) when the trail matters most.

---

## 11. Acceptance criteria references

| Behaviour | AC |
|---|---|
| Audit record exists for every state transition | AC-AUD-001 |
| Audit records cannot be updated or deleted | AC-AUD-002 |
| Audit written in same transaction (no orphan) | AC-AUD-003 |
| Override requires reason and appears in report | AC-AUD-004 |
| Audit log restricted and self-audited | AC-AUD-006 |
| Hash chain detects tampering | AC-AUD-003 |
| Retention job archives without loss | AC-AUD-005 |

---

## 12. Cross-references

- Requirements: [`../PRD.md`](../PRD.md) §6.9, §7
- Data model: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §9
- Override mechanism: [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md) §6
- Numbering audit events: [`./DOCUMENT_NUMBERING.md`](./DOCUMENT_NUMBERING.md) §8
- Decision: [`../decisions/ADR-004-audit-logging-approach.md`](../decisions/ADR-004-audit-logging-approach.md)
