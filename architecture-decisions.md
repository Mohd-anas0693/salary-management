# Architecture Decision Records — ACME Employee Salary Management

**Status:** Draft v2 for review · 2026-11-22
Each record states the decision, what was genuinely considered against it, what it costs,
and the condition under which it should be revisited. ADR-001 … ADR-010 were accepted in
v1; **ADR-004, ADR-005 and ADR-006 are revised** and **ADR-011 … ADR-019 are new**.
Companion: [architecture.md](architecture.md) · [database-design.md](database-design.md) ·
[api-design.md](api-design.md)

---

## ADR-001 — PostgreSQL rather than MongoDB

**Context.** The domain is a ledger of monetary events with referential integrity, exact
decimal arithmetic, multi-dimensional aggregation (per currency, per department, per
country), and invariants that must hold for years.

**Decision.** PostgreSQL 16.

**Alternatives considered.**

| Option | Why rejected |
|---|---|
| MongoDB | No exact decimal in the arithmetic path: `Decimal128` exists for storage, but aggregation and the driver's JS types push values toward floats, and the ecosystem's default is `Double`. No foreign keys, so "an employee with salary history cannot be deleted" and "audit rows are never orphaned" become application conventions that a single bad script breaks. Multi-document transactions exist but are a bolt-on requiring a replica set. No triggers or `CHECK` constraints, so append-only and decimal-scale validation could not be enforced below the application. Aggregation pipelines can group and compute percentiles, but window functions, `DISTINCT ON`, `width_bucket` and `percentile_disc` are either absent or awkward — and those four are precisely what this product's analytics are built from |
| MySQL 8 | A defensible second choice: exact `DECIMAL`, triggers, window functions. Rejected for weaker `JSONB` querying, no `DISTINCT ON`, no partial or expression indexes of the same quality, no `CITEXT`, and a weaker extension story (`pg_trgm`, ADR-014) |
| SQLite | Fine for tests, unfit for a shared multi-user system with concurrent writes |

**Consequences.** Exact `NUMERIC` end to end; FK `RESTRICT` plus withheld `DELETE` grants
give database-level protection for history; triggers and grants make append-only real
(ADR-003); the hardest analytics are single queries. Cost: an operational relational
database, and a schema that must be migrated deliberately rather than drifting per document.

**Revisit when.** Never for this domain.

---

## ADR-002 — Modular monolith rather than microservices

**Context.** One user role, ~10,000 employees, low write volume (salary changes are a
monthly-to-annual event per employee), read-heavy analytics, and a small team.

**Decision.** A single Express process with enforced module boundaries
(architecture.md §2), one database, one deployment. No queue, no cache, no second service.

**Alternatives considered.**

| Option | Why rejected |
|---|---|
| Microservices (employees / salaries / analytics / audit) | The product's central guarantee is that a salary change and its audit record commit together. Across services that becomes a distributed transaction or a saga with compensating actions — in an **audit** system, where a compensating action is itself a falsification risk. We would be trading a `BEGIN … COMMIT` for an eventual-consistency protocol to solve a scaling problem we do not have |
| Serverless functions per endpoint | Connection-pool pressure against PostgreSQL, cold starts on an interactive HR UI, and streamed CSV export fits badly in a request-response function budget |
| Single-layer monolith (controllers calling Prisma directly) | Fastest to write, but the domain rules (resolution, correction, currency, decimal scale) would be scattered across route handlers, which is how "superseded events must not count" eventually gets violated in one forgotten query |

**Consequences.** Atomic multi-table writes; one place to run migrations; trivial local
development. Cost: a single process to scale (vertically, plus read replicas if ever
needed) and boundary discipline that must be lint-enforced.

**Revisit when.** A module needs genuinely independent scaling or a separate release
cadence — the likeliest candidate is a future review-cycle module with ~1,000 concurrent
managers. Because boundaries are already interfaces, that extraction is mechanical.

---

## ADR-003 — Append-only salary events rather than a mutable salary field

**Context.** Requirements demand effective-dated history, correction without loss,
reconstruction of salary for any date, and an audit trail that withstands scrutiny.

**Decision.** `salary_event` is append-only. Current salary **and current currency** are
derived, never stored. A mistake is corrected by inserting a new event linked through
`supersedes_event_id`. Enforced by trigger and by withheld `UPDATE`/`DELETE` grants
(database-design.md §8).

**Alternatives considered.**

| Option | Why rejected |
|---|---|
| `employee.salary` column + a separate history table written by application code | The two can disagree, and they eventually do. Whichever one a given query trusts becomes an accident of which developer wrote it. (The v1 `employee.salary_currency` column was a smaller instance of exactly this flaw — see ADR-011) |
| Mutable rows with `valid_from`/`valid_to` maintained on update | Requires updating the previous row's `valid_to` on every change — a write path that rewrites history, so a bug silently alters the past. Also serializes concurrent inserts for one employee |
| Event sourcing with a separate projection store | The correct shape at a far larger scale. Here it adds a projection to rebuild, a consistency lag, and a staleness bug class (ADR-006) for a table of tens of thousands of rows |
| Soft delete (`is_deleted`) on salary rows | A deleted row is still a rewrite of what the record said, and it gives no way to express *why* the number changed |

**Consequences.** Nothing is ever lost; `recorded_at` plus the correction chain makes the
system answer not only "what was she paid" but "what did we believe, and when"
(database-design.md §5.2). Concurrency becomes almost trivial: appends never conflict, and
the one real conflict — two corrections of one event — is caught by a unique index. Costs:
current salary needs a resolution query rather than a column read (one composite index), the
UI must explain superseded rows, and the resolution rule must be unambiguous — which is why
`effective_date`, `recorded_at`, `id` is specified to the last tiebreaker.

---

## ADR-004 — Exact decimal storage at `NUMERIC(20,4)`, currency-scaled strings on the wire

*(revised in v2 — was `NUMERIC(14,2)`)*

**Context.** Monetary amounts across currencies with **0, 2 and 3 minor units** (JPY;
USD/EUR; KWD/BHD/OMR/TND), summed and averaged for payroll totals HR will reconcile against
payroll system output.

**Decision.**

1. Storage is `NUMERIC(20,4)` — one uniform type with headroom past every ISO-4217 scale in
   use, and 16 integer digits for 0-decimal currencies.
2. The *permitted* decimals for a given amount come from `currency.minor_units`, enforced by
   trigger as `amount = round(amount, minor_units)` and by Zod at the API boundary
   (database-design.md §3.5).
3. The wire format is a **string at the currency's own precision** — `"85000.00"` (USD),
   `"9000000"` (JPY), `"12345.678"` (KWD). Lossless, because stored amounts can never carry
   more decimals than their currency permits.
4. **Averages are not cast to a money scale.** They are computed as exact `sum / count` and
   rounded explicitly to `minor_units + 2`, labelled a derived statistic in response `meta`.

**Alternatives considered.**

| Option | Why rejected |
|---|---|
| `NUMERIC(14,2)` (v1) | Cannot represent a 3-decimal currency at all: a KWD salary of `12345.678` silently becomes `12345.68`, which is a wrong number in a payroll system. The fixed scale 2 also made the type, not the currency, the arbiter of precision |
| `double precision` / JS `number` | Binary floating point cannot represent 0.1. Summing 9,640 salaries accumulates error, and a payroll total off by cents destroys trust in the product. The failure is also non-deterministic across summation orders |
| Integer minor units (store as `BIGINT`) | Genuinely good, and a close second. Rejected because every read and write needs a scale conversion keyed on currency, which puts a unit-conversion bug between the database and every display — now worse, with three distinct scales in play |
| Per-currency scale in the column type | Impossible: one column has one type. Hence a uniform storage scale plus a semantic scale in reference data |
| `::numeric(14,2)` cast on averages (v1) | Threw precision away silently **and** mislabelled a statistic as a money amount. For a 0-decimal currency it also implied two decimals that cannot exist |
| JSON numbers on the wire | `JSON.parse` turns the number into an IEEE-754 double at the client boundary, discarding the exactness the database preserved |
| Money as a formatted display string (`"KWD 12,345.678"`) | Unparseable, locale-dependent, breaks CSV re-import, and invites cross-currency string concatenation |

**Consequences.** Exactness from storage to screen for every currency ACME pays in; one
explicit, documented rounding point (averages) instead of several silent ones. Analytics
must avoid float-returning functions, which is why medians use `percentile_disc`
(ADR-016). The frontend never parses an amount into `number`.

---

## ADR-005 — Offset pagination on every collection, with honest guarantees

*(revised in v2 — adds salary history and upcoming events; corrects the stability claim)*

**Context.** 10,000 employees; a searchable sortable table; salary histories that grow with
career length and corrections; an organization-wide list of future-dated events;
NFR-1 p95 < 500 ms.

**Decision.** Offset pagination (`page`, `pageSize` 25/100) on **every** collection,
including `GET /employees/:id/salary-events` and `GET /salary-events/upcoming`. Filters and
sort are evaluated in SQL against an allow-list; `totalItems` comes from a windowed count in
the same query; every sort is totalized with a trailing `id`. Sorting by `currentSalary`
requires a `currency` filter (ADR-011).

**What total ordering guarantees — corrected.** v1 claimed a total order means "page 2 can
never repeat or skip a row that page 1 showed". That is wrong, and the correction matters
because it sets expectations for the UI. A total order makes a *single query's* result
sequence unambiguous. It does **not** make successive offset requests a consistent snapshot:
each request sees its own committed state, so a concurrent insert, deactivation, or sort-key
change can shift a row across a page boundary and cause a duplicate or a miss. Keyset
pagination would narrow the window to insertions; nothing eliminates it, because the data
genuinely changed between requests.

**Alternatives considered.**

| Option | Why rejected |
|---|---|
| Ship all rows and filter in the browser | Megabytes per load, re-downloaded on every reload, and it puts the entire salary table of the company into browser memory and cache — a data-exposure decision disguised as a performance one. For salary history it would also hide unbounded growth |
| Client-side sort of a server-paginated page | Sorts only the visible rows while appearing to sort the dataset. Actively misleading, and invisible in demos |
| Keyset / cursor pagination | Better at large scale and strictly better page-to-page behaviour. Rejected for MVP because user-chosen multi-column sorts make cursor encoding fiddly, jumping to page 40 is a legitimate HR action, and at 10,000 rows `OFFSET` with a matching index is inside budget. The upgrade path is confined to one repository |
| Unpaginated salary history "because it is one employee" | Not a size guarantee. A long career plus correction chains is unbounded in principle, and the endpoint is also used by the export path |
| Unpaginated upcoming events | Organization-wide and therefore unbounded; during a merit cycle it could hold thousands of rows |

**Consequences.** Constant response size, predictable queries, URL-shareable list state.
Page drift under concurrent writes is accepted and mitigated where it matters: lists display
`meta.asOf`, and CSV export streams a **single query** so the exported file is one
consistent snapshot rather than a walk across pages (api-design.md §7).

**Revisit when.** The employee table passes roughly 200,000 rows, or deep offsets appear in
slow-query logs.

---

## ADR-006 — Analytics aggregated in SQL, not in application code

*(revised in v2 — average rounding and the derived-currency cost)*

**Context.** Dashboard KPIs, per-currency payroll and averages, department and country
comparisons with medians, and per-currency distribution buckets — all over current salaries
derived from an event table.

**Decision.** Every aggregate is computed by PostgreSQL (`GROUP BY`, `DISTINCT ON`,
`percentile_disc`, `width_bucket`, `round(sum/count, minor_units + 2)`) through hand-written
SQL in the analytics repository, with results Zod-parsed at the boundary. The API returns
finished numbers (database-design.md §10).

**Alternatives considered.**

| Option | Why rejected |
|---|---|
| Fetch rows into Node and aggregate in JavaScript | Moves ~10,000 rows per dashboard load across the wire, then sums money in a language whose only numeric type is a double unless every value is wrapped in `decimal.js` — slower and a correctness hazard at once. Medians would need a full in-memory sort per group |
| Aggregate in the browser | Same arithmetic problem, plus shipping the whole salary table to the client (ADR-005) |
| A materialized view refreshed on a schedule | Tempting for speed, but a future-dated salary event must become current the instant the calendar rolls over (UTC, ADR-012); a materialized projection is wrong from midnight until refresh. Kept as the **documented escape hatch** if the derived-currency filter (ADR-011) misses NFR-1, with refresh on both writes and a daily date-rollover job |
| A separate analytics store / warehouse | Needs a pipeline, a second copy of compensation data to secure, and a consistency lag, for a dataset that fits comfortably in the operational database |
| Prisma's `groupBy`/`aggregate` API | Used where it fits, but it cannot express `DISTINCT ON`, window functions, `percentile_disc`, `width_bucket`, or the per-currency bucket-bounds CTE. Forcing it would mean several round trips stitched in JavaScript |

**Consequences.** One round trip per panel, exact decimal arithmetic in the engine that owns
the data, and the no-cross-currency rule expressed where it cannot be bypassed — `currency`
is in every `GROUP BY`, and comparison responses nest results **under** currency so a client
cannot flatten them by accident. Cost: raw SQL is not type-checked by Prisma, so each
query's result shape is Zod-validated and covered by an integration test against the seeded
dataset; and because currency is now derived (ADR-011), filtered-by-currency queries resolve
the population before filtering — a claim that must be measured, not assumed
(database-design.md §11).

---

## ADR-007 — Opaque server-side sessions rather than JWT

**Context.** One role, a browser SPA, highly sensitive data, and a hard requirement that an
access revocation takes effect immediately.

**Decision.** Random 32-byte token in an `httpOnly` `Secure` `SameSite=Lax` cookie; only
its SHA-256 hash is stored; validity checked against `hr_session` on every request
(architecture.md §6).

**Alternatives considered.** A stateless JWT in a cookie or in `localStorage`. Rejected
because a JWT is valid until it expires: revoking access needs a denylist, which reinstates
the database lookup the JWT was supposed to avoid — so we pay the complexity and still do
the query. `localStorage` additionally exposes the token to any XSS. There is no
third-party API consumer here to justify stateless tokens, and a session lookup on a local
index is sub-millisecond.

**Consequences.** Instant logout and "log out everywhere"; no token-rotation machinery; the
server knows who is logged in, which is itself an audit asset. Cost: one indexed lookup per
request and a session table to prune — and because the application role holds no `DELETE`,
pruning is an owner-role task (architecture.md §10.2, item 8).

---

## ADR-008 — Prisma for structure, hand-written SQL for invariants

**Context.** Prisma cannot express triggers, `CHECK` constraints, views, functions,
generated columns, extensions, partial indexes, or grants — and several of those carry the
product's core guarantees, including the decimal-scale rule and the prospective-currency
rule introduced in v2.

**Decision.** Prisma Migrate owns tables, columns, relations and plain indexes. A reviewed
SQL block appended to each generated migration owns everything else
(prisma-schema-draft.prisma header). Integration tests attempt `UPDATE salary_event`,
`DELETE FROM audit_log` and `UPDATE export_log` and assert each fails, plus one test per
insert-validation rule — so every invariant is verified rather than assumed.

**Alternatives considered.** Application-only enforcement (one forgotten code path and
history is mutable — unacceptable for an audit system); a pure SQL migration tool with
Prisma as a read-only client (loses the typed client's main benefit for ordinary CRUD);
`prisma db push` (no reviewable migration artifact, unusable for production).

**Consequences.** Full type safety where it helps, real database guarantees where they
matter. Cost: migrations are partly hand-written and must be reviewed as code, and
`prisma migrate dev` regeneration must not silently drop the appended SQL — the test suite
catches that. The schema draft also pins the two Prisma preview features it relies on
(`views`, `postgresqlExtensions`) and documents a fallback for each.

---

## ADR-009 — Supersession derived from a unique pointer, not a stored flag

**Context.** Superseded events must not participate in current-salary calculations, and
salary events must be immutable.

**Decision.** The correcting event carries `supersedes_event_id`, which is `UNIQUE`. "Is
superseded" is derived by a `LEFT JOIN`/`NOT EXISTS` on that column. There is no
`is_superseded` or `superseded_by` column on the corrected row (database-design.md §4.2).

**Alternatives considered.** An `is_superseded` boolean or `superseded_by_event_id` on the
corrected row would require an `UPDATE` to an append-only table — directly violating the
immutability rule, and forcing the trigger to allow one narrow mutation, which is a hole
that later grows. A `salary_event_status` side table avoids the update but adds a second
writable source of truth for the same fact.

**Consequences.** Strictly zero `UPDATE` paths on `salary_event`, so the trigger can be
absolute. The unique index does triple duty: it makes the join provably 1:1, forbids forked
correction chains, and turns concurrent corrections into a clean `409` instead of two
competing truths. Combined with `CHECK (supersedes_event_id < id)`, cycles are structurally
impossible. Cost: one extra index probe per resolution, and "is this superseded" is a join
rather than a column read.

---

## ADR-010 — One resolution function as the single source of salary truth

**Context.** Current salary, historical as-of salary, the employee list's `currentSalary`,
the dashboard, comparisons, and CSV export must all agree — about the amount **and the
currency**. Six independently written queries would eventually disagree.

**Decision.** One documented rule (requirements D-1, generalized in database-design.md
§5.1), implemented once as `salary_as_of(employee_id, as_of, known_at)` for single-employee
reads and once as the `employee_current_salary` view for set-based reads, with identical
`ORDER BY` and exclusion logic. No other query in the system resolves salary; everything
else joins the view. The worked examples in database-design.md §6 are the shared test table
for both implementations.

**Alternatives considered.** Resolving in the service layer (correct, but then analytics SQL
would need its own copy, defeating ADR-006); a single parameterized view (impossible — views
take no parameters; a set-returning function per employee would be an N+1 in list queries).

**Consequences.** Two implementations instead of one, which is a real duplication —
mitigated by a shared test table that asserts both return the same answer for every case in
§6. In exchange, the hardest rule in the product is written down twice and tested, rather
than reimplemented six times and tested nowhere.

---

## ADR-011 — Currency belongs to the salary event, not the employee *(new)*

**Context.** v1 stored `salary_currency` on `employee` *and* `currency` on every
`salary_event`, with a trigger forcing them equal. That is two sources of truth for one
fact, and it made an employee's currency permanent — so a relocation had no legal path, and
a mis-entered currency could only be fixed by mutating the employee row, which is not where
the history lives.

**Decision.** Remove `employee.salary_currency` entirely. Currency lives on
`salary_event` only.

1. Current currency is **derived** from the current salary event, via
   `employee_current_salary` (database-design.md §5.3).
2. Historical events keep the currency they were paid in, permanently — they are immutable.
3. A **prospective** currency change is a new event with the new currency,
   `change_reason = 'CURRENCY_CHANGE'`, and `effective_date >= today (UTC)`.
4. A **backdated** currency change is rejected (`422 BACKDATED_CURRENCY_CHANGE`): money
   actually paid in one currency cannot retroactively become another.
5. A **mis-entered** currency is fixed by a correction, which may change currency and
   carries a mandatory `correction_reason` (database-design.md §4.3).
6. An employee with no salary event has **no currency**, and the API reports
   `currentSalary: null`.

**Alternatives considered.**

| Option | Why rejected |
|---|---|
| Keep both columns, trigger-enforced equal (v1) | Two sources of truth; the trigger is a patch over a modelling error. It also made "what currency were they paid in in 2023" ambiguous between the employee row and the event |
| Keep `employee.salary_currency` as a derived cache, trigger-maintained | Requires updating `employee` on every salary insert — which is a write amplification *and* a staleness window at the UTC date boundary when a scheduled currency change takes effect, exactly the bug class ADR-006 rejects |
| Allow backdated currency changes | Would rewrite what was actually paid, and would make a historical total in one currency change retroactively. Corrections exist for genuine errors and carry a reason; a "change" is not a correction |
| Let any event change currency without a distinguished reason | A currency switch would be indistinguishable from a raise in history, and could happen by typo rather than intent |

**Consequences.** One source of truth; relocation is now expressible; the inconsistency
class is gone. The costs are real and recorded: (a) filtering or sorting employees by
currency resolves current salary for the population first rather than seeking an indexed
column, which must be measured against NFR-1 (database-design.md §11); (b) the index
`employee (status, salary_currency)` is gone; (c) the API must handle a history spanning
multiple currencies (`meta.currenciesInHistory`) and a `null` currency; (d) sorting by
`currentSalary` now requires a `currency` filter, since amounts in different currencies
cannot be ranked together (api-design.md §1.4).

**Revisit when.** Only if the measurement in (a) misses NFR-1 — and the answer then is the
ADR-006 materialized view, not a denormalized column.

---

## ADR-012 — UTC as the system-wide effective-date boundary *(new)*

**Decision.** Every date boundary — the current-salary view, `salary_as_of`'s default, the
prospective-currency rule, dashboard `asOf`, "upcoming" — uses
`(now() AT TIME ZONE 'UTC')::date`, written once in SQL. The UI labels derived figures
`as of <date> UTC`.

**Alternatives considered.** A per-user timezone would mean two HR Managers in different
offices disagree about an employee's current salary for up to a day at every change
boundary, and an export taken in Dubai would not reconcile with one taken in New York. An
organization-level configured timezone was also considered; it is a single setting today but
becomes a migration if ACME ever changes it, and it buys nothing a label does not.

**Consequences.** One definition of "today" in the whole system. Cost: an HR Manager in
UTC+4 entering an event late in their day may see it classed as effective "tomorrow"; the
UI shows the UTC date on the form to make this visible rather than surprising.

---

## ADR-013 — Duplicate effective dates permitted with deterministic precedence *(new)*

**Decision.** Two events for one employee may share an `effective_date`. Precedence is the
later `recorded_at`, then the higher `id`; the loser stays in history as `HISTORICAL`. The
API returns `meta.warnings: ["DUPLICATE_EFFECTIVE_DATE"]` and the UI warns at entry.

**Alternatives considered.** A `UNIQUE (employee_id, effective_date)` constraint would block
legitimate same-day re-entries and corrections that restate a date, and it would convert a
routine HR action into an error the user cannot resolve without understanding the data
model. Rejecting at the service layer has the same problem with extra indirection.

**Consequences.** No input is ever refused for being on a date already used, and the outcome
is still deterministic (D-1). Cost: an HR Manager must be told which of two same-day events
wins — hence the warning being part of the API contract rather than a UI nicety.

---

## ADR-014 — `pg_trgm` for substring search *(new)*

**Decision.** A generated `search_text` column (`first_name ‖ last_name ‖ email ‖
employee_code`) with a GIN trigram index serves the `q` filter.

**Alternatives considered.** `ILIKE '%term%'` without an index (a sequential scan per
keystroke, which at 10,000 rows is survivable but wasteful and degrades as the table grows);
a `tsvector` full-text index (handles prefixes and whole words, but not mid-word substrings —
searching "khan" would miss "Khanna" only in some configurations and, more importantly, an
employee code fragment like "4021" is not a lexeme); client-side filtering (ADR-005).

**Consequences.** Fast substring search on names, emails and codes. Cost: a hosting
dependency — `CREATE EXTENSION pg_trgm` must be permitted, which is listed as an open item
(architecture.md §10.2, item 4). The `tsvector` variant is the documented fallback.

---

## ADR-015 — Salary events are rejected for inactive employees in MVP *(new)*

**Decision.** `POST .../salary-events` on an `INACTIVE` employee returns
`422 EMPLOYEE_INACTIVE`, enforced in the service and again in the insert trigger. The error
names reactivation as the path.

**Alternatives considered.** Permitting events for inactive employees would support
genuine back-pay and final-settlement cases, but it also means an inactive employee can
acquire a new "current" salary while being excluded from payroll totals — a state the
analytics basis does not describe. Allowing it only for backdated events was considered and
rejected as a rule too subtle to explain in a UI error message.

**Consequences.** A clean, explainable rule, and payroll totals that always match the active
population. Cost: a real back-pay case requires reactivate → add event → deactivate, which
is three audited actions instead of one. If HR reports that this happens regularly, the rule
is worth revisiting with their actual cases in hand.

---

## ADR-016 — `percentile_disc` for medians *(new)*

**Decision.** Medians use `percentile_disc(0.5) WITHIN GROUP (ORDER BY amount)`.

**Alternatives considered.** `percentile_cont(0.5)` interpolates between the two middle
values — but it accepts and returns `double precision`, so it would put a floating-point
number into a monetary response and break the exactness rule in ADR-004. Casting its result
back to `numeric` does not undo the float arithmetic, it only hides it. Computing the median
in application code would mean sorting every group in memory (ADR-006).

**Consequences.** Medians stay exact `numeric` and are always a real employee's salary. For
an even-sized group, `percentile_disc` returns the lower of the two middle values rather
than their mean; the API states this in `meta.medianNote` so a reader cannot mistake it for
an interpolated median.

---

## ADR-017 — Employees without salary events count toward headcount only *(new)*

**Decision.** Every analytics query joins `employee_current_salary`, so an employee with no
resolvable event contributes to headcount and to nothing else — no total, average, median or
bucket. The dashboard reports them explicitly, split into `neverRecorded` and
`scheduledOnly` (api-design.md §5).

**Alternatives considered.** Treating a missing salary as zero (which would corrupt every
average and total while looking plausible); excluding such employees from headcount too
(which would make the headcount disagree with the employee list and hide the gap entirely);
reporting one undifferentiated "missing" count (which would conflate a data-entry error with
a new hire whose first event is legitimately in the future).

**Consequences.** Totals describe exactly the population they claim to, and the gap is
visible. Cost: one more number on the dashboard, and the `scheduledOnly` split requires a
second small query.

---

## ADR-018 — `knownAt` stays internal *(new)*

**Decision.** `salary_as_of` takes a `known_at` parameter, defaulted to `now()`. It is used
by tests and support investigations. It is not a client-settable API parameter and is not
reported in responses.

**Alternatives considered.** Exposing `?knownAt=` would make the bitemporal capability
available to HR, but it also makes a confusing distinction ("as of" vs "as known at")
part of the public contract, invites screenshots of two different "correct" salaries for the
same date, and adds a parameter nobody has asked for. Removing the parameter from the
function entirely was also rejected: it costs nothing to keep, and it is the only way to
answer "what did we report in February" when an auditor asks.

**Consequences.** A simpler API with the capability retained where it is needed. Promoting
it later is additive and breaks nothing.

---

## ADR-019 — No deletion path for employees, anywhere *(new)*

**Context.** v1 exposed `DELETE /api/employees/:id` for employees with no salary history, on
the reasoning that nothing would be lost.

**Decision.** Removed. There is no deletion endpoint, no service method that deletes an
employee, and the application database role holds **no `DELETE` privilege on any table**.
An employee created in error is deactivated with a reason such as "created in error".
`DELETE /api/employees/:id` returns `405`.

**Alternatives considered.** Keeping the narrow "no history" case was defensible on data
grounds, but it means the codebase contains a deletion path that must be guarded forever,
and the guard is a runtime condition (`has no salary events`) rather than a structural
impossibility. The FK `RESTRICT` protects history; it does not protect the audit trail of
the record's own creation, which deleting the row would orphan. Keeping `DELETE` for
convenience in an audited HR system is not worth one permanently dangerous code path.

**Consequences.** Deletion is structurally impossible rather than conditionally forbidden,
which also makes the privilege model simpler to state and to test (database-design.md §8).
Cost: a handful of deactivated rows that are genuinely junk will persist. That is the
cheaper mistake.
