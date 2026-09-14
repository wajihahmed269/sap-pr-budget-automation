# SAP PR Budget Automation — Classifier Rules v1

**Status:** Working specification  
**Purpose:** Single source of truth for classifying SAP Purchase Requisition emails into **RM**, **Consumables**, **Skip**, or **Review** before production implementation.

---

## 1. Scope

This automation only manages:

1. **RM — Repair & Maintenance Budget**
2. **Consumables Budget**

Anything outside those two budgets is not posted to the budget workbook. Out-of-scope items are still recorded in the audit log with a Skip reason.

Primary outputs:

- RM posting
- Consumables posting
- Skip — Project/DIES
- Skip — CAPEX/Asset
- Skip — Service/Fee
- Review

**Review is an exception bucket, not the default.**  
The system should automate as many PRs/lines as reasonably possible using deterministic rules and historical references.

---

## 2. Core Routing Priority

### Rule 1 — Explicit RM code has highest priority

If the PR header explicitly contains a valid RM budget code such as:

- `E-4`
- `C-11`
- `G-2`
- `J-2`
- `B1(C)` / equivalent valid RM reference
- any other valid code found in the RM workbook

then the PR or specified line(s) must be posted to that RM code.

The material/item description **does not have to match** the description written against that RM budget line.

This is intentional because management may choose to charge a particular purchase to a specific RM code.

### Important override

Even if the PR is project-related, DIES-related, or otherwise looks out of scope, an **explicit valid RM code still wins**.

---

## 3. Single RM Code vs Multiple RM Codes

### 3.1 One explicit RM code for the PR

If one valid RM code is clearly given and no item-by-item split is specified:

> `Budget: E-4`

then the whole in-scope PR amount is posted to **E-4**.

The posting amount is the **SAP Total shown in the email**.

---

### 3.2 Multiple explicit RM codes tied to specific items

If the PR header explicitly assigns different RM codes to different items, post each SAP line total to the stated code.

#### Real example — PR#2000005510

Header:

- `G-2 (Sander 5" U-55)`
- `C-11 (Walkie Talkie)`

SAP lines:

| Item | SAP Line Total | RM Code | Posting |
|---|---:|---|---:|
| Double Action Sander 5" U-55 | PKR 62,500 | G-2 | PKR 62,500 |
| Walkie Talkie 16 Channel | PKR 13,530 | C-11 | PKR 13,530 |

Control total:

`62,500 + 13,530 = 76,030`

which must reconcile to the SAP PR Total of **PKR 76,030**.

#### Real example — PR#2000005477

If the header states:

- C-11 for Polishing Stone
- C-11 for Foot Mat
- G-2 for Angle Grinder

then each line is posted to its explicit RM code using the SAP line total.

---

### 3.3 Partially mapped multi-RM PR

If some lines have clear RM codes and another line does not:

- post the clearly mapped lines automatically
- intelligently match the unmapped line against the RM workbook/history
- if the unmapped line still cannot be matched confidently, put **only that line** into Review

Do **not** hold the entire PR just because one line is uncertain.

---

## 4. `PR SCM Local ASC` Rule

If there is **no explicit RM code** and the subject contains:

`PR SCM Local ASC`

then the **whole PR is routed to Consumables**.

This is a client-defined business rule.

### Real example — PR#2000005428

Because the subject is `PR SCM Local ASC` and there is no RM code, the whole PR belongs to **Consumables**, even if individual lines look mixed or unusual.

Do not split such a PR into RM / Production Material / other buckets unless a higher-priority explicit RM instruction exists.

---

## 5. Consumables Matching

Once a PR is routed to Consumables, classify the individual material lines inside the Consumables budget.

### 5.1 Existing material

For each line:

1. check SAP Material Number
2. check Description
3. compare against the existing `Consuambles` workbook sheet

If Material Number exists and Description reasonably agrees:

- update that existing Consumables row
- use the SAP line/PR amount as applicable
- update utilization and remaining budget according to the workbook structure

Material Number + Description is a high-confidence match.

---

### 5.2 New Consumable material

If a genuine Consumable material is not already in the sheet:

- automatically create a new row
- preserve its SAP Material Number
- preserve its Description
- preserve UOM and other usable SAP details
- place it under the most appropriate existing Consumables section/category

Do **not** send a new material to Review merely because it is new.

---

### 5.3 Consumables section/category inference

Use the workbook and historical decisions as the reference library.

The matcher should consider:

1. exact Material Number
2. description similarity
3. keywords/synonyms
4. neighboring/similar items already in the same section
5. historical PR decisions
6. department/subject context where useful

Example:

- new Uniform item → same section as existing Uniform items
- new stationery item → same section as existing Printing & Stationery-type items
- new packaging material → same section as similar packaging items

If a new item could belong to more than one section, choose the strongest historical/workbook match.

Only genuinely unclear items should enter Review.

---

## 6. Project / DIES

Project-based PRs are outside the automation unless an explicit RM code overrides them.

Typical indicators include combinations such as:

- `Project AIL`
- `Project PSMC`
- `Project IMC`
- `Local Dies`
- `Import Dies`
- die-project tooling
- `DIES-*` materials
- project-specific dies/components/tooling

### Real examples

PRs such as:

- PR#2000005438
- PR#2000005439
- PR#2000005440
- PR#2000005511

are Project/DIES examples and should be:

**Skip — Project/DIES**

when no explicit RM code is present.

### Unknown conflicting combinations

If a future email contains a combination never seen in the historical 228-email set, for example:

- subject `PR SCM Local ASC`
- strong Project/DIES wording
- no RM code

and the rules conflict, place it in **Review** rather than inventing a new automatic rule.

Review is only for such unusual conflicts, not for normal emails.

---

## 7. CAPEX / Assets

CAPEX and asset purchases are outside the automation.

Examples:

- Yaris / passenger vehicles
- Hino / trucks
- laptops
- desktops
- major IT hardware
- major machinery/equipment
- clearly capital construction/improvements
- other capital assets

Result:

**Skip — CAPEX/Asset**

These PRs should still be written to the audit log but should make **zero budget change** to RM or Consumables.

---

## 8. Services and Fees

General professional, administrative, manpower, and similar services are outside scope.

Examples:

- consultant fee
- professional consultancy charges
- legal/professional fee
- retainership fee
- security guard/service
- manpower service
- advertisement service
- similar non-maintenance service fees

Result:

**Skip — Service/Fee**

---

## 9. Technical Maintenance Services

Do **not** assume all services are skipped.

Technical maintenance services can belong to RM when they match the RM budget or have an explicit RM code.

Examples from the RM workbook include:

- Weigh Bridge Calibration & Service Contract
- Vibration Analysis
- Compressor efficiency analysis
- Horiuchi contract
- Solar Maintenance Annual Contract
- CCTV annual service contract
- Lift annual service contract
- other technical machine/plant maintenance contracts

Therefore:

**technical maintenance service + valid RM instruction/reference → RM**

while:

**general professional/admin/manpower service → Skip**

---

## 10. Blank `Budget ()` / Missing RM Code

If the PR contains a blank budget reference such as:

`Budget ()`

do not automatically send it to Review.

First compare the PR description against the RM workbook.

If there is a strong/exact match:

- auto-map to the matching RM line

Example:

`Budget () + Vibration Analysis`

can map to the RM line for Vibration Analysis.

If the match is weak or genuinely ambiguous:

- Review

---

## 11. SAP Amount Rule

### Authoritative booking amount

Always use the **SAP Total amount exactly as shown in the email**.

Do not recalculate the booking amount from:

`Quantity × Price`

Quantity × Price may be used only as a validation check.

### For multi-line RM splits

Use the **SAP line Total** for each explicitly assigned line.

The overall PR Total is used as a reconciliation/control amount.

Example:

- line 1 = PKR 62,500
- line 2 = PKR 13,530
- PR Total = PKR 76,030

Required control:

`sum(line postings) = SAP PR Total`

---

## 12. Duplicate Emails and Revisions

PR Number is the main deduplication key.

Multiple SAP emails may exist for the same PR because of:

- approval levels
- revisions
- re-release
- unrelease
- content changes

The system must preserve revision/history records and must never blindly double-book the same PR.

For audit/comparison, the latest version is treated as the active version.

### Still pending client confirmation

The following production behavior is **not finalized yet**:

> If an RM or Consumables PR was already posted and SAP later sends it as Unreleased, Cancelled, Deleted, or with a revised/reduced amount, should the system automatically reverse/update the previous utilization or require another treatment?

Until the client answers, this behavior must remain configurable / pending.

---

## 13. Foreign Currency

Some RM PRs are in currencies such as USD.

### Still pending client confirmation

Need confirmation on how foreign-currency amounts should affect a PKR budget:

- convert to PKR using a defined exchange rate, or
- follow the client's existing workbook/manual treatment

Do not hard-code an FX method until the client answers.

---

## 14. Review Rules

Review should be used only for genuinely unresolved cases.

Examples:

- conflicting high-priority signals not covered by established rules
- an RM line that cannot be confidently matched after all reference checks
- a partially mapped multi-RM PR where one specific line remains unresolved
- genuinely ambiguous RM vs CAPEX situations
- genuinely ambiguous future cases not represented in the historical dataset

### Important

If 2 out of 3 lines are known:

- post those 2 lines automatically
- Review only the unresolved 3rd line

Do not hold the entire PR.

---

## 15. Skip Audit Behavior

Skipped PRs must still be stored in the audit database.

Suggested fields:

- PR Number
- email date/time
- subject
- header note
- SAP total
- currency
- Skip reason
- rule that caused the Skip
- source email/message identifier
- revision/version information

Skipped PRs make **zero changes** to RM/Consumables budgets.

On the dashboard, skipped PRs should appear lower down / secondary so they do not dominate the main finance view.

---

## 16. Classification Audit Trail

Every automated decision should store:

- PR Number
- PR line/item number where applicable
- Material Number
- Description
- SAP line total
- SAP PR total
- currency
- classification
- RM code or Consumables section
- reason
- matched rule
- confidence type:
  - Rule-based
  - Exact workbook match
  - Historical match
  - Inferred match
  - Manual review
- source email ID
- revision/version
- processed timestamp

---

## 17. Deterministic Decision Flow

Recommended production order:

1. Parse PR Number, subject, header, line items, totals, currency, status/revision
2. Detect explicit RM code(s)
3. If one explicit RM code → RM
4. If multiple explicit item-specific RM codes → split by SAP line totals
5. For unmapped lines in a partially mapped RM PR → intelligently match; Review only unresolved lines
6. If no RM code and subject contains `PR SCM Local ASC` → whole PR to Consumables
7. If no earlier rule matched, detect Project/DIES → Skip
8. Detect CAPEX/Asset → Skip
9. Detect general Service/Fee → Skip
10. Try blank-budget / description-based RM matching
11. Try Consumables reference matching where applicable
12. Only genuinely unresolved/conflicting cases → Review
13. Record every decision in the audit ledger
14. Never double-book the same active PR/version

---

## 18. Known Real PR References

Use these as regression-test examples.

### PR#2000005510
Expected:
- Sander line → RM G-2
- Walkie Talkie line → RM C-11
- line postings reconcile to PKR 76,030

### PR#2000005477
Expected:
- item-specific RM split according to the explicit header assignments

### PR#2000005428
Expected:
- `PR SCM Local ASC`
- no explicit RM code
- whole PR → Consumables

### PR#2000005438 / 5439 / 5440 / 5511
Expected:
- Project/DIES
- no explicit RM code
- Skip — Project/DIES

### PR#2000005421
Expected:
- RM E-1
- useful regression case for revision/unrelease behavior once the client confirms the final rule

### PR#2000005497
Expected:
- explicit G-4 RM instruction
- RM G-4 even if material wording alone could suggest something else

---

## 19. Production Goal

The classifier should be designed for:

**Maximum safe automation + minimum manual Review**

The system should not ask AI to classify every PR from scratch.

Priority should be:

1. deterministic client rules
2. explicit SAP/header instructions
3. exact workbook matches
4. historical mappings
5. intelligent similarity matching
6. Review only when necessary

Manual corrections should later become reusable mappings so the same type of item is not repeatedly reviewed.

---

## 20. Pending Client Answers

Only the following major business rules remain pending:

### A. Revision / cancellation handling
If an already-booked RM or Consumables PR is later Unreleased, Cancelled, Deleted, or revised, should the previous budget posting be automatically reversed/updated?

### B. Foreign-currency handling
How should USD/other foreign-currency PRs be converted/posted against the PKR budget?

Until confirmed, these must remain configurable and must not be guessed.

---

**End of Classifier Rules v1**
