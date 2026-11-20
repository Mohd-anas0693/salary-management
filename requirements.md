# ACME Employee Salary Management — Requirements

**Status:** Draft v2 for approval · 2026-11-20 · No code until approved.
**User:** HR Manager — one role, managing a 10,000-employee workforce.
**Product in one sentence:** an HR Manager signs in, finds any employee among 10,000,
records and updates salaries in the employee's own currency, and sees a correct,
auditable history and analytics of what the organisation pays.

**Three invariants everything else serves:** salary history is append-only and
effective-dated (current salary is a projection, never the source of truth); money is an
exact decimal plus a currency code — never a float; amounts in different currencies are
never summed.

## 1. Functional requirements (MVP)

| # | Requirement |
|---|---|
| FR-1 | HR authentication: sign in/out, session handling, password hashed. All data access requires an authenticated HR session. |
| FR-2 | Employee management: create, view, update, deactivate. Fields include name, employee ID, email, department, country, job title, hire date, status. |
| FR-3 | Employee list with free-text search (name / ID / email), filters (department, country, status, currency), multi-column sort, and **server-side pagination** — no endpoint returns the full 10,000. |
| FR-4 | Create a salary record for an employee: amount (exact decimal), currency, effective date, change reason, optional note. |
| FR-5 | Update salary = append a new effective-dated record. Existing records are immutable; a mistake is fixed by a correction record, never an edit. |
| FR-6 | Salary history per employee, newest first, showing amount, currency, effective date, reason, and who made the change. Current salary is derived from the record effective today. |
| FR-7 | Future-dated records are accepted and become current on their effective date. The system can answer "what was this employee paid on date D." |
| FR-8 | Multi-currency: each salary is stored and displayed in its own currency. Totals, averages and comparisons are **always grouped by currency**; no conversion and no cross-currency arithmetic anywhere. |
| FR-9 | Dashboard: total/active employee count, total payroll per currency, average salary per currency, and salary analytics (distribution by band-free range buckets, recent salary changes, headcount by department). |
| FR-10 | Comparisons: average and median salary by **department** and by **country**, each per currency, sortable. |
| FR-11 | Audit log of every salary create/update: actor, timestamp, employee, before/after amount and currency, reason. Append-only, filterable by employee and date, exportable. |
| FR-12 | CSV export of the current filtered employee list and of an employee's salary history, respecting the active filters. |
| FR-13 | Deterministic seed script generating 10,000 employees with salary history across multiple departments, countries and currencies — fixed seed, identical output on every run. |

## 2. Non-functional requirements

| # | Requirement |
|---|---|
| NFR-1 | Employee list, search and dashboard queries: p95 < 500 ms at 10,000 employees. Indexed, paginated, aggregates computed in the database — not in application memory. |
| NFR-2 | Monetary values stored as exact decimals with a currency code. No floating point in storage, API or arithmetic. |
| NFR-3 | Authorization and validation enforced **server-side** on every request. Client-side validation is convenience only; no endpoint trusts the client. |
| NFR-4 | Salary history append-only, enforced at the data layer, not only in application code. |
| NFR-5 | Seed script completes in under 2 minutes and is idempotent/re-runnable. |
| NFR-6 | Locale-correct currency and date formatting; amounts always shown with their currency. |
| NFR-7 | No salary amounts in logs or error traces. Secrets in environment config, never in the repo. |
| NFR-8 | Responsive desktop-first web UI; keyboard-usable forms and tables. |

## 3. Explicitly out of scope — and why

| Excluded | Why |
|---|---|
| Compensation review cycles, manager proposals, approval workflows | A second product: budgets, delegation, multi-step state machines, ~1,000 extra users. Needs a trustworthy salary ledger underneath it first — that ledger is this MVP. |
| Salary bands, grades, compa-ratio | Requires a job architecture ACME has not defined. Without real bands, compa-ratio is a number computed from an invented denominator. |
| Pay-equity reporting | Legally sensitive and jurisdiction-specific; depends on demographic attributes not in scope. Wrong output here is worse than no output. |
| Background/bulk import pipelines | Job queues, progress tracking, partial-failure semantics. The seed script covers the 10,000-employee demonstration; CSV *export* covers the real hand-off need. |
| Currency conversion / FX rates | Conversion needs an authoritative rate source and as-of snapshotting, and a converted total invites decisions on stale rates. Grouping by currency is correct and honest. |
| Payroll processing, tax, payslips | Regulated per country. We are the record of *what is paid*; payroll systems consume it via export. |
| Benefits, leave, performance, recruiting | Adjacent HR domains with their own data models. No bearing on recording a salary. |
| Employee self-service | 10,000 read-only accounts triple the auth and support surface and add nothing for the HR Manager. |
| Bonus and equity components | Modelled later as separate components; folding them into base salary would corrupt the history. |
| Configurable workflow / custom report builder | Generalising before any real variation is observed is the classic enterprise over-build. |
| SSO/SAML, multi-tenancy, mobile apps | One organisation, one role, desktop usage. SSO sits behind the auth interface as a later swap. |

## 4. Assumptions

1. Employee and salary data is created and maintained in this system; no external HRIS integration is in scope.
2. Base salary is the only compensation component; pay frequency (annual) is uniform and displayed as such.
3. HR Managers are trusted equally — no per-department scoping of visibility within the role.
4. Salary records are entered in the employee's existing currency; the currency does not change retroactively.
5. Browser-based desktop use on modern browsers.

## 5. Open decisions

1. **Salary visibility scoping** — do all HR Managers see all salaries, or should visibility be limited by department? Affects the authorization model. *Recommend: all, per Assumption 3.*
2. **Can an effective date be backdated?** *Recommend: yes, with a reason — real HR corrections arrive late — and the audit log records when it was entered versus when it takes effect.*
3. **Average vs median emphasis on the dashboard** — medians resist outliers but are costlier to compute. *Recommend: show both for comparisons, average only on the dashboard tiles.*
4. **Deactivated employees in analytics** — included or excluded from payroll totals. *Recommend: excluded, with the count shown separately.*

**Approval needed on §1 and §3 before implementation begins.**
