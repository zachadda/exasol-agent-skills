# INFER_JOIN_PATHS — catalog-only fact/dim discovery and join emission

`INFER_JOIN_PATHS(SCHEMA_NAME)` is the empty-schema profiler. Reads the catalog only — no data scans, no joins, no row counts. Classifies every table (fact / dimension / unknown), then emits join rows from three tiers in priority order. Designed to answer "what would the cube look like if I built one right now?" without burning any compute.

## When to invoke

- Just loaded a schema and want a candidate star model before touching data.
- Schema has zero declared FKs and ANALYZE_CONSTRAINTS' SQL is too aggressive for the situation.
- Building a cube-generator preview UI that needs a fast, deterministic answer.
- Comparing pre-optimize and post-optimize join inferences (run before, run after).

## Call

```sql
EXECUTE SCRIPT EXA_OPTIMIZE.INFER_JOIN_PATHS('INVENTORY');
```

The script `RETURNS TABLE`; the rows arrive as a result set. Wrapping in `SELECT * FROM (EXECUTE SCRIPT …)` is rejected by the parser.

## Returned columns

| Column | Meaning |
|---|---|
| `FACT_TABLE` | The fact side of the inferred join |
| `FACT_KEY` | The FK column on the fact |
| `DIM_TABLE` | The dimension side |
| `DIM_KEY` | The PK column (or best-guess key column) on the dim |
| `INFERENCE_SOURCE` | `declared_fk` (tier 1) · `stem_match` (tier 2) · `same_name_fallback` (tier 3) |

No `SEVERITY`. No `SQL_TEXT`. Caller decides what to do with the rows — typically feed them straight into a cube DDL generator or a join-graph visualization.

## Classification (the `classify_table` heuristic)

For every table not matched by name (`FACT_*`, `FT_*`, `FCT_*`, `F_*` → fact; `DIM_*`, `D_*`, `LU_*`, `LKP_*`, `REF_*` → dimension), the UDF inspects column shape:

- Count declared FK columns.
- Count declared PK columns.
- Count "numeric non-key" columns (DECIMAL, INTEGER, FLOAT, DOUBLE, NUMERIC, NUMBER, excluding the PK and any declared FK).
- Count "text non-key" columns (VARCHAR, CHAR, CLOB, excluding PK and FKs).

Decision tree:

1. `fk_count > 0` and `numeric_non_key == 0` and `text_non_key > 0` → **dimension** (a junction-like dim with only text attributes and FKs back to other dims).
2. `numeric_non_key > 2` → **fact** (measures dominate).
3. `pk_count == 1` and `fk_count == 0` and `text_non_key > 0` → **dimension** (single-PK, attribute-only).
4. Otherwise → **unknown** (no emission for this table).

The classifier is conservative on purpose. It deliberately under-recalls in ambiguous cases — wrongly classifying a dim as a fact emits noise into the join graph, while leaving a borderline table unclassified just means it doesn't show up in the cube preview.

## Tier 1 — declared FKs

Read every row in `SYS.EXA_ALL_CONSTRAINT_COLUMNS` with `CONSTRAINT_TYPE = 'FOREIGN KEY'` and emit `INFERENCE_SOURCE = 'declared_fk'`. Always wins over the other tiers for the same `(fact, fact_key, dim, dim_key)` quad (deduped via `seen` set).

## Tier 2 — stem match

For each fact table:
- Build a dim lookup keyed by `strip_dim_prefix(dim_table)` (strips `DIM_`, `DIM`, `D_`, `LU_`, `LOOKUP_`, `LKP_`, `REF_`).
- Resolve each dim's PK column: declared PK first, then by stem-matched column, then first `_KEY`/`_ID`/`KEY`/`ID` tail.
- Also register `stem:gsub('_', '')` compact variant so `DIM_CUSTOMER_TYPE` matches `CUSTOMERTYPE_KEY`.

For each fact column whose `strip_key_suffix` is non-empty, look up that stem in the dim index. Emit `stem_match` rows.

## Tier 3 — same-name fallback

Only runs for facts that emitted **zero** tier-2 rows. Prevents rich schemas with declared FKs from picking up a spurious tail of weaker matches.

For each remaining fact column with a non-empty stripped stem:
- Look up the column name (verbatim, upper-cased) in `dim_by_col`, a reverse index of every dim's key-shaped columns.
- Emit only when **exactly one** dim exposes that column name. Ambiguous matches (≥ 2 dims) are dropped silently.

This is what catches schemas where every dim shares the same `CUSTOMER_ID` column and there's no `DIM_CUSTOMER` table at all — `FACT_TIER3_FALLBACK.CUSTOMER_ID → DIM_<exactly_one>.CUSTOMER_ID`.

## What it deliberately doesn't do

- **No data scans.** No row-count check, no subset-of-values check, no NULL count. Catalog-only. If you need value verification, use `ANALYZE_CONSTRAINTS`.
- **No FK ALTER generation.** The output is a join graph, not a migration plan.
- **No role-playing date detection.** That logic lives in `ANALYZE_CONSTRAINTS`. `INFER_JOIN_PATHS` will see all three date FKs only if they're declared or named with stems matching a date dim.
- **No view content inspection.** Views are read from `SYS.EXA_ALL_VIEWS` but only their column shape — the query is never parsed. A view re-exposing a dim's key column will join-emit only if the column name still matches the stem rules (and the smoke fixture `VW_DIM_SHADOW` asserts this doesn't inflate the count).

## Reading the output

The expected output for a normal star is:
- One `declared_fk` row per existing FK.
- One `stem_match` row per fact column ending in `_KEY` whose stem matches a `DIM_*` table.
- Zero or a handful of `same_name_fallback` rows for facts whose key columns aren't stem-shaped (`SHIP_TO_ZIP`, etc.).

If the output is empty, classification failed for every candidate. Common cause: no FACT_ prefix and column counts didn't trip rule (2). Re-name a table to `FACT_*` or add measures and re-run.

If the output is much larger than expected, check for:
- A misnamed dim that's classifying as a fact (e.g. `DIM_TRANSACTION_HISTORY` with three integer FKs).
- A junk attribute dim (`DIM_FLAGS`) with stem-shaped column names matching every fact.

## Pairing with ANALYZE_CONSTRAINTS

Recommended order for a fresh schema:
1. `INFER_JOIN_PATHS` → fast preview, decide whether the model looks right.
2. If yes → run `ANALYZE_CONSTRAINTS` to get the PK/FK ALTERs.
3. Apply via `DRY_RUN_PLAN` for the safety net.
4. Re-run `INFER_JOIN_PATHS` → expect tier-1 (`declared_fk`) for everything previously emitted as tier-2/3.

See also `smoke-fact-detection.md` for the deeper dive on how `classify_table` makes its calls and what to do when it gets one wrong.
