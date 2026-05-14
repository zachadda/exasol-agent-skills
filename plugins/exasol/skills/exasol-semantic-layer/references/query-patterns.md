# Querying the cube

Canonical SELECT shapes against a live SQLCube virtual schema. Reflects the live `exanano-sqlcube` adapter behavior — verified via smoke test 2026-05-13.

## Schema reference for examples

Assume the multi-domain layer `SQLCUBE_ADVENTUREWORKS` with one model `factinternetsales`:

```
MODELS:
  factinternetsales (FACT_SCHEMA=ADVENTUREWORKS, FACT_TABLE=FACTINTERNETSALES)

DIMENSIONS (selection):
  Englishproductname     → ADVENTUREWORKS.DIMPRODUCT.ENGLISHPRODUCTNAME       (alias dim)
  Color                  → ADVENTUREWORKS.DIMPRODUCT.COLOR                    (alias dim)
  Firstname              → ADVENTUREWORKS.DIMCUSTOMER.FIRSTNAME               (alias dim1)
  Englishmonthname       → ADVENTUREWORKS.DIMDATE.ENGLISHMONTHNAME            (alias dim2)
  Calendaryear           → ADVENTUREWORKS.DIMDATE.CALENDARYEAR                (alias dim2)
  Salesterritoryregion   → ADVENTUREWORKS.DIMSALESTERRITORY.SALESTERRITORYREGION (alias dim4)

MEASURES:
  Extended Amount       SUM            f.EXTENDEDAMOUNT
  Order Count           COUNT_DISTINCT f.SALESORDERNUMBER
  Gross Margin %        AVG            (f.UNITPRICE - f.TOTALPRODUCTCOST) / NULLIF(f.UNITPRICE, 0)
```

## The query shape

Cube queries are flat SELECTs against a single virtual table:

```sql
SELECT <columns_from_DIMENSIONS_or_MEASURES>
FROM "<VIRTUAL_SCHEMA>"."<lowercase_model_id>"
[WHERE <predicate>]
[GROUP BY ...]
[ORDER BY ...]
[LIMIT ...]
```

The user lists **what** they want; the adapter handles **how** (joins, aggregations, group-by).

**Critical syntax detail**: the virtual table is lowercase and lives inside the virtual schema. **Always use quoted identifiers**:

```sql
SELECT * FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales" LIMIT 1;
```

Unquoted `SELECT * FROM SQLCUBE_ADVENTUREWORKS.factinternetsales` does NOT work — Exasol uppercases the table reference and the lowercase virtual table name no longer matches.

## Pattern 1: Just sample data

```sql
SELECT * FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales" LIMIT 5;
```

Returns a **wide denormalized** row: all visible DIMENSIONS columns + all visible MEASURES, with measures aggregated over the natural fact-table grain (no group-by means one row per fact-table grain, which is usually overkill — LIMIT first).

Use this to confirm the cube is wired correctly before issuing real queries.

## Pattern 2: KPI — measure with no group-by

```sql
SELECT
  "Extended Amount" AS revenue,
  "Order Count" AS orders
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales";
```

Returns one row. The adapter sees no DIMENSIONS columns in the SELECT, so it skips GROUP BY entirely and aggregates over the whole fact.

Underlying SQL (adapter-generated, schematically):

```sql
SELECT SUM(f.EXTENDEDAMOUNT)            AS "Extended Amount",
       COUNT(DISTINCT f.SALESORDERNUMBER) AS "Order Count"
FROM ADVENTUREWORKS.FACTINTERNETSALES f;
```

No dims, no joins. Cheapest query you can write.

## Pattern 3: Group by one dimension

```sql
SELECT "Englishproductname", "Extended Amount"
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
GROUP BY "Englishproductname"
ORDER BY "Extended Amount" DESC
LIMIT 10;
```

The adapter sees one DIMENSIONS column and one MEASURE. It generates:

```sql
SELECT dim.ENGLISHPRODUCTNAME                          AS "Englishproductname",
       SUM(f.EXTENDEDAMOUNT)                           AS "Extended Amount"
FROM ADVENTUREWORKS.FACTINTERNETSALES f
LEFT JOIN ADVENTUREWORKS.DIMPRODUCT dim ON f.PRODUCTKEY = dim.PRODUCTKEY
GROUP BY dim.ENGLISHPRODUCTNAME
ORDER BY "Extended Amount" DESC
LIMIT 10;
```

The dim alias (`dim`) comes from the DIMENSIONS row. The JOIN line comes from the JOINS row with `DIM_ALIAS='dim'`.

## Pattern 4: Multi-dimensional group

```sql
SELECT "Salesterritoryregion", "Calendaryear", "Extended Amount", "Order Count"
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
GROUP BY "Salesterritoryregion", "Calendaryear"
ORDER BY "Salesterritoryregion", "Calendaryear";
```

The adapter walks dims and measures, infers the join graph from JOINS rows, emits the SQL. The user never writes joins.

## Pattern 5: Filtered KPI

```sql
SELECT "Extended Amount"
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
WHERE "Calendaryear" = 2013;
```

Predicates on dimension columns push through to the joined dim table:

```sql
SELECT SUM(f.EXTENDEDAMOUNT) AS "Extended Amount"
FROM ADVENTUREWORKS.FACTINTERNETSALES f
LEFT JOIN ADVENTUREWORKS.DIMDATE dim2 ON f.ORDERDATEKEY = dim2.DATEKEY
WHERE dim2.CALENDARYEAR = 2013;
```

## Pattern 6: Top-N by measure

```sql
SELECT "Firstname", "Lastname", "Extended Amount"
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
GROUP BY "Firstname", "Lastname"
ORDER BY "Extended Amount" DESC
LIMIT 10;
```

Standard. ORDER BY + LIMIT push down — the adapter doesn't materialize all rows.

## Pattern 7: Derived measure

If `DERIVED_MEASURES` has `"Gross Profit"` with formula `"Extended Amount" - "Total Product Cost"`:

```sql
SELECT "Englishproductname", "Gross Profit"
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
GROUP BY "Englishproductname"
ORDER BY "Gross Profit" DESC
LIMIT 10;
```

Adapter resolves the formula by expanding to the underlying base measures:

```sql
SELECT dim.ENGLISHPRODUCTNAME,
       SUM(f.EXTENDEDAMOUNT) - SUM(f.TOTALPRODUCTCOST) AS "Gross Profit"
FROM ...
```

## Pattern 8: Role-playing date

When two JOINS rows reference the same dim (different aliases), DIMENSIONS rows distinguish via the `DIM_TABLE_ALIAS`:

If DIMENSIONS has:
- `Orderdate Year` → `DIMDATE.CALENDARYEAR` via alias `dim2`
- `Shipdate Year` → `DIMDATE.CALENDARYEAR` via alias `dim3`

```sql
SELECT "Orderdate Year", "Shipdate Year", "Order Count"
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
GROUP BY 1, 2
ORDER BY 1, 2;
```

Adapter emits two LEFT JOINs to DIMDATE with different aliases (`dim2`, `dim3`) and different ON clauses (`ORDERDATEKEY` vs `SHIPDATEKEY`).

If the registry doesn't have distinct DIM_ALIAS rows for the two roles, role-playing breaks silently — query returns wrong numbers (or zero rows, if the aliases collide).

## What the adapter does NOT support (v0.1)

| Pattern | Why | Workaround |
|---|---|---|
| Subqueries in SELECT that reference virtual columns | Adapter parses top-level only | Wrap in CTE, fully resolve in inner SELECT |
| Window functions over measures | Measures are aggregates already; window over aggregate needs explicit grain | CTE wrap, then RANK/ROW_NUMBER over the materialized view |
| Cross-product joins between two virtual tables | Adapter mounts each model independently; no inter-model join inference yet | UNION on shared dims, then aggregate |
| INSERT / UPDATE / DELETE through the virtual schema | Read-only by design | Modify the underlying physical tables |
| `EXPLAIN PLAN` returning underlying SQL | Adapter doesn't expose the generated SQL via EXPLAIN | Read adapter logs (LOG_LEVEL='DEBUG' in WITH clause not exposed by current adapter; use Lua logs on Exasol cluster) |

Workaround for window-over-aggregate (rank top customers by revenue):

```sql
WITH rev AS (
  SELECT "Firstname" || ' ' || "Lastname" AS customer,
         "Extended Amount" AS revenue
  FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
  GROUP BY "Firstname", "Lastname"
)
SELECT customer, revenue,
       RANK() OVER (ORDER BY revenue DESC) AS rk
FROM rev
ORDER BY rk
LIMIT 10;
```

## Verifying a fresh cube

After CREATE VIRTUAL SCHEMA + registry writes, run this validation sequence:

```sql
-- 1. List the virtual tables in the schema
SELECT TABLE_NAME, TABLE_OBJECT_ID
FROM SYS.EXA_ALL_TABLES
WHERE TABLE_SCHEMA = 'SQLCUBE_<LAYER>';

-- 2. Describe the virtual table
OPEN SCHEMA "SQLCUBE_<LAYER>";
DESCRIBE "<lowercase_model_id>";

-- 3. Sample
SELECT * FROM "<lowercase_model_id>" LIMIT 3;

-- 4. KPI shape
SELECT "<MEASURE_1>", "<MEASURE_2>" FROM "<lowercase_model_id>";

-- 5. Group-by shape
SELECT "<DIM_COL>", "<MEASURE_1>"
FROM "<lowercase_model_id>"
GROUP BY 1
ORDER BY 2 DESC
LIMIT 5;
```

All five pass → cube is fully wired.

If (1) fails (table list empty) → CREATE VIRTUAL SCHEMA didn't pick up the MODEL_IDS. Check the WITH clause matches MODELS rows.

If (2) fails (DESCRIBE errors) → adapter's `set_capabilities` failed on first reach. Check that all DIMENSIONS rows reference DIM_ALIAS values that match JOINS rows.

If (3) fails (SELECT errors) → adapter generated SQL that doesn't run. Most commonly: PHYSICAL_EXPR references a column that doesn't exist. See `troubleshooting.md` §3.

If (4) passes but (5) fails → JOINS row is broken (wrong DIM_KEY or FACT_FK).

## Performance notes

The adapter doesn't cache. Every query against the virtual schema reads the registry (cheap — KBs) and emits a physical SQL (expensive, depends on data size).

For dashboards firing multiple panel queries against the same cube: prefer **one wide query** returning all needed measures and dims, GROUP BY everything at once. The adapter is smart enough to fuse measures into a single SELECT — issuing N separate single-measure queries pays N round-trips.

```sql
-- One query, multiple measures + dims:
SELECT "Englishproductname", "Calendaryear",
       "Extended Amount", "Order Count", "Gross Margin %"
FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales"
GROUP BY "Englishproductname", "Calendaryear"
ORDER BY "Englishproductname", "Calendaryear";

-- vs three separate queries — same data, three round-trips, three rewrites.
```

## Casing in identifiers — common pitfalls

The adapter is strict about identifier casing because Exasol is strict:

| Reference | Casing in the registry | Casing in queries |
|---|---|---|
| Virtual schema name | `SQLCUBE_<LAYER_UPPER>` | Quoted or unquoted; Exasol will uppercase if unquoted |
| Virtual table name | lowercase model_id (e.g., `factinternetsales`) | **MUST be quoted lowercase** |
| Virtual column name | as stored in DIMENSIONS.VIRTUAL_COL | **MUST be quoted in mixed case**; Exasol uppercases unquoted |
| Physical schema/table/column in registry | uppercase (Exasol native) | n/a (adapter generates) |

Always quote virtual-table and virtual-column names in SQL emitted by the agent.

## When the cube returns surprising rows

If a query returns way more rows than expected, the most common cause is **DIMENSIONS pointing at a dim that's not joined** — DIM_TABLE_ALIAS doesn't match any JOINS row. The adapter falls back to a CROSS JOIN in that case (or in some versions, returns NULL for the dim column with full fact-table count). Check JOINS rows for the model.

If query returns FEWER rows than expected, the most common cause is **JOIN_TYPE=INNER on a JOINS row** — unmatched fact rows are dropped. Default to LEFT unless you have a specific reason.

Both diagnostics surface as the same SQL the adapter generated. Get that SQL by enabling adapter logging on the cluster (currently only available via direct Lua log inspection — see `troubleshooting.md` §2).
