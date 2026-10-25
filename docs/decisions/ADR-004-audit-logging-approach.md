# ADR-004 — Audit logging approach

| Field | Value |
|---|---|
| Status | **Proposed** |
| Date | 2026-10-25 |
| Deciders | Technical lead, business owner |
| Related | [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md), ADR-003 |

---

## Context

**[Assumption]** P7: today there is no reliable way to answer "who approved this purchase", "who changed this date", or "who voided this invoice". Answers come from scrolling WhatsApp.

The system must provide attributable history for approvals, issuances, payments, overrides, and permission changes. Two failure modes are unacceptable:

1. A state change commits without an audit record — an unattributable change, permanently.
2. An audit record can be altered or deleted — the trail becomes worthless exactly when it is needed.

The team is small, so the mechanism must not require operational effort to maintain.

---

## Decision

### D1 — Audit records written in the same transaction as the state change

```
BEGIN;
  <state change>
  <audit insert>
COMMIT;
```

Both succeed or both fail. There is no asynchronous audit path.

### D2 — Append-only enforced by database privileges

`UPDATE` and `DELETE` are revoked from the application role on `audit_events`. The role has `INSERT` and `SELECT` only.

### D3 — Hash chaining for tamper evidence

Each record stores `prev_hash` and `record_hash`, forming a chain. A periodic verification job recomputes and reports the first divergence.

### D4 — Diff-only before/after state

Only changed fields are recorded, not full entity snapshots.

### D5 — Effective roles captured at event time

`actor_role_codes` is stored on the record, not joined from current role tables.

### D6 — Database triggers on high-risk tables

For `invoices`, `payments`, `purchase_orders`, `purchase_approvals`, `issued_documents`, and `document_number_allocations`, a trigger asserts an audit row exists for the transaction and aborts if not. Lower-risk tables rely on application-level writes plus test coverage.

### D7 — Audit log access is itself audited

Every view of the audit log produces an audit record.

---

## Consequences

### Positive

- **No orphan changes.** A committed change always has its audit record, proven by rollback tests.
- **Immutability is a privilege, not a convention.** An ORM method that "doesn't expose update" is not a control; a revoked privilege is.
- **Tampering is detectable**, which is the realistic guarantee.
- **Historical role context is preserved.** A later demotion does not rewrite history.
- **Small storage footprint** from diff-only recording.

### Negative

- **Every audited transaction writes an extra row.** Irrelevant at ~18 users; would matter at very high write volume.
- **Trigger maintenance.** High-risk triggers must be reviewed when tables change.
- **Hash chaining adds a verification job** and a failure mode to monitor.
- **Tamper-evidence, not tamper-proofing.** Someone with full database access and the hashing key could rewrite the chain. Mitigated only by external anchoring, which is out of scope.

### Neutral

- Audit records are not a backup and not a reporting source. Business history lives in separate `*_status_transitions` tables (DATA_MODEL §10).

---

## Alternatives considered

### Alternative A — Asynchronous audit write after commit

**Rejected.** A crash between commit and audit write produces a permanent unattributable change. This is precisely the failure the system exists to prevent.

### Alternative B — Queue-based audit (message broker)

**Rejected.** Adds infrastructure for a small team, introduces ordering ambiguity and lag during incident investigation, and can lose messages. The transactional approach is simpler and strictly more correct.

### Alternative C — Application-level immutability only

**Rejected.** A convention is not a control. A future migration, a debugging session, or a careless admin script would silently violate it. Revoked privileges are enforceable and testable.

### Alternative D — Full entity snapshots in audit records

**Rejected.** Multiplies storage, makes records unreadable, and duplicates data already available via the snapshot tables. Diff-only keeps records legible and small.

### Alternative E — Generic database change-data-capture

**Rejected.** CDC captures row changes but not intent, actor identity in business terms, reasons, or overrides. It would produce a log of column changes, not a record of decisions.

### Alternative F — No hash chaining

**Rejected.** Without chaining, an altered or deleted record is undetectable. Chaining is inexpensive and converts "we believe the log is intact" into "we can demonstrate it".

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Every state transition produces a record | AC-AUD-001 |
| V2 | `UPDATE audit_events` as app role fails | AC-AUD-002 |
| V3 | `DELETE FROM audit_events` as app role fails | AC-AUD-002 |
| V4 | Forced audit failure rolls back the state change | AC-AUD-003 |
| V5 | Altered record breaks chain verification | AC-AUD-003 |
| V6 | Deleted record produces a detectable gap | AC-AUD-003 |
| V7 | Override without reason refused | AC-AUD-004 |
| V8 | Technician and ops admin denied audit access | AC-AUD-006 |

**V4 is the decisive test.** It proves the system refuses to act when it cannot record.

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-AUD-01 | Retention period per event class | 10 years financial, 2 years auth (Q-OPS-01) |
| Q-AUD-02 | Is external hash anchoring required? | No for MVP |
| Q-AUD-03 | Who besides the owner may hold `audit.view`? | No one by default |
| Q-AUD-04 | Should audit records be exportable to a client-owned archive? | On request, audited |
