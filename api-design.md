# API Design — ACME Employee Salary Management

**Status:** Draft v3 for review · 2026-11-22 · Contract design only, no implementation.
**Base path:** `/api` · **Format:** JSON (`application/json; charset=utf-8`), UTF-8
Companion: [architecture.md](architecture.md) · [database-design.md](database-design.md)

---

## 1. Conventions

### 1.1 Response envelope

Collections and single resources share one shape, so clients have one parser:

```json
{ "data": { }, "meta": { } }
```

`meta` is present on collections (pagination, sort, applied filters) and on analytics
responses (`asOf`, `generatedAt`, rounding notes). Errors never carry `data` — see §1.7.

### 1.2 Monetary serialization (requirements D-5)

| Type | Wire format | Example |
|---|---|---|
| Money amount | **string**, rounded to the currency's `minor_units` | `"85000.00"` (USD, 2) · `"9000000"` (JPY, 0) · `"12345.678"` (KWD, 3) |
| Money value object | always amount **and** currency together | `{ "amount": "85000.00", "currency": "USD" }` |
| Derived average | **string** with `minor_units + 2` decimals, flagged in `meta` | `"100093.7542"` (USD) |
| Identifier (BigInt) | string | `"4021"` |
| Effective date | `YYYY-MM-DD`, UTC calendar (§5 `asOf`) | `"2026-01-01"` |
| Recorded timestamp | ISO-8601 UTC with `Z` | `"2026-02-03T09:14:22Z"` |
| Absent salary | `null`, never `0` or `"0.00"` | `"currentSalary": null` |

Storage is `NUMERIC(20,4)`, but **display precision is the currency's**: amounts are
serialized at exactly `minor_units` decimal places. This is lossless, because a stored
amount can never carry more decimals than its currency permits (database-design.md §3.5),
so `85000.0000` → `"85000.00"` discards nothing.

**Averages are the single documented exception.** An average is a division and is not
exactly representable, so it is rounded to `minor_units + 2` and labelled a derived
statistic in `meta.rounding`. Sums, medians and individual amounts are exact at
`minor_units`. **The API always returns the full `minor_units + 2` value; the UI decides how
to round it for display** — KPI tiles round to `minor_units`, comparison tables may show
more. The contract carries the precision; presentation owns the rounding (ADR-004).

A JSON number is never used for money in either direction. Request bodies must send
amounts as strings; a numeric `amount` is rejected with `400 VALIDATION_FAILED` rather than
silently coerced, because `0.1 + 0.2` arriving from a client is unrecoverable. An amount
with more decimals than its currency allows is rejected with `400 INVALID_DECIMAL_SCALE`
and a field-level message naming the permitted scale. Zod applies the **strict** rule
(`"9000000.00"` is rejected for JPY, which takes no decimals) so the client error is
unambiguous; the database applies the significant-precision rule via `min_scale()`, so it
rejects only values whose arithmetic would change (ADR-020). Malformed input
(`"85,000.00"`, `"8.5e4"`, a negative) is `400 INVALID_DECIMAL_FORMAT`; a value past the
column's range is `400 AMOUNT_OUT_OF_RANGE`.

### 1.3 Pagination

Offset pagination on **every** collection, including salary history and upcoming events
(ADR-005).

| Param | Default | Max | Notes |
|---|---|---|---|
| `page` | 1 | — | 1-indexed |
| `pageSize` | 25 | 100 | values above the max are rejected, not clamped silently |

```json
"meta": {
  "page": 2, "pageSize": 25, "totalItems": 10000, "totalPages": 400,
  "sort": "lastName:asc", "filters": { "status": "ACTIVE", "departmentId": "12" },
  "asOf": "2026-11-22"
}
```

Every sort is made **total** by appending `id` as the final key. `totalItems` comes from a
windowed `count(*) OVER ()` in the same query, not a second round trip.

**What total ordering does and does not guarantee** *(corrected in v2)*. A total order
means a single query returns rows in one well-defined sequence, so a page is never
internally ambiguous. It does **not** make offset pages stable across a changing dataset:
each request is a separate snapshot, so if rows are inserted, deactivated, or re-sorted
between two page requests, a row can be seen twice or missed entirely as it shifts across a
page boundary. Keyset pagination would narrow this to insertions only; neither technique
eliminates it, because the underlying data genuinely changed.

This is accepted rather than engineered around: write volume is low (salary changes are
monthly-to-annual per employee), the reader is a single HR Manager, and no decision in the
product depends on two pages being a consistent snapshot of one instant. Mitigations: the
`meta.asOf` date is displayed on every list, and CSV export — the one place where
completeness matters — streams a single query rather than walking pages (§8).

### 1.4 Sorting

`?sort=<field>:<asc|desc>[,<field>:<asc|desc>]` — at most two keys. Fields are checked
against a per-resource **allow-list** and mapped to column names server-side; an unknown
field is a `400`, never interpolated into SQL.

Employee sort fields: `lastName`, `firstName`, `employeeCode`, `department`, `country`,
`hireDate`, `status`, `currentSalary`.

**`sort=currentSalary` requires a `currency` filter** *(new in v2)*. Amounts in different
currencies are not comparable (requirements D-6), so sorting the whole population by salary
is a meaningless operation rather than a slow one. Without `currency`:

```json
400 Bad Request
{ "error": { "code": "CURRENCY_REQUIRED_FOR_SORT",
             "message": "Sorting by currentSalary requires a currency filter",
             "details": [{ "path": "currency",
               "message": "Add ?currency=USD — salaries in different currencies cannot be ranked together" }],
             "requestId": "01JD3K8QW2ZC4T0M" } }
```

Salary history is sorted `effectiveDate:desc, recordedAt:desc, id:desc` and is not
client-sortable: it is a timeline, and the resolution rule's ordering *is* its meaning.

### 1.5 Filtering

Filters are explicit named parameters, never a generic query language.

| Param | Type | Semantics |
|---|---|---|
| `q` | string | substring match on name, email, employee code (trigram index) |
| `departmentId` | id | exact |
| `countryCode` | ISO-2 | exact |
| `currency` | ISO-3 | matches the employee's **current** currency, derived from their current salary event (§1.6). Employees with no current salary are excluded when this filter is used |
| `status` | `ACTIVE\|INACTIVE\|ALL` | defaults to `ACTIVE` |
| `hiredFrom` / `hiredTo` | date | inclusive range |

### 1.6 Currency is a property of the salary, not the employee *(new in v2)*

There is no employee-level currency field anywhere in this API — not in `POST /api/employees`,
not in `PATCH`, not in the employee representation. An employee's currency is reported inside
`currentSalary`, and is `null` when they have no current salary event (ADR-011).

| Operation | How currency is handled |
|---|---|
| First salary for an employee | the event's `currency` sets it, freely |
| Prospective change (relocation) | `POST .../salary-events` with the new currency, `changeReason: "CURRENCY_CHANGE"`, and `effectiveDate >= today (UTC)` |
| Backdated change | rejected, `422 BACKDATED_CURRENCY_CHANGE` — money actually paid in one currency cannot retroactively become another |
| Mis-entered currency | `POST /api/salary-events/:id/corrections` with the right currency and a mandatory `correctionReason` |
| Historical events | keep their original currency forever; they are immutable |

### 1.7 Errors

Single shape for every failure (architecture.md §7):

```json
{ "error": { "code": "EVENT_ALREADY_SUPERSEDED",
             "message": "This salary event has already been corrected",
             "details": [{ "path": "eventId", "message": "Superseded by event 205" }],
             "requestId": "01JD3K8QW2ZC4T0M" } }
```

| Status | Codes |
|---|---|
| 400 | `VALIDATION_FAILED`, `INVALID_DECIMAL_SCALE`, `INVALID_DECIMAL_FORMAT`, `AMOUNT_OUT_OF_RANGE`, `CURRENCY_REQUIRED_FOR_SORT` |
| 401 | `AUTH_REQUIRED`, `INVALID_CREDENTIALS` |
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 409 | `STALE_VERSION`, `EVENT_ALREADY_SUPERSEDED`, `IMMUTABLE_RECORD` |
| 422 | `EMPLOYEE_INACTIVE`, `BACKDATED_CURRENCY_CHANGE`, `CURRENCY_CHANGE_REASON_REQUIRED`, `CURRENCY_INACTIVE`, `RULE_VIOLATION` |
| 429 | `RATE_LIMITED` |
| 500 | `INTERNAL` |

### 1.8 Cross-cutting

- Auth: `httpOnly` session cookie on every request; mutations also require
  `X-Requested-With: acme-salary-ui` (CSRF, architecture.md §6).
- `X-Request-Id` is echoed on every response and appears in logs and `audit_log`.
- No endpoint returns more than `pageSize` rows. CSV export streams instead (§8).
- `Idempotency-Key` on mutations is **deferred to P1** (open decision).

---

## 2. Authentication

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/auth/login` | exchange credentials for a session cookie |
| `POST` | `/api/auth/logout` | revoke the current session |
| `GET` | `/api/auth/me` | current user; the SPA's auth probe |

```http
POST /api/auth/login
{ "email": "hr.manager@acme.com", "password": "..." }
```

```http
200 OK
Set-Cookie: acme_session=<opaque>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=28800
```
```json
{ "data": { "id": "1", "email": "hr.manager@acme.com",
            "fullName": "Dana Whitfield", "role": "HR_MANAGER" } }
```

Failure is always `401 INVALID_CREDENTIALS` with the same message and timing for an unknown
email as for a wrong password (no user enumeration). **Both outcomes are audited**, including
a failed attempt on an address that belongs to no user: the audit row has a `null` actor and
records the attempted address in `actorEmailAttempted` (database-design.md §3.6). The
submitted password is never stored or logged.

---

## 3. Employees

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/employees` | list — search, filter, sort, paginate |
| `POST` | `/api/employees` | create |
| `GET` | `/api/employees/:id` | detail, including current salary and currency |
| `PATCH` | `/api/employees/:id` | update mutable attributes (optimistic concurrency) |
| `POST` | `/api/employees/:id/deactivate` | status → INACTIVE, reason required |
| `POST` | `/api/employees/:id/reactivate` | status → ACTIVE |

**There is no `DELETE` endpoint** *(removed in v2)*. Employee records are never physically
deleted — not by this API, not by any service method, and not by the application database
role, which holds no `DELETE` privilege (database-design.md §8, §9). An employee created in
error is deactivated with a reason such as "created in error". `DELETE /api/employees/:id`
returns `405 Method Not Allowed`.

```http
GET /api/employees?q=khan&departmentId=12&status=ACTIVE&sort=lastName:asc&page=1&pageSize=25
```
```json
{
  "data": [
    { "id": "4021", "employeeCode": "ACME-004021",
      "firstName": "Ayesha", "lastName": "Khan",
      "email": "ayesha.khan@acme.com", "jobTitle": "Senior Analyst",
      "department": { "id": "12", "name": "Finance" },
      "countryCode": "AE", "hireDate": "2021-04-12", "status": "ACTIVE",
      "currentSalary": { "amount": "240000.00", "currency": "AED",
                         "effectiveDate": "2026-07-01", "salaryEventId": "88021" },
      "version": 3 },
    { "id": "7740", "employeeCode": "ACME-007740",
      "firstName": "Noah", "lastName": "Khanna",
      "email": "noah.khanna@acme.com", "jobTitle": "Analyst",
      "department": { "id": "12", "name": "Finance" },
      "countryCode": "US", "hireDate": "2026-11-18", "status": "ACTIVE",
      "currentSalary": null,
      "version": 1 }
  ],
  "meta": { "page": 1, "pageSize": 25, "totalItems": 2, "totalPages": 1,
            "sort": "lastName:asc", "asOf": "2026-11-22" }
}
```

`currentSalary` is resolved per employee by the `employee_current_salary` view in the same
query (database-design.md §5.3) — not N+1 lookups — and carries the **currency**, which is
the only place an employee's currency appears. It is `null` for an employee with no
resolvable event, as shown in the second row.

```http
POST /api/employees
{ "employeeCode": "ACME-010001", "firstName": "Ayesha", "lastName": "Khan",
  "email": "ayesha.khan@acme.com", "jobTitle": "Senior Analyst",
  "departmentId": "12", "countryCode": "AE", "hireDate": "2021-04-12",
  "initialSalary": { "amount": "200000.00", "currency": "AED",
                     "effectiveDate": "2021-04-12", "changeReason": "HIRE" } }
```

There is no employee-level currency field in this body. `initialSalary` is optional; when
present, the employee row and its first salary event are created in **one transaction** with
both audit rows — an employee is never half-created. When absent, the employee exists with
no currency and no salary until their first event.

```http
PATCH /api/employees/4021
{ "version": 3, "jobTitle": "Lead Analyst", "departmentId": "12" }
```

`version` is mandatory; a mismatch returns `409 STALE_VERSION` with the current
representation so the client can re-render rather than guess. Neither salary nor currency is
accepted here — compensation changes only through §4, so no path changes pay without
creating an event.

---

## 4. Salary events, history, and historical queries

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/employees/:id/salary-events` | **paginated** history, newest first, with correction links |
| `POST` | `/api/employees/:id/salary-events` | append a new salary event |
| `POST` | `/api/salary-events/:eventId/corrections` | append a correcting event that supersedes `:eventId` |
| `GET` | `/api/employees/:id/salary?asOf=YYYY-MM-DD` | resolved salary as of a date |
| `GET` | `/api/salary-events/upcoming` | **paginated** future-dated events across the organization |

### 4.1 Creating an event

```http
POST /api/employees/4021/salary-events
{ "amount": "255000.00", "currency": "AED", "effectiveDate": "2027-01-01",
  "changeReason": "MERIT", "note": "2026 cycle" }
```
```json
201 Created
{ "data": { "id": "88142", "employeeId": "4021",
            "amount": "255000.00", "currency": "AED",
            "effectiveDate": "2027-01-01", "recordedAt": "2026-11-22T10:02:41Z",
            "changeReason": "MERIT", "note": "2026 cycle",
            "recordedBy": { "id": "1", "fullName": "Dana Whitfield" },
            "isSuperseded": false, "supersedesEventId": null,
            "status": "SCHEDULED" } }
```

`status` is derived for the client's benefit: `CURRENT` (resolves today), `HISTORICAL`
(effective, superseded by a later effective date), `SCHEDULED` (`effectiveDate >` today UTC),
`SUPERSEDED` (corrected). It is computed, never stored.

**Rejections**

| Condition | Response |
|---|---|
| `currency` differs from the prior currency at that point in the timeline **and** `effectiveDate` is in the past | `422 BACKDATED_CURRENCY_CHANGE` |
| currency change without `changeReason: "CURRENCY_CHANGE"` | `422 CURRENCY_CHANGE_REASON_REQUIRED` |
| employee is `INACTIVE` | `422 EMPLOYEE_INACTIVE` |
| `effectiveDate` before `hireDate` | `422 RULE_VIOLATION` |
| more decimals than the currency permits | `400 INVALID_DECIMAL_SCALE` |
| `amount` ≤ 0, or sent as a JSON number | `400 VALIDATION_FAILED` |

Backdating an amount (as opposed to a currency) is permitted and requires a `note`;
`recordedAt` is always server-assigned, so a backdated entry can never disguise when it was
made. A **duplicate effective date** is accepted: the response carries
`meta.warnings: ["DUPLICATE_EFFECTIVE_DATE"]` naming the event that will be superseded in
precedence, and the UI surfaces it at entry time (database-design.md §6.2).

A prospective currency change looks like this:

```http
POST /api/employees/4021/salary-events
{ "amount": "72000.00", "currency": "EUR", "effectiveDate": "2027-02-01",
  "changeReason": "CURRENCY_CHANGE", "note": "Relocation to Dublin office" }
```

### 4.2 Correcting an event

```http
POST /api/salary-events/88000/corrections
{ "amount": "86000.00", "currency": "EUR", "effectiveDate": "2026-01-01",
  "correctionReason": "Entered in USD; the employee was paid in EUR" }
```
```json
201 Created
{ "data": { "id": "88143", "supersedesEventId": "88000",
            "changeReason": "CORRECTION",
            "correctionReason": "Entered in USD; the employee was paid in EUR",
            "amount": "86000.00", "currency": "EUR",
            "effectiveDate": "2026-01-01", "recordedAt": "2026-11-22T10:05:12Z",
            "status": "CURRENT" },
  "meta": { "superseded": { "id": "88000", "amount": "86000.00", "currency": "USD",
                            "effectiveDate": "2026-01-01",
                            "recordedAt": "2026-01-02T08:30:00Z" },
            "warnings": ["CURRENCY_CHANGED_BY_CORRECTION"] } }
```

A correction may change the **amount, the effective date, and the currency**;
`correctionReason` is mandatory, so a currency correction always carries its justification.
It may not change the employee. A correction of an already-corrected event returns
`409 EVENT_ALREADY_SUPERSEDED` — enforced by a unique index, not a race-prone application
check (database-design.md §4.2). Correcting a correction is allowed and extends the chain;
the original event is returned in `meta` so the UI can show what changed without a second
request.

There is deliberately no `PUT`, `PATCH` or `DELETE` on a salary event anywhere in this API.

### 4.3 History (paginated)

```http
GET /api/employees/4021/salary-events?page=1&pageSize=25
```
```json
{ "data": [
    { "id": "88143", "amount": "86000.00", "currency": "EUR",
      "effectiveDate": "2026-01-01", "recordedAt": "2026-11-22T10:05:12Z",
      "changeReason": "CORRECTION", "status": "CURRENT",
      "supersedesEventId": "88000",
      "correctionReason": "Entered in USD; the employee was paid in EUR",
      "isSuperseded": false,
      "recordedBy": { "id": "1", "fullName": "Dana Whitfield" } },
    { "id": "88000", "amount": "86000.00", "currency": "USD",
      "effectiveDate": "2026-01-01", "recordedAt": "2026-01-02T08:30:00Z",
      "changeReason": "MERIT", "status": "SUPERSEDED",
      "supersedesEventId": null, "isSuperseded": true,
      "supersededBy": { "id": "88143", "recordedAt": "2026-11-22T10:05:12Z" },
      "recordedBy": { "id": "7", "fullName": "Ravi Menon" } }
  ],
  "meta": { "employeeId": "4021", "page": 1, "pageSize": 25,
            "totalItems": 2, "totalPages": 1,
            "sort": "effectiveDate:desc,recordedAt:desc,id:desc",
            "currentCurrency": "EUR", "currenciesInHistory": ["EUR", "USD"],
            "asOf": "2026-11-22" } }
```

Paginated in v2: an employee with a long career plus corrections can accumulate dozens of
events, and "it is only one employee" is not a size guarantee. Ordering is the resolution
order (`effectiveDate desc, recordedAt desc, id desc`), which is total, so pages do not
overlap within a request.

`meta` reports `currentCurrency` and the full set of `currenciesInHistory` — because
currency now lives on the event, a history can legitimately contain more than one, and the
client must not assume a single currency for the employee. Superseded events are returned,
clearly flagged, and linked in both directions; the client renders them struck through and
never aggregates them.

### 4.4 Historical ("as of") query

```http
GET /api/employees/4021/salary?asOf=2026-03-15
```
```json
{ "data": { "amount": "86000.00", "currency": "EUR",
            "effectiveDate": "2026-01-01",
            "salaryEventId": "88143", "changeReason": "CORRECTION" },
  "meta": { "asOf": "2026-03-15",
            "resolution": "greatest effective_date <= asOf among non-superseded events",
            "dateBoundary": "UTC" } }
```

Backed by `salary_as_of()` (database-design.md §5.2). `asOf` defaults to today (UTC); a date
before the employee's first event returns `"data": null` with `200`, because "not paid yet"
is a fact, not an error.

The `knownAt` axis — "what did the system report on date T" — exists in the resolution
function and is used by tests and support investigations, but is **not a client-settable
parameter and is not reported in responses** (confirmed, ADR-018).

### 4.5 Upcoming events (paginated)

```http
GET /api/salary-events/upcoming?page=1&pageSize=25&sort=effectiveDate:asc
```
```json
{ "data": [
    { "id": "88142", "employee": { "id": "4021", "employeeCode": "ACME-004021",
                                   "name": "Ayesha Khan" },
      "amount": "255000.00", "currency": "AED", "effectiveDate": "2027-01-01",
      "changeReason": "MERIT", "recordedAt": "2026-11-22T10:02:41Z",
      "status": "SCHEDULED" } ],
  "meta": { "page": 1, "pageSize": 25, "totalItems": 134, "totalPages": 6,
            "sort": "effectiveDate:asc,id:asc", "asOf": "2026-11-22",
            "note": "Events with effectiveDate > today (UTC)." } }
```

Organization-wide and therefore unbounded in principle — paginated in v2 for the same
reason as the employee list. Default sort `effectiveDate:asc`, totalized by `id`.

---

## 5. Dashboard and analytics

Every response is **grouped by currency** and no field anywhere sums across currencies
(requirements D-6). There is no `totalPayroll` scalar in this API — only
`payrollByCurrency`.

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/analytics/dashboard` | headline KPIs |
| `GET` | `/api/analytics/distribution?currency=USD` | salary buckets for one currency |
| `GET` | `/api/analytics/by-department` | average + median per department per currency |
| `GET` | `/api/analytics/by-country` | average + median per country per currency |
| `GET` | `/api/analytics/recent-changes?limit=10` | latest salary events |

```http
GET /api/analytics/dashboard
```
```json
{ "data": {
    "employeeCount": { "total": 10000, "active": 9640, "inactive": 360,
                       "withoutCurrentSalary": { "neverRecorded": 12, "scheduledOnly": 3 } },
    "payrollByCurrency": [
      { "currency": "USD", "minorUnits": 2, "employeeCount": 5120,
        "totalPayroll": "512480000.00", "averageSalary": "100093.7500" },
      { "currency": "EUR", "minorUnits": 2, "employeeCount": 2210,
        "totalPayroll": "174590000.00", "averageSalary": "79000.0000" },
      { "currency": "JPY", "minorUnits": 0, "employeeCount": 1180,
        "totalPayroll": "10620000000", "averageSalary": "9000000.00" },
      { "currency": "KWD", "minorUnits": 3, "employeeCount": 94,
        "totalPayroll": "2256000.000", "averageSalary": "24000.00000" }
    ],
    "headcountByDepartment": [ { "departmentId": "12", "name": "Finance", "count": 840 } ]
  },
  "meta": { "asOf": "2026-11-22", "dateBoundary": "UTC",
            "basis": "ACTIVE employees with a current effective salary",
            "rounding": "totalPayroll exact at minorUnits; averageSalary rounded to minorUnits + 2",
            "generatedAt": "2026-11-22T10:31:02Z" } }
```

Note the per-currency display precision in action: JPY totals have no decimals, KWD has
three, and each `averageSalary` carries two more places than its currency — the only derived
figure that does.

`inactive` and `withoutCurrentSalary` sit next to the totals rather than being hidden.
Employees with no current salary count toward **headcount only** and never toward payroll,
average, median or buckets (confirmed, ADR-017); splitting "never recorded" from
"scheduled only" distinguishes a data-entry gap from a legitimate new hire whose first event
is still in the future. The KPI tiles show average; comparisons show both average and
median (requirements §5.3).

```http
GET /api/analytics/by-department?sort=averageSalary:desc
```
```json
{ "data": [
    { "currency": "USD", "minorUnits": 2,
      "departments": [
        { "departmentId": "12", "name": "Finance", "employeeCount": 420,
          "averageSalary": "112400.0000", "medianSalary": "108000.00" },
        { "departmentId": "7", "name": "Support", "employeeCount": 980,
          "averageSalary": "64250.0000", "medianSalary": "61000.00" } ] },
    { "currency": "EUR", "minorUnits": 2, "departments": [ ] }
  ],
  "meta": { "asOf": "2026-11-22", "medianMethod": "percentile_disc",
            "medianNote": "An actual employee salary; for an even-sized group, the lower of the two middle values.",
            "rounding": "averageSalary rounded to minorUnits + 2; medianSalary exact",
            "note": "Currencies are reported separately and are never combined." } }
```

Nesting departments **inside** currency makes the no-mixing rule structural: a client cannot
render a cross-currency league table without deliberately flattening the response.
`by-country` has the identical shape keyed on `countryCode`.

```http
GET /api/analytics/distribution?currency=USD&buckets=8
```
```json
{ "data": { "currency": "USD", "minorUnits": 2, "bucketCount": 8,
            "lowerBound": "42000.00", "upperBound": "310000.00",
            "buckets": [ { "bucket": 1, "from": "42000.00", "to": "75500.00",
                           "employeeCount": 1840 } ] },
  "meta": { "asOf": "2026-11-22",
            "note": "Bounds are derived from this currency's own distribution." } }
```

`currency` is **required** here. There is no all-currency distribution endpoint, because the
bucket boundaries would be meaningless (requirements FR-9).

---

## 6. Audit log

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/audit-logs` | filterable, paginated audit trail |

Filters: `employeeId`, `actorId`, `action`, `entityType`, `from`, `to`. Default sort
`recordedAt:desc`, totalized by `id`. There is no write endpoint — audit rows are written
only inside the transaction of the change they describe (architecture.md §5).

```json
{ "data": [
    { "id": "99812", "action": "SALARY_CORRECT", "entityType": "SALARY_EVENT",
      "entityId": "88143",
      "employee": { "id": "4021", "employeeCode": "ACME-004021", "name": "Ayesha Khan" },
      "actor": { "id": "1", "fullName": "Dana Whitfield" },
      "before": { "amount": "86000.00", "currency": "USD", "effectiveDate": "2026-01-01" },
      "after":  { "amount": "86000.00", "currency": "EUR", "effectiveDate": "2026-01-01" },
      "reason": "Entered in USD; the employee was paid in EUR",
      "recordedAt": "2026-11-22T10:05:12Z", "requestId": "01JD3K8QW2ZC4T0M" },
    { "id": "99790", "action": "LOGIN_FAILURE", "entityType": "SESSION",
      "entityId": null, "employee": null,
      "actor": null, "actorEmailAttempted": "unknown.person@example.com",
      "before": null, "after": null, "reason": "No such user",
      "recordedAt": "2026-11-22T09:58:03Z", "requestId": "01JD3K7PQ1AB2C9Z" }
  ],
  "meta": { "page": 1, "pageSize": 25, "totalItems": 2, "totalPages": 1 } }
```

`actor` is `null` only for `LOGIN_FAILURE` on an unrecognised address, where
`actorEmailAttempted` carries what was typed (database-design.md §3.6). Amounts in
`before`/`after` are strings like everywhere else, and the rendered log shows `recordedAt`
(when it was entered) separately from `effectiveDate` (when it applies) — the distinction
that makes a backdated change auditable.

---

## 7. CSV export

| Method | Path | Produces |
|---|---|---|
| `GET` | `/api/exports/employees.csv` | the employee list under the **same filters and sort** as `GET /api/employees` |
| `GET` | `/api/exports/employees/:id/salary-history.csv` | one employee's full history including superseded events |

```http
GET /api/exports/employees.csv?status=ACTIVE&departmentId=12&sort=lastName:asc
```
```http
200 OK
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="acme-employees-2026-11-22.csv"
Transfer-Encoding: chunked
```
```csv
employee_code,first_name,last_name,email,job_title,department,country_code,hire_date,status,salary_amount,salary_currency,salary_effective_date
ACME-004021,Ayesha,Khan,ayesha.khan@acme.com,Senior Analyst,Finance,AE,2021-04-12,ACTIVE,240000.00,AED,2026-07-01
ACME-007740,Noah,Khanna,noah.khanna@acme.com,Analyst,Finance,US,2026-11-18,ACTIVE,,,
```

Rules: `pageSize` does not apply — the full filtered set is **streamed from a single
query**, so the file is one consistent snapshot (unlike paging through §1.3) and memory is
constant at 10,000 rows. Amounts are unquoted plain decimals at the currency's precision,
with a separate currency column, never a combined `"AED 240,000"` string, so the file
re-imports cleanly. An employee with no current salary exports with those three fields
empty, as in the third row. The salary-history export includes a `currency` column per row,
since a history may span currencies. Every cell beginning `=`, `+`, `-`, `@`, tab or CR is
prefixed with `'` to defeat CSV formula injection (architecture.md §8).

**Export auditing (v3).** Each export writes **two** append-only `export_log` rows sharing
the response's `request_id`: a `STARTED` row before the first byte is streamed (actor,
export type, filters, `row_count` NULL) and a `COMPLETED` row after the last (same fields
plus the actual `row_count`). Because `export_log` cannot be updated, a count discovered
after streaming cannot be written back into the first row — and an export that aborts
mid-stream must still leave a trace, since a partially delivered bulk export is the
exfiltration-relevant case. A `STARTED` row with no matching `COMPLETED` row is therefore a
reviewable signal, not a gap (ADR-022).

---

## 8. Endpoint summary

| # | Method & path | Auth | Notes |
|---|---|---|---|
| 1 | `POST /api/auth/login` | public | rate limited 5/min/IP; failures audited with null actor |
| 2 | `POST /api/auth/logout` | session | revokes server-side |
| 3 | `GET /api/auth/me` | session | |
| 4 | `GET /api/employees` | HR | search, filter, sort, paginate; `currency` filter matches current currency |
| 5 | `POST /api/employees` | HR | optional `initialSalary`, one transaction; no employee-level currency |
| 6 | `GET /api/employees/:id` | HR | includes `currentSalary` (amount + currency) or `null` |
| 7 | `PATCH /api/employees/:id` | HR | `version` required; no salary or currency fields |
| 8 | `POST /api/employees/:id/deactivate` | HR | reason required |
| 9 | `POST /api/employees/:id/reactivate` | HR | |
| 10 | `GET /api/employees/:id/salary-events` | HR | **paginated**; includes superseded, flagged |
| 11 | `POST /api/employees/:id/salary-events` | HR | append-only; prospective currency change allowed |
| 12 | `POST /api/salary-events/:eventId/corrections` | HR | may correct amount, date, currency; 409 if already superseded |
| 13 | `GET /api/employees/:id/salary?asOf=` | HR | historical resolution; `knownAt` not exposed |
| 14 | `GET /api/salary-events/upcoming` | HR | **paginated** future-dated events |
| 15 | `GET /api/analytics/dashboard` | HR | per-currency only; no-salary employees in headcount only |
| 16 | `GET /api/analytics/distribution` | HR | `currency` required |
| 17 | `GET /api/analytics/by-department` | HR | avg + median, nested under currency |
| 18 | `GET /api/analytics/by-country` | HR | avg + median, nested under currency |
| 19 | `GET /api/analytics/recent-changes` | HR | |
| 20 | `GET /api/audit-logs` | HR | read-only by design |
| 21 | `GET /api/exports/employees.csv` | HR | streamed, single snapshot, audited |
| 22 | `GET /api/exports/employees/:id/salary-history.csv` | HR | streamed, audited |
| 23 | `GET /health`, `GET /health/ready` | public | liveness / DB round-trip |

22 functional endpoints plus health, no `PUT`, and **no destructive verb anywhere** — not on
salary, not on audit records, not on employees.
