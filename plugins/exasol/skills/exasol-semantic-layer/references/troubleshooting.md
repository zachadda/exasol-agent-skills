# Troubleshooting cubes

Failure modes during creation, refresh, and query against a SQLCube virtual schema. Ordered by what the agent will likely encounter first.

## Decision tree

```
CREATE VIRTUAL SCHEMA fails              → §1
REFRESH VIRTUAL SCHEMA fails             → §2
SELECT * FROM cube.X works, but…
  …a measure errors                      → §3
  …a join returns no rows / cross-product → §4
  …role-playing date returns garbage     → §5
LLM enrichment produces nonsense          → §6
MCP doesn't see the cube                  → §7
```

## §1 CREATE VIRTUAL SCHEMA fails

Most common cold-start errors and their causes.

### "Model not registered: <CUBE_NAME>"

**Cause:** SQLCUBE_META.MODELS row not inserted before the CREATE statement.

**Fix:** INSERT the MODELS row first. The adapter's `set_capabilities` callback runs on CREATE and reads MODELS to bootstrap; without it there's nothing to mount.

### "Source schema not found: <SOURCE_SCHEMA>"

**Cause:** Either the schema doesn't exist, or `MODELS.SOURCE_SCHEMA` value doesn't match what was passed in the CREATE WITH clause.

**Fix:**
```sql
SELECT * FROM SYS.EXA_ALL_SCHEMAS WHERE SCHEMA_NAME = '<SOURCE_SCHEMA>';
SELECT SOURCE_SCHEMA FROM SQLCUBE_META.MODELS WHERE MODEL_NAME = '<CUBE_NAME>';
```

If schema missing → migrate hasn't run, or it ran to a different target. If values disagree → UPDATE MODELS row before retry.

### "Dimension table not found: DIM_X"

**Cause:** SQLCUBE_META.DIMENSIONS references a `PHYSICAL_TABLE` that doesn't exist in `SOURCE_SCHEMA`. Often: typo in INSERT, or table was DROPped between META writes and CREATE.

**Fix:**
```sql
SELECT DIM_NAME, PHYSICAL_TABLE FROM SQLCUBE_META.DIMENSIONS WHERE MODEL_NAME='<CUBE_NAME>';
SELECT TABLE_NAME FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA='<SOURCE_SCHEMA>';
```

Reconcile and re-INSERT.

### "Primary key column not found: <PK_COLUMN> on <DIM_TABLE>"

**Cause:** DIMENSIONS.PK_COLUMN refers to a column that doesn't exist on the table. Often: optimize discovered the wrong PK, or someone manually edited DIMENSIONS.

**Fix:** Verify against the constraints catalog:
```sql
SELECT COLUMN_NAME
FROM SYS.EXA_ALL_CONSTRAINT_COLUMNS
WHERE CONSTRAINT_SCHEMA='<SOURCE_SCHEMA>'
  AND CONSTRAINT_TABLE='<DIM_TABLE>'
  AND CONSTRAINT_TYPE='PRIMARY KEY';
```

If 0 rows → PK was never declared (re-run exasol-optimize or ALTER TABLE manually). Fix the PK_COLUMN value, retry.

### "Adapter UDF not callable: SQLCUBE.ADAPTER"

**Cause:** Either the UDF isn't installed (precondition `sqlcube_installed` failed), or its language container isn't loaded.

**Fix:**
```sql
SELECT * FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_NAME='ADAPTER' AND SCRIPT_SCHEMA='SQLCUBE';
```

If empty → install via `references/install.md` (or factory-foundation's deployment script).

If present but errors at call time → check Exasol's UDF runtime: probably out of memory or the Lua container is corrupt. `docker restart exanano-sqlcube` and retry.

## §2 REFRESH VIRTUAL SCHEMA fails

`REFRESH VIRTUAL SCHEMA <CUBE_NAME>` rebuilds the adapter's internal catalog. Failures here are typically:

### Newly-added measure references a column that doesn't exist

**Cause:** SQLCUBE_META.MEASURES.EXPRESSION points at a column not on the fact (or referenced dimension).

**Fix:** Manual validation query:
```sql
-- Approximate: list measures and check expressions actually parse
SELECT FACT_NAME, MEASURE_NAME, EXPRESSION
FROM SQLCUBE_META.MEASURES
WHERE MODEL_NAME='<CUBE_NAME>';
```

For each, try:
```sql
SELECT <EXPRESSION> FROM <SOURCE_SCHEMA>.<PHYSICAL_FACT> LIMIT 0;
```

Any expression that errors → broken. DELETE the row from MEASURES, REFRESH again.

### Newly-added relationship has incompatible PK/FK types

**Cause:** `FACT_FK_COLUMN` and `DIM_PK_COLUMN` exist but their data types don't match (e.g., VARCHAR vs DECIMAL after a type narrowing run).

**Fix:** Cast in the source data or restore the type via optimize re-run. Adapter won't auto-cast — joins on mismatched types are silently wrong.

```sql
SELECT
  (SELECT DATA_TYPE FROM SYS.EXA_ALL_COLUMNS
   WHERE COLUMN_TABLE='<FACT>' AND COLUMN_NAME='<FK>') AS fk_type,
  (SELECT DATA_TYPE FROM SYS.EXA_ALL_COLUMNS
   WHERE COLUMN_TABLE='<DIM>' AND COLUMN_NAME='<PK>') AS pk_type;
```

## §3 Query against cube errors on a measure

### "Function SUM does not exist for column X" — or similar parse errors

**Cause:** Measure EXPRESSION is non-numeric (SUM on VARCHAR), or AGGREGATION doesn't match expression shape.

**Fix:** Inspect:
```sql
SELECT MEASURE_NAME, EXPRESSION, AGGREGATION
FROM SQLCUBE_META.MEASURES
WHERE MODEL_NAME='<CUBE_NAME>' AND FACT_NAME='<FACT>' AND MEASURE_NAME='<NAME>';
```

If `AGGREGATION='SUM'` and the EXPRESSION is `CONCAT(FIRST_NAME, LAST_NAME)` → broken. Update or DELETE the row, REFRESH.

### Divide-by-zero in measure

**Cause:** Expression is a ratio without NULLIF / ZEROIFNULL defense.

**Fix:** UPDATE EXPRESSION to wrap the divisor:
```sql
UPDATE SQLCUBE_META.MEASURES
SET EXPRESSION = '(REVENUE - COST) / NULLIF(REVENUE, 0)'
WHERE MODEL_NAME='<CUBE_NAME>' AND MEASURE_NAME='Margin Rate';
```

Then REFRESH.

## §4 Joins produce wrong rows

### Cross-product (NxM rows where you expected N)

**Cause:** RELATIONSHIPS row missing or `FACT_FK_COLUMN` / `DIM_PK_COLUMN` swapped.

**Fix:** Check the relationships graph for the cube:
```sql
SELECT RELATIONSHIP_NAME, FACT_NAME, DIM_NAME, FACT_FK_COLUMN, DIM_PK_COLUMN, ROLE_PLAYING_AS
FROM SQLCUBE_META.RELATIONSHIPS
WHERE MODEL_NAME='<CUBE_NAME>'
ORDER BY FACT_NAME, DIM_NAME;
```

Compare against the optimize-discovered FK graph:
```sql
SELECT CONSTRAINT_TABLE AS fact, CONSTRAINT_COLUMN AS fk_col,
       REFERENCED_TABLE AS dim, REFERENCED_COLUMN AS pk_col
FROM SYS.EXA_ALL_CONSTRAINT_COLUMNS
WHERE CONSTRAINT_TYPE='FOREIGN KEY' AND CONSTRAINT_SCHEMA='<SOURCE_SCHEMA>';
```

If a known FK doesn't appear in RELATIONSHIPS → INSERT it and REFRESH.

### Zero rows when you expected some

**Cause:** Join column values don't actually overlap (e.g., FK was declared but data was loaded with empty/null values). Or the IS_HIDDEN flag is masking the rows.

**Fix:** Check raw overlap:
```sql
SELECT COUNT(*) FROM <SOURCE>.<FACT>
WHERE <FK_COL> IN (SELECT <PK_COL> FROM <SOURCE>.<DIM>);
```

If 0 → data problem, not cube problem. Defer to migrate / optimize logs.

## §5 Role-playing date returns garbage

### Both roles return the same data

**Cause:** RELATIONSHIPS has two rows pointing at DIM_DATE but `ROLE_PLAYING_AS` is null on one or both. Adapter can't disambiguate, picks first match.

**Fix:**
```sql
SELECT RELATIONSHIP_NAME, FACT_FK_COLUMN, DIM_PK_COLUMN, ROLE_PLAYING_AS
FROM SQLCUBE_META.RELATIONSHIPS
WHERE MODEL_NAME='<CUBE_NAME>' AND DIM_NAME='Date';
```

Each row should have a distinct `ROLE_PLAYING_AS` (`OrderDate`, `ShipDate`, etc.). If null → UPDATE:
```sql
UPDATE SQLCUBE_META.RELATIONSHIPS
SET ROLE_PLAYING_AS = 'OrderDate'
WHERE MODEL_NAME='<CUBE_NAME>' AND RELATIONSHIP_NAME='Order Date';
```

REFRESH after.

### Query references `OrderDate.Year` but adapter errors

**Cause:** Adapter is matching on RELATIONSHIP_NAME instead of ROLE_PLAYING_AS in some versions. Workaround: align the two.

```sql
UPDATE SQLCUBE_META.RELATIONSHIPS
SET RELATIONSHIP_NAME = ROLE_PLAYING_AS
WHERE MODEL_NAME='<CUBE_NAME>' AND ROLE_PLAYING_AS IS NOT NULL;
```

REFRESH. Then query `OrderDate.Year` should resolve.

## §6 LLM enrichment produces nonsense

### Hallucinated columns / tables

**Cause:** LLM cherry-picked names that "should exist" but don't. Validation should have caught this — see `llm-enrichment.md` §Validation.

**Fix:** If validation didn't run, run it now:
```sql
SELECT m.MEASURE_NAME, m.EXPRESSION
FROM SQLCUBE_META.MEASURES m
WHERE m.MODEL_NAME='<CUBE_NAME>';
```

For each EXPRESSION, dry-run:
```sql
SELECT <EXPRESSION> FROM <SOURCE>.<FACT> LIMIT 0;
```

DELETE rows that error.

### Wrong-domain naming (LLM thinks it's a different schema)

**Cause:** Domain hint absent or misleading. LLM saw `ORDERS` table and assumed e-commerce when it's a logistics shipment table.

**Fix:** Easiest path — disable enrichment for now (`llm_enrichment=false`), generate the cube structurally, hand-write a `dimensions_overrides` YAML, re-run with overrides.

Don't try to tune the LLM prompt for a single use case. Lock in via YAML.

## §7 MCP doesn't see the cube

### `mcp__exasol_db__list_exasol_schemas` doesn't list the cube

**Cause:** MCP server caches the schema list. Hot-reload not triggered, or MCP server config doesn't include SQLCUBE virtual schemas.

**Fix:**
1. Verify with direct SQL: `SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME='<CUBE>'`. If 0 → cube doesn't actually exist, look at §1.
2. If cube exists → MCP server needs reload. For exanano-sqlcube's bundled MCP: `docker exec exanano-sqlcube sv reload mcp` (or restart container).
3. Check MCP config: `~/.config/claude/mcp_servers.json` (or wherever) — the `exasol_db` entry should NOT filter out virtual schemas. If it does (some configs use `schema_filter: physical_only`), remove the filter.

### MCP sees the cube but queries return "schema not found"

**Cause:** Caching mismatch — MCP fetched a schema list before cube creation, then a query referenced it.

**Fix:** Re-trigger schema discovery. In most clients: send `mcp__exasol_db__list_exasol_schemas` once, then the query.

## Total bricking

If multiple cubes are dysfunctional after a complex sequence of changes:

```sql
-- Wipe just this cube:
DROP VIRTUAL SCHEMA <CUBE_NAME> CASCADE;
DELETE FROM SQLCUBE_META.RELATIONSHIPS WHERE MODEL_NAME='<CUBE_NAME>';
DELETE FROM SQLCUBE_META.MEASURES      WHERE MODEL_NAME='<CUBE_NAME>';
DELETE FROM SQLCUBE_META.COLUMNS       WHERE MODEL_NAME='<CUBE_NAME>';
DELETE FROM SQLCUBE_META.FACTS         WHERE MODEL_NAME='<CUBE_NAME>';
DELETE FROM SQLCUBE_META.DIMENSIONS    WHERE MODEL_NAME='<CUBE_NAME>';
DELETE FROM SQLCUBE_META.MODELS        WHERE MODEL_NAME='<CUBE_NAME>';
```

Then re-run `exasol-semantic-layer` from scratch. The source schema isn't touched.

Never blanket-wipe SQLCUBE_META across cubes — other cubes share the schema and will break.
