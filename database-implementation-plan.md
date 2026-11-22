# Phase 3 — Database Implementation Plan

**Status:** v2, revised against review feedback · 2026-11-22 ·
**No migrations and no application code until final approval.**
**Scope:** database only. No API endpoints, no frontend, no service/controller code. The one
TypeScript artifact proposed here is the seed script (requirements FR-13); see §9.
**Inputs:** [requirements.md](requirements.md) · [architecture.md](architecture.md) ·
[database-design.md](database-design.md) (v3) · [api-design.md](api-design.md) (v3) ·
[architecture-decisions.md](architecture-decisions.md) (ADR-001…022) ·
[prisma-schema-draft.prisma](prisma-schema-draft.prisma) (v3)
**Companion:** [sql-test-matrix.md](sql-test-matrix.md) (195 cases)

---

## 1. Platform pinning

### 1.1 PostgreSQL — one pinned patch version, asserted at runtime

`postgres:16-alpine` is a **floating tag**: it moves with every patch release and is not a
pin. The pin is an exact patch version held in one file and used by every consumer.

```
.postgres-version        ->  16.10          # proposed; replace with the value production reports
```

| Consumer | How it uses the pin |
|---|---|
| CI / Testcontainers | image tag `postgres:16.10-alpine`, read from `.postgres-version` — never a bare `16` or `16-alpine` |
| Local development | the same tag in `docker-compose.yml`, read from the same file |
| Production | the managed instance's version, which **must** report the same `major.minor` |
| Test suite | `ENV-01` asserts `SHOW server_version` starts with the pinned value and **fails the suite** otherwise |
| Release | a mismatch between `.postgres-version` and production's reported version is a release blocker, surfaced by a pre-deploy check |

**Needed from you (§11.1, B1):** the patch version your managed instance reports
(`SHOW server_version`), so the pin and production agree. `16.10` above is a placeholder, and
CI will fail against a different patch until you confirm it.

Minimum workable major is **13** (`min_scale()`); everything else in this design predates it.
We pin 16 because production will run 16 — a dev machine on 15 would pass every test here,
which is exactly the drift the pin prevents.

| Feature | Introduced | Used for |
|---|---|---|
| `min_scale(numeric)`, `trim_scale()` | **13** | pre-coercion precision validation (§2) |
| `scale(numeric)` | 9.6 | strict scale in Zod's mirror rule, tests |
| Generated (stored) columns | 12 | `employee.search_text` |
| `percentile_disc`, `width_bucket(numeric,…)`, `DISTINCT ON` | ≤ 9.5 | analytics (ADR-006, ADR-016) |
| `BEFORE TRUNCATE` statement triggers | ≤ 9.0 | append-only protection (§7) |

### 1.2 Prisma 6.x

| Assumption | Detail |
|---|---|
| Version | `prisma` / `@prisma/client` **6.x**, exact version in `package.json`, no `^` |
| `previewFeatures` | **`["postgresqlExtensions"]` only.** `views` is dropped (approved): both views are read with `$queryRaw` + Zod |
| Why `postgresqlExtensions` stays | `citext` is a *column type*, so `prisma migrate dev` must create it in the **shadow database** too. Declaring extensions in the datasource is what makes that happen; creating them only in M0 breaks `migrate dev` |
| `Decimal` | `numeric` ↔ `decimal.js`; `$queryRaw` returns `numeric` as a **string**, which matches the money-as-string DTO contract directly |
| `BigInt` | ids are `BigInt`, not JSON-serializable → string in every DTO |
| `DateTime @db.Date` | JS `Date` at UTC midnight; local getters shift the day at negative offsets → §3 |
| Relation mode | default `foreignKeys`; the FK `RESTRICT` rules are load-bearing |
| Enums | none, by design (ADR-008) |
| Salary inserts | **not** via `prisma.salaryEvent.create()` — the app role has no `INSERT` (§2) |

---

## 2. Concern A — original monetary precision (corrected)

### 2.1 The defect

PostgreSQL applies the column typmod at coercion, **before** `BEFORE` triggers and `CHECK`
constraints. `NEW.amount` is already scale 4, so `amount = round(amount, minor_units)` only
catches over-precision that survives rounding:

| Input | After coercion | Trigger verdict |
|---|---|---|
| `85000.001` USD | `85000.0010` | rejected — correct |
| `85000.00001` USD | `85000.0000` | **accepted — silent alteration** |
| `9000000.00004` JPY | `9000000.0000` | **accepted** |
| `12345.678004` KWD | `12345.6780` | **accepted** |

`scale()` cannot detect it afterwards: every stored value has scale exactly 4.

### 2.2 The hardened append function

```sql
-- Illustrative design, not the migration.
CREATE FUNCTION public.salary_event_append(
  p_employee_id         bigint,
  p_amount_text         text,            -- the client's exact digits, uncoerced
  p_currency            text,            -- text, not char(3): function signatures carry no typmod
  p_effective_date      date,
  p_change_reason       text,
  p_note                text   DEFAULT NULL,
  p_supersedes_event_id bigint DEFAULT NULL,
  p_correction_reason   text   DEFAULT NULL,
  p_recorded_by         bigint DEFAULT NULL
) RETURNS public.salary_event
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = pg_catalog, pg_temp     -- pg_temp LAST; unlisted, it is searched FIRST
AS $function$
DECLARE
  v_amount    numeric;                    -- unconstrained: preserves the input's true scale
  v_minor     smallint;
  v_status    text;
  v_hire_date date;
  v_row       public.salary_event;
BEGIN
  -- 1. Format, before any cast can mask it.
  IF p_amount_text IS NULL OR p_amount_text !~ '^[0-9]+(\.[0-9]+)?$' THEN
    RAISE EXCEPTION 'amount must be a plain decimal string, received %', p_amount_text
      USING ERRCODE = 'P0001', HINT = 'INVALID_DECIMAL_FORMAT';
  END IF;
  v_amount := p_amount_text::numeric;     -- min_scale('85000.00001') = 5

  -- 2. Serialize appends for THIS employee and read its state under the lock (ADR-021).
  SELECT e.status, e.hire_date INTO v_status, v_hire_date
  FROM   public.employee e
  WHERE  e.id = p_employee_id
  FOR    UPDATE;
  IF NOT FOUND THEN
    RAISE EXCEPTION 'employee % does not exist', p_employee_id
      USING ERRCODE = 'P0001', HINT = 'NOT_FOUND';
  END IF;
  IF v_status <> 'ACTIVE' THEN
    RAISE EXCEPTION 'employee % is not active', p_employee_id
      USING ERRCODE = 'P0001', HINT = 'EMPLOYEE_INACTIVE';
  END IF;

  -- 3. Currency and its permitted precision.
  SELECT c.minor_units INTO v_minor
  FROM   public.currency c
  WHERE  c.code = upper(p_currency) AND c.is_active;
  IF NOT FOUND THEN
    RAISE EXCEPTION 'currency % is unknown or inactive', p_currency
      USING ERRCODE = 'P0001', HINT = 'CURRENCY_INACTIVE';
  END IF;

  IF min_scale(v_amount) > v_minor THEN
    RAISE EXCEPTION '% permits % decimal places; received %',
      upper(p_currency), v_minor, p_amount_text
      USING ERRCODE = 'P0001', HINT = 'INVALID_DECIMAL_SCALE';
  END IF;

  IF v_amount <= 0 OR v_amount >= 10::numeric ^ 16 THEN
    RAISE EXCEPTION 'amount % is out of range', p_amount_text
      USING ERRCODE = 'P0001', HINT = 'AMOUNT_OUT_OF_RANGE';
  END IF;

  -- 4. Actor must be a real, active HR user; recorded_at and id stay server-assigned.
  IF p_recorded_by IS NULL
     OR NOT EXISTS (SELECT 1 FROM public.hr_user u
                     WHERE u.id = p_recorded_by AND u.is_active) THEN
    RAISE EXCEPTION 'recording user % is unknown or inactive', p_recorded_by
      USING ERRCODE = 'P0001', HINT = 'RULE_VIOLATION';
  END IF;

  INSERT INTO public.salary_event (
      employee_id, amount, currency, effective_date, change_reason,
      note, supersedes_event_id, correction_reason, recorded_by_hr_user_id)
  VALUES (p_employee_id, v_amount, upper(p_currency)::char(3), p_effective_date,
          p_change_reason, p_note, p_supersedes_event_id, p_correction_reason,
          p_recorded_by)
  RETURNING * INTO v_row;    -- the BEFORE INSERT trigger still runs: cross-row rules

  RETURN v_row;
END
$function$;

ALTER FUNCTION public.salary_event_append(bigint, text, text, date, text, text,
                                          bigint, text, bigint) OWNER TO acme_owner;
```

### 2.3 Why `acme_app` cannot bypass it

| Bypass route | Closed by |
|---|---|
| `INSERT INTO salary_event` directly | `REVOKE ALL`, then `GRANT SELECT` only — no `INSERT` (§8). Test P-01 |
| Advancing the id sequence to forge a row | no `USAGE ON SEQUENCE salary_event_id_seq` for `acme_app`; the sequence is advanced inside the definer context |
| Defining its own `SECURITY DEFINER` function | `acme_app` has `USAGE` but **not `CREATE`** on schema `public`, and is not the owner of anything. Test P-10 |
| Calling the function as an unrelated role | `REVOKE EXECUTE … FROM PUBLIC` with the exact signature, then `GRANT` to `acme_app` only. Test P-03 |
| Shadowing `public.salary_event` or `public.currency` with a temp table | `SET search_path = pg_catalog, pg_temp` with `pg_temp` **last**, plus full schema qualification of every object |
| Inheriting owner rights more broadly | `acme_owner` is **not a superuser**; definer rights are exactly the table owner's. Test P-12 |
| A later overload of the function with laxer checks | every grant names the full signature, so a new overload is not covered by the existing grant |
| `UPDATE` / `DELETE` / `TRUNCATE` on the ledger | triggers plus absent privileges (§7, §8). Tests I-01…I-10 |

### 2.4 Alternatives considered

| Option | Why not |
|---|---|
| Bare `numeric` column, or `CREATE DOMAIN money_amount AS numeric CHECK (scale(VALUE) <= 4)` | Strongest — protects `acme_owner` too. Rejected as primary because it changes the confirmed `NUMERIC(20,4)` decision and Prisma can express neither (bare `Decimal` emits `decimal(65,30)`), so every later `migrate diff` reports drift. Still available on your word |
| Zod only | Layer 3 already; cannot protect the seed, a future import, or a hurried `$executeRaw` |
| `CHECK (amount = round(amount, 4))` | Always true after coercion; enforces nothing |
| `trim_scale()` on write | Normalizes silently; the client is never told its input changed |

**Trade-off, stated plainly:** salary appends leave the typed Prisma client (one
`$queryRaw` per append; reads unaffected), and error mapping reads the exception `HINT`.
In exchange there is exactly one validated write path, enforced by privilege.

---

## 3. Concern B — every date comparison explicitly UTC

```sql
CREATE FUNCTION public.utc_today() RETURNS date
LANGUAGE sql STABLE PARALLEL SAFE AS $$ SELECT (now() AT TIME ZONE 'UTC')::date $$;
```

| Site | Status |
|---|---|
| `employee_current_salary` predicate | `public.utc_today()` (database-design.md §5.3, corrected) |
| `salary_as_of(p_as_of DEFAULT …)` | `public.utc_today()` (corrected) |
| Prospective-currency trigger rule | `public.utc_today()` — **was `CURRENT_DATE`**, the one session-dependent expression in v2 |
| Upcoming-events query, derived `status`, dashboard `asOf` | `public.utc_today()`; the API reports the database's date, never Node's |

Controls: `ALTER DATABASE … SET timezone='UTC'` and `ALTER ROLE acme_app SET timezone='UTC'`
as defense in depth, never relied upon; a **CI grep** over `prisma/migrations/**/*.sql` and
`src/**/*.sql` failing on `CURRENT_DATE`, `current_date`, `now()::date`,
`CURRENT_TIMESTAMP::date`, `localtimestamp` and any `AT TIME ZONE` with a non-UTC zone, with
one allow-listed definition site; Node carries dates as `'YYYY-MM-DD'` strings only; and the
suite runs under three session timezones (series **U**).

`utc_today()` is `STABLE`, so it cannot appear in a `CHECK` or an index predicate — which is
why the prospective rule is a trigger and the upcoming index is plain `(effective_date, id)`.

---

## 4. Migration order

| # | Migration | Owner | Contents | Why here |
|---|---|---|---|---|
| **M0** | `000_bootstrap` | DBA, manual | `CREATE ROLE acme_owner` (not superuser), `CREATE ROLE acme_app`; database; `ALTER DATABASE … SET timezone='UTC'`; `REVOKE CREATE ON SCHEMA public FROM PUBLIC`; `CREATE EXTENSION citext, pg_trgm` | Roles and extensions precede Prisma; `citext` is a column type used in M1 |
| **M1** | `001_reference_and_auth` | Prisma | `currency`, `department`, `hr_user`, `hr_session` + indexes | No FK dependencies; `currency` precedes `salary_event` |
| **M2** | `002_employee` | Prisma | `employee` + indexes | Needed by `salary_event` and `audit_log` |
| **M3** | `003_salary_event` | Prisma | `salary_event` + indexes, self-FK, `UNIQUE (supersedes_event_id)` | After `employee`, `currency` |
| **M4** | `004_audit_and_export` | Prisma | `audit_log` (nullable actor), `export_log` (**`status`, nullable `row_count`, `request_id`** — ADR-022) + indexes | After `hr_user`, `employee` |
| **M5** | `005_constraints` | SQL | all `CHECK`s incl. the `export_log` pair and the `audit_log` actor pair; `employee.search_text` generated column; GIN trigram index; partial `hr_session` index | Prisma cannot express these; must precede data |
| **M6** | `006_functions_and_views` | SQL | `utc_today()`, `salary_as_of()`, `employee_current_salary`, `salary_event_resolved` | Views depend on tables |
| **M7** | `007_triggers_and_append_fn` | SQL | `reject_mutation()` + 3 row triggers; **`reject_truncate()` + 3 statement triggers**; `salary_event_validate()` + `BEFORE INSERT` trigger; `salary_event_append()` | Trigger functions call `utc_today()` from M6 |
| **M8** | `008_privileges` | SQL | the full matrix (§8), signatures exact | **Last**, so nothing created earlier escapes it |
| **M9** | seed | script | reference data, HR user, 10,000 employees (§9) | After every constraint and trigger exists |

M1–M4 are Prisma-reversible; M5–M8 ship explicit `DOWN` SQL; M0 is manual and documented.
CI runs `prisma migrate diff --exit-code` against a freshly migrated database to assert zero
drift, which also catches an M5–M8 statement lost to a `migrate dev` regeneration.

---

## 5. Tables and relationships

Authoritative in [database-design.md §2–§3.3](database-design.md). Execution-order summary:

| Table | Rows at seed | Key FKs (all `RESTRICT` unless noted) |
|---|---|---|
| `currency` | ~6 | — |
| `department` | ~12 | — |
| `hr_user` | 1–3 | — |
| `hr_session` | 0 | → `hr_user` (**CASCADE**, the only cascade) |
| `employee` | 10,000 | → `department` |
| `salary_event` | ~40,000 | → `employee`, `currency`, `hr_user`, `salary_event` (self, `UNIQUE`) |
| `audit_log` | ~40,000 | → `hr_user` (**nullable**), `employee` (nullable) |
| `export_log` | 0 | → `hr_user`; two rows per export (ADR-022) |

---

## 6. Constraints and indexes

**M5 checklist:** `salary_event` — `amount > 0`, upper-case currency,
`supersedes_event_id < id`, correction-reason pairing, `change_reason` list including
`CURRENCY_CHANGE`; `currency.minor_units IN (0,2,3)`; `employee` status list, status/
`deactivated_at` pairing, `country_code` shape; `audit_log` actor pair;
**`export_log`** `status` list, `export_type` list, and
`((status = 'STARTED') = (row_count IS NULL))`.

**Index build order:** plain indexes with their tables (M1–M4); the GIN trigram index after
the generated column (M5); `export_log (request_id)` in M4. **Indexes are built before the
seed** (approved) — correctness and realistic plans over a few seconds of seed time.

**Measurement, not assertion.** The ADR-011 consequence — filtering or sorting employees by
currency resolves current salary for the population first — is verified by **latency** at
seed scale, with plans captured for human review (§12, series **PERF**). A specific scan type
is not asserted: on a 10,000-row table the planner may legitimately prefer a sequential scan
or a bitmap heap scan depending on cache state, statistics and `work_mem`, and a test that
demands one shape fails for the wrong reason.

---

## 7. Functions, views and immutability objects

| Object | Kind | Notes |
|---|---|---|
| `public.utc_today()` | `STABLE` SQL | single definition of "today" (§3) |
| `public.salary_as_of(bigint, date, timestamptz)` | `STABLE` SQL | both temporal axes; `known_at` internal (ADR-018) |
| `employee_current_salary` | view | `DISTINCT ON`; carries amount **and** currency; read via `$queryRaw` + Zod |
| `salary_event_resolved` | view | derived supersession via the 1:1 `LEFT JOIN` |
| `public.reject_mutation()` | trigger fn | `BEFORE UPDATE OR DELETE … FOR EACH ROW` on all three append-only tables |
| `public.reject_truncate()` | trigger fn | **`BEFORE TRUNCATE … FOR EACH STATEMENT`** on all three — row triggers do not fire on `TRUNCATE`, so without this one statement erases the ledger |
| `public.salary_event_validate()` | trigger fn | `BEFORE INSERT` on `salary_event`: 6 cross-row rules, rule 6 now using `utc_today()` |
| `public.salary_event_append(bigint, text, text, date, text, text, bigint, text, bigint)` | `SECURITY DEFINER` | the app's only write path; pre-coercion validation + per-employee lock (§2) |

Division of labour: the **append function** validates what exists only before coercion and
takes the lock; the **insert trigger** validates cross-row state and remains the backstop for
`acme_owner` writes (migrations, seed, DBA) that bypass the function entirely.

---

## 8. Permissions — exact signatures throughout

```sql
-- Schema.
GRANT  USAGE  ON SCHEMA public TO acme_app;          -- USAGE only, never CREATE
REVOKE CREATE ON SCHEMA public FROM PUBLIC, acme_app;

-- Append-only tables: read all three, append to two.
REVOKE ALL    ON salary_event, audit_log, export_log FROM acme_app;
GRANT  SELECT ON salary_event, audit_log, export_log TO   acme_app;
GRANT  INSERT ON audit_log, export_log               TO   acme_app;   -- NOT salary_event
GRANT  USAGE  ON SEQUENCE audit_log_id_seq, export_log_id_seq TO acme_app;
-- Deliberately no USAGE on salary_event_id_seq.

-- The only salary append path.
REVOKE EXECUTE ON FUNCTION
  public.salary_event_append(bigint, text, text, date, text, text, bigint, text, bigint)
  FROM PUBLIC;
GRANT  EXECUTE ON FUNCTION
  public.salary_event_append(bigint, text, text, date, text, text, bigint, text, bigint)
  TO acme_app;

-- Read helpers.
REVOKE EXECUTE ON FUNCTION public.utc_today() FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION public.salary_as_of(bigint, date, timestamptz) FROM PUBLIC;
GRANT  EXECUTE ON FUNCTION public.utc_today() TO acme_app;
GRANT  EXECUTE ON FUNCTION public.salary_as_of(bigint, date, timestamptz) TO acme_app;

-- Trigger functions are never called directly.
REVOKE EXECUTE ON FUNCTION public.reject_mutation()      FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION public.reject_truncate()      FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION public.salary_event_validate() FROM PUBLIC;

-- Mutable tables.
GRANT SELECT, INSERT, UPDATE ON employee, hr_user, hr_session, department TO acme_app;
GRANT USAGE ON SEQUENCE employee_id_seq, hr_user_id_seq, hr_session_id_seq,
                        department_id_seq TO acme_app;

-- Read-only.
GRANT SELECT ON currency, employee_current_salary, salary_event_resolved TO acme_app;

-- Future objects are not world-executable.
ALTER DEFAULT PRIVILEGES FOR ROLE acme_owner IN SCHEMA public
  REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;

-- Nothing, anywhere, grants DELETE or TRUNCATE to acme_app (ADR-019).
```

Every function grant names its full signature. An unqualified `GRANT EXECUTE ON FUNCTION
name` silently extends to any later overload — for the append function that would mean
extending it to a variant with weaker validation. `REVOKE … FROM PUBLIC` matters because
`SECURITY DEFINER` functions are world-executable by default, which would hand any role the
owner's insert rights.

Test series **P** verifies the matrix with `has_table_privilege` / `has_function_privilege`
rather than by inspecting the migration text.

---

## 9. Seed data strategy (precision claim corrected)

| Dimension | Plan |
|---|---|
| Determinism | seeded PRNG (`mulberry32`, seed `20261122`), never `Math.random()`; **explicit `recorded_at` on every event**, because `DEFAULT now()` is not reproducible |
| Date anchor | `SEED_ANCHOR`, default the fixed date `2026-01-01`; `SEED_ANCHOR=today` is the documented demo mode (approved) |
| Currencies | 6 across all three scales: JPY (0), USD/EUR/AED/GBP (2), KWD (3) |
| Employees | 10,000: 9,640 `ACTIVE`, 360 `INACTIVE`; 12 with no salary event; 3 with only a future-dated event |
| Salary events | ~40,000: one `HIRE` each plus 0–5 later; ~3% correction chains (incl. depth 2 and 3); ~1% duplicate effective dates; ~2% future-dated; ~1.5% prospective `CURRENCY_CHANGE` |
| Append-function coverage | ~50 events inserted through `salary_event_append()` (approved), so the guarded path is exercised by the seed as well as by tests |
| HR users | one `HR_MANAGER`, Argon2id hash of `SEED_HR_PASSWORD`; refuses to run against a non-empty database or with `NODE_ENV=production` unless `--force` |
| Performance | staged + set-based, one transaction, run as `acme_owner`; target well under NFR-5's 2 minutes |

### 9.1 Amount strings are validated **before** insertion

The correction: `min_scale()` on a `numeric(20,4)` **column** cannot recover precision
already lost to coercion. A post-seed assertion over stored values proves only that stored
values conform to their currency — it **cannot** detect that an input string had extra digits
that rounded away. Validation must therefore happen on the strings, before they reach the
column:

```sql
-- 1. Stage amounts as TEXT (owner-side, inside the seed transaction).
CREATE TEMP TABLE seed_salary_stage (
  employee_id bigint, amount_text text, currency char(3), effective_date date,
  recorded_at timestamptz, change_reason text, note text,
  supersedes_event_id bigint, correction_reason text, recorded_by bigint
);
-- … bulk-load generated rows …

-- 2. Validate the ORIGINAL strings pre-coercion. Must return zero rows.
SELECT s.employee_id, s.amount_text, s.currency
FROM   seed_salary_stage s
JOIN   public.currency c ON c.code = s.currency
WHERE  s.amount_text !~ '^[0-9]+(\.[0-9]+)?$'
   OR  min_scale(s.amount_text::numeric) > c.minor_units      -- text::numeric is unconstrained
   OR  s.amount_text::numeric <= 0
   OR  s.amount_text::numeric >= 10::numeric ^ 16;

-- 3. Only then insert; the cast is now lossless by construction.
INSERT INTO public.salary_event (…) SELECT …, s.amount_text::numeric, … FROM seed_salary_stage s;
```

`s.amount_text::numeric` in step 2 is an **unconstrained** numeric, so `min_scale()` sees the
true scale — the same guarantee the append function gets, applied set-based. The generator
also emits amounts with exactly `minor_units` decimals, so step 2 is a guard against
generator bugs, not a routine filter.

### 9.2 Post-seed assertions — and what they do *not* prove

| Assertion | Proves |
|---|---|
| 10,000 employees; 9,640 active | shape |
| every stored amount satisfies `min_scale(amount) <= minor_units` | stored values **conform to their currency**. It does **not** prove each original input was free of excess precision — coercion is irreversible, so that can only be established by §9.1 before insert |
| no superseded event resolves as current, for any employee | ADR-009 |
| `salary_as_of(e, utc_today())` equals `employee_current_salary` for all 10,000 | ADR-010 |
| exactly 12 employees with no events; exactly 3 scheduled-only | ADR-017 fixtures present |
| ≥ 1 depth-3 correction chain, ≥ 1 duplicate effective date, ≥ 1 future `CURRENCY_CHANGE` | the interesting cases exist |
| no `effective_date < hire_date` | trigger rule held |
| ~50 events have been created through the append function | guarded path exercised |
| two seeds into fresh databases produce an identical `md5` over an ordered dump | FR-13 |

The seed must never `ALTER TABLE … DISABLE TRIGGER`; CI greps for it. Row-level insert
triggers fire on the set-based insert, so every cross-row rule is enforced during seeding.

### 9.3 Historical currency changes remain unsupported (approved)

ADR-011 forbids backdated currency changes and the trigger enforces it, so no seeded employee
has a *historical* currency switch; multi-currency histories appear only as future-dated
changes or corrections. The consequence to keep in view: if ACME's real history contains
employees genuinely paid in different currencies over time, that history cannot be imported
without a privileged path — a Phase-4+ decision, deliberately not built now.

---

## 10. Contradiction register

| # | Contradiction | Status |
|---|---|---|
| 1 | `CURRENT_DATE` in the prospective-currency rule vs the UTC mandate | **Resolved** — `utc_today()`; database-design.md §4.4 and §5.4 updated; CI guard |
| 2 | The `round()` check described as enforcing per-currency precision | **Resolved** — database-design.md §3.5 rewritten; ADR-004 point 5; ADR-020 |
| 3 | §8 granting `acme_app` `INSERT` on `salary_event` | **Resolved** — revoked (approved); database-design.md §8 and §8 above agree |
| 4 | Average precision stated as settled in api-design.md, open in architecture.md | **Resolved** — API returns `minor_units + 2`, UI owns display rounding; ADR-004, api-design.md §1.2, architecture.md §10.1 |
| 5 | Byte-identical seed vs future-dated events | **Resolved** — fixed anchor by default, `SEED_ANCHOR=today` demo mode (approved) |
| 6 | Append-only `export_log` vs a row count known only after streaming | **Resolved** — `STARTED`/`COMPLETED` pair (ADR-022); schema shape needs your nod (§11.1 B3) |
| 7 | `audit_log.action = 'EXPORT'` overlapping `export_log` | **Decided** — keep both (approved); `export_log` carries the detail |
| 8 | Row triggers do not fire on `TRUNCATE` | **Resolved** — statement triggers on all three tables (§7); tests I-07…I-09 |
| 9 | Seed precision guarantee overstated | **Resolved** — §9.1/§9.2 |
| 10 | Currency-timeline rule evaluated on an unlocked read | **Resolved** — per-employee row lock (ADR-021); tests CONC-01…09 |
| 11 | `postgres:16-alpine` described as a pin | **Resolved** — §1.1, exact patch in `.postgres-version`, asserted by ENV-01 |
| 12 | PERF cases asserting a specific scan type | **Resolved** — latency gates plus captured plans (§6, §12) |

No other divergence found: error codes agree across architecture.md §7, api-design.md §1.7
and the trigger/function `HINT` set; the Prisma draft matches the ER diagram column for
column; all ADR cross-references resolve.

---

## 11. Decisions

### 11.1 Blocking — I need an answer before migrations

| # | Decision | Recommendation |
|---|---|---|
| **B1** | **PostgreSQL patch version** to pin, and the version production reports (§1.1) | Supply the managed instance's `SHOW server_version`; CI pins exactly that and ENV-01 enforces it. `16.10` is a placeholder |
| **B2** | **`pg_trgm` and `citext` availability** — `CREATE EXTENSION` on the primary **and** in the Prisma shadow database (§1.2) | Confirm with the provider; this blocks M0. Fallback for search is a `tsvector` column (prefix only, no mid-word substrings) |
| **B3** | **`export_log` shape** for ADR-022: `status text CHECK (STARTED|COMPLETED)`, `row_count int NULL`, `request_id text` + index | Confirm the shape; the behaviour is approved, this changes the Prisma model and ER diagram |
| **B4** | **Keep `postgresqlExtensions` preview** (my recommendation, reversed from last round) | **Keep it.** It is what creates `citext` in the shadow database for `migrate dev`. Dropping it means replacing `migrate dev` with `migrate diff` + `migrate deploy` in CI — a workflow change, so it is your call |

### 11.2 Defaults I will take unless you object

| # | Decision | Default |
|---|---|---|
| 5 | Locking mechanism for ADR-021 | `SELECT … FOR UPDATE` on the `employee` row (closes the deactivation race too) rather than `pg_advisory_xact_lock` |
| 6 | One salary append per transaction; batch appends must lock in ascending `employee_id` order | adopted as a working rule (architecture.md §10.3) |
| 7 | DB uses `min_scale()`, Zod uses strict `scale()` | approved; S-07 pins the behaviour |
| 8 | Plan-shape checks in PERF are advisory warnings; latency is the gate | §6, §12 |
| 9 | `p_currency` typed `text` in the function signature | function signatures carry no typmod, so `char(3)` would appear as `character` and make the signature ambiguous to read; `upper()` + cast happens inside |

### 11.3 Open, not blocking Phase 3

`CURRENCY_CHANGE` as a *required* reason (architecture.md §10.2 item 1), currency-filter
latency (item 2 — measured in PERF-03), correction changing currency on a non-latest event
(item 4), `Idempotency-Key` (item 5), audit/export retention (item 6), session pruning
(item 7).

---

## 12. Phase 3 exit criteria

1. M0–M8 applied to a clean database; `prisma migrate diff --exit-code` reports no drift.
2. All **195** cases in [sql-test-matrix.md](sql-test-matrix.md) pass, including the three
   session-timezone runs and the concurrency series.
3. `ENV-01` confirms the server's `major.minor` matches `.postgres-version`.
4. Seed: 10,000 employees under 2 minutes, §9.1 pre-insert validation returning zero rows,
   all §9.2 assertions green, two runs byte-identical.
5. **Latency gates** in series PERF met over ≥ 20 warm iterations, with
   `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` captured as PR artifacts for human review. Plan
   shape is reviewed, not asserted.
6. `UPDATE`, `DELETE` and `TRUNCATE` fail on all three append-only tables; a direct `INSERT`
   into `salary_event` as `acme_app` fails; the privilege matrix matches §8 as checked by
   `has_*_privilege`.
7. No API endpoint, service, controller or frontend file exists.
