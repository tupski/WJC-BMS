# ADR-005 — Authorisation model: roles, scopes, and logged overrides

| Field | Value |
|---|---|
| Status | **Proposed** |
| Date | 2026-10-25 |
| Deciders | Business owner, technical lead |
| Related | [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md), ADR-004 |

---

## Context

**[Confirmed]** There are ~3 office employees and ~15 technicians. The brief explicitly warns that:

- Not all technicians need identical access.
- Not every office employee has the same responsibilities.
- The owner approves purchases and may personally transfer funds.

**[Assumption]** At least one person will hold both operations and finance responsibilities, because three people cannot staff four distinct functions.

This creates a genuine tension. Standard practice says separate duties: the person who approves spending should not be the person who pays, and the person who verifies work should not be the person who did it. But with three office staff, enforcing four-eyes control everywhere would stall the business in its first week — and a control that stops work gets bypassed informally, which is worse than no control because it is invisible.

---

## Decision

### D1 — Permissions are data, keyed by action and resource

Permissions are `resource.action` codes stored in tables (`permissions`, `role_permissions`, `user_roles`). Code evaluates `can(user, action, resource)`. No `if role === 'owner'` branching in business logic.

### D2 — Five roles, combinable per user

`OWNER`, `OPS_ADMIN`, `FINANCE`, `TECHNICIAN`, `TEAM_LEADER`. A user may hold several; effective permissions are the union.

### D3 — Technician access is assignment-scoped, structurally

Technician-facing queries are dedicated scoped functions that cannot return unassigned rows. Not general queries with an optional filter.

### D4 — Team leader is a per-project designation, not a company role

Granted through `project_assignments.role_on_project`, scoped to that project's team. Avoids inventing an org chart the business may not have.

### D5 — Separation of duties where practical; logged override where not

Conflicting permission pairs (approve+pay, record+confirm, verify own work, issue+void) are blocked by default. When one person must hold both, the action requires an **override**: mandatory reason, recorded, surfaced in an owner report.

### D6 — Override bypasses authorisation conflicts, never data invariants

An override may permit a person to approve their own request. It may never permit a duplicate document number, an over-reservation, or an edit to an issued invoice.

### D7 — Server-side enforcement at two layers

Application-layer policy check plus row-scoped data access. UI hiding is not a control.

---

## Consequences

### Positive

- **Technician isolation is structural**, not a filter that someone might forget.
- **The small-team reality is acknowledged honestly.** Rather than pretending four-eyes control exists, the system makes every bypass visible and attributable.
- **Control shifts from prevention to detection**, which is the only workable model at this scale — and detection requires the owner to look, which is why the override report is a first-class deliverable.
- **Roles can be recombined** as staffing changes without schema change.
- **Auditability is complete** for every privileged action.

### Negative

- **The owner can do anything.** No technical control exists above `OWNER`. Inherent to an owner-operated business.
- **Override is available, so it will be used.** Mitigated by mandatory reasons, owner visibility, and override-frequency reporting; not eliminated.
- **No delegation.** If the owner is unavailable, approvals wait.
- **Union of permissions can produce unexpected combinations.** Mitigated by explicit conflict-pair rules that apply regardless of how roles were combined.

### Neutral

- Role names are job-adjacent but not job-identical. A person's title does not determine their permissions; their assigned roles do.

---

## Alternatives considered

### Alternative A — Strict separation of duties with no override

**Rejected.** With three office staff, this blocks legitimate work. The predictable outcome is shared accounts or work performed outside the system — both worse than a logged override, because both destroy attribution entirely.

### Alternative B — Role checks hard-coded in the UI

**Rejected.** Direct API calls bypass it. Hiding a button is a usability decision, not a security control (NFR-SEC-002).

### Alternative C — Single admin role for all office staff

**Rejected.** Gives the office administrator invoice-issuance and payment-confirmation rights. Invoicing is a financial and legal act; the strongest available separation in a small team is to keep it with finance/owner.

### Alternative D — Per-user granular permissions without roles

**Rejected.** With ~18 users, per-user permissions become unmaintainable and unauditable. Roles provide a reviewable bundle; individual grants can be added where genuinely needed.

### Alternative E — Team leader as a company-wide role

**Rejected.** Would grant review rights over every project, which is broader than the operating model justifies. Per-project designation is narrower and matches how the owner actually assigns work.

### Alternative F — Attribute-based access control (ABAC)

**Rejected for MVP.** Powerful but requires a policy language, testing strategy, and expertise the project does not have. Role + scope + override covers the confirmed needs.

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Technician direct-requests another's work order | 403/404, no existence leak — AC-TECH-009 |
| V2 | Technician requests invoice list | 403 — AC-SEC-006 |
| V3 | OPS_ADMIN issues invoice | 403 — AC-SEC-002 |
| V4 | FINANCE approves purchase | 403 — AC-SEC-002 |
| V5 | Multi-role union applies correctly | AC-SEC-003 |
| V6 | Override without reason refused | AC-AUD-004 |
| V7 | Override appears in owner report | AC-AUD-004 |
| V8 | Override cannot bypass a constraint | AC-MATL-006, AC-DOC-004 |

**V8 is the boundary that must never erode.** An override that can bypass a database constraint converts a controlled exception into a data-integrity hole.

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-ROLE-01 | Does a team leader role exist in practice, and do they review reports? | Per-project designation, review step optional |
| Q-SEC-01 | Should the owner be able to delegate approval temporarily? | No in MVP; documented as operational risk |
| Q-SEC-02 | Should `audit.view` be grantable beyond the owner? | No by default |
| Q-SEC-03 | Is MFA required for owner accounts? | Out of scope for MVP; session model must not preclude it |
