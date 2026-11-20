# ACME Compensation Management Platform — Requirements

**Status:** Draft for review · **Date:** 2026-11-20 · **Author:** Product Engineering
**Scope of this document:** problem framing, requirements, MVP boundary. No implementation.

---

## 1. What the assessment is actually asking for

Read literally, the brief is "build compensation management for ACME." Read as an
engineering assessment, it tests five things, and the requirements below are shaped to
demonstrate them:

| What is being tested | How this document answers it |
|---|---|
| Scoping judgement | An MVP that is one coherent workflow end-to-end, not a feature list (§5, §6) |
| Domain modelling | Effective-dated compensation with immutable history — the hard part of comp (§4.1) |
| Scale thinking | 10,000 employees is the design constraint, not a footnote (§7) |
| Handling sensitive data | Comp data is the most leak-sensitive HR data; access control is a core feature, not a wrapper (§4.6, §7.3) |
| Trade-off articulation | Every exclusion carries a reason (§6); unresolved calls are escalated, not guessed (§10) |

**Core insight driving scope:** compensation is not "a salary field on the employee
record." It is a time-series of approved change events, each with a reason, an approver,
and a currency, that must be reconstructable years later for audit and pay-equity
defence. Everything else is reporting on top of that ledger.

---

## 2. Users and their jobs

| Persona | Volume at ACME | Primary job to be done |
|---|---|---|
| **Compensation Manager** (primary) | 1–5 | Run the annual review cycle; keep bands current; answer "are we paid fairly and within budget?" |
| **HR Business Partner** | ~20–50 | Make and justify off-cycle changes for their population |
| **People Manager** | ~1,000 | Propose increases for a direct team within a given budget |
| **Finance** | ~5 | See committed and projected payroll cost by org, country, currency |
| **Employee** | 10,000 | (Out of MVP — see §6) view own compensation |

The primary persona is the Compensation Manager handling 10,000 employees across
countries. Every design decision resolves in their favour.

---

## 3. Problem statement

ACME runs compensation in spreadsheets emailed between HR, managers, and Finance. This
produces: no reliable salary history, no audit trail for why someone was raised, budget
overruns discovered after the fact, inconsistent currency handling across countries, and
compensation data sitting in uncontrolled files. The platform must become the single
source of truth for what every employee is paid, why, and who approved it.

---

## 4. Functional requirements

Priority: **P0** = MVP, **P1** = next, **P2** = later.

### 4.1 Compensation record and history (P0)
- **FR-1** Every employee has exactly one *current* compensation record derived from an
  append-only series of compensation events.
- **FR-2** A compensation event carries: effective date, amount, currency, pay frequency,
  change reason (merit / promotion / market adjustment / demotion / correction /
  new hire), FTE, actor, approver, timestamp, optional note.
- **FR-3** Events are immutable once approved. A mistake is fixed by a *correction* event,
  never by editing history.
- **FR-4** Future-dated events are supported and become effective automatically; the
  system can answer "what was X paid on any given date."
- **FR-5** Full compensation timeline is viewable per employee.

### 4.2 Compensation structure (P0)
- **FR-6** Job grades/levels with salary bands (min / mid / max) defined per grade **per
  country-currency**, with validity dates.
- **FR-7** Compa-ratio and band penetration computed for every employee.
- **FR-8** Out-of-band compensation is permitted but flagged and requires a justification
  note — HR reality, not a hard block.

### 4.3 Compensation review cycle (P0)
This is the workflow that makes the product worth buying.
- **FR-9** Compensation Manager creates a cycle: name, effective date, eligible
  population (filter-based), budget per org unit, submission deadline.
- **FR-10** Managers see their eligible reports with current comp, compa-ratio, band, and
  last increase, and enter a proposed new amount or percentage.
- **FR-11** Live budget consumption against allocation, blocking submission over budget
  unless explicitly overridden with a reason.
- **FR-12** Approval chain: manager submits → HR/Comp reviews → approve, reject with
  comment, or adjust. Configurable to one or two approval steps; nothing deeper in MVP.
- **FR-13** Cycle finalisation converts every approved proposal into a dated compensation
  event in one atomic, re-runnable operation.
- **FR-14** Cycle status dashboard: submitted vs outstanding by manager, budget used,
  total cost impact.

### 4.4 Bulk operations (P0 — required by 10k scale)
- **FR-15** CSV/XLSX import of employees and of compensation events, with dry-run
  validation producing a per-row error report before anything is written.
- **FR-16** Imports are idempotent and all-or-nothing per batch; a partially bad file
  writes nothing.
- **FR-17** Export of any filtered employee or cycle view, respecting the requester's
  permissions.

### 4.5 Search, segmentation, reporting (P0)
- **FR-18** Server-side search/filter/sort across 10k employees by org unit, country,
  grade, manager, currency, compa-ratio range, band status, last-change date.
- **FR-19** Saved segments reusable as cycle eligibility rules.
- **FR-20** Fixed reports: budget utilisation, band distribution, cost of cycle by
  country/currency, compensation change log, out-of-band exceptions.
- **FR-21** Pay-equity view: median/mean compensation by grade sliced by one chosen
  attribute (e.g. gender), with gap percentage. Reporting only — no legal determination.

### 4.6 Access control and audit (P0)
- **FR-22** Roles: Comp Admin, HR Business Partner (scoped to population), Manager
  (scoped to own reporting line), Finance (aggregates only, no individual amounts),
  Auditor (read-only everything).
- **FR-23** Scope enforced server-side on every query. A manager cannot read a peer's
  compensation by any route, including export and API.
- **FR-24** Immutable audit log of every read of individual compensation and every write,
  with actor, timestamp, before/after. Queryable and exportable.

### 4.7 Multi-currency (P0)
- **FR-25** Compensation is stored in its local currency; the amount is never converted in
  place.
- **FR-26** A reporting currency is selectable; conversion uses an FX rate **snapshotted
  on the cycle or report** so numbers do not change retroactively. Rates are entered or
  imported manually in MVP — no live FX feed.

### 4.8 Deferred functional requirements
- **FR-27 (P1)** Employee self-service compensation statement.
- **FR-28 (P1)** Variable pay: bonus targets, actual bonus, pro-ration.
- **FR-29 (P1)** Notifications/reminders for cycle deadlines.
- **FR-30 (P2)** Equity/long-term incentive grants and vesting.
- **FR-31 (P2)** Market benchmark ingestion and compa-ratio against market.

---

## 5. MVP scope (what we build first)

**One sentence:** a Compensation Manager can load 10,000 employees with history, maintain
grades and bands per country, run a budgeted annual review cycle through manager proposal
and HR approval, finalise it into dated compensation records, and report on cost, bands,
and equity — with every action access-controlled and audited.

MVP = FR-1 … FR-26. Nothing else.

**MVP is "done" when, with 10,000 seeded employees:**
1. A full cycle runs end-to-end: create → propose → approve → finalise, with budget
   enforced throughout.
2. An employee's compensation on any past date is reconstructable from history alone.
3. A manager's session cannot reach compensation outside their reporting line, via UI,
   export, or direct API call.
4. A 10,000-row import validates and reports errors without writing partial data.
5. Employee list queries stay responsive under the §7 targets at 10k scale.

---

## 6. Deliberate exclusions

| Excluded | Why |
|---|---|
| **Payroll processing, payslips, tax, statutory filings** | A different regulated product per country; ACME has a payroll system. We are the source of *what to pay*, and integrate by export. Building this would consume the entire budget on compliance, not product. |
| **Benefits administration, leave, time & attendance** | Adjacent HR domains with their own data models. No bearing on the compensation decision workflow. |
| **Performance management** | Ratings are an *input* to merit decisions. We accept an imported rating field; we do not run review forms, calibration, or goals. |
| **Recruiting / offer management** | New-hire compensation enters as a compensation event. ATS integration is a connector, not a feature. |
| **Equity / stock administration** | Genuinely valuable, but vesting, exercise, and tax treatment is a second product-sized domain. P2. |
| **Employee self-service portal** | 10,000 read-only users triples the auth, access-control, and support surface while adding nothing to the primary persona's workflow. P1, once the ledger is trustworthy. |
| **Configurable workflow / rule engine / custom report builder** | Classic enterprise over-build. Two fixed approval steps and fixed reports cover the stated need; generalise only after seeing real variation. |
| **AI/ML increase recommendations** | Not credible without pay history and market data we do not yet hold, and automated pay recommendations carry discrimination risk. Needs the ledger first. |
| **Native mobile apps** | Managers approve on desktop. A responsive web UI is sufficient. |
| **Multi-tenancy / white-label SaaS** | ACME is one tenant. Multi-tenancy is an architectural tax paid for a customer that does not exist. |
| **Live FX feed, real-time integrations, event-streaming architecture** | Compensation changes are low-frequency (monthly cycles, occasional off-cycle). Batch and request/response are correct here; streaming is complexity without a driver. |
| **SSO/SAML and SCIM provisioning** | Required for real enterprise rollout, excluded from MVP build: ACME's IdP is not available during the assessment. Auth is abstracted behind a provider interface so SSO is a swap, not a rewrite. Flagged in §10. |

---

## 7. Non-functional requirements

### 7.1 Scale and performance (10,000 employees, ~1,000 managers)
- **NFR-1** Employee list and search: p95 < 500 ms, server-side pagination, no unbounded
  queries. No endpoint ever returns all 10,000 records to a browser.
- **NFR-2** Compensation history read for one employee: p95 < 200 ms.
- **NFR-3** 10,000-row import: validated and committed in < 2 minutes, processed as a
  background job with progress and a downloadable error report.
- **NFR-4** Cycle finalisation of 10,000 proposals: single transactional/batched
  operation, resumable, idempotent on retry.
- **NFR-5** Peak concurrency: ~300 managers during the final week of a cycle.
- **NFR-6** Reports over the full population render in < 3 s, from pre-aggregated data
  where needed.

### 7.2 Correctness and integrity
- **NFR-7** Monetary values stored as exact decimals with currency — never floats.
- **NFR-8** Optimistic locking on compensation records and cycle proposals; concurrent
  edits surface a conflict rather than silently overwriting.
- **NFR-9** Append-only compensation history enforced at the data layer, not only in
  application code.
- **NFR-10** Every state-changing endpoint is idempotent under client retry.

### 7.3 Security and privacy
- **NFR-11** Compensation amounts encrypted at rest; TLS in transit.
- **NFR-12** Deny-by-default authorisation evaluated server-side per request; scope is
  part of the query, not a UI filter.
- **NFR-13** Reads of individual compensation are logged (see FR-24) — HR must be able to
  prove who looked.
- **NFR-14** No compensation amounts in application logs, error traces, or analytics.
- **NFR-15** Non-production environments use anonymised/synthetic compensation data.
- **NFR-16** GDPR-shaped obligations: export and erasure request support for an
  individual's personal data, with legally-required compensation records retained.

### 7.4 Operability
- **NFR-17** Daily automated backup with a tested restore path; RPO 24 h, RTO 4 h.
- **NFR-18** Availability target 99.5% business hours across ACME regions. Compensation
  is not a 24/7 transactional system; cycle deadlines are the real availability risk.
- **NFR-19** Structured logs, request tracing, and error reporting from day one.
- **NFR-20** Schema migrations are versioned, reversible, and run without downtime.

### 7.5 Global and usability
- **NFR-21** Locale-correct currency, number, and date formatting; UTC storage with
  per-user display timezone.
- **NFR-22** UI copy externalised for translation; English-only strings shipped in MVP.
- **NFR-23** WCAG 2.1 AA for the manager and HR workflows.
- **NFR-24** A manager completes a proposal for a 10-person team in under 5 minutes
  without training.
- **NFR-25** Data residency is a known constraint for EU employee data and is addressed in
  §10, not silently ignored.

---

## 8. Domain model sketch

Entities only — no schema, no technology choice.

```
Organisation ──< OrgUnit ──< Employee
                                │
                                ├──< EmploymentRecord (grade, FTE, manager, country, dates)
                                └──< CompensationEvent  (append-only, effective-dated)
                                            ▲
Grade ──< SalaryBand (country, currency, min/mid/max, validity)
                                            │
ReviewCycle ──< CycleEligibility ──< Proposal ──┘ (finalisation emits events)
            └──< BudgetAllocation (per OrgUnit)

User ──< RoleAssignment (role + scope: org unit / reporting line / global)
AuditEntry (actor, action, entity, before, after, timestamp)   -- append-only
FxRate (from, to, rate, as-of)                                  -- snapshotted
```

The two invariants worth defending in review:
1. `CompensationEvent` is append-only and effective-dated; current compensation is a
   projection, never a source of truth.
2. Access scope is an attribute of the *query*, so there is no code path that can read
   outside it.

---

## 9. Assumptions

These are stated, not verified. Each is cheap to correct now and expensive later.

1. **A-1** Employee master data (identity, org, manager, country, grade) is owned by an
   external HRIS and imported. We own compensation only.
2. **A-2** Performance ratings are imported as a value; we do not compute them.
3. **A-3** One annual review cycle plus ad-hoc off-cycle changes. No overlapping
   concurrent cycles in MVP.
4. **A-4** Approval depth is at most two steps (manager → HR/Comp). Finance consumes
   results; it does not gate them.
5. **A-5** Base salary is the only MVP compensation component. Bonus and equity are
   modelled as separate future components, not shoehorned into the base amount.
6. **A-6** Approximately 20 countries and 15 currencies; band structures differ per
   country.
7. **A-7** Comp is stored monthly or annually per country convention; pay frequency is an
   attribute, not a conversion.
8. **A-8** Finance sees aggregates only. If Finance needs individual amounts, §4.6 roles
   change.
9. **A-9** Reporting FX rates are supplied by Finance monthly; no live feed.
10. **A-10** Browser-based desktop usage; modern evergreen browsers only.
11. **A-11** Single deployment region to start, with EU residency treated as a risk (§10).

---

## 10. Unresolved product decisions

Each needs an owner's answer. My recommendation is given, but these are not mine to
settle.

1. **The assessment brief itself.** No assessment document was found in the repository;
   this draft is derived from the verbal brief. If a written brief exists with explicit
   requirements, I will reconcile before any code is written. — *Blocking for final scope.*
2. **EU data residency.** Does ACME require EU employee data to remain in the EU? This is
   an architecture-level decision (single region vs regional partitioning) and cannot be
   retrofitted cheaply. *Recommend: confirm before build starts.*
3. **Merit matrix automation.** Should the system *suggest* increase percentages from
   rating × compa-ratio, or only record manager input? *Recommend: record-only in MVP;
   suggestion logic is a fast follow once real cycle data exists.*
4. **Budget enforcement hardness.** Is an over-budget submission blocked outright, or
   permitted with justification and HR escalation? *Recommend: permitted with reason —
   hard blocks drive work back into spreadsheets.*
5. **Visibility of bands to managers.** Do managers see the band min/mid/max for their
   reports, or only a compa-ratio indicator? Pay-transparency posture, not a technical
   question. *Recommend: compa-ratio indicator in MVP; full band visibility
   configurable later.*
6. **Pay-equity attributes.** Which attributes may be used for equity slicing, and who
   may see those reports? Jurisdiction-dependent and legally sensitive. *Recommend:
   gender only in MVP, restricted to Comp Admin.*
7. **Payroll hand-off contract.** Is a scheduled export file sufficient, or does ACME
   require an API integration with the payroll vendor? Affects whether FR-17 is the whole
   answer. *Recommend: export file in MVP.*
8. **Off-cycle approval path.** Does an off-cycle change require the same two-step
   approval as a cycle change, or can an HRBP apply it directly? *Recommend: same
   approval path — identical audit guarantees.*
9. **Historical data depth.** How many years of existing compensation history must be
   migrated? Drives import effort and storage. *Recommend: 3 years.*
10. **SSO timing.** Is SSO required for the assessment deliverable, or acceptable as a
    documented integration point? *Recommend: integration point, per §6.*

---

## 11. What I need to proceed

1. Confirmation or correction of §5 (MVP scope) and §6 (exclusions).
2. Answers to items 1, 2, and 4 in §10 — the rest can be defaulted to my recommendation.
3. Any technology constraints ACME imposes (stack, hosting, existing HRIS vendor) that
   should shape the design before implementation begins.

No application code will be written until this document is reviewed.
