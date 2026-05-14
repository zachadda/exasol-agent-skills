# Troubleshooting cubes

Failure modes during creation, registry upsert, and query against a SQLCube virtual schema. Ordered by what the agent will likely encounter first.

Schema name convention in this doc: registry = `SQLCUBE_REGISTRY` (override if customized); virtual schema = `SQLCUBE_<LAYER_ID>`.

## Decision tree

```
CREATE VIRTUAL SCHEMA fails                          → §1
Registry upsert fails                                → §2
SELECT * FROM cube works, but…
  …a measure errors                                  → §3
  …a join returns cross-product / wrong row count    → §4
  …role-playing dim returns garbage                  → §5
LLM enrichment produces invalid rows                  → §6
MCP / list_schemas doesn't see the cube              → §7
Identifier casing errors at query time                → §8
```

## §1 CREATE VIRTUAL SCHEMA fails

### "Script SQLCUBE.ADAPTER not found"

**Cause:** Adapter not installed.

**Fix:** Route to `install.md`. Don't try to fall back.

### "Property MODEL_ID is required" or "LAYER_ID / MODEL_IDS not set"

**Cause:** WITH clause shape doesn't match what the adapter expects. The adapter accepts either:

```
MODEL_ID = '<id>'                     # single-model legacy
```

OR

```
LAYER_ID = '<id>' MODEL_IDS = '<csv>' # multi-domain
```

**Fix:** Pick a mode. The skill's `mode` parameter (`multi_domain` default, `single_model` legacy) controls which shape lands. If you mixed both (e.g., supplied MODEL_IDS but the adapter version pre-dates multi-domain), upgrade the adapter or drop to single-model.

### "MODELS row not found for MODEL_ID 'x'"

**Cause:** Registry MODELS row absent when CREATE VIRTUAL SCHEMA runs. Adapter validates referenced model_ids at `set_capabilities`.

**Fix:** Insert MODELS row first. Registry write order matters — see `cube-creation.md`.

### "Schema SQLCUBE_<LAYER> already exists"

**Cause:** Previous deploy left a virtual schema; CREATE without DROP fails.

**Fix:** Skill should always emit `DROP VIRTUAL SCHEMA IF EXISTS "<NAME>" CASCADE;` before CREATE. If running manually:

```sql
DROP VIRTUAL SCHEMA IF EXISTS "SQLCUBE_ADVENTUREWORKS" CASCADE;
```

CASCADE drops dependent objects (none should exist, but be safe).

### Adapter fails silently during CREATE — no error, but virtual schema has zero tables

**Cause:** Adapter's `set_capabilities` short-circuited due to bad registry data. Most common: MODEL_IDS in the WITH clause references model_ids that don't exist in MODELS table.

**Fix:**

```sql
-- Confirm MODELS rows exist for each id in the WITH clause:
SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS
WHERE MODEL_ID IN ('factinternetsales', 'factresellersales');

-- Confirm the virtual schema landed:
SELECT TABLE_NAME FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA = 'SQLCUBE_ADVENTUREWORKS';
```

If MODELS missing → fix registry. If MODELS present but virtual schema has zero tables → the adapter logs to Exasol's auditing system; check `SELECT * FROM SYS.EXA_STATISTICS_SESSION` or your DBA's adapter log destination.

## §2 Registry upsert fails

### "Object SQLCUBE_REGISTRY.MODELS not found"

**Cause:** Registry schema or tables missing.

**Fix:** Route to `install.md`. The skill's precondition check should have caught this; if it didn't, the agent may have been pointed at the wrong cluster.

### "Cannot delete row: foreign key constraint violated"

**Cause:** Wrong DELETE order. SQLCUBE_REGISTRY has FK constraints (MODELS → DOMAINS, DIMENSIONS/MEASURES/JOINS → MODELS). DELETE must happen child-first.

**Fix:** Use the canonical order from `meta-model.md`:

```
DERIVED_MEASURES, JOINS, MEASURES, DIMENSIONS, MODELS, JOIN_PATHS, ATTRIBUTES, DOMAINS
```

If you're getting this error, the skill emitted DELETEs in the wrong order. Inspect the generated SQL.

### "Cannot insert: duplicate key on (MODEL_ID, VIRTUAL_COL)"

**Cause:** DIMENSIONS or MEASURES has a duplicate row from a prior incomplete deploy.

**Fix:** Re-run the full upsert. DELETE-by-domain_id should clear the duplicates. If a single domain owns rows that overlap with another domain's slug, that's a deeper data-integrity issue — investigate which deploys ran historically.

### "Object MODEL_ID does not exist" during INSERT

**Cause:** Inserting DIMENSIONS / MEASURES / JOINS before the parent MODELS row exists.

**Fix:** Order matters in INSERT too: DOMAINS → ATTRIBUTES → JOIN_PATHS → MODELS → DIMENSIONS → MEASURES → DERIVED_MEASURES → JOINS.

## §3 Measure error at query time

### "Column UNITPRICE not found"

**Cause:** `PHYSICAL_EXPR` references a column the underlying fact table doesn't have. Common after a column rename in the source schema.

**Fix:**

```sql
-- Verify each measure expression dry-runs:
SELECT MEASURE_ID, VIRTUAL_COL, PHYSICAL_EXPR
FROM SQLCUBE_REGISTRY.MEASURES
WHERE MODEL_ID = 'factinternetsales';
```

For each, run:

```sql
SELECT <PHYSICAL_EXPR> FROM <FACT_SCHEMA>.<FACT_TABLE> f LIMIT 0;
```

Anything that errors → update or DELETE the measure row, re-run.

### "Function SUM cannot be applied to type VARCHAR"

**Cause:** `AGG_TYPE='SUM'` on a non-numeric expression.

**Fix:**

```sql
UPDATE SQLCUBE_REGISTRY.MEASURES
SET AGG_TYPE = 'COUNT_DISTINCT'   -- or COUNT, or change PHYSICAL_EXPR
WHERE MODEL_ID = '<m>' AND VIRTUAL_COL = '<col>';
```

### "Division by zero"

**Cause:** Ratio measure without NULLIF defense.

**Fix:** Update the expression:

```sql
UPDATE SQLCUBE_REGISTRY.MEASURES
SET PHYSICAL_EXPR = '(f.MARGIN) / NULLIF(f.REVENUE, 0)'
WHERE MODEL_ID = '<m>' AND VIRTUAL_COL = 'Margin Rate';
```

### "Reference to alias 'f' is ambiguous"

**Cause:** The expression uses an alias the adapter didn't expect to generate. The adapter generates `FROM <FACT_SCHEMA>.<FACT_TABLE> f` — measures must use `f.` for fact columns. Using `<TABLE_NAME>.<COL>` instead also fails.

**Fix:**

```sql
UPDATE SQLCUBE_REGISTRY.MEASURES
SET PHYSICAL_EXPR = 'f.EXTENDEDAMOUNT'    -- not FACTINTERNETSALES.EXTENDEDAMOUNT
WHERE MODEL_ID = '<m>' AND VIRTUAL_COL = '<col>';
```

## §4 Wrong row counts

### Cross-product (NxM rows when N expected)

**Cause:** A DIMENSIONS row references a DIM_TABLE_ALIAS that doesn't have a matching JOINS row. The adapter doesn't know how to bring the dim in, and depending on adapter version either joins on nothing (Cartesian) or returns nulls.

**Fix:**

```sql
-- Find DIMENSIONS rows missing a JOINS counterpart:
SELECT D.MODEL_ID, D.VIRTUAL_COL, D.PHYSICAL_TABLE, D.DIM_TABLE_ALIAS
FROM SQLCUBE_REGISTRY.DIMENSIONS D
LEFT JOIN SQLCUBE_REGISTRY.JOINS J
  ON D.MODEL_ID = J.MODEL_ID AND D.DIM_TABLE_ALIAS = J.DIM_ALIAS
WHERE J.JOIN_ID IS NULL
  AND D.IS_VISIBLE
ORDER BY D.MODEL_ID, D.VIRTUAL_COL;
```

Anything returned → add a JOINS row, or fix the DIM_TABLE_ALIAS, or hide the DIMENSIONS row (`IS_VISIBLE=FALSE`).

### Zero rows when some expected

**Cause:** `JOIN_TYPE='INNER'` on a JOINS row, and the join key doesn't match in the data.

**Fix:**

```sql
-- Sanity-check the join keys overlap:
SELECT COUNT(*) FROM <FACT> f
WHERE f.<FACT_FK> IN (SELECT <DIM_KEY> FROM <DIM>);
```

If zero → data problem upstream. If non-zero but query returns zero → JOIN_TYPE='INNER' is dropping rows you wanted; switch to LEFT:

```sql
UPDATE SQLCUBE_REGISTRY.JOINS
SET JOIN_TYPE = 'LEFT'
WHERE MODEL_ID = '<m>' AND JOIN_ID = <id>;
```

## §5 Role-playing dim returns garbage

### Both date roles return the same data

**Cause:** Two DIMENSIONS rows for the same dim column with the same `DIM_TABLE_ALIAS`. Adapter picks one and ignores the other.

**Fix:**

```sql
SELECT MODEL_ID, VIRTUAL_COL, PHYSICAL_TABLE, PHYSICAL_COL, DIM_TABLE_ALIAS
FROM SQLCUBE_REGISTRY.DIMENSIONS
WHERE MODEL_ID = '<m>' AND PHYSICAL_TABLE = 'ADVENTUREWORKS.DIMDATE';
```

Each role must have a distinct DIM_TABLE_ALIAS. Fix by updating the rows or re-running the structural pass with the role-playing detection enabled.

### "Cannot resolve column dim2.CALENDARYEAR"

**Cause:** DIMENSIONS row references `DIM_TABLE_ALIAS='dim2'` but there's no JOINS row with `DIM_ALIAS='dim2'` for that model.

**Fix:** Add the JOINS row. Pattern for an order-date join:

```sql
INSERT INTO SQLCUBE_REGISTRY.JOINS
  (JOIN_ID, MODEL_ID, DIM_TABLE, DIM_ALIAS, DIM_KEY, FACT_FK, JOIN_TYPE, JOIN_ORDER)
VALUES (
  <next_id>, '<MODEL>',
  'ADVENTUREWORKS.DIMDATE', 'dim2', 'DATEKEY', 'ORDERDATEKEY', 'LEFT', 30
);
```

## §6 LLM enrichment produces invalid rows

### Hallucinated columns / tables

**Cause:** LLM proposed a PHYSICAL_TABLE / PHYSICAL_COL that doesn't exist. Should have been caught by validation in `llm-enrichment.md`, but if it wasn't:

```sql
-- Find DIMENSIONS pointing at nonexistent physical columns:
SELECT D.MODEL_ID, D.VIRTUAL_COL, D.PHYSICAL_TABLE, D.PHYSICAL_COL
FROM SQLCUBE_REGISTRY.DIMENSIONS D
LEFT JOIN SYS.EXA_ALL_COLUMNS C
  ON UPPER(D.PHYSICAL_TABLE) = C.COLUMN_SCHEMA || '.' || C.COLUMN_TABLE
 AND UPPER(D.PHYSICAL_COL) = C.COLUMN_NAME
WHERE C.COLUMN_NAME IS NULL;
```

Anything returned → DELETE those rows, re-run validation.

### Wrong-domain naming

LLM saw schema `ORDERS` and assumed e-commerce when it's actually logistics shipments. The cube technically works, but business labels are misleading.

**Fix:** Set `llm_enrichment=false`, supply a `dimensions_overrides.yaml` with correct domain labels, re-run.

## §7 MCP doesn't see the cube

### `mcp__exasol_db__list_exasol_schemas` doesn't list it

**Cause:** MCP server caches the schema list, or filters virtual schemas.

**Fix:**

1. Confirm SQL-side: `SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = 'SQLCUBE_<LAYER>'`. If 0, cube doesn't actually exist; see §1.
2. Reload MCP. For exanano-sqlcube's bundled MCP, `docker exec exanano-sqlcube sv reload mcp` (or container restart).
3. Check MCP config — `~/.config/claude/mcp_servers.json` or platform-equivalent. Some configs filter virtual schemas via `schema_filter` — remove or extend.

### MCP sees the cube but query errors with "schema not found"

**Cause:** MCP fetched the schema list before the cube existed.

**Fix:** Re-list schemas (`mcp__exasol_db__list_exasol_schemas`), then re-issue the query.

## §8 Identifier casing errors

### "Object SQLCUBE_ADVENTUREWORKS.FACTINTERNETSALES does not exist"

**Cause:** Unquoted reference to the virtual table — Exasol uppercased `factinternetsales` to `FACTINTERNETSALES`, which doesn't match.

**Fix:** Always quote:

```sql
SELECT * FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales" LIMIT 1;
```

### "Object SQLCUBE_ADVENTUREWORKS.factinternetsales does not exist" (even with quotes)

**Cause:** The MODEL_IDS WITH-clause property didn't include this model. Multi-domain mode only mounts the models listed in MODEL_IDS.

**Fix:**

```sql
-- See what's mounted:
SELECT TABLE_NAME FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA = 'SQLCUBE_ADVENTUREWORKS';

-- Re-create with the right MODEL_IDS:
DROP VIRTUAL SCHEMA "SQLCUBE_ADVENTUREWORKS" CASCADE;
CREATE VIRTUAL SCHEMA "SQLCUBE_ADVENTUREWORKS" USING SQLCUBE.ADAPTER
  WITH IS_LOCAL='true' LAYER_ID='ADVENTUREWORKS'
       MODEL_IDS='factinternetsales,factresellersales';
```

### "Column 'Extended Amount' not found"

**Cause:** Unquoted column name with a space → Exasol parse error or wrong identifier resolution.

**Fix:**

```sql
SELECT "Extended Amount" FROM "SQLCUBE_ADVENTUREWORKS"."factinternetsales";
```

## Total wipe and rebuild

If a cube is fundamentally broken and the user accepts wiping its registry rows:

```sql
DROP VIRTUAL SCHEMA IF EXISTS "SQLCUBE_<LAYER>" CASCADE;

-- Per domain — repeat for each:
DELETE FROM SQLCUBE_REGISTRY.DERIVED_MEASURES WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<DOM>');
DELETE FROM SQLCUBE_REGISTRY.JOINS            WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<DOM>');
DELETE FROM SQLCUBE_REGISTRY.MEASURES         WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<DOM>');
DELETE FROM SQLCUBE_REGISTRY.DIMENSIONS       WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<DOM>');
DELETE FROM SQLCUBE_REGISTRY.MODELS           WHERE DOMAIN_ID = '<DOM>';
DELETE FROM SQLCUBE_REGISTRY.JOIN_PATHS       WHERE DOMAIN_ID = '<DOM>';
DELETE FROM SQLCUBE_REGISTRY.ATTRIBUTES       WHERE DOMAIN_ID = '<DOM>';
DELETE FROM SQLCUBE_REGISTRY.DOMAINS          WHERE DOMAIN_ID = '<DOM>';
```

Then re-run the skill from scratch. The source schema isn't touched. Other domains in the same registry are untouched.

NEVER blanket-wipe the entire SQLCUBE_REGISTRY — other cubes share it.

## When to call DBA

- Adapter UDF won't deploy (Lua compile / permissions). Get cluster privileges.
- BucketFS upload fails (auth / bucket missing). Get bucket credentials.
- Registry schema gets corrupted somehow (FKs out of sync). Restore from backup or `EXPORT SCHEMA` from a known-good environment and re-import.

The adapter source lives in `exasol-factory-foundation/sqlcube/adapter/` — the studio team owns it. File issues against that repo, not this skill.
