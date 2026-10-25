# ADR-004 — Audit logging approach

| Field | Value |
|---|---|
| Status | **Accepted (design) — amended in place 2026-10-25 for MySQL/MariaDB enforcement limits** |
| Date | 2026-10-25 |
| Revision | Rev 2 — enforcement stack restated; decision unchanged |
| Deciders | Technical lead, business owner |
| Related | [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md), ADR-003, ADR-008 |

> **Revision note.** The decision — **same-transaction append-only audit, privilege-enforced, hash-chained** — is unchanged. Rev 2 replaces an assumption of PostgreSQL-style enforcement (a rule system with security-definer parity) with what MySQL/MariaDB can actually do, and states the resulting limits honestly rather than overstating the guarantee.

---

## Context

**[Assumption]** P7: today there is no reliable way to answer "who approved this purchase", "who changed this date", or "who voided this invoice". Answers come from scrolling WhatsApp.

The system must provide attributable history for approvals, issuances, payments, overrides, permission changes, **session corrections, and settings changes**. Two failure modes are unacceptable:

1. A state change commits without an audit record — permanently unattributable.
2. An audit record can be altered or deleted — the trail becomes worthless exactly when it is needed.

The team is small, so the mechanism must not require operational effort to maintain.

---

## Decision

### D1 — Same transaction as the state change

```
BEGIN;  <state change>  <audit insert>  COMMIT;
```

Both succeed or both fail. There is no asynchronous audit path.

### D2 — Append-only enforced by privileges **and** triggers

`UPDATE` and `DELETE` are not granted to the application database user. Additionally, `BEFORE UPDATE` and `BEFORE DELETE` triggers on `audit_events` raise `SIGNAL SQLSTATE '45000'`, so a connection that somehow retains the privilege still cannot mutate a row by accident.

### D3 — Hash chaining for tamper evidence

Each record stores `prev_hash` and `record_hash`. A periodic job recomputes and reports the first divergence. **[Proposed]** chain per entity class rather than globally, to avoid serialising every insert on one previous hash.

### D4 — Diff-only before/after state

Only changed fields are recorded, not full entity snapshots.

### D5 — Effective roles captured at event time

`actor_role_codes` is stored on the record, not joined from current role tables. MySQL has no array type, so this is a `JSON` array.

### D6 — Triggers on high-risk tables

For `invoices`, `payments`, `purchase_orders`, `purchase_approvals`, `issued_documents`, `number_allocations`, `work_sessions`, and `settings`, an `AFTER INSERT`/`AFTER UPDATE` trigger asserts an audit row exists for the transaction and aborts otherwise. Lower-risk tables rely on application-level writes plus test coverage.

### D7 — Audit-log access is itself audited

Every view of the audit log produces an audit record.

### D8 — Secrets are never audited

Settings changes are recorded by **key**, never by value (NFR-SEC-009).

---

## Consequences

### Positive

- **No orphan changes.** A committed change always has its audit record, proven by rollback tests.
- **Immutability is a privilege and a trigger, not a convention.**
- **Tampering is detectable** — the realistic guarantee.
- **Historical role context is preserved.** A later demotion does not rewrite history.
- **Small storage footprint** from diff-only recording.

### Negative

- **Every audited transaction writes an extra row.** Irrelevant at ~18 users.
- **Trigger maintenance.** High-risk triggers must be reviewed when tables change; a migration that drops one must fail a test rather than pass silently.
- **Hash chaining adds a verification job** and a failure mode to monitor.
- **Tamper-evidence, not tamper-proofing.** Someone with full database access and the hashing key could rewrite the chain.

### Negative — engine-specific limits, stated explicitly

| Limitation | Consequence |
|---|---|
| No rule system comparable to `CREATE RULE … DO INSTEAD NOTHING` | Immutability cannot be declared declaratively; it relies on privileges plus triggers |
| Triggers are **not** security-definer in the PostgreSQL sense | A privileged user (DBA, `root`) can drop the trigger and then mutate rows |
| A DDL-capable user can `DROP TRIGGER` | The guarantee is against the **application path**, not a hostile DBA |
| External anchoring is out of scope | A full-database attacker can rewrite the chain undetected by the application |

**Therefore:** no document in this set may describe the audit log as tamper-proof, legally conclusive, or immutable against a database administrator. It is **tamper-evident and attributable**, and that is what the design claims.

### Neutral

- Audit records are not a backup and not a reporting source. Business history lives in separate `*_status_transitions` tables and in `work_sessions`.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Asynchronous audit write after commit | **Rejected** — a crash between commit and audit write produces a permanent unattributable change |
| Queue-based audit | **Rejected** — adds infrastructure, introduces ordering ambiguity and lag during incident investigation, and can lose messages |
| Application-level immutability only | **Rejected** — a convention is not a control; a migration, a debugging session, or a careless admin script would violate it silently |
| Full entity snapshots in audit records | **Rejected** — multiplies storage, makes records unreadable, duplicates the snapshot tables |
| Generic change-data-capture | **Rejected** — captures row changes but not intent, actor identity in business terms, reasons, or overrides |
| No hash chaining | **Rejected** — an altered or deleted record would be undetectable |
| External hash anchoring in MVP | **Rejected for MVP, retained as Q-AUD-02** — would close the privileged-attacker gap but adds an operational dependency the business has not asked for |

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Every state transition produces a record | AC-AUD-001 |
| V2 | `UPDATE audit_events` as the application user fails | AC-AUD-002 |
| V3 | `DELETE FROM audit_events` as the application user fails | AC-AUD-002 |
| V4 | `UPDATE`/`DELETE` as a privileged user refused by the trigger | AC-AUD-002 |
| V5 | Forced audit failure rolls back the state change | AC-AUD-003 |
| V6 | Altered record breaks chain verification | AC-AUD-003 |
| V7 | Deleted record produces a detectable gap | AC-AUD-003 |
| V8 | Override without a reason refused | AC-AUD-004 |
| V9 | `TEKNISI` and `ADMIN` denied audit access | AC-AUD-006 |
| V10 | Settings change audited without the secret value | AC-SYS-001 |

**V5 is the decisive test.** It proves the system refuses to act when it cannot record.

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-AUD-01 | Retention period per event class | 10 years financial, 2 years auth (Q-OPS-01) |
| Q-AUD-02 | Is external hash anchoring required? | No for MVP; revisit if the threat model includes privileged insiders |
| Q-AUD-03 | Who besides `PEMILIK` and `SUPER_ADMIN` may hold `audit.view`? | No one by default |
| Q-AUD-04 | Exportable to a client-owned archive? | On request, audited |
| Q-AUD-05 | Single global chain or per-entity-class chains? | Per entity class, to avoid insert serialisation |

---

## Cross-references

- Standard: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)
- Data model: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §10
- Authorisation: [`./ADR-005-authorisation-model.md`](./ADR-005-authorisation-model.md)
- Platform: [`./ADR-008-laravel-platform-baseline.md`](./ADR-008-laravel-platform-baseline.md)
