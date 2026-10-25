# Procurement and Materials — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed |
| Version | 0.1 |
| Scope owner | This document owns material requirements, stock status, reservations, the movement ledger, purchasing, approval, goods receipt, issue/consumption/return, and material readiness status |
| Requirement source | [`../PRD.md`](../PRD.md) §6.3, §6.4, §9.3, §9.4, §9.5, §9.6, §9.7, §8 |
| Data model | [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §5, §6 |

---

## 1. Problem statement and design position

**[Confirmed]** operating conditions that shape this design:

- There is **no centralised inventory system**. Materials are prepared and tracked per project.
- **Not all required materials are on hand** at office/warehouse; some must be purchased before work proceeds.
- Office administration handles purchasing and purchase-order preparation.
- The owner sometimes approves purchases and may personally transfer funds to suppliers.

The business does not need a warehouse ERP. It needs to answer, per project, with confidence:

1. What materials does this project require?
2. Do we actually have them — and do we *know* we have them?
3. If not, what must be bought, and where is that purchase?
4. What did we hand to the technician, and what came back?

Everything in this document serves those four questions. Anything that serves a fifth question (bin locations, lot tracking, cycle counting) is deliberately deferred (Q-INV-01).

### 1.1 The MVP position: project-scoped control, not warehouse management

**[Proposed]** MVP models material **against projects**, with a shared item catalogue and a movement ledger. There is no location hierarchy, no bin, no serial number, no valuation method.

| In MVP | Deferred |
|---|---|
| Item catalogue | Multi-location stock |
| Requirement lines per project | Bin/shelf locations |
| Stock status `UNKNOWN`/`VERIFIED` | Lot/batch tracking |
| Reservations | Serial-number tracking |
| Purchase request → approval → PO → receipt | Barcode scanning |
| Issue / consume / return / write-off | Cycle counting schedules |
| Movement ledger + reconciliation | Stock valuation (FIFO/average) |
| Single implicit location | Stock transfers between locations |

---

## 2. The unknown-stock policy

This is the most important rule in the document, and it exists because of a specific failure mode.

### 2.1 The failure it prevents

A spreadsheet cell says `20` next to "Kabel UTP". Nobody knows whether that is 20 boxes, 20 metres, or a number typed eight months ago. A project reserves 20. A second project also reserves 20. Both proceed. One aborts on site. The abort costs a technician's day, a customer's trust, and an emergency purchase at retail price.

The root cause is not a wrong number — it is that **a number of unknown provenance was treated as verified fact**.

### 2.2 The policy

| Rule | Statement |
|---|---|
| **U1** | Every material requirement line carries `stock_status` ∈ {`UNKNOWN`, `VERIFIED`}. New lines default to `UNKNOWN`. (FR-MATL-002) |
| **U2** | An `UNKNOWN` line's quantity is **never** treated as available. It cannot be reserved, cannot count toward readiness, and cannot offset a purchase shortfall. |
| **U3** | A line becomes `VERIFIED` only through an explicit verification action recording the observed quantity, the verifying user, and the timestamp. (FR-MATL-003) |
| **U4** | Verification is a physical or authoritative act. The system does not infer it from a purchase order, a delivery note, or a previous project's usage. |
| **U5** | A reservation may only exist on a `VERIFIED` line. The system refuses to create one otherwise, naming the blocking condition and the required action. (FR-MATL-005) |
| **U6** | Verification may later be invalidated (stock recounted and found different). Invalidation returns the line to `UNKNOWN` and forces re-evaluation of its reservations. |

**Explicitly rejected:** a "default to available" behaviour, a "trust the spreadsheet" import flag, or an "assume 0 is fine" shortcut. Each of these reintroduces the failure the system exists to prevent.

### 2.3 What verification looks like in practice

The office administrator physically checks the shelf, or checks a trusted record, and enters: item, observed quantity, unit, location note, optional note. That single action is the moment the business converts an assumption into a fact. It takes seconds and is the cheapest possible control against a costly failure.

**[Assumption]** Verification is performed by office staff at the warehouse/office. If material is stored elsewhere, or if a technician must verify on arrival, the permission `material.verify` can be granted more widely — but the *act* of verification remains mandatory regardless of who performs it.

---

## 3. Material requirement → reservation → purchase: the full chain

### 3.1 Stage A — Requirement

```
[Office] Add requirement lines to project
   item, required_quantity, unit, specification_note
   → stock_status = UNKNOWN   (always)
   → fulfilment_status = DRAFT
```

A requirement line is a statement of need. It carries no availability implication. Requirements are editable while the project is in `DRAFT`/`PLANNED`; once materials are reserved or ordered, changing a requirement requires releasing reservations first.

### 3.2 Stage B — Verification

```
[Office] Verify each line
   observed_quantity, location_note, note
   → stock_status = VERIFIED
   → verified_on_hand_quantity = observed
   → stock_verifications row appended (history preserved)
```

Verification is per line, not per project. Partial verification is normal: five lines verified, two still unknown. The project's readiness status reflects this honestly (`REQUIREMENTS_INCOMPLETE`).

### 3.3 Stage C — Coverage calculation

For each verified line, the system computes:

| Quantity | Definition |
|---|---|
| `required` | What the project needs |
| `on_hand_verified` | What was observed at verification |
| `reserved_by_others` | Active reservations on the same item held by other projects |
| `available_to_this_line` | `on_hand_verified − reserved_by_others` |
| `covered` | `min(required, max(0, available_to_this_line))` |
| `shortfall` | `required − covered` |

The `shortfall` is what must be purchased (FR-MATL-007). This is the single calculation that connects materials to procurement, and it must be computed under a lock so two projects cannot both claim the same units (§5.3).

### 3.4 Stage D — Reservation

```
[Office] Reserve covered quantity
   → reservations row (ACTIVE)
   → requirement.fulfilment_status = RESERVED
```

Reservation semantics are precise and worth stating plainly:

> A reservation is a **claim on verified stock**, not a physical movement. It reduces what other projects may claim. It does not move anything, does not change physical location, and does not appear as a quantity movement in the ledger.

This distinction is why reservations live in their own table rather than as a movement type. Conflating "reserved" with "moved" produces systems where nobody can tell what is physically on the shelf.

### 3.5 Stage E — Purchase request

```
[Office] Shortfall → purchase request
   → purchase_requests (DRAFT) with lines referencing requirement lines
   → submit → status = SUBMITTED
```

A purchase request may aggregate shortfalls across multiple projects when the same item is needed by several (e.g. one cable order for three jobs). This is normal and must be supported, but each request line retains its link back to the requirement line so cost can be attributed.

### 3.6 Stage F — Approval

```
[Owner] Review → APPROVED or REJECTED (with reason)
   → purchase_approvals row + policy_snapshot
```

**The state distinction that matters most** (BR-PROC-001):

| Event | Means | Does **not** mean |
|---|---|---|
| Approval | A human with authority said yes | Money spent, goods ordered, goods received |
| PO issued | A document was sent to a supplier | Goods delivered, invoice received, money paid |
| Goods received | Physical items arrived and were counted | Invoice received, money paid |
| Payable recorded | Supplier billed us | Money paid |
| Payment recorded | A transfer reportedly happened | Payment verified |
| Payment confirmed | A second person verified the transfer | — |

Collapsing these is the operational failure the business currently lives with. Each is a separate row, a separate timestamp, a separate actor.

### 3.7 Stage G — Purchase order

```
[Office] Approved request → create PO (DRAFT)
   → verify supplier, prices, quantities
   → issue → allocate document number (server-side, atomic)
            → freeze snapshot of supplier + lines
            → generate PDF
            → status = ISSUED
```

Numbering follows [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md). A PO without a number is a draft and must be visibly distinguishable from an issued PO everywhere it appears.

### 3.8 Stage H — Delivery and receipt

```
[Supplier] delivers (possibly in parts, possibly late, possibly short)
   → [Office] post goods receipt (per line, per delivery)
   → status PARTIALLY_RECEIVED or RECEIVED
   → RECEIPT_IN movements written atomically
   → stock becomes available NOW (not at PO issuance)
```

**BR-MATL-005 is absolute:** a PO does not increase available stock. Only a posted goods receipt does. A system that shows stock as available when a PO is issued will cause the exact double-promising failure described in §2.1.

Partial receipt is a first-class case, not an edge case. Real suppliers deliver in parts. Each receipt is its own record with its own date, its own lines, its own discrepancies.

### 3.9 Stage I — Supplier invoice and payable

```
[Office/Finance] Record supplier invoice
   → payables row (RECORDED) linked to PO/receipts
   → due date, amount, optional attachment
```

A supplier invoice may arrive before, with, or after delivery. The model allows all three; the payable exists independently of the receipt. Discrepancies (invoice ≠ PO price, invoice for undelivered goods) are recorded, not silently accepted.

### 3.10 Stage J — Payment

```
[Finance] Record payment (amount, date, method, reference, allocation)
   → payments row, status = RECORDED
   → allocation rows against payables
[Owner/Finance] Confirm payment
   → status = CONFIRMED
```

**No bank integration exists.** The system records what a human reports and requires a second confirmation (BR-PROC-002, BR-PROC-004). Partial payment is normal and supported.

**[Confirmed]** the owner may personally transfer funds to suppliers. In that case the recording user is the owner and the confirming user should be a different authorised person; where that is impossible, the override path applies with reason and owner visibility.

### 3.11 End-to-end chain

```
Requirement (UNKNOWN)
   → Verify (VERIFIED, observed qty)
      → Compute coverage
         ├─ covered → Reserve ──────────────────────┐
         └─ shortfall → Purchase Request            │
                          → Approval (APPROVED)     │
                             → PO (ISSUED)          │
                                → Goods Receipt ────┤
                                   (PARTIAL/RECEIVED)
                                                    ▼
                                            Available stock
                                                    ▼
                                        Issue to technician
                                                    ▼
                              ┌──────────────┬──────────────┬─────────────┐
                          Consumed       Returned       Written off    Remaining
                              │              │              │              │
                              ▼              ▼              ▼              ▼
                        Project cost   Back to stock   Loss recorded   Still issued
```

---

## 4. Reservation lifecycle

### 4.1 States

| State | Meaning | Exits |
|---|---|---|
| `ACTIVE` | Quantity claimed; reduces availability for other projects | → `RELEASED`, → `CONSUMED` |
| `RELEASED` | Claim withdrawn; quantity returns to availability | terminal |
| `CONSUMED` | Reserved quantity was actually issued and used | terminal |

### 4.2 Transitions and rules

| From | To | Trigger | Rule |
|---|---|---|---|
| — | `ACTIVE` | Office reserves | Line must be `VERIFIED`; available ≥ requested |
| `ACTIVE` | `RELEASED` | Project cancelled, requirement reduced, or material no longer needed | Release reason required |
| `ACTIVE` | `CONSUMED` | Material issued to technician | Issue quantity ≤ reserved quantity, or the excess is unreserved and re-checked |

### 4.3 Release rules

| Event | Reservation effect |
|---|---|
| Project cancelled | All `ACTIVE` reservations released (BR-PROJ-005 flow) |
| Requirement quantity reduced | Excess reservation released first, then the reduction applies |
| Requirement line removed | Reservation must be released before removal is permitted |
| Work order cancelled | No automatic effect; reservations are project-scoped, not work-order-scoped |
| Project closed | Any `ACTIVE` reservation must be resolved before closure (closure condition 2) |

### 4.4 Why reservations are not movements

| | Movement | Reservation |
|---|---|---|
| Physical meaning | Something moved | Nothing moved |
| Affects shelf count | Yes | No |
| Affects availability for others | Indirectly | Directly |
| Reversible | Only by compensating entry | By release |
| Audited | Yes | Yes |

Keeping these separate means the physical ledger stays true even when commercial plans change. A project that reserves and then releases leaves the ledger untouched, which is correct: nothing physically happened.

---

## 5. Preventing double-counting across projects

This is the second-highest-risk area after numbering, because the failure is silent.

### 5.1 The three failure modes

| # | Failure | Cause | Consequence |
|---|---|---|---|
| F1 | Same units promised to two projects | No reservation concept, or reservations not netted | One project aborts on site |
| F2 | Same cost counted twice | Receipt, supplier invoice, and payment all counted as separate project costs | Inflated project cost, wrong margin |
| F3 | Quantity drifts from reality | Direct quantity edits, untracked issues/returns | Stock figure is fiction |

### 5.2 Controls for F1 — quantity

| Control | Mechanism |
|---|---|
| Verified-only reservation | Reservation requires `VERIFIED` line (U5) |
| Net-of-others availability | Coverage computed as verified on-hand minus other projects' active reservations (§3.3) |
| Locked evaluation | Availability computed under `SELECT … FOR UPDATE` on the item's balance row; check and insert in one transaction |
| Database constraint | Active reservation sum per item cannot exceed verified available; violations raise, never warn |
| Reconciliation | Periodic job recomputes availability from ledger + reservations and reports variance (FR-MATL-013) |

**Concurrency test obligation:** two simultaneous reservations for the last available units must produce exactly one success and one refusal. A sequential test proves nothing (AC-MATL-006).

### 5.3 Controls for F2 — cost

| Control | Mechanism |
|---|---|
| Single recognition point | Cost is recognised at **goods receipt** (proposed default), not at PO, not at payment |
| Unique source constraint | `UNIQUE (project_id, source_type, source_id)` on `project_cost_entries` |
| Other sources excluded, not added | Supplier invoice and payment are *views* of the same cost, recorded for AP/AR purposes, not added to project cost |
| Variance handling | If supplier invoice differs from receipt value, an explicit adjustment entry is recorded with reason — never a silent overwrite |
| Cost composition report | FR-FIN-007 shows cost by source so a double count is visible |

**[Open Question] Q-COST-01 — Recognition point.** Receipt (accrual at delivery) vs supplier invoice (accrual at billing) vs payment (cash basis). **Proposed default:** goods receipt, because the business cares about cost as soon as material exists on site, and cash-basis reporting understates cost during payment delays. **Decision trigger:** accountant input (Q-FIN-01). Whichever is chosen, only **one** point recognises cost; the others are excluded by constraint.

### 5.4 Controls for F3 — quantity integrity

| Control | Mechanism |
|---|---|
| Append-only ledger | `material_movements` has no update/delete path (FR-MATL-012) |
| No editable quantity | No UI or API exposes a quantity field for direct edit |
| Compensating corrections | Wrong entry → new reversal row referencing the original |
| Issue-line arithmetic | `consumed + returned + written_off ≤ issued`, enforced by CHECK (BR-MATL-004) |
| Reconciliation checkpoints | Per-project and per-item recomputation with variance reporting (FR-MATL-013) |

### 5.5 Reconciliation checkpoints

| Checkpoint | When | Compares |
|---|---|---|
| Pre-issue | Before handing material to a technician | Reserved vs required vs available |
| Post-return | When an issue is closed | Issued vs consumed + returned + written off |
| Pre-closure | At project closure | All lines resolved to a terminal fulfilment state |
| Periodic | Weekly/scheduled | Ledger sum vs cached balance per item |
| Post-migration | After spreadsheet import | Imported quantities vs first physical count |

A variance is never auto-corrected. It is reported, investigated, and resolved with a documented adjustment.

---

## 6. Issue, consumption, return, damage

### 6.1 Issue

```
[Office] Issue materials to technician for project
   → material_issues (OPEN) + issue lines
   → ISSUE_OUT movements
   → reservation CONSUMED (where applicable)
```

An issue is a **handover of custody**. From that moment the technician is accountable for the quantity. This is why issue records name a technician and a project, and why closure requires every issue to be resolved.

### 6.2 Consumption

```
[Technician or Office] Record consumption
   → consumed_quantity increases on the issue line
   → CONSUME movement
```

Consumption is the point at which material becomes project cost (subject to Q-COST-01). Unconsumed material is still in the technician's custody, not yet a cost.

### 6.3 Return

```
[Technician or Office] Return unused material
   → returned_quantity increases
   → RETURN_IN movement
   → availability restored
```

Return restores availability. Returned material is usable by another project — but only after verification if its condition is in doubt. **[Proposed]** returned material does not require re-verification unless flagged as condition-unknown; requiring re-verification of every return would create friction that leads to returns being skipped entirely.

### 6.4 Damage, loss, write-off

```
[Office] Record write-off with reason code
   → written_off_quantity increases
   → WRITE_OFF movement
```

Reason codes: `DAMAGED_ON_SITE`, `DAMAGED_IN_TRANSIT`, `LOST`, `OBSOLETE`, `THEFT`, `OTHER`.

Write-off is not a silent deletion. The quantity remains visible in the ledger and in the project's material report, marked as lost. This is what makes loss visible to the owner — the opposite of the current situation where material simply stops being mentioned.

### 6.5 Invariant

For every issue line, at all times:

```
consumed_quantity + returned_quantity + written_off_quantity ≤ issued_quantity
```

Remaining (still in technician custody) = `issued − consumed − returned − written_off`.

A project cannot close while any issue line has remaining > 0 (closure condition 2).

---

## 7. Material readiness status (project level)

Defined in [`../PRD.md`](../PRD.md) §9.3. **Derived, never hand-set.**

| Status | Derivation |
|---|---|
| `NOT_ASSESSED` | No requirement lines |
| `REQUIREMENTS_INCOMPLETE` | ≥1 line with `stock_status = UNKNOWN` |
| `AWAITING_PROCUREMENT` | All lines verified but ≥1 has unresolved shortfall |
| `READY_TO_START` | All non-optional lines covered by verified stock or received goods |
| `PARTIALLY_ISSUED` | Some quantity issued, not all |
| `FULLY_ISSUED` | All required quantity issued |
| `CLOSED` | All issued quantity consumed/returned/written off |

### 7.1 Interaction with project status

| Project status | Readiness | Permitted? |
|---|---|---|
| `SCHEDULED` | `READY_TO_START` | Yes — normal |
| `SCHEDULED` | `AWAITING_PROCUREMENT` | Yes — scheduling ahead of purchasing is normal |
| `IN_PROGRESS` | `READY_TO_START` / `PARTIALLY_ISSUED` | Yes |
| `IN_PROGRESS` | `AWAITING_PROCUREMENT` / `REQUIREMENTS_INCOMPLETE` | **Only with override + reason** (BR-PROJ-003) |
| `CLOSED` | not `CLOSED` | **Refused** (closure condition 2) |

The override exists because site work sometimes starts with partial materials (cabling before cameras arrive). It is logged and visible to the owner, which converts a normal workaround into a monitored pattern. If overrides become frequent, that is evidence the readiness model needs adjustment — not evidence the override should be removed.

---

## 8. Purchasing and approval detail

### 8.1 Approval policy

**[Open Question] Q-PROC-01 — Approval thresholds.** Not confirmed. The brief states the owner *sometimes* approves purchases, implying not always, but no threshold is known.

**[Proposed] default policy, configurable:**

| Condition | Approval required |
|---|---|
| Amount above a configurable threshold (default: any amount, until configured) | Yes |
| Supplier not in an approved supplier list | Yes |
| Purchase for a project already flagged | Yes |
| Requested by a user who also holds `purchase.approve` | Yes, by a different holder where one exists |
| No policy configured at all | **Yes — fail closed** (FR-PROC-003) |

**Fail-closed is deliberate.** A system that defaults to "no approval needed" when misconfigured will eventually let unapproved spending through. A system that defaults to "approval needed" merely causes a minor inconvenience.

**Rationale for the default:** the confirmed fact is that the owner approves purchases. The unconfirmed part is *when*. Defaulting to "always" matches the confirmed behaviour and can only be loosened with the owner's explicit configuration.

### 8.2 Approval mechanics

| Aspect | Behaviour |
|---|---|
| Who may approve | `OWNER`; `OPS_ADMIN` only via override |
| Self-approval | Blocked when the requester holds approval permission, unless override with reason |
| Rejection | Requires a reason; terminal for that request (a new request is created if needed) |
| Re-submission | A rejected request is not edited back to `SUBMITTED`; a new request is raised, preserving the rejection |
| Policy snapshot | Stored with the decision so the historical rule is explainable after thresholds change |
| Time limit | No auto-approval, no timeout escalation in MVP — an unapproved request simply stays `SUBMITTED` and appears in an ageing report |

### 8.3 Partial delivery and partial payment

| Case | Handling |
|---|---|
| Partial delivery | Multiple goods receipts against one PO; PO becomes `PARTIALLY_RECEIVED` |
| Over-delivery | Blocked unless an explicit over-receipt approval is recorded (BR-PROC-003) |
| Under-delivery never completed | PO remains `PARTIALLY_RECEIVED`; ageing report surfaces it |
| Wrong item delivered | Recorded as `WRONG_ITEM` discrepancy; does not increase usable stock |
| Damaged on arrival | Recorded as `DAMAGED`; stock increases only for accepted quantity |
| Partial payment | Multiple allocations against one payable; status `PARTIALLY_PAID` |
| Overpayment | Recorded as unapplied credit, never a negative balance (BR-FIN-003) |
| Payment before invoice | Allowed; the payable is created and immediately allocated |

### 8.4 Supplier invoice discrepancies

| Discrepancy | Handling |
|---|---|
| Price higher than PO | Recorded; requires resolution (accept with reason, or dispute) |
| Quantity higher than received | Blocked until goods are received or the invoice is disputed |
| Invoice for cancelled PO | Blocked; requires explicit handling |
| Duplicate supplier invoice number | Warned and flagged, not silently accepted |

**[Open Question] Q-PROC-02 — Supplier delivery and goods-receipt practices.** Whether suppliers provide delivery notes, whether the office signs anything, and how discrepancies are currently handled is unconfirmed. **Proposed default:** capture an optional delivery-note reference and support discrepancy recording; do not require a delivery note. **Decision trigger:** Phase 0 interviews with office staff.

---

## 9. Migration path: project-scoped → centralised stock

The brief requires an incremental path, so the MVP must not paint the business into a corner.

### 9.1 What stays the same

The movement ledger, item catalogue, reservation model, and reconciliation logic are already correct for centralised stock. The ledger records `item_id` and quantity regardless of location. Adding locations later means adding a `location_id` dimension, not rebuilding the model.

### 9.2 What changes at the trigger point

| Area | MVP | Centralised |
|---|---|---|
| Location | Single implicit | Multiple, with transfers |
| Balance | Per item | Per item per location |
| Reservation | Per item | Per item per location |
| Verification | Per requirement line | Per stock count at a location |
| Replenishment | Reactive (per project) | Min/max levels, reorder points |
| Valuation | None | FIFO or weighted average (**[Open Question]** Q-INV-01) |
| Reporting | Project material status | Stock-on-hand, ageing, turnover |

### 9.3 Migration trigger

Move to centralised stock when **any** of the following holds:

1. Two or more concurrent projects repeatedly contend for the same item and reservations block work.
2. The owner wants a stock value figure that the project view cannot produce.
3. Physical stock is stored in more than one place and staff lose track of which.
4. Purchase decisions require knowing company-wide consumption history.

**Rationale for a trigger rather than a date:** building warehouse features before the business needs them adds complexity to every screen the office staff uses daily. The trigger is observable, so the decision is made on evidence.

### 9.4 Data migration at the trigger

1. Introduce `locations`; create the default location representing today's office/warehouse.
2. Backfill `location_id` on all existing movements to the default location (a single, verifiable update).
3. Perform a full physical stock count per item; record as opening balances.
4. Reconcile against the ledger; investigate every variance before enabling multi-location.
5. Only then enable transfers and per-location reservations.

**The physical count is not optional.** Without it, the centralised model inherits every inaccuracy of the project-scoped era and gains new ones.

---

## 10. Reporting obligations

| Report | Content | Requirement |
|---|---|---|
| Project material status | Per line: required, stock status, reserved, purchased, received, issued, consumed, returned, written off, remaining | FR-RPT-003 |
| Procurement pipeline | Requests by state, POs by state, ageing | FR-RPT-005 |
| Outstanding deliveries | POs `PARTIALLY_RECEIVED` with days outstanding | FR-RPT-005 |
| Material exceptions | Discrepancies unresolved, write-offs, over-receipts | FR-RPT-003 |
| Reconciliation variances | Cached vs ledger differences | FR-MATL-013 |
| Readiness overrides | Projects started with incomplete materials | FR-AUD-004 |

---

## 11. Acceptance criteria references

| Behaviour | AC |
|---|---|
| Requirement line defaults to `UNKNOWN` | AC-MATL-002 |
| Reservation refused on unverified line | AC-MATL-005 |
| Concurrent reservations cannot exceed verified stock | AC-MATL-006 |
| PO does not increase available stock | AC-PROC-005 |
| Partial goods receipt accumulates correctly | AC-PROC-007 |
| Receipt discrepancy blocks line completion | AC-PROC-008 |
| Issue/consume/return arithmetic cannot exceed issued | AC-MATL-008, AC-MATL-010 |
| Write-off remains visible | AC-MATL-011 |
| No direct quantity edit path exists | AC-MATL-012 |
| Reconciliation detects injected variance | AC-MATL-013 |
| Approval required by policy, fail-closed | AC-PROC-003 |
| Payment recorded ≠ confirmed | AC-FIN-009 |
| Project cost not double counted | AC-FIN-007 |

---

## 12. Cross-references

- Status definitions: [`../PRD.md`](../PRD.md) §9.3–9.7
- Entities and constraints: [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §5, §6
- Project states: [`./PROJECT_LIFECYCLE.md`](./PROJECT_LIFECYCLE.md)
- Payment and payable flow: [`./FINANCE_AND_DOCUMENTS.md`](./FINANCE_AND_DOCUMENTS.md)
- Who may approve or issue: [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md)
