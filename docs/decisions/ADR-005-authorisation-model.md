# ADR-005 — Authorisation model: four roles, scopes, and logged overrides

| Field | Value |
|---|---|
| Status | **Accepted (design) — rewritten 2026-10-25 for the four-role model** |
| Date | 2026-10-25 |
| Revision | Rev 2 — role set replaced; decision structure retained |
| Deciders | Business owner, technical lead |
| Related | [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md), ADR-004 |

> **Revision note.** Rev 1 defined five roles: `OWNER`, `OPS_ADMIN`, `FINANCE`, `TECHNICIAN`, `TEAM_LEADER`. The client has fixed the role set at exactly four: `SUPER_ADMIN`, `PEMILIK`, `ADMIN`, `TEKNISI`. This revision replaces the role set and its permission consequences. The **mechanism** — permissions as data, structural scoping, logged overrides — is unchanged, because it was never role-specific.

---

## Context

**[Confirmed]** ~3 office employees and ~15 technicians. The brief states:

- Not all technicians need identical access; not every office employee has the same responsibilities.
- **The owner approves every purchase** and may personally transfer funds.
- A system administrator role exists for users, settings, numbering, mail, and company profile.

**[Assumption]** At least one person will hold both `SUPER_ADMIN` and `ADMIN`, because three people cannot staff four distinct functions.

This creates genuine tension. Standard practice separates duties: the approver should not be the payer, and the verifier should not be the doer. But with three office staff, enforcing four-eyes control everywhere would stall the business in its first week — and a control that stops work gets bypassed informally, which is worse than no control because it is invisible.

---

## Decision

### D1 — Permissions are data, keyed by action and resource

Permissions are `resource.action` codes stored in tables (`permissions`, `role_permissions`, `user_roles`). Code evaluates a policy. No role-name branching in business logic.

### D2 — Exactly four roles, combinable per user

| Role | Defines |
|---|---|
| `SUPER_ADMIN` | Full application access **plus** system administration: users, roles, settings, SMTP, numbering configuration, company profile, bank accounts |
| `PEMILIK` | Business authority and the **mandatory purchase approver**; no system configuration |
| `ADMIN` | Full operational access including purchasing **preparation**; **no approval authority** |
| `TEKNISI` | Assignment-scoped operational access only |

### D3 — `FINANCE` is not a role

Its capabilities are now granular permissions (`invoice.issue`, `payment.record`, `payment.confirm`, `payable.record`, `report.finance`) granted to `ADMIN` and/or `SUPER_ADMIN`. Financial authority is expressed through **permissions plus approval requirements plus audit**, never a role name.

This is an improvement over Rev 1: with `FINANCE` removed, the hard rule "`OPS_ADMIN` cannot issue invoices" became unnecessary. `ADMIN` may now issue invoices — an operational document act — while remaining unable to approve a purchase, which is the spending decision that needs separation.

### D4 — `TEAM_LEADER` is not a role

Team leadership is `project_members.is_team_leader` plus two narrowly scoped permissions: submit a team-level daily summary, and view that project's team reports. No financial access, no material-issue authority, no verification authority, no company-wide read. If it later proves a distinct function, that may justify a fifth role — **Q-ROLE-01**, explicitly not decided here.

### D5 — Technician access is assignment-scoped, structurally

Technician-facing queries are dedicated scoped functions that cannot return unassigned rows — not general queries with an optional filter.

### D6 — Separation of duties where practical; logged override where not

Conflicting pairs are blocked by default: record + confirm a payment, issue + void an invoice, verify one's own work, record a payable + pay it. When one person must hold both, the action requires an **override**: mandatory reason, recorded, surfaced to `PEMILIK`.

### D7 — Override bypasses authorisation conflicts, never invariants, never purchase approval

An override may permit confirming a payment one recorded. It may never permit a duplicate number, an over-reservation, an edited issued document, a backdated session timestamp, **or a PO without `PEMILIK` approval**.

**D7 is narrower than in Rev 1**, which allowed an override to permit approving one's own purchase request on the reasoning that a three-person office cannot avoid it. The client has since stated that `PEMILIK` approval is mandatory for **every** purchase, converting the rule from an overridable authorisation conflict into a non-overridable business control. `ADMIN` no longer holds `purchase.approve` at all, so no conflict remains to override — the separation is structural rather than compensatory.

### D8 — Administrative privilege is not financial authority

`SUPER_ADMIN` does **not** hold `purchase.approve`. This is the single most important cell in the matrix: whoever can change every setting still cannot commit company money.

### D9 — Server-side enforcement at two layers

Application-layer policy plus row-scoped data access. UI hiding is not a control.

---

## Consequences

### Positive

- **Technician isolation is structural**, not a filter someone might forget.
- **The small-team reality is acknowledged honestly** — every remaining bypass is visible and attributable.
- **Purchase approval is stronger than in Rev 1** — not an overridable conflict, but a permission only `PEMILIK` holds.
- **Removing `FINANCE` simplified the model without weakening separation**, because the separation that matters (spending vs operations) is preserved.
- **Roles can be recombined** as staffing changes without schema change.

### Negative

- **`SUPER_ADMIN` can do almost anything operational.** Protections are audit, the invariant boundary, and the absence of approval authority. Inherent to an owner-operated business with an administrator.
- **Override is available, so it will be used.** Mitigated by mandatory reasons, `PEMILIK` visibility, and override-frequency reporting; not eliminated.
- **No delegation of purchase approval.** If the owner is unavailable, purchases wait — a client-stated consequence, not an oversight.
- **`SUPER_ADMIN` and `ADMIN` may be the same person.** The model supports it; audit and override make it survivable.
- **Four roles are fewer than five**, so some previously distinct bundles are now permission subsets. More flexible, but a permission grant must be reviewed rather than assumed.

### Neutral

- Role names are job-adjacent but not job-identical. A title does not determine permissions; assigned roles do.

---

## Alternatives considered

| Alternative | Verdict |
|---|---|
| Strict separation of duties, no override | **Rejected** — blocks legitimate work in a 3-person office; the predictable outcome is shared accounts or work outside the system, both destroying attribution |
| Role checks hard-coded in the UI | **Rejected** — direct requests bypass them |
| Single admin role for all office staff | **Rejected** — would give the administrator purchase-approval rights |
| Per-user permissions without roles | **Rejected** — unmaintainable at ~18 users |
| Retain `FINANCE` as a role | **Rejected by client instruction.** It also implied a separation that did not exist in a three-person office |
| Retain `TEAM_LEADER` as a role | **Rejected** — a company-wide role grants review rights over every project |
| Give `SUPER_ADMIN` approval authority | **Rejected** — collapses the only separation the client asked for |
| ABAC | **Rejected for MVP** — needs a policy language and expertise the project lacks |

---

## Verification obligations

| # | Test | Proves |
|---|---|---|
| V1 | Technician direct-requests another's work order | 403/404, no existence leak — AC-TECH-009 |
| V2 | Technician requests invoice list | 403 — AC-SEC-006 |
| V3 | `ADMIN` attempts to approve a purchase | 403 — AC-PROC-011 |
| V4 | `SUPER_ADMIN` attempts to approve a purchase | 403 — AC-PROC-011 |
| V5 | `PEMILIK` attempts SMTP or numbering configuration | 403 — AC-SEC-002 |
| V6 | Multi-role union applies correctly | AC-SEC-003 |
| V7 | Override without a reason refused | AC-AUD-004 |
| V8 | Override appears in the `PEMILIK` report | AC-AUD-004 |
| V9 | Override cannot bypass a constraint | AC-MATL-006, AC-DOC-004 |
| V10 | Override cannot waive purchase approval | AC-PROC-011 |
| V11 | Override cannot backdate a session timestamp | AC-TECH-012 |

**V9, V10, and V11 are the boundary that must never erode.** An override that can bypass a database constraint, the client's approval rule, or timestamp authority converts a controlled exception into a data-integrity hole.

---

## Open questions

| ID | Question | Default |
|---|---|---|
| Q-ROLE-01 | Does the team-leader designation stay useful, and should it ever become a role? | Keep as a per-project designation; a future role is not decided |
| Q-FIN-04 | Does `PEMILIK` perform financial actions beyond approval, and which destructive actions are prohibited? | May record and confirm payments and view all reports; prohibited: editing issued documents, deleting audit records, altering numbering history |
| Q-SEC-04 | Temporary delegation of approval? | No in MVP; documented as an operational risk |
| Q-SEC-02 | `audit.view` beyond `PEMILIK` and `SUPER_ADMIN`? | No by default |
| Q-SEC-03 | MFA for owner and administrator accounts? | Out of scope for MVP; session model must not preclude it |

---

## Cross-references

- Permission matrix: [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md) §3.2
- Roles in the PRD: [`../PRD.md`](../PRD.md) §4
- Audit and overrides: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md) §7
- Purchase approval rule: [`../workflows/PROCUREMENT_AND_MATERIALS.md`](../workflows/PROCUREMENT_AND_MATERIALS.md) §8
