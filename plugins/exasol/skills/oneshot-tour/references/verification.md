# Phase 4 — Verification

After Phase 3 finishes the six-step execution, the orchestrator must independently verify that the cube is live, queryable, and returning real data — not just trust the skills' success exits. This is the contract gate before handing off to the user.

## Goal

Prove `cube_live` actually holds by running a deterministic sample query against the virtual schema. If the query fails or returns zero rows, halt with a clear diagnosis.

## What it does

1. Confirm the virtual schema exists in the catalog.
2. Pick a representative model + measure from the cube and run one SELECT.
3. Confirm the result is non-empty.
4. Capture the result for the dashboard render in Phase 5.

## Verification SQL pattern

The orchestrator generates a query from the cube metadata in `SQLCUBE_REGISTRY`, not the source schema. Registry shape (canonical: `studio/backend/services/sqlcube_builder.py:build_registry_ddl()`):

- `SQLCUBE_REGISTRY.MODELS(MODEL_ID, DOMAIN_ID, MODEL_LABEL, FACT_SCHEMA, FACT_TABLE, ...)`
- `SQLCUBE_REGISTRY.MEASURES(MEASURE_ID, MODEL_ID, VIRTUAL_COL, AGG_TYPE, PHYSICAL_EXPR, DATA_TYPE, ...)`
- `SQLCUBE_REGISTRY.DIMENSIONS(DIM_ID, MODEL_ID, VIRTUAL_COL, PHYSICAL_TABLE, PHYSICAL_COL, DATA_TYPE, ...)`
- `SQLCUBE_REGISTRY.JOINS(JOIN_ID, MODEL_ID, DIM_TABLE, DIM_KEY, FACT_FK, JOIN_TYPE, JOIN_ORDER)`

Virtual-schema table names = `MODEL_ID` (case-sensitive, quoted). Virtual-schema column names = `VIRTUAL_COL` (display labels with spaces, quoted).

```sql
-- 1. Confirm virtual schema exists
SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = :target_cube_name;

-- 2. Pick the first model whose FACT_SCHEMA matches the source we imported
SELECT MODEL_ID, MODEL_LABEL, FACT_TABLE
FROM SQLCUBE_REGISTRY.MODELS
WHERE FACT_SCHEMA = :source_schema
  AND IS_ACTIVE = TRUE
ORDER BY MODEL_ID
LIMIT 1;

-- 3. Pull one numeric measure on that model.
--    IS_VISIBLE = TRUE is required — the SQLCube adapter only projects visible
--    columns onto the virtual schema. Registry rows with IS_VISIBLE=FALSE exist
--    but won't appear in the virtual schema and a query against them errors
--    `object <col> not found`.
SELECT VIRTUAL_COL, AGG_TYPE
FROM SQLCUBE_REGISTRY.MEASURES
WHERE MODEL_ID = :model_id
  AND IS_VISIBLE = TRUE
  AND AGG_TYPE IN ('SUM', 'COUNT', 'COUNT_DISTINCT', 'AVG')
ORDER BY MEASURE_ID
LIMIT 1;

-- 4. Pull one date-shaped visible dim on that model.
SELECT VIRTUAL_COL
FROM SQLCUBE_REGISTRY.DIMENSIONS
WHERE MODEL_ID = :model_id
  AND IS_VISIBLE = TRUE
  AND (UPPER(DATA_TYPE) LIKE 'DATE%' OR UPPER(DATA_TYPE) LIKE 'TIMESTAMP%')
ORDER BY DIM_ID
LIMIT 1;

-- 5. Compose + run sample query.
--    Both `MODEL_ID` and `VIRTUAL_COL` are case-preserving — quote them.
--    Quote the result aliases too: `value` is a reserved keyword in Exasol
--    (ISO SQL), so `AS value` parses as a syntax error. Use `AS "value"` or a
--    non-reserved alias like `AS "amount"` / `AS "bucket"`.
SELECT
  "<dim_virtual_col>"                   AS "bucket",
  <agg_type>("<measure_virtual_col>")   AS "amount"
FROM "<target_cube_name>"."<model_id>"
GROUP BY 1
ORDER BY 1
LIMIT 10;
```

## What "live" actually means

`cube_live` requires ALL of:

- Virtual schema row exists in `SYS.EXA_VIRTUAL_SCHEMAS`
- At least one `MODEL` row in `SQLCUBE_REGISTRY.MODELS` whose `FACT_SCHEMA` matches the source
- The sample SELECT executes without error
- The sample SELECT returns at least one row

If any of those fails, the cube is broken even if the skills all returned success.

## Narration

```
agent:
  ### Step 6 of 6 — Verify + render dashboard

  Verifying cube `SQLCUBE_ADVENTUREWORKS`...

    tool: SYS.EXA_VIRTUAL_SCHEMAS lookup ✓
    tool: SQLCUBE_REGISTRY.MODELS for FACT_SCHEMA=ADVENTUREWORKS → internet_sales (FACTINTERNETSALES)
    tool: SQLCUBE_REGISTRY.MEASURES → "Sales Amount" (SUM)
    tool: SQLCUBE_REGISTRY.DIMENSIONS → "Calendar Year" (DECIMAL)
    tool: Sample query:

      SELECT "Calendar Year" AS "bucket", SUM("Sales Amount") AS "amount"
      FROM "SQLCUBE_ADVENTUREWORKS"."internet_sales"
      GROUP BY 1 ORDER BY 1 LIMIT 10;

  ┌──────────────────┬──────────────────────────┐
  │  Calendar Year   │  Sales Amount            │
  ├──────────────────┼──────────────────────────┤
  │  2010            │            43,421.04     │
  │  2011            │         7,075,439.06     │
  │  2012            │         5,842,485.14     │
  │  2013            │        16,351,176.85     │
  └──────────────────┴──────────────────────────┘

  ✓ Cube is live and returning data.
```

## Capture for Phase 5

Phase 5 (dashboard) needs:

- The verification query SQL text
- The verification query result rows (column names + row tuples)
- The model_id, model_label, and the (dim, measure) pair used

Stash these in `tour_state.verification_sample` so Phase 5 doesn't re-query unnecessarily.

```yaml
tour_state:
  verification_sample:
    model_id:        "internet_sales"
    model_label:     "Internet Sales"
    dim_virtual_col: "Calendar Year"
    measure_virtual_col: "Sales Amount"
    agg_type:        SUM
    sql:             "SELECT \"Calendar Year\" AS \"bucket\", SUM(\"Sales Amount\") AS \"amount\" ..."
    columns:         ["bucket", "amount"]
    rows:
      - [2010,       43421.04]
      - [2011,    7075439.06]
      - [2012,    5842485.14]
      - [2013,   16351176.85]
```

## Halt conditions

| Failure | User message |
|---|---|
| `SYS.EXA_VIRTUAL_SCHEMAS` returns no row | "Virtual schema didn't materialize. Cube create reported success but catalog says otherwise. Re-run step 5 in step_by_step mode." |
| `SQLCUBE_REGISTRY.MODELS` empty for FACT_SCHEMA | "Virtual schema is live but no MODEL was registered. Semantic-layer skill misbehaved." |
| `SQLCUBE_REGISTRY.MEASURES` has rows but none with `IS_VISIBLE = TRUE` | "Model `<id>` has measures registered but all are hidden. Re-run step 5 with `llm_enrichment: true`, or update IS_VISIBLE on at least one measure." |
| `SQLCUBE_REGISTRY.DIMENSIONS` has rows but none with `IS_VISIBLE = TRUE` | Same as above for dims. Pick `COUNT(*)` as the measure and report "no visible dims; verified at fact-row-count level only". |
| Sample query fails to compile | Print the actual error verbatim; suggest checking adapter logs (`SQLCUBE.sqlcube_adapter` script). Common cause: unquoted reserved-keyword alias (e.g. `AS value` instead of `AS "value"`). |
| Sample query returns 0 rows | "Cube is live but query returned no rows. Possible model error: dim/measure don't connect to facts. Inspect `SQLCUBE_REGISTRY.JOINS` for the model." |

On halt, Phase 5 does NOT run. Tour ends without a dashboard.

## Edge cases

| Situation | Action |
|---|---|
| Cube has multiple models for the source | Use the first one (lowest `MODEL_ID`). User can inspect others post-tour |
| No numeric measure exists | Pick `COUNT(*)` and an arbitrary dim |
| No date dim exists | Pick first non-numeric dim by `DIM_ID` |
| Verification query is fast but returns suspicious data (e.g., all nulls) | Surface a warning but don't halt — let user catch this on the dashboard |
