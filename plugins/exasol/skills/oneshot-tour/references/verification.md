# Phase 4 — Verification

After Phase 3 finishes the six-step execution, the orchestrator must independently verify that the cube is live, queryable, and returning real data — not just trust the skills' success exits. This is the contract gate before handing off to the user.

## Goal

Prove `cube_live` actually holds by running a deterministic sample query against the virtual schema. If the query fails or returns zero rows, halt with a clear diagnosis.

## What it does

1. Confirm the virtual schema exists in the catalog.
2. Pick a representative fact + measure from the cube and run one SELECT.
3. Confirm the result is non-empty.
4. Capture the result for the dashboard render in Phase 5.

## Verification SQL pattern

The orchestrator generates a query from the cube metadata, not the source schema. Use SQLCUBE_META to discover what's queryable:

```sql
-- 1. Confirm virtual schema exists
SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = :target_cube_name;

-- 2. Pull the first model from the cube
SELECT MODEL_NAME, FACT_TABLE FROM SQLCUBE_META.MODELS
WHERE TARGET_SCHEMA = :target_cube_name
ORDER BY MODEL_ID
LIMIT 1;

-- 3. Pull one numeric measure + one date-shaped dim attribute on that model
SELECT MEASURE_NAME FROM SQLCUBE_META.MEASURES
WHERE MODEL_NAME = :model_name AND AGG_FN IN ('SUM', 'COUNT', 'AVG')
ORDER BY MEASURE_ID
LIMIT 1;

SELECT DIM_ATTR_NAME FROM SQLCUBE_META.DIMENSION_ATTRIBUTES
WHERE MODEL_NAME = :model_name AND DATA_KIND = 'DATE'
ORDER BY DIM_ATTR_ID
LIMIT 1;

-- 4. Compose + run sample query
SELECT
  "<dim_attr>" AS bucket,
  <agg_fn>("<measure>") AS value
FROM "<target_cube_name>"."<model_name>"
GROUP BY 1
ORDER BY 1
LIMIT 10;
```

## What "live" actually means

`cube_live` requires ALL of:

- Virtual schema row exists in `SYS.EXA_VIRTUAL_SCHEMAS`
- At least one MODEL row in `SQLCUBE_META.MODELS` for the target schema
- The sample SELECT executes without error
- The sample SELECT returns at least one row

If any of those fails, the cube is broken even if the skills all returned success.

## Narration

```
agent:
  ### Step 6 of 6 — Verify + render dashboard

  Verifying cube `CUBE_ACME_ADVENTUREWORKS`...

    tool: SYS.EXA_VIRTUAL_SCHEMAS lookup ✓
    tool: SQLCUBE_META.MODELS for target → "Internet Sales" (FACT_INTERNET_SALES)
    tool: Sample query — first numeric measure × first date dim:

      SELECT "Order Year", SUM("Internet Sales Amount") AS "Internet Sales"
      FROM "CUBE_ACME_ADVENTUREWORKS"."Internet Sales"
      GROUP BY 1 ORDER BY 1 LIMIT 10;

  ┌──────────────┬──────────────────────────┐
  │  Order Year  │  Internet Sales          │
  ├──────────────┼──────────────────────────┤
  │  2010        │            43,421.04     │
  │  2011        │         7,075,439.06     │
  │  2012        │         5,842,485.14     │
  │  2013        │        16,351,176.85     │
  └──────────────┴──────────────────────────┘

  ✓ Cube is live and returning data.
```

## Capture for Phase 5

Phase 5 (dashboard) needs:

- The verification query SQL text
- The verification query result rows (column names + row tuples)
- The model name and the (dim, measure) pair used

Stash these in `tour_state.verification_sample` so Phase 5 doesn't re-query unnecessarily.

```yaml
tour_state:
  verification_sample:
    model_name:      "Internet Sales"
    dim_attr:        "Order Year"
    measure_name:    "Internet Sales Amount"
    agg_fn:          SUM
    sql:             "SELECT \"Order Year\", SUM(\"Internet Sales Amount\") ..."
    columns:         ["Order Year", "Internet Sales"]
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
| `SQLCUBE_META.MODELS` empty for target | "Virtual schema is live but no MODEL was registered. Semantic-layer skill misbehaved." |
| Sample query fails to compile | Print the actual error verbatim; suggest checking adapter logs |
| Sample query returns 0 rows | "Cube is live but query returned no rows. Possible model error: dim/measure names don't match facts. Inspect SQLCUBE_META.JOINS." |

On halt, Phase 5 does NOT run. Tour ends without a dashboard.

## Edge cases

| Situation | Action |
|---|---|
| Cube has multiple models | Use the first one (lowest MODEL_ID). User can inspect others post-tour |
| No numeric measure exists | Pick `COUNT(*)` and an arbitrary dim attribute |
| No date dim exists | Pick first text-typed dim attribute |
| Verification query is fast but returns suspicious data (e.g., all nulls) | Surface a warning but don't halt — let user catch this on the dashboard |
