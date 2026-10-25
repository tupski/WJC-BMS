# Finance and Documents — WJ-CCTV Business Operations System

| Field | Value |
|---|---|
| Status | Proposed |
| Version | 0.1 |
| Scope owner | This document owns quotations, invoices, payments, payables, expenses, AR/AP, document generation and history, and correction procedures |
| Requirement source | [`../PRD.md`](../PRD.md) §6.6, §9.6–9.10, §10 |
| Numbering authority | [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) |
| Data model | [`../architecture/DATA_MODEL.md`](../architecture/DATA_MODEL.md) §7, §8 |

---

## 1. Three layers, kept separate

PRD §10 defines the layers. This document implements them without letting them bleed into each other.

| Layer | What it answers | MVP | Tables |
|---|---|---|---|
| **L1 — Operational finance and cash tracking** | What came in, what went out, what did this project cost? | In scope | `payments`, `expenses`, `project_cost_entries` |
| **L2 — AR/AP** | Who owes us, whom do we owe, how overdue? | In scope | `invoices`, `payables`, `payment_allocations` |
| **L3 — Formal accounting** | What are the statutory statements? | **[Out of Scope]** (Q-FIN-01) | none |

### 1.1 Why the boundary is drawn here

The business needs to chase receivables and pay suppliers. It does not currently have, and has not confirmed needing, journals, a chart of accounts, or tax filings produced by this system.

Building a general ledger without an accountant's requirements produces statements nobody can file and a maintenance burden nobody asked for. Building AR/AP and cash tracking produces immediate operational value.

**The boundary is a decision, not an oversight.** When Q-FIN-01 is answered affirmatively, L3 is scoped as a separate module that *consumes* L1/L2 data — it does not replace them.

---

## 2. Document types

| Type | Series | Issued when | Immutable after issuance |
|---|---|---|---|
| Quotation | `QUO` | Sent to customer | Yes |
| Supplier Purchase Order | `PO` | Sent to supplier | Yes |
| Sales Invoice | `INV` | Billed to customer | Yes |
| Credit Note | `CN` | Correcting an invoice | Yes |
| Receipt | `RCP` | Payment received from customer | Yes |
| Delivery/Work Order document | `WO` | Work assigned (optional print) | Yes |
| Supplier Payable reference | `PAY` (optional) | Supplier invoice recorded | Internal only |

**Customer POs are not our documents.** A customer's PO number is recorded as data (`customer_purchase_orders.po_reference`) and never enters our numbering series (DATA_MODEL §7.1). Confusing the two is a real risk in small businesses and produces numbering conflicts.

**Numbering rules are not restated here.** See [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md) — one authority, one place.

---

## 3. Quotation → invoice flow

```
[Office] Create quotation (DRAFT)
   → lines, prices, validity
   → issue → number allocated → PDF stored → SENT
        ↓
   [Customer] accepts or rejects
        ↓
   ACCEPTED ──► project may reference it (optional, BR-PROJ-001)
   REJECTED ──► terminal
   EXPIRED  ──► validity passed without decision
        ↓
[Finance] Issue invoice against project
   → number allocated (INV series)
   → snapshot frozen
   → PDF stored
   → ISSUED
```

### 3.1 Optionality

BR-PROJ-001: a project may exist without a quotation. Many small jobs are agreed verbally or over WhatsApp. The system must support that path without friction — no warning wall, no forced "no quotation" reason. The quotation link is simply empty.

**Why this matters:** systems that require a quotation for every project get abandoned by businesses that do small jobs. The requirement is to *support* quotations, not to *require* them.

### 3.2 Quotation is not revenue

An accepted quotation creates no receivable and no revenue. Only an issued invoice does. A quotation is an offer; treating it as a financial event inflates reported revenue and creates phantom receivables.

---

## 4. Invoice issuance

### 4.1 Issuance sequence (single transaction)

```
1. Validate: lines present, totals consistent, due date ≥ invoice date
2. Authorise: invoice.issue permission
3. Allocate number atomically  ← DOCUMENT_NUMBERING §5
4. Freeze snapshot (customer, address, lines, prices, totals)
5. Render PDF, store immutably, record hash
6. Set status = ISSUED, issued_at, issued_by
7. Write audit event
8. COMMIT — all or nothing
```

Any failure in steps 3–7 rolls back everything. A number cannot be consumed without a document, and a document cannot exist without a number.

### 4.2 The snapshot

FR-FIN-003 requires that later master-data edits do not alter issued documents. The snapshot captures, at issuance:

| Captured | Source |
|---|---|
| Customer name, address, contact | Live customer record at that moment |
| Line descriptions, quantities, unit prices | Invoice lines at that moment |
| Totals, discount, tax | Computed at that moment |
| Currency | Configured |
| Invoice date, due date | As issued |

**Why denormalisation is correct here:** normalisation exists to avoid update anomalies in mutable data. An issued invoice is not mutable data — it is a historical fact. Storing the fact as it was is the only correct representation.

**Failure this prevents:** a customer renames their company, or an item price changes, and a reprinted invoice no longer matches what the customer received. In a dispute, the business must be able to produce exactly what was issued. This is why M4 requires byte-identical re-download.

### 4.3 Draft vs issued

| | Draft | Issued |
|---|---|---|
| Document number | None | Allocated, permanent |
| Editable | Yes | **No** |
| Appears in AR | No | Yes |
| Has PDF | No (preview only) | Yes, stored |
| Counts in reports | No | Yes |

The distinction must be visible everywhere: lists, PDFs, reports, exports. A draft that looks like an invoice causes accidental double-billing.

---

## 5. Payments and allocation

### 5.1 Recording

```
[Finance] Record payment
   amount, date, method, reference, party
   → payments row (RECORDED)
   → allocate to invoices/payables
```

### 5.2 Recording vs confirming

**BR-PROC-002** — the distinction is not bureaucratic:

| | Recorded | Confirmed |
|---|---|---|
| Who | Finance/Admin | A second authorised person |
| Meaning | "A payment reportedly happened" | "Someone verified it happened" |
| Effect on AR/AP | Reduces balance | Reinforces; enables reporting confidence |
| Reversible | Yes (reverse with reason) | Yes (reverse with reason) |

**Why two steps:** the confirmed operating condition is that the owner sometimes transfers funds personally. The person recording the payment may be the same person who received a WhatsApp message saying "already transferred". Without a confirmation step, an unverified claim reduces receivables and hides a real debt.

In a three-person office, the confirmer is often the owner. Where the recorder is also the only available confirmer, an override with reason applies and the owner sees it in the override report.

### 5.3 Allocation rules

| Rule | Behaviour |
|---|---|
| Partial payment | Allowed; invoice becomes `PARTIALLY_PAID` |
| Multiple invoices | One payment may allocate across several |
| Overpayment | Recorded as unapplied credit, never a negative balance (BR-FIN-003) |
| Allocation exceeds balance | Refused with the remaining balance shown |
| Unallocated payment | Allowed; appears as an unapplied credit pending allocation |
| Reversal | Creates a compensating record; original retained |

### 5.4 Outstanding balance

```
outstanding = total_amount − SUM(allocations on non-reversed payments)
```

Computed, not stored as an editable field. `amount_paid` is a cache with a reconciliation path (DATA_MODEL §12).

### 5.5 Overdue

`OVERDUE` is a **derived display state**, not a stored transition (PRD §9.8):

```
overdue = due_date < today AND status ∈ {ISSUED, PARTIALLY_PAID}
```

Deriving it means no nightly job can leave a stale flag, and no user can accidentally clear an overdue status.

---

## 6. Payables (supplier side)

```
[Finance] Record supplier invoice
   → payables (RECORDED)
   → link to PO / goods receipts
   → due date, amount, attachment
        ↓
[Finance] Record payment → allocate
   → PARTIALLY_PAID / PAID
        ↓
[Owner/Finance] Confirm payment
```

### 6.1 Payable without PO

**[Assumption]** some supplier purchases happen without a PO (urgent retail purchase by a technician). The model allows a payable without a PO link, flagged as `no_po` so it appears in a control report. Blocking such payables outright would push these purchases outside the system entirely — worse than recording them imperfectly.

### 6.2 Disputes

A payable may be `DISPUTED` with a reason. Disputed payables are excluded from "due now" totals but remain visible. This prevents both premature payment and silent forgetting.

---

## 7. Expenses

| Type | Attribution |
|---|---|
| Project expense | Linked to a project; enters project cost |
| Operational expense | Not project-linked; enters company overhead |

Examples: fuel and tolls for site visits, equipment rental, consumables, office costs.

**[Assumption]** technicians may incur small cash expenses. **[Proposed]** MVP allows office staff to record expenses on a technician's behalf; technician self-service expense entry is deferred. Rationale: reimbursement workflow (approval, receipt capture, payout) is a distinct feature; recording expenses is enough for cost visibility in MVP.

---

## 8. Correction procedures

This is where document control and finance intersect, and where the numbering standard's rules become operational.

### 8.1 The rule

**An issued document is never edited.** Not by the owner, not by an administrator, not by a database fix in normal operation. (BR-FIN-001, BR-FIN-004)

### 8.2 Correction paths

| Situation | Correct procedure | Never do this |
|---|---|---|
| Wrong amount on issued invoice | Issue credit note for the full amount, issue a replacement invoice | Edit the invoice |
| Wrong customer name | Credit note + replacement invoice, or a documented addendum | Edit the snapshot |
| Invoice issued to wrong customer | Void with reason, reissue to correct customer | Reassign the invoice |
| Customer disputes one line | Credit note for that line only | Edit the line |
| Invoice number rejected by client | Void, reissue under a format the client accepts (after Q-NUM-01) | Change the number on the issued document |
| Payment recorded in error | Reverse with reason; record correct payment | Delete the payment |
| Duplicate invoice created | Void the duplicate with reason referencing the original | Delete it |

### 8.3 Why voiding beats deletion

A voided invoice keeps its number. That number is never reused (BR-FIN-002). The audit trail shows: issued at T1 by A, voided at T2 by B with reason R. If the document were deleted, the business would have no evidence it ever existed — and a customer producing a copy would create an unexplainable discrepancy.

**Gaps in the sequence are acceptable and expected.** A gap means "a document was issued and voided", which is a normal business event. Chasing gap-free numbering leads businesses to reuse numbers, which is precisely the failure mode that caused the original conflict.

### 8.4 Interaction with payments

| Scenario | Handling |
|---|---|
| Invoice voided after partial payment | Payment allocations are reversed; the payment becomes unapplied credit; refund or reallocation is a separate recorded action |
| Credit note issued after full payment | Customer has a credit balance; refund recorded as an outbound payment |
| Invoice voided after full payment | Same as above; the cash fact remains |
| Replacement invoice issued | Links to the original via `replaced_by_invoice_id`; the original is marked `CREDITED` |

**Principle:** voiding a document never erases a cash movement. Money that arrived is a fact independent of the paperwork describing it.

### 8.5 Interaction with audit

Every issuance, void, credit note, and replacement produces an audit event (FR-AUD-001) and a document number allocation record (DATA_MODEL §8). The chain is: allocation → issuance → void/credit → replacement, each with actor and timestamp. See [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md).

---

## 9. Document generation and history

### 9.1 PDF generation

| Aspect | Rule |
|---|---|
| When | At issuance only |
| Storage | Private object storage, immutable |
| Hash | SHA-256 recorded at generation (FR-DOC-003) |
| Regeneration | Permitted only as a new derived artefact; never replaces original bytes (FR-DOC-008) |
| Layout | Company header, document number, date, party, lines, totals, terms, signature block |
| Language | Indonesian (NFR-OPS-007) |
| Currency | IDR |

**Why store the PDF rather than regenerate on demand:** regeneration from live data is how issued documents silently change. Storing the bytes makes the historical artefact verifiable against its hash. If the template is later redesigned, old documents keep their original appearance — which is what a customer expects when they open a file they received last year.

### 9.2 Document history

FR-DOC-005 requires retrieval by number with full history:

| Shown | Content |
|---|---|
| Document | Number, type, status, issue date, actor |
| Snapshot | Customer, lines, totals as issued |
| PDF | Original file, downloadable |
| Related | Credit notes, replacements, payments, project |
| Timeline | Issued → paid/partially paid → voided/credited, with actors |

### 9.3 Retrieval

Retrieval is by document number (human-facing) or by entity (project, customer). Document numbers are unique per series (NFR-DATA-002); a number without its series prefix may be ambiguous across types, so lookup requires type or resolves across series with the type shown.

---

## 10. Receivables and payables reporting

| Report | Content | Requirement |
|---|---|---|
| AR ageing | Per customer: current, 1–30, 31–60, 61–90, 90+ | FR-FIN-005, FR-RPT-004 |
| AP ageing | Per supplier: same buckets | FR-FIN-010, FR-RPT-004 |
| Outstanding invoices | Per invoice: issued, paid, outstanding, days overdue | FR-FIN-005 |
| Project billing status | Per project: billed, paid, outstanding | FR-RPT-002 |
| Unbilled completed work | Completed projects with unbilled scope | PROJECT_LIFECYCLE §6.3 |
| Override report | All financial overrides with reasons | FR-AUD-004 |

### 10.1 Ageing basis

**[Proposed]** Ageing is computed from **due date**, not invoice date, because due date is what determines lateness. Both are displayed so the office can see the difference.

---

## 11. Project cost

### 11.1 Sources

| Source | Recognised? | Notes |
|---|---|---|
| Goods receipt | **Yes** (proposed default) | Cost exists when material exists |
| Supplier invoice | No (view of same cost) | Recorded for AP |
| Payment | No (view of same cost) | Recorded for cash |
| Material issue/consumption | No | Physical movement, not a new cost |
| Expense | Yes | Distinct cost |
| Labour | **No** in MVP | **[Open Question]** whether labour cost is needed for profitability |

### 11.2 Double-counting control

`UNIQUE (project_id, source_type, source_id)` on `project_cost_entries` (DATA_MODEL §7.1, BR-FIN-005).

**The subtle failure this prevents:** a project showing material cost of 3× the real value because receipt, invoice, and payment were each added. This produces a business that appears unprofitable on profitable jobs and leads to wrong pricing decisions.

### 11.3 Cost composition

FR-FIN-007 requires composition by source, so a double count would be visible rather than hidden in a total.

---

## 12. Tax and statutory considerations

**[Open Question] Q-FIN-01** — tax obligations are unconfirmed.

| Question | Status |
|---|---|
| Is PPN (VAT) applicable and at what rate? | **[Open Question]** |
| Must invoices show a tax invoice (faktur pajak) format? | **[Open Question]** |
| Is PPh withholding required? | **[Open Question]** |
| Are there e-invoicing (e-Faktur) integration requirements? | **[Open Question]** |

**[Proposed] default:** the invoice model carries `tax_amount` and a configurable rate, displayed as a line. No statutory format, no e-Faktur integration, no filing. **Decision trigger:** accountant confirmation in Phase 0.

**Why not guess:** tax formatting errors on issued invoices create legal exposure. A neutral, correct invoice without statutory claims is safer than a wrong imitation of one.

---

## 13. Failure behaviours

| Failure | Required behaviour |
|---|---|
| Number allocation fails | No invoice created; clear error; no partial state |
| PDF generation fails | Transaction rolls back; number not consumed |
| Duplicate submission of issuance | Idempotent; one invoice, one number |
| Payment exceeds balance | Refused with remaining balance shown |
| Concurrent payments on one invoice | Serialised; total allocation never exceeds balance |
| Snapshot missing on issued invoice | Refused at issuance (cannot issue without snapshot) |
| File storage unavailable at issuance | Issuance fails; no numbered document without its PDF |

---

## 14. Acceptance criteria references

| Behaviour | AC |
|---|---|
| Quotation issued with number and PDF | AC-FIN-001 |
| Invoice issued with allocated number | AC-FIN-002 |
| Issued invoice immutable after master-data edit | AC-FIN-003 |
| Partial payments and accurate outstanding | AC-FIN-004 |
| Overdue derived from due date | AC-FIN-005 |
| Project cost not double counted | AC-FIN-007 |
| Payment recorded ≠ confirmed | AC-FIN-009 |
| Void retains number and record | AC-FIN-011 |
| Credit note and replacement preserve original | AC-FIN-012 |
| Private storage, signed URLs only | AC-DOC-001 |
| Issued PDF immutable | AC-DOC-008 |

---

## 15. Cross-references

- Numbering: [`../standards/DOCUMENT_NUMBERING.md`](../standards/DOCUMENT_NUMBERING.md)
- Audit: [`../standards/AUDIT_LOGGING.md`](../standards/AUDIT_LOGGING.md)
- Payables from procurement: [`./PROCUREMENT_AND_MATERIALS.md`](./PROCUREMENT_AND_MATERIALS.md) §3.9–3.10
- Project closure and billing: [`./PROJECT_LIFECYCLE.md`](./PROJECT_LIFECYCLE.md) §6
- Finance permissions: [`../architecture/SECURITY_AND_ACCESS_CONTROL.md`](../architecture/SECURITY_AND_ACCESS_CONTROL.md) §3.2
