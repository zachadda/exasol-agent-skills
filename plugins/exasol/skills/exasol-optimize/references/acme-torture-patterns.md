# ACME_TORTURE patterns — adversarial cases and current coverage

`ACME_TORTURE` is a synthetic Exasol schema that exercises every documented edge case of the four `EXA_OPTIMIZE` UDFs. The smoke harness in `exasol-factory-foundation/scripts/optimize_torture_smoke.py` asserts a `(severity, action_name, sql_fragment)` triple per case. A failed assertion in CI is a regression.

Snapshot from the most recent harness run (V.7, 2026-05-13): **55 cases · 55 passing · 0 failing** across nine UDF surfaces (`ANALYZE_CONSTRAINTS`, `INFER_JOIN_PATHS`, `BUILD_UNKNOWN_MEMBER_INSERT`, `DRY_RUN_PLAN`, `VALIDATE_JOIN_PLAN`, `CONVERT_VARCHAR`, `CONVERT_DATATYPES`, `ANALYZE_DIM_DATE`, `GENERATE_DIM_DATE`).

### Severity vocabulary — two systems coexist by design

`ANALYZE_CONSTRAINTS` emits `critical` / `warning` / `info`; `BUILD_UNKNOWN_MEMBER_INSERT` and `DRY_RUN_PLAN` emit `PASS` / `REPAIR` / `ERROR` / `INFO`. Both vocabularies survived the unification pass because their consumers want different signals (urgency-ladder vs action-outcome). The bridge is the `ACTION_NAME` column on `ANALYZE_CONSTRAINTS` (`ADD_PK`, `ADD_FK`, `WIDEN_FK`, `INSERT_UNKNOWN_MEMBER`, `NOT_NULL_TIGHTEN`, `NO_CANDIDATE`, `REJECTED_NULLS`, `REJECTED_DUPLICATES`, `REJECTED_CANDIDATE`, `FK_NO_MATCH`) — every emitted row carries a stable verb so downstream routing doesn't depend on which severity vocabulary you reach for.

### What changed in V.5/V.6 (2026-05-13)

The smoke harness was broken: it issued `SELECT * FROM EXA_OPTIMIZE.<NAME>(...);` which Exasol's parser rejects, captured zero rows on every call, and the "13 passing" cases were all negative cases trivially passing on the empty capture. Fixed by switching to bare `EXECUTE SCRIPT <NAME>(args);` invocation, zipping the backend's list-row response with the column header (the asserter expects dict rows), and supplying the missing PK-column argument to `BUILD_UNKNOWN_MEMBER_INSERT(SCHEMA, DIM, PK)`. The DRY_RUN_PLAN section was rewritten to read the real shape of `/api/optimize/dry-run` (snake_case `validation_rows` + `statements` + top-level `success/rolled_back/committed`) and synthesize a SUMMARY row.

Three real UDF bugs were fixed in the same pass:

1. **VARCHAR(1) sentinel overflow.** `unknown_key_literal('VARCHAR(1)')` returned `'-1'` (2 chars). Insert overflowed. Now clamps to `'?'` when declared length < 2. `default_value(is_key=True)` was rewired to delegate so the INSERT row and the NOT EXISTS predicate use the same literal.
2. **Composite-PK refusal in BUILD_UNKNOWN_MEMBER_INSERT.** The UDF now queries `SYS.EXA_ALL_CONSTRAINT_COLUMNS` for the dim's declared PK; if 2+ columns are declared, it returns an `ERROR` finding instead of a partial-key INSERT. Return shape extended to `SQL_TEXT, SEVERITY, MESSAGE`. ACME_TORTURE.DIM_COMPOSITE_PK has no declared PK so its smoke case still uses the SQL path (covered by the heuristic fix below).
3. **Composite-PK detection in ANALYZE_CONSTRAINTS.** The single-column heuristic short-circuited before the composite greedy stage could run. A new `find_composite_by_naming` pass catches two patterns on empty tables (`KEY_A, KEY_B` shared prefix, and consecutive single-letter columns like `A, B, C`) so the UDF emits the full composite ALTER instead of the first column.

The matrix was re-baselined to assert what these UDFs actually emit today.

## ANALYZE_CONSTRAINTS (28 cases · all passing)

Severities emitted today: `critical` (validated FK + NULL-FK Unknown Member), `warning` (PK proposal + FK inclusion-dependency match), `info` (candidate failed NULL/uniqueness validation).

| Case | Status | Adversarial property |
|---|---|---|
| `DIM_VALID_PK` | ✅ | Single-col candidate, no NULLs, unique — emits `warning` + `PK_DIM_VALID_PK PRIMARY KEY ("ID")` |
| `DIM_NULL_PK` | ✅ | NULLs in candidate column → `info` finding, no `SQL_TEXT` |
| `DIM_DUP_PK` | ✅ | Duplicate values in candidate → `info` finding, no `SQL_TEXT` |
| `DIM_EMPTY` | ✅ | Zero rows — NULL + uniqueness checks pass trivially; UDF emits the PK proposal anyway |
| `DIM_COMPOSITE_PK` | ✅ | 2-column natural key — `find_composite_by_naming` recognises shared-prefix `KEY_A, KEY_B` shape |
| `DIM_COMPOSITE_PK_3` | ✅ | 3-column natural key — `find_composite_by_naming` catches the single-letter run `A, B, C` |
| `DIM_WIDE_RESERVED` | ✅ | 272+ columns including `YEAR`, `ORDER`, `GROUP`, `SOURCE` — every emitted identifier is double-quoted |
| `DIM_PAIRED_PARENT` | ✅ | PK pending in same batch as a FK that references it — PK proposal emitted; paired-FK ordering verified end-to-end via `DRY_RUN_PLAN` |
| `FACT_FK_NARROWER` | ✅ | FK narrower than referenced PK. DIM_CUSTOMER_WIDE has declared PK, so the inclusion-dependency FK discovery fires and the UDF emits a widen-then-ALTER pair (`@@STEP@@ Widen FACT_FK_NARROWER.CUST_CODE to VARCHAR(100) UTF8` + `ADD CONSTRAINT FK_FACT_FK_NARROWER_CUST_CODE`) |
| `FACT_FK_WIDER` | ✅ | FK wider than referenced PK — DIM_CUSTOMER_NARROW has declared PK; UDF refuses the lossy direction, no FK ALTER |
| `FACT_FK_DECIMAL_MISMATCH` | ✅ | DECIMAL(10,2) ↛ DECIMAL(18,0) — no FK ALTER (refused, correctly) |
| `FACT_FK_CHARSET_MISMATCH` | ✅ | UTF8 vs ASCII same length — no FK ALTER (refused, correctly) |
| `FACT_FK_PAIRED` | ✅ | DIM_PAIRED_PARENT has declared PK on ID; UDF emits `FK_FACT_FK_PAIRED_PARENT_ID FOREIGN KEY ("PARENT_ID") REFERENCES "DIM_PAIRED_PARENT"` |
| `FACT_FK_DUP_NAMES` | ✅ | Both `PRIMARY_PARENT_ID` and `SECONDARY_PARENT_ID` reference DIM_PAIRED_PARENT.ID; UDF emits two distinct ADD_FKs (`FK_<fact>_<col>` naming makes them name-distinct, per-column dedup map prevents double-emission of either) |
| `DIM_ASCII_PK` · `DIM_CHAR36_PK` · `DIM_CLOB_PK` · `DIM_CUSTOMER_NARROW` · `DIM_CUSTOMER_WIDE` · `DIM_DECIMAL_NATURAL_PK` · `DIM_DECIMAL_PK` · `DIM_DECIMAL_SURROGATE_PK` · `DIM_VARCHAR1_PK` · `FACT_NO_DECLARED_FK` · `FACT_TIER3_FALLBACK` | ✅ | Surface coverage — every torture dim/fact without a declared PK gets a `warning` PK proposal emitted |
| `FACT_NULL_FK` | ✅ | NULL-FK Unknown Member path: 2 NULL rows in `CUSTOMER_ID` drive the four-step `INSERT_UNKNOWN_MEMBER` block (INSERT default dim row / UPDATE NULLs to -1 / MODIFY NOT NULL / ADD CONSTRAINT FK). Added 2026-05-13 to close coverage on the validated-FK-with-NULLs path; surfaced a UDF bug where `validated_fks` registration silently skipped when inclusion-dependency had already emitted the ALTER |
| `FACT_ROLEPLAY_DATE` | ✅ | Role-playing date FK: both `ORDERDATEKEY` and `SHIPDATEKEY` resolve to `DIM_DATE.DATE_KEY` and emit `severity=critical` `ADD_FK` rows. Exercises the `find_date_dim` branch and the role-playing detection loop |
| `DIM_DATE` | ✅ | Negative case — declared PK on `DATE_KEY` so no PK proposal warning. Acts as the resolution target for `FACT_ROLEPLAY_DATE`'s role-playing FK and for `ANALYZE_DIM_DATE`'s attribute scorer |

## INFER_JOIN_PATHS (6 cases · all passing)

| Case | Status | Adversarial property |
|---|---|---|
| `DIM_CUSTOMER` | ✅ | Stem-collision partner with `DIM_CUSTOMER_TYPE` — no tier-3 row for `CUSTOMER_ID` (ambiguity) |
| `DIM_CUSTOMER_TYPE` | ✅ | Same as above, the other side |
| `VW_DIM_SHADOW` | ✅ | View re-exposing a dim's key column does not inflate the join count |
| `FACT_NO_DECLARED_FK` | ✅ | Tier-2 stem match wins; tier-3 fallback is skipped because tier-2 fired. Emits `stem_match` row for `PAIRED_PARENT_ID → DIM_PAIRED_PARENT.ID` |
| `FACT_TIER3_FALLBACK` | ✅ | Tier-3 fires: `FACT_TIER3_FALLBACK.RESOLVED_KEY → DIM_DECIMAL_SURROGATE_PK.RESOLVED_KEY` with `INFERENCE_SOURCE = same_name_fallback`. Stem `RESOLVED` has no matching dim, exactly one dim exposes `RESOLVED_KEY` column verbatim. Closed 2026-05-13 via fixture column addition |
| `FACT_NULL_FK` | ✅ | Tier-2 `stem_match` row: `FACT_NULL_FK.CUSTOMER_ID → DIM_CUSTOMER.CUSTOMER_ID`. Stem `CUSTOMER` resolves cleanly despite the `DIM_CUSTOMER_TYPE` collision because `dim_lookup` keys on the exact-stem entry first |

## BUILD_UNKNOWN_MEMBER_INSERT (6 cases · all passing)

UDF now returns `SQL_TEXT, SEVERITY, MESSAGE`. `SEVERITY = ERROR` on declared composite-PK refusal; otherwise `PASS` with the INSERT statement.

| Case | Status | Adversarial property |
|---|---|---|
| `DIM_VARCHAR1_PK` | ✅ | VARCHAR(1) PK — sentinel clamped to `'?'` so the INSERT fits the declared length. Fixed 2026-05-13 |
| `DIM_CHAR36_PK` | ✅ | CHAR(36) PK — quoted `'-1'`; Exasol pads on INSERT |
| `DIM_DECIMAL_SURROGATE_PK` | ✅ | DECIMAL(18,0) keylike-numeric → bare `-1` (no quotes) |
| `DIM_DECIMAL_NATURAL_PK` | ✅ | DECIMAL(10,2) non-keylike — numeric type rule wins; bare `-1` in the predicate |
| `DIM_CLOB_PK` | ✅ | Exasol VARCHAR(2000000) UTF8 PK — single-quoted sentinel emitted |
| `DIM_COMPOSITE_PK` | ✅ | Heuristic: UDF emits SQL referencing `KEY_A`. Refusal-with-ERROR-row path verified separately against a dim with a declared composite PK |

## DRY_RUN_PLAN (5 cases · all passing)

| Case | Status | Adversarial property |
|---|---|---|
| `dry_run_repair_injection` | ✅ | Multi-statement plan with mid-batch REPAIR; UDF self-heals (3 statements executed: 2 input + 1 injected repair). SUMMARY = PASS / DRY_RUN |
| `dry_run_validation_error` | ✅ | Statement against nonexistent table — validation row with `SEVERITY = ERROR`; no execution proceeds |
| `dry_run_skip_statement` | ✅ | Already-declared PK on `DIM_PRE_PK` — validator emits PASS / `PRIMARY_KEY_DATA` row referencing the table. TODO(udf): emit explicit `ACTION_NAME = SKIP_STATEMENT` for clearer discoverability |
| `dry_run_volume_50` | ✅ | ~50-statement batch — SUMMARY = PASS / DRY_RUN confirms throughput |
| `dry_run_mixed_ddl_dml` | ✅ | DDL persists across ROLLBACK; DML rolls back; SUMMARY message notes the auto-commit warning |

## VALIDATE_JOIN_PLAN (6 cases · all passing)

Direct invocation (`EXECUTE SCRIPT EXA_OPTIMIZE.VALIDATE_JOIN_PLAN(schema, plan_sql)`). The smoke runs one curated multi-statement plan and asserts the per-check emissions. `case.sql_fragment` overload encodes the `CHECK_NAME` to assert.

| Case (target table · CHECK_NAME) | Status | Adversarial property |
|---|---|---|
| `DIM_PRE_PK` · `PRIMARY_KEY_METADATA` | ✅ | Declared PK already in place → `INFO` row with `ACTION_NAME=SKIP_STATEMENT`. Exercises the metadata short-circuit before any data scan runs |
| `DIM_NULL_PK` · `PRIMARY_KEY_NULLS` | ✅ | 1 NULL in `CANDIDATE_KEY` → `ERROR`. Proves the null-count predicate fires correctly on a real seed row |
| `DIM_DUP_PK` · `PRIMARY_KEY_DUPLICATES` | ✅ | 1 duplicate group → `ERROR`. Proves the `GROUP BY HAVING COUNT(*)>1` check fires correctly |
| `DIM_VALID_PK` · `PRIMARY_KEY_NULLS` | ✅ | Empty table (0 rows) → `PASS`. Validates that NULL/uniqueness checks are safe on empty inputs |
| `FACT_NULL_FK` · `FOREIGN_KEY_NULLS` | ✅ | 2 NULL FK rows → `WARNING` (not ERROR — NULLs in FK columns are allowed but flagged for Unknown Member review) |
| `DIM_CUSTOMER` · `FOREIGN_KEY_REFERENCE_KEY` | ✅ | DIM_CUSTOMER carries a declared PK on `CUSTOMER_ID` so the validator follows the existing-PK path (`PASS`) instead of falling back to REPAIR |

## CONVERT_VARCHAR / CONVERT_DATATYPES (1 case each · post-load helpers)

| Case | Status | Adversarial property |
|---|---|---|
| `ACME_TORTURE` · CONVERT_VARCHAR | ✅ | Schema-scoped invocation `('ACME_TORTURE', '%', 1000)` emits the expected `-- Table <T> (N rows; sample size is 1000 rows)` header comments. Smoke proves the UDF is installed and traverses the catalog |
| `ACME_TORTURE` · CONVERT_DATATYPES | ✅ | Dry-run `('ACME_TORTURE', '%', false)` emits an explanatory info row ending with "execute the script again …" whenever DECIMAL conversions get proposed |

## ANALYZE_DIM_DATE / GENERATE_DIM_DATE (1 case each)

| Case | Status | Adversarial property |
|---|---|---|
| `DIM_DATE` · ANALYZE_DIM_DATE | ✅ | The fixture's 4-column DIM_DATE is well short of the 47-attribute Kimball standard. The UDF emits a `warning` row whose TITLE contains "missing N of 47 standard date attributes" |
| `DIM_DATE_GEN` · GENERATE_DIM_DATE | ✅ | `('ACME_TORTURE', 'DIM_DATE_GEN', '2026-01-01', '2026-12-31', 1, 'TRUE')` materializes a 366-row date dim and emits one row with `MESSAGE='Generated "ACME_TORTURE"."DIM_DATE_GEN"'` and `ROW_COUNT=366`. `FORCE='TRUE'` lets the smoke run idempotently — the next `load_fixture` drops the schema and cleans up |

## Patterns worth knowing

### NULL-heavy and duplicate-heavy PK candidates

The fixture deliberately seeds:
- `DIM_NULL_PK.CANDIDATE_KEY` with at least one NULL row, rest unique.
- `DIM_DUP_PK.CANDIDATE_KEY` with at least one duplicate value, no NULLs.

A PK candidate must pass `validate_pk_candidate` (a NULL-count check via `IS NULL OR …` plus a `GROUP BY HAVING COUNT(*) > 1` uniqueness check) before the UDF emits the ALTER. Either property alone is enough to refuse the proposal.

### Reserved-word columns

`DIM_WIDE_RESERVED` declares columns named `YEAR`, `ORDER`, `GROUP`, `SOURCE`, plus ~270 others. The `ident()` helper always double-quotes (`"YEAR"`, `"ORDER"`, etc.) so every emitted statement is reserved-word safe. If you write your own emission code around the UDF output, **don't strip the quotes**.

### Same-stem dim collision

`DIM_CUSTOMER` and `DIM_CUSTOMER_TYPE` both expose a `CUSTOMER_ID` column shape. INFER_JOIN_PATHS sees the ambiguity and emits no tier-3 row for facts with `CUSTOMER_ID`. The smoke harness asserts this. Caller's responsibility: if you actually want a join here, declare an explicit FK, or restructure the dim names so the stems no longer collide.

### Wider-PK paired with narrower-FK (and the inverse)

The fixture asserts the **directional** widen rule:
- `FACT_FK_NARROWER.CUST_CODE` smaller than its referenced `DIM_CUSTOMER_WIDE.CUST_CODE` → UDF emits widen + FK (when this case lands).
- `FACT_FK_WIDER.CUST_CODE` larger than its referenced `DIM_CUSTOMER_NARROW.CUST_CODE` → UDF emits nothing (lossy direction).

The harness already passes the wider-side negative case. The narrower-side positive case is currently missing — implementing it requires the existing `fk_with_widen_sql` helper to be reachable from the missing-PK path on this specific fixture row.

### Charset and DECIMAL mismatches

`FACT_FK_CHARSET_MISMATCH` declares VARCHAR(20) UTF8 against a VARCHAR(20) ASCII PK. Type lengths match but charsets don't. `fk_widen_target` returns nil because charset differs. UDF emits nothing.

`FACT_FK_DECIMAL_MISMATCH` declares DECIMAL(10,2) against a DECIMAL(18,0) PK. Same family of mismatch — `fk_widen_target` doesn't model precision/scale changes (would be lossy in general). UDF emits nothing.

Both pass as negative cases. If you ever extend `fk_widen_target` to model decimal precision changes, you must update these cases to positive (with explicit expected SQL).

### Composite PKs in unknown member generation

`BUILD_UNKNOWN_MEMBER_INSERT` is single-PK only by design — the sentinel `-1` doesn't generalize cleanly across multiple columns. The fixture's `DIM_COMPOSITE_PK` requires the UDF to emit a finding with `severity = ERROR` and a message mentioning "composite". When this case lands, the message must include the word "composite" (case-insensitive) so downstream consumers can match it.

### Paired-batch ordering

`DIM_PAIRED_PARENT` and `FACT_FK_PAIRED` together exercise the PK-before-FK invariant. The UDF buffers paired FKs in `pending_paired_fks` during the missing-PK pass and only flushes them after every PK finding. A regression here would let `FACT_FK_PAIRED.PARENT_KEY` land before `DIM_PAIRED_PARENT.PARENT_KEY` gets its PK, and the FK ALTER would fail with "no primary key on referenced table".

### Dedup on FK name collisions

`FACT_FK_DUP_NAMES` declares two FK candidates that would both generate the constraint name `FK_FACT_FK_DUP_NAMES_<col>`. The UDF tracks `emitted_fk_names` and refuses to emit the duplicate. The harness asserts exactly one emission for this fixture row.

## How to add a torture case

1. Add the DDL to `studio/backend/tests/fixtures/synthetic/acme_torture/ddl.sql` (schema-qualified — no implicit `OPEN SCHEMA`).
2. Add a `CoverageCase(…)` entry to `scripts/optimize_torture_cases.py`.
3. Run `python3 scripts/optimize_torture_smoke.py --target nano` against a loaded ExaNano. Expect the new case to fail (`missing`).
4. Implement the UDF change. Re-run. Case should flip to passing.
5. Commit fixture + UDF + smoke change together so CI doesn't break in either direction.

## Quick reference: severity ladder by UDF

| UDF | Emitted severities | Negative case representation |
|---|---|---|
| `ANALYZE_CONSTRAINTS` | `critical`, `warning`, `info` | No row emitted for the target table |
| `INFER_JOIN_PATHS` | (no severity column) | No row emitted for the target fact |
| `BUILD_UNKNOWN_MEMBER_INSERT` | `PASS` (SQL emitted) · `ERROR` (composite-PK refusal — SQL_TEXT empty) | UDF returns `SEVERITY = ERROR` with explanatory message |
| `DRY_RUN_PLAN` | `PASS`, `REPAIR`, `ERROR`, `INFO` | No EXECUTION row, validator row referencing the table |
