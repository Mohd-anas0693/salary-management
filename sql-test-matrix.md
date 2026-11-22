# SQL Test Matrix — Salary Event Resolution and Invariants

**Status:** v2, revised against review feedback · 2026-11-22 · Specification of tests, not
the tests themselves. **195 cases.**
**Companion:** [database-implementation-plan.md](database-implementation-plan.md) ·
[database-design.md](database-design.md)
**Harness:** Vitest + a real PostgreSQL pinned to the exact patch in `.postgres-version`
(Testcontainers tag `postgres:<pinned>-alpine`, never `16-alpine`), one transaction per case
rolled back after it, fixtures loaded once per suite as `acme_owner`. Series **CONC** is the
exception: it needs two real connections and commits, so it cleans up explicitly. Cases marked **app** run on a
connection authenticated as `acme_app` to exercise the privilege model.
**Every series except PERF runs three times**, with session `TimeZone` set to `UTC`,
`Asia/Kolkata` (+05:30) and `America/Los_Angeles` (−08:00). Identical results are required
(series **U** exists to make a timezone regression fail loudly rather than subtly).

---

## 1. Fixtures

Reference data: currencies `USD`(2), `EUR`(2), `JPY`(0), `KWD`(3), `GBP`(2, `is_active =
false`); departments `Finance`, `Support`; HR users `U1`, `U2`.

`T` denotes the suite's frozen "today" in UTC. Every fixture event has an **explicit**
`recorded_at`, never `DEFAULT now()`, so ordering is deterministic.

| Employee | Status | Hire | Events (id · effective · amount · currency · recorded_at · supersedes) |
|---|---|---|---|
| **E1** | ACTIVE | T−900 | `101` · T−365 · 80000.00 USD · T−370 · — |
| **E2** | ACTIVE | T−900 | `201` · T−365 · 80000.00 USD · T−370 · — ; `202` · T−365 · 85000.00 USD · T−300 · **201** ; `203` · T−365 · 86000.00 USD · T−200 · **202** |
| **E3** | ACTIVE | T−900 | `301` · T−180 · 70000.00 USD · T−185 · — ; `302` · T−180 · 72000.00 USD · T−100 · — *(duplicate effective date, later recorded_at)* |
| **E4** | ACTIVE | T−900 | `401` · T−180 · 60000.00 USD · T−100 · — ; `402` · T−180 · 61000.00 USD · **T−100** · — *(identical effective_date **and** recorded_at; id is the only tiebreaker)* |
| **E5** | ACTIVE | T−900 | `501` · T−365 · 90000.00 USD · T−370 · — ; `502` · **T+60** · 78000.00 EUR · T−10 · — `CURRENCY_CHANGE` |
| **E6** | ACTIVE | T−900 | `601` · T−365 · 9000000 JPY · T−370 · — |
| **E7** | ACTIVE | T−900 | `701` · T−365 · 12345.678 KWD · T−370 · — |
| **E8** | ACTIVE | T−30 | *(none)* |
| **E9** | ACTIVE | T−30 | `901` · **T+30** · 50000.00 USD · T−5 · — |
| **E10** | INACTIVE (T−10) | T−900 | `1001` · T−365 · 55000.00 USD · T−370 · — |
| **E11** | ACTIVE | T−900 | `1101` · T−365 · 40000.00 USD · T−370 · — ; `1102` · T−200 · 42000.00 USD · T−210 · — ; `1103` · T−200 · 43000.00 USD · T−205 · **1102** *(correction of a non-latest event)* |

---

## 2. Series R — core resolution (requirements D-1)

| # | Case | Query | Expected |
|---|---|---|---|
| R-01 | Single effective event | `salary_as_of(E1, T)` | `101`, 80000.00 USD |
| R-02 | As-of exactly the effective date | `salary_as_of(E1, T−365)` | `101` |
| R-03 | As-of one day before the first event | `salary_as_of(E1, T−366)` | **0 rows** (never `0.00`) |
| R-04 | As-of before the hire date | `salary_as_of(E1, T−901)` | 0 rows |
| R-05 | Employee with no events | `salary_as_of(E8, T)` | 0 rows |
| R-06 | Only a future event | `salary_as_of(E9, T)` | 0 rows |
| R-07 | Future event, future as-of | `salary_as_of(E9, T+30)` | `901`, 50000.00 USD |
| R-08 | Future event excluded from the view | `employee_current_salary WHERE employee_id = E9` | **no row** |
| R-09 | Future currency change not yet current | `employee_current_salary WHERE employee_id = E5` | `501`, 90000.00 **USD** |
| R-10 | Future currency change after its date | `salary_as_of(E5, T+60)` | `502`, 78000.00 **EUR** |
| R-11 | Boundary: day before a scheduled change | `salary_as_of(E5, T+59)` | `501` USD |
| R-12 | Latest of several effective events wins | `salary_as_of(E11, T)` | `1103`, 43000.00 USD |
| R-13 | Mid-history as-of | `salary_as_of(E11, T−300)` | `1101`, 40000.00 USD |
| R-14 | Inactive employee still resolves | `salary_as_of(E10, T)` | `1001`, 55000.00 USD |
| R-15 | View returns exactly one row per employee with a current salary | `count(*) FROM employee_current_salary` | 9 of the 11 fixtures (E8, E9 absent) |
| R-16 | View carries currency, not just amount | `employee_current_salary WHERE employee_id = E6` | 9000000 **JPY** |
| R-17 | **Function and view agree** (ADR-010) | full outer join of `salary_as_of(e, utc_today())` over all employees against the view | **zero mismatching rows** |
| R-18 | Resolution plan is captured for review | `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` on the view | plan stored as a PR artifact; **advisory** warning if it contains a `Sort` node or a `Seq Scan` on `salary_event`. The pass/fail gate is PERF-01 latency, because the planner may legitimately choose differently with different statistics or cache state |

---

## 3. Series C — corrections and supersession (D-2, D-3, ADR-009)

| # | Case | Action | Expected |
|---|---|---|---|
| C-01 | Chain tail resolves | `salary_as_of(E2, T)` | `203`, 86000.00 |
| C-02 | Superseded events excluded | `salary_as_of(E2, T)` | never `201` or `202` |
| C-03 | Depth-3 chain in history | `salary_event_resolved WHERE employee_id = E2` | 3 rows; `201`,`202` `is_superseded = true`; `203` false |
| C-04 | Supersession is derived, not stored | `\d salary_event` | **no** `is_superseded` / `superseded_by` column |
| C-05 | Correction of a superseded event | insert superseding `201` again | **fails**, unique violation `23505` → `EVENT_ALREADY_SUPERSEDED` |
| C-06 | Forked chain impossible | two inserts both superseding `203`, serially | second fails `23505` |
| C-07 | **Concurrent** corrections of one event | two sessions, both superseding `203`, commit order A then B | A commits; B fails `23505`; exactly one successor exists |
| C-08 | Self-supersession | insert with `supersedes_event_id = id` | impossible by construction; `CHECK (supersedes_event_id < id)` rejects any backward-pointing violation `23514` |
| C-09 | Cycle impossible | attempt to supersede a **higher** id | `CHECK` violation `23514` |
| C-10 | Cross-employee supersession | event for E1 superseding `203` (E2) | trigger `P0001` → `RULE_VIOLATION` |
| C-11 | Missing correction reason | supersede `101` with `correction_reason = NULL` | `CHECK` violation `23514` |
| C-12 | `CORRECTION` reason without a pointer | `change_reason = 'CORRECTION'`, `supersedes_event_id = NULL` | `CHECK` violation `23514` |
| C-13 | Correction changes the amount | correct `101` → 81000.00 USD | accepted; `salary_as_of(E1, T)` = 81000.00; `101` retains 80000.00 |
| C-14 | Correction changes the effective date | correct `101` → effective T−300 | accepted; `salary_as_of(E1, T−350)` now **0 rows** |
| C-15 | Correction of a **non-latest** event does not become current | `salary_as_of(E11, T)` | `1103` is current only because T−200 > T−365 — assert `1101` unaffected and still resolves at T−300 |
| C-16 | Correction preserves the original author | `salary_event_resolved` for the chain | each row keeps its own `recorded_by_hr_user_id` |
| C-17 | Correcting an event of an **inactive** employee | correct `1001` (E10) | trigger `P0001` → `EMPLOYEE_INACTIVE` (consistent with ADR-015) |

---

## 4. Series D — duplicate effective dates (ADR-013)

| # | Case | Query | Expected |
|---|---|---|---|
| D-01 | Tiebreak by later `recorded_at` | `salary_as_of(E3, T)` | `302`, 72000.00 |
| D-02 | Loser remains in history | history of E3 | both `301` and `302`; `301` not superseded, never current |
| D-03 | Tiebreak by `id` when `recorded_at` ties | `salary_as_of(E4, T)` | `402`, 61000.00 (higher id) |
| D-04 | Determinism under repetition | run D-01 and D-03 100× | identical results every time |
| D-05 | Determinism under physical reordering | `VACUUM FULL` / re-insert fixtures in reverse id order | same winner; no reliance on heap order |
| D-06 | Duplicate dates are permitted | insert a third event for E3 on T−180 | **accepted** (no unique constraint on `(employee_id, effective_date)`) |
| D-07 | Duplicate plus correction | correct `302`; then resolve | the correction wins, `301` still never current |

---

## 5. Series B — backdated events

| # | Case | Action | Expected |
|---|---|---|---|
| B-01 | Backdated amount accepted | E1, effective T−200, 82000.00 USD, `MERIT` | accepted; `salary_as_of(E1, T−100)` = 82000.00 |
| B-02 | Backdating does not alter `recorded_at` | same row | `recorded_at ≈ now()`, `effective_date = T−200`; the two differ |
| B-03 | Backdated before hire date | E1, effective T−1000 | trigger `P0001` → `RULE_VIOLATION` |
| B-04 | Backdated on the hire date | E1, effective T−900 | accepted (inclusive bound) |
| B-05 | Backdated event earlier than an existing one | E1, effective T−800 | accepted; does not change the current salary |
| B-06 | Backdated **same currency** after a scheduled change | E5, effective T−100, USD | accepted — prior currency at T−100 was USD |
| B-07 | Backdated **different currency** | E5, effective T−100, EUR | trigger `P0001` → `BACKDATED_CURRENCY_CHANGE` |
| B-08 | Backdated as-of before and after | B-01 then `salary_as_of(E1, T−300)` | still `101` 80000.00 |

---

## 6. Series CUR — currency rules (ADR-011)

| # | Case | Action | Expected |
|---|---|---|---|
| CUR-01 | Prospective change accepted | E1, effective T+10, EUR, `CURRENCY_CHANGE` | accepted |
| CUR-02 | Change effective **today** accepted | E1, effective T, EUR, `CURRENCY_CHANGE` | accepted (bound is `>= utc_today()`) |
| CUR-03 | Change effective yesterday rejected | E1, effective T−1, EUR, `CURRENCY_CHANGE` | `P0001` → `BACKDATED_CURRENCY_CHANGE` |
| CUR-04 | Prospective change with the wrong reason | E1, effective T+10, EUR, `MERIT` | `P0001` → `CURRENCY_CHANGE_REASON_REQUIRED` |
| CUR-05 | Same-currency future event needs no special reason | E1, effective T+10, USD, `MERIT` | accepted |
| CUR-06 | First event sets currency freely | E8, effective T−10, JPY, `HIRE` | accepted (no prior currency) |
| CUR-07 | Correction may change currency | correct `101` → EUR with a reason | accepted |
| CUR-08 | Correction changing currency without a reason | same, `correction_reason = NULL` | `CHECK` violation `23514` |
| CUR-09 | History retains the original currency | after CUR-07 | `101` still USD; `salary_event_resolved` shows both |
| CUR-10 | As-of before a change returns the old currency | `salary_as_of(E5, T)` then `salary_as_of(E5, T+60)` | USD, then EUR |
| CUR-11 | Inactive currency rejected | E1, GBP (`is_active = false`) | `P0001` → `CURRENCY_INACTIVE` |
| CUR-12 | Unknown currency rejected | E1, `'XXX'` | FK violation `23503` → `RULE_VIOLATION` |
| CUR-13 | Lower-case currency rejected | `'usd'` | `CHECK` violation `23514` |
| CUR-14 | No employee-level currency exists | `\d employee` | **no** `salary_currency` column |
| CUR-15 | Mixed-currency history is representable | E5 at T+60 and E5 at T | two currencies across one employee's timeline |
| CUR-16 | Reversal is prospective-legal | after CUR-01, add effective T+20 USD `CURRENCY_CHANGE` | accepted (prior currency at T+20 is EUR) |
| CUR-17 | Prior currency is evaluated **at the timeline position**, not today | E5, effective T+70, EUR, `MERIT` | accepted — prior currency at T+70 is already EUR, so this is not a change |

---

## 7. Series S — monetary precision (ADR-004, plan §2)

All **app**-role cases go through `salary_event_append(... p_amount_text ...)`.

| # | Case | Input | Expected |
|---|---|---|---|
| S-01 | USD two decimals | `"85000.00"` | accepted, stored `85000.0000`, serializes `"85000.00"` |
| S-02 | USD over-precision, survives rounding | `"85000.001"` | rejected `INVALID_DECIMAL_SCALE` |
| S-03 | **USD over-precision beyond scale 4** | `"85000.00001"` | **rejected** — the gap this plan closes; without the function it would be silently stored as `85000.0000` |
| S-04 | JPY integral | `"9000000"` | accepted |
| S-05 | JPY with a fractional part | `"9000000.5"` | rejected |
| S-06 | **JPY beyond scale 4** | `"9000000.00004"` | **rejected** |
| S-07 | JPY trailing zeros | `"9000000.00"` | **accepted** under `min_scale()` (plan §11.2 decision 7); rejected if strict `scale()` is chosen — this case pins whichever you approve |
| S-08 | KWD three decimals | `"12345.678"` | accepted |
| S-09 | KWD four decimals | `"12345.6781"` | rejected |
| S-10 | **KWD beyond scale 4** | `"12345.678004"` | **rejected** |
| S-11 | Zero | `"0.00"` | rejected `AMOUNT_OUT_OF_RANGE` |
| S-12 | Negative | `"-100.00"` | rejected `INVALID_DECIMAL_FORMAT` (regex) |
| S-13 | Thousands separators | `"85,000.00"` | rejected `INVALID_DECIMAL_FORMAT` |
| S-14 | Scientific notation | `"8.5e4"` | rejected `INVALID_DECIMAL_FORMAT` |
| S-15 | Empty / non-numeric | `""`, `"abc"`, `"NaN"`, `"Infinity"` | all rejected `INVALID_DECIMAL_FORMAT` |
| S-16 | Magnitude limit | `"10000000000000000.00"` (10¹⁶) | rejected `AMOUNT_OUT_OF_RANGE`, **not** a raw `22003` overflow |
| S-17 | Largest valid | `"9999999999999999.99"` | accepted |
| S-18 | Leading zeros | `"0085000.00"` | accepted, stored `85000.0000` |
| S-19 | Column type is unchanged | `information_schema.columns` | `numeric`, precision 20, scale 4 (ADR-004 intact) |
| S-20 | Sum of exact amounts is exact | sum 9,640 seeded USD salaries twice in different orders | identical to the cent; no float drift |
| S-21 | Average rounding is explicit | dashboard aggregate for USD | `round(sum/count, 4)` = `minor_units + 2`; never a `double precision` type in the result |
| S-22 | Median type | `percentile_disc` over E-fixtures | result type `numeric`, value equals an actual stored amount |
| S-23 | No float anywhere in analytics | `pg_typeof` over every analytics projection | no `double precision` column |
| S-24 | **Seed staging rejects over-precision pre-insert** | load `"85000.00001"` into `seed_salary_stage`, run the §9.1 validation query | returns the offending row → seed aborts **before** any coercion |
| S-25 | Seed staging accepts conforming strings | `"85000.00"` USD, `"9000000"` JPY, `"12345.678"` KWD | validation query returns zero rows |
| S-26 | **Post-seed assertion scope is honest** | insert `85000.00001` directly as `acme_owner` (bypassing stage and function), then run the post-seed `min_scale(amount) <= minor_units` assertion | the assertion **passes** — documenting that it proves stored-value conformance and cannot detect precision already lost to coercion (plan §9.2). Only S-24's pre-insert check can |

---

## 8. Series U — UTC date boundary (ADR-012, plan §3)

| # | Case | Setup | Expected |
|---|---|---|---|
| U-01 | `utc_today()` is session-independent | `SET TimeZone='Asia/Kolkata'` then `'America/Los_Angeles'` | same date in both, equal to `(now() AT TIME ZONE 'UTC')::date` |
| U-02 | Resolution unaffected by session zone | run series R under all three zones | identical results |
| U-03 | Event effective "tomorrow UTC" | insert effective `utc_today() + 1`; read the view from a +05:30 session | **not** current |
| U-04 | Event effective "today UTC" | insert effective `utc_today()`; read from a −08:00 session | **is** current |
| U-05 | Currency rule boundary under a positive offset | CUR-02 under `Asia/Kolkata` | accepted, as under UTC |
| U-06 | Currency rule boundary under a negative offset | CUR-03 under `America/Los_Angeles` | rejected, as under UTC |
| U-07 | No session-dependent expression survives | grep migrations for `CURRENT_DATE`, `current_date`, `now()::date`, `localtimestamp`, `AT TIME ZONE` with a non-UTC zone | only `utc_today()` defines today |
| U-08 | `utc_today()` volatility | `pg_proc.provolatile` | `s` (STABLE) — so it cannot be used in a `CHECK` or index predicate |
| U-09 | `recorded_at` is absolute | insert from two session zones | both store the same instant; `recorded_at` is `timestamptz` |
| U-10 | DATE round-trip has no off-by-one | write `T+1`, read back via the client, compare the `YYYY-MM-DD` string | equal under all three zones |
| U-11 | The trigger itself uses `utc_today()` | inspect `pg_get_functiondef` for `salary_event_validate` and `salary_event_append` | contains `utc_today`, and none of `CURRENT_DATE`, `now()::date`, `localtimestamp` |

---

## 9. Series I — immutability (D-2, ADR-003) and P — privileges (ADR-019)

| # | Case | Action | Expected |
|---|---|---|---|
| I-01 | Update a salary event | `UPDATE salary_event SET amount = 1 WHERE id = 101` | `P0001`, message names the table and `UPDATE` |
| I-02 | Delete a salary event | `DELETE FROM salary_event WHERE id = 101` | `P0001` |
| I-03 | Update an audit row | `UPDATE audit_log …` | `P0001` |
| I-04 | Delete an audit row | `DELETE FROM audit_log …` | `P0001` |
| I-05 | Update an export row | `UPDATE export_log …` | `P0001` |
| I-06 | Delete an export row | `DELETE FROM export_log …` | `P0001` |
| I-07 | **Truncate `salary_event`** | `TRUNCATE salary_event` as `acme_owner` | `P0001` from `reject_truncate()` — row triggers do not fire on `TRUNCATE`, so the statement-level trigger is what stops it |
| I-08 | **Truncate `audit_log`** | `TRUNCATE audit_log` as `acme_owner` | `P0001` |
| I-09 | **Truncate `export_log`** | `TRUNCATE export_log` as `acme_owner` | `P0001` |
| I-10 | Owner is bound by the mutation triggers too | I-01 as `acme_owner` | `P0001` — triggers are not role-conditional |
| P-01 **app** | Direct insert into `salary_event` | `INSERT INTO salary_event …` | `42501` insufficient privilege (plan §2.2) |
| P-02 **app** | Append via the function | `salary_event_append(...)` | accepted |
| P-03 **app** | Function is not world-executable | `EXECUTE` as an unrelated role | `42501` |
| P-04 **app** | Delete an employee | `DELETE FROM employee WHERE id = E8` | `42501` — no `DELETE` grant anywhere |
| P-05 owner | Delete an employee **with** history | as `acme_owner` | `23503` FK `RESTRICT` |
| P-06 owner | Delete an employee **without** history | E8 as `acme_owner` | succeeds — the API has no such path (ADR-019); documents that the last line of defence is the privilege model |
| P-07 **app** | Update a `currency` row | `UPDATE currency …` | `42501` |
| P-08 **app** | Read the views | `SELECT FROM employee_current_salary` | permitted |
| P-09 | Future tables are not auto-granted | create a table as owner, then `SELECT` as app | `42501` (`ALTER DEFAULT PRIVILEGES` held) |
| P-10 **app** | No `CREATE` on schema `public` | `CREATE TABLE t(…)`; `CREATE FUNCTION …` | `42501` both — so the app cannot define a definer function of its own |
| P-11 **app** | Trigger functions are not world-executable | `SELECT public.reject_mutation()`, `reject_truncate()`, `salary_event_validate()` | `42501` (revoked from PUBLIC), not "can only be called as a trigger" |
| P-12 | Definer rights are not superuser rights | `pg_roles` for `acme_owner` | `rolsuper = false`; the function's definer context is exactly the table owner |
| P-13 | Privilege matrix matches plan §8 exactly | `has_table_privilege` for every table × privilege, and `has_function_privilege` with the **full signature** for each function | matches the matrix cell for cell, including `INSERT` = false on `salary_event`, `DELETE` = false everywhere, `TRUNCATE` = false everywhere |
| P-14 **app** | No `USAGE` on the salary sequence | `SELECT nextval('salary_event_id_seq')` | `42501` |
| P-15 **app** | Signature-scoped grants | `has_function_privilege('acme_app', 'public.salary_event_append(bigint,text,text,date,text,text,bigint,text,bigint)', 'EXECUTE')` | true; and a hypothetical differing signature is **not** granted |

---

## 10. Series H — bitemporal / historical queries (ADR-018)

| # | Case | Query | Expected |
|---|---|---|---|
| H-01 | Known-at before any correction | `salary_as_of(E2, T−365, T−350)` | `201`, 80000.00 |
| H-02 | Known-at between corrections | `salary_as_of(E2, T−365, T−250)` | `202`, 85000.00 |
| H-03 | Known-at after all corrections | `salary_as_of(E2, T−365, now())` | `203`, 86000.00 |
| H-04 | Known-at before the event was recorded | `salary_as_of(E1, T−365, T−380)` | 0 rows |
| H-05 | Known-at ignores a later-recorded backdated event | insert B-01, then `salary_as_of(E1, T−100, T−150)` | the pre-insert answer, unchanged |
| H-06 | Both axes move independently | matrix of `as_of ∈ {T−365, T−200}` × `known_at ∈ {T−350, T−250, now()}` | 6 documented answers, asserted individually |
| H-07 | Default `known_at` | `salary_as_of(E2, T−365)` | equals H-03 |
| H-08 | `known_at` is not exposed | API contract review (api-design.md §4.4) | no `knownAt` request parameter and none in responses |

---

## 11. Series A — analytics (D-6, ADR-016, ADR-017)

| # | Case | Query | Expected |
|---|---|---|---|
| A-01 | Grouping is per currency | dashboard aggregate over fixtures | one row per currency; no row aggregates two currencies |
| A-02 | No cross-currency total exists | scan every analytics projection | no column sums across currencies |
| A-03 | Inactive employees excluded from payroll | totals with E10 present | E10's 55000.00 absent; inactive count = 1 |
| A-04 | No-salary employee counted in headcount only | E8 | in headcount; absent from totals, averages, medians, buckets |
| A-05 | Scheduled-only employee classified separately | E9 | `scheduledOnly`, not `neverRecorded` |
| A-06 | Future events excluded from current totals | E5 | contributes 90000.00 USD, not 78000.00 EUR |
| A-07 | Median is `percentile_disc` | odd-sized USD group | an actual member value |
| A-08 | Median of an even-sized group | 4 USD salaries | the **lower** middle value (documented in `meta.medianNote`) |
| A-09 | Median per currency | mixed fixtures | JPY and KWD medians computed within their own groups |
| A-10 | Buckets are per currency | distribution for USD and JPY | different bounds; no shared axis |
| A-11 | Bucket bounds from the currency's own min/max | USD distribution | `lowerBound`/`upperBound` equal that currency's min/max |
| A-12 | Single-member currency group | KWD (one employee) | one bucket, count 1, no division error |
| A-13 | Empty currency group | currency with no current salaries | row absent, not a zero row |
| A-14 | Department comparison nests under currency | by-department aggregate | currency is the outer grouping |
| A-15 | Superseded events never reach analytics | E2 | only 86000.00 contributes |

---

## 12. Series DE — deactivation (D-7, ADR-015, ADR-019)

| # | Case | Action | Expected |
|---|---|---|---|
| DE-01 | New event for an inactive employee | E10, any valid event | `P0001` → `EMPLOYEE_INACTIVE` |
| DE-02 | History readable after deactivation | E10 | full history and `salary_as_of` work normally |
| DE-03 | Deactivate → reactivate → append | E10 reactivated | append now accepted; both status changes audited |
| DE-04 | Status/timestamp consistency | `status='INACTIVE', deactivated_at=NULL` | `CHECK` violation `23514` |
| DE-05 | Reactivation clears the timestamp | reactivate E10 | `deactivated_at IS NULL` |
| DE-06 | Audit survives deactivation | audit rows for E10 | intact and queryable |

---

## 13. Series AUD — audit log (database-design.md §3.6)

| # | Case | Action | Expected |
|---|---|---|---|
| AUD-01 | Failed login, unknown email | insert `LOGIN_FAILURE`, `actor = NULL`, `actor_email_attempted` set | accepted |
| AUD-02 | Failed login without an attempted email | `LOGIN_FAILURE`, both null | `CHECK` violation `23514` |
| AUD-03 | Null actor on a non-login action | `SALARY_CREATE`, `actor = NULL` | `CHECK` violation `23514` |
| AUD-04 | Failed login, known email | `actor` set and `actor_email_attempted` set | accepted |
| AUD-05 | Audit row is in the same transaction as its change | roll back a transaction containing append + audit | neither row persists |
| AUD-06 | Unknown action value | `action = 'SOMETHING'` | `CHECK` violation `23514` |
| AUD-07 | No password material stored | scan `audit_log` columns and payloads | no password, hash, or token field |

---

## 14. Series EXP — export audit records (ADR-022)

| # | Case | Action | Expected |
|---|---|---|---|
| EXP-01 | `STARTED` row has no count | insert `status='STARTED'`, `row_count=NULL` | accepted |
| EXP-02 | `STARTED` row with a count | `status='STARTED'`, `row_count=10` | `CHECK` violation `23514` |
| EXP-03 | `COMPLETED` row without a count | `status='COMPLETED'`, `row_count=NULL` | `CHECK` violation `23514` |
| EXP-04 | Completed export writes a pair | `STARTED` then `COMPLETED` with one `request_id` | two rows, same `request_id`, `recorded_at` ordered |
| EXP-05 | **Aborted export leaves a trace** | write `STARTED`, then abandon (no `COMPLETED`) | the `STARTED` row persists and is findable by "STARTED with no COMPLETED sibling" — the exfiltration-relevant signal |
| EXP-06 | Unknown status / export type | `status='PARTIAL'`; `export_type='EVERYTHING'` | `CHECK` violation `23514` each |

---

## 15. Series CONC — concurrency (ADR-021)

Two committed connections per case, with a barrier so the interleaving is deterministic.
`C1` and `C2` are sessions; `→ commit` marks the commit point.

| # | Case | Interleaving | Expected |
|---|---|---|---|
| CONC-01 | Two appends, same employee | C1 append E1 · C2 append E1 (both before either commits) | C2 **blocks** on the `employee` row lock until C1 commits, then proceeds. Both rows exist; D-1 decides which is current, deterministically |
| CONC-02 | **The currency write skew** | committed history USD. C1 appends T+10 EUR `CURRENCY_CHANGE`; C2 appends T+20 USD `MERIT`; C1 → commit; C2 proceeds | C2 is **rejected** with `CURRENCY_CHANGE_REASON_REQUIRED` — it re-reads the committed timeline under the lock and sees EUR as the prior currency. **Without the lock this pair commits an invalid USD → EUR → USD timeline**, which is the regression this case guards |
| CONC-03 | Append racing deactivation (deactivate first) | C1 `UPDATE employee SET status='INACTIVE'` → commit; C2 append | C2 fails `EMPLOYEE_INACTIVE` — it reads the committed status under the lock, not a stale snapshot |
| CONC-04 | Append racing deactivation (append first) | C1 append → commit; C2 deactivate | both succeed; the event belongs to a then-active employee and history is unaffected |
| CONC-05 | Concurrent corrections of one event | C1 and C2 both supersede `203` | one succeeds, the other fails `23505` `EVENT_ALREADY_SUPERSEDED` (cross-reference C-07) |
| CONC-06 | Different employees never contend | C1 append E1 · C2 append E2, interleaved | neither blocks; both commit. Asserted by elapsed time, not just outcome |
| CONC-07 | No deadlock under load | 50 concurrent appends across 10 random employees | all succeed or fail on a documented rule; **zero** `40P01` deadlocks |
| CONC-08 | Lock released on rollback | C1 append then `ROLLBACK`; C2 append same employee | C2 proceeds immediately; no row from C1 |
| CONC-09 | `READ COMMITTED` suffices | run CONC-01…08 at the default isolation level | all pass with no `40001` serialization failures and no retry logic |

---

## 16. Series ENV — platform pinning (plan §1.1)

| # | Case | Check | Expected |
|---|---|---|---|
| ENV-01 | **Server version matches the pin** | `SHOW server_version` vs `.postgres-version` | identical `major.minor`; mismatch **fails the suite** |
| ENV-02 | Required extensions present | `pg_extension` | `citext` and `pg_trgm` installed, in the test database and the shadow database |
| ENV-03 | Explicit UTC does not depend on the server setting | run U-01 with the database's `timezone` deliberately set to `Asia/Tokyo` | `utc_today()` still returns the UTC date — the `ALTER DATABASE … SET timezone='UTC'` is defense in depth, not a dependency |

---

## 17. Series PERF — latency gates, plans reviewed not asserted (NFR-1, plan §6)

Run against the full 10,000-employee seed. **Method:** 3 warm-up iterations, then ≥ 20
measured iterations; the gate is the **p95 latency** below. `EXPLAIN (ANALYZE, BUFFERS,
FORMAT JSON)` is captured for every case and attached to the PR for human review.

**No case asserts a scan type or the absence of a `Sort` node.** On a table of this size the
planner may legitimately choose a sequential scan, a bitmap heap scan or an index scan
depending on statistics, cache state, `work_mem` and the CI machine — and a test that demands
one shape fails for reasons unrelated to correctness. Plan **properties** are recorded as
advisory warnings instead: a `Seq Scan` on `salary_event`, a `Sort` in the current-salary
path, or `Rows Removed by Filter` above 10× the returned rows each raise a review flag in the
PR comment without failing the build. A latency gate miss does fail.

| # | Case | Gate (p95) | Advisory flags |
|---|---|---|---|
| PERF-01 | `employee_current_salary` full scan | < 150 ms | `Sort` present; `Seq Scan` on `salary_event` |
| PERF-02 | Employee list, page 1, no filter | < 500 ms | over-fetch beyond `pageSize` |
| PERF-03 | **Employee list filtered by currency** — the ADR-011 cost | < 500 ms | rows examined ≫ rows returned. **The gating measurement**: if it misses, escalate to the ADR-006 materialized view, not a denormalized currency column |
| PERF-04 | Employee list sorted by `currentSalary` + currency filter | < 500 ms | `Sort` over the whole population |
| PERF-05 | Free-text `q` search | < 500 ms | trigram index unused |
| PERF-06 | Dashboard aggregate, all currencies | < 500 ms | — |
| PERF-07 | Department + country comparisons with medians | < 3 s | — |
| PERF-08 | One employee's history, page 1 | < 200 ms | `Seq Scan` on `salary_event` |
| PERF-09 | Deep offset (page 200) | recorded, not gated | feeds the ADR-005 revisit trigger |
| PERF-10 | Seed runtime | < 2 min (NFR-5) | — |
| PERF-11 | Seed determinism | two fresh seeds → identical `md5` over an ordered dump | — |
| PERF-12 | Append latency through the guarded function | < 100 ms per append, and CONC-06 shows no cross-employee contention | lock wait time > 0 for different employees |

---

## 18. Coverage summary

| Series | Cases | Covers |
|---|---|---|
| R | 18 | D-1 resolution, future events, boundaries, function/view equality |
| C | 17 | D-2, D-3, correction chains, forks, cycles |
| D | 7 | duplicate effective dates and determinism |
| B | 8 | backdated events, hire-date bound, currency interaction |
| CUR | 17 | D-8 / ADR-011 in full |
| S | 26 | ADR-004, pre-coercion precision (ADR-020), seed staging |
| U | 11 | ADR-012 under three session timezones |
| I | 10 | immutability: UPDATE, DELETE **and TRUNCATE** on all three tables |
| P | 15 | the privilege matrix, verified with `has_*_privilege` |
| H | 8 | bitemporal queries, `knownAt` internal |
| A | 15 | D-6, medians, buckets, no-salary employees |
| DE | 6 | deactivation rules |
| AUD | 7 | nullable actor and audit integrity |
| EXP | 6 | export `STARTED`/`COMPLETED` records (ADR-022) |
| CONC | 9 | concurrent appends, the currency write skew, deactivation race (ADR-021) |
| ENV | 3 | version pin and extension presence |
| PERF | 12 | NFR-1, NFR-5, the ADR-011 measurement |
| **Total** | **195** | every binding rule in requirements §1 (D-1…D-8) |

**Deliberately not covered here:** API status codes, request validation, response shapes, CSV
formatting and streaming — those belong to the Phase 4 Supertest suite. This matrix stops at
the database boundary; each case asserts a SQLSTATE plus the `HINT` that Phase 4 maps to an
HTTP code.

**Cases that encode a correction from this review round**, and would have failed the v1
design: I-07…I-09 (`TRUNCATE` bypassed row triggers), S-03/S-06/S-10 and S-24/S-26
(pre-coercion precision, and the honest scope of the post-seed assertion), CONC-02 (the
currency write skew), P-13…P-15 (signature-exact privileges), U-11 (`CURRENT_DATE` in the
trigger) and ENV-01 (a floating image tag instead of a pin).
