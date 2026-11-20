# ACME Employee Salary Management — Requirements

**Status:** v3 — final for approval · 2026-11-20 · No code until approved.
**User:** HR Manager — one role, organization-wide visibility, 10,000-employee workforce.
**Product in one sentence:** an HR Manager signs in, finds any employee among 10,000,
records and updates salaries in the employee's own currency, and sees a correct,
auditable history and analytics of what the organisation pays.

## 1. Domain rules (binding — every requirement below obeys these)

| # | Rule |
|---|---|
| D-1 | **Deterministic resolution.** An employee's salary on date *D* is the event with the greatest `effective_date ≤ D`; ties are broken by later `created_at`, then by higher event id. Exactly one event resolves for any date, and the same inputs always resolve the same way. |
| D-2 | **Append-only.** Salary events are never updated or deleted. No write path can silently overwrite an existing event; enforced at the data layer, not only in application code. |
| D-3 | **Corrections preserve the original.** A correction is a *new* event that references the event it corrects (`corrects_event_id`) and carries a mandatory reason. The original remains stored and visible, marked superseded; both appear in history with the correction relationship shown. |
| D-4 | **As-of queries.** Salary history answers "what was this employee paid on date *D*" for any past or present date under D-1. Future-dated events are stored and excluded from "current" until their effective date arrives. |
| D-5 | **Exact money.** Amounts are exact decimals (fixed scale, e.g. `NUMERIC(14,2)`) paired with an ISO-4217 currency code. No floating-point type in storage, arithmetic, or transport; decimal values are serialized as **strings** in API responses. |
| D-6 | **No cross-currency arithmetic.** Every total, average, median and bucket is grouped by currency. Amounts in different currencies are never summed, averaged together, or converted. |
| D-7 | **No destructive deletion.** Deactivation is a status change. An employee with salary history is never physically deleted; their salary and audit records are retained and remain queryable after deactivation. |

## 2. Functional requirements (MVP)

| # | Requirement |
|---|---|
| FR-1 | HR authentication: sign in/out, server-side session handling, hashed passwords. Every data request requires an authenticated HR session. |
| FR-2 | Employee management: create, view, update, deactivate/reactivate. Fields include name, employee ID, email, department, country, job title, hire date, status. Deactivation per D-7. |
| FR-3 | Employee list with free-text search (name / ID / email), filters (department, country, status, currency), multi-column sort, and **server-side pagination** — no endpoint returns the full 10,000. |
| FR-4 | Create a salary event: amount, currency, effective date, change reason, optional note. Backdated effective dates are permitted and require a reason; the audit log records both entry time and effective date. |
| FR-5 | Update salary = append a new effective-dated event (D-2). Fixing a mistake means issuing a correction (D-3), never editing. |
| FR-6 | Salary history per employee, newest first: amount, currency, effective date, reason, actor, and superseded/correction markers. Current salary is derived, not stored (D-1). |
| FR-7 | Multi-currency: each salary lives in its own currency and is always displayed with it; all aggregation is per-currency (D-6). |
| FR-8 | Dashboard: total and active employee count, total payroll per currency, average salary per currency (KPI tiles show average), recent salary changes, and headcount by department. Active payroll totals exclude deactivated employees, whose count is shown separately. |
| FR-9 | Salary distribution: range buckets computed **independently per currency** from that currency's own distribution. Bucket boundaries are never shared or compared across currencies. |
| FR-10 | Comparisons: **average and median** salary by department and by country, each per currency, sortable, with employee counts. |
| FR-11 | Audit log of every salary create and correction: actor, entry timestamp, effective date, employee, before/after amount and currency, reason. Append-only, filterable by employee and date range, exportable. |
| FR-12 | CSV export of the current filtered employee list and of an employee's full salary history, honouring active filters and D-5 formatting. |
| FR-13 | Deterministic seed script: 10,000 employees with salary history across multiple departments, countries and currencies — fixed seed, byte-identical output on every run. |

## 3. Non-functional requirements

| # | Requirement |
|---|---|
| NFR-1 | Employee list, search and dashboard queries: p95 < 500 ms at 10,000 employees. Indexed, paginated, aggregates computed in the database — never in application memory. |
| NFR-2 | Authorization and validation enforced **server-side** on every request; no endpoint trusts the client. Client-side validation is convenience only. |
| NFR-3 | Salary history and audit log immutability enforced by database constraints/permissions (D-2), so a future code path cannot violate it. |
| NFR-4 | Seed script completes in under 2 minutes and is re-runnable to the same state. |
| NFR-5 | Locale-correct currency and date formatting; an amount is never rendered without its currency. |
| NFR-6 | No salary amounts in logs or error traces. Secrets in environment config, never in the repo. |
| NFR-7 | Responsive desktop-first web UI; keyboard-usable forms and tables. |

## 4. Explicitly out of scope — and why

| Excluded | Why |
|---|---|
| Review cycles, manager proposals, approval workflows | A second product — budgets, delegation, multi-step state machines, ~1,000 extra users. It needs a trustworthy salary ledger underneath it; that ledger is this MVP. |
| Salary bands, grades, compa-ratio | Requires a job architecture ACME has not defined. Without real bands, compa-ratio is computed from an invented denominator. |
| Pay-equity reporting | Legally sensitive and jurisdiction-specific, and depends on demographic attributes not in scope. Wrong output is worse than none. |
| Background/bulk import pipelines | Job queues, progress tracking, partial-failure semantics. The seed script covers the 10,000-employee demonstration; CSV export covers the real hand-off. |
| Currency conversion / FX rates | Needs an authoritative rate source and as-of snapshotting; a converted total silently depends on which rate was used when. Per-currency grouping is correct and honest (D-6). |
| Payroll processing, tax, payslips | Regulated per country. We are the record of *what is paid*; payroll consumes it via export. |
| Benefits, leave, performance, recruiting | Adjacent HR domains with their own data models. No bearing on recording a salary. |
| Employee self-service | 10,000 read-only accounts triple the auth and support surface and add nothing for the HR Manager. |
| Bonus and equity components | Later, as separate components. Folding them into base salary would corrupt the history. |
| Configurable workflows / custom report builder | Generalising before any real variation is observed is the classic enterprise over-build. |
| SSO/SAML, multi-tenancy, mobile apps | One organisation, one role, desktop use. SSO sits behind the auth interface as a later swap. |

## 5. Confirmed decisions

1. **Visibility** — all HR Managers have organization-wide salary visibility; no per-department scoping.
2. **Backdating** — permitted, with a mandatory reason, audited with both entry and effective dates (FR-4).
3. **Statistics** — average *and* median in department/country comparisons; average on dashboard KPI tiles (FR-8, FR-10).
4. **Deactivated employees** — excluded from active payroll totals, counted separately, and fully accessible historically (D-7, FR-8).

## 6. Assumptions

1. Employee and salary data originates in this system; no external HRIS integration.
2. Base salary is the only compensation component; amounts are annual.
3. Salary events are entered in the employee's existing currency.
4. Browser-based desktop use on modern browsers.

**No open decisions remain. Awaiting approval to begin implementation.**
