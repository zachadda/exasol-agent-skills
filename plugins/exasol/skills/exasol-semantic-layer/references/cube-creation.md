# Cube creation

The CREATE VIRTUAL SCHEMA flow. Assumes SQLCUBE_META rows already exist (or are being written in parallel — see `meta-model.md`).

## Statement shape

```sql
CREATE VIRTUAL SCHEMA <CUBE_NAME>
USING SQLCUBE.ADAPTER
WITH
  MODEL_NAME           = '<CUBE_NAME>'
  SOURCE_SCHEMA        = '<SOURCE_SCHEMA>'
  REFRESH_ON_CREATE    = TRUE
  LOG_LEVEL            = 'INFO';
```

What each property does:

| Property | Purpose |
|---|---|
| `MODEL_NAME` | Key into SQLCUBE_META.MODELS. Must already exist or REFRESH_ON_CREATE will fail. |
| `SOURCE_SCHEMA` | Physical schema the adapter reads from. The adapter validates this against MODELS.SOURCE_SCHEMA — must match. |
| `REFRESH_ON_CREATE` | If TRUE, adapter immediately builds the virtual catalog. If FALSE, you must `REFRESH VIRTUAL SCHEMA` before queries work. Default to TRUE. |
| `LOG_LEVEL` | `DEBUG` / `INFO` / `WARN` / `ERROR`. Adapter logs land in SYS audit. INFO is enough for most uses. |

## Sequencing

In order:

1. **MODELS row first.** No `CREATE VIRTUAL SCHEMA` without a MODELS row matching `MODEL_NAME`. Otherwise the adapter throws "Model not registered".
2. **DIMENSIONS / FACTS / MEASURES / RELATIONSHIPS / COLUMNS next.** These can be written in any order relative to each other.
3. **CREATE VIRTUAL SCHEMA** last. The adapter reads everything during REFRESH.

If you're re-running:

```sql
DROP VIRTUAL SCHEMA <CUBE_NAME> CASCADE;
DELETE FROM SQLCUBE_META.RELATIONSHIPS WHERE MODEL_NAME = '<CUBE_NAME>';
DELETE FROM SQLCUBE_META.MEASURES      WHERE MODEL_NAME = '<CUBE_NAME>';
DELETE FROM SQLCUBE_META.FACTS         WHERE MODEL_NAME = '<CUBE_NAME>';
DELETE FROM SQLCUBE_META.DIMENSIONS    WHERE MODEL_NAME = '<CUBE_NAME>';
DELETE FROM SQLCUBE_META.COLUMNS       WHERE MODEL_NAME = '<CUBE_NAME>';
DELETE FROM SQLCUBE_META.MODELS        WHERE MODEL_NAME = '<CUBE_NAME>';
-- then re-INSERT and re-CREATE
```

Order matters for the DELETEs because of FK constraints in SQLCUBE_META itself.

Only do the DROP CASCADE when `drop_existing=true` parameter is set AND the user has confirmed. Drop is irreversible.

## Refresh vs recreate

| Situation | Operation |
|---|---|
| New table added to source schema | `REFRESH VIRTUAL SCHEMA <CUBE_NAME>` (after updating SQLCUBE_META) |
| Measure renamed | Same |
| New cube from scratch | `CREATE VIRTUAL SCHEMA ...` |
| Cube structure broken | DROP + recreate |
| Underlying table dropped | REFRESH (with manifest reconciliation in metadata) |

Refresh is cheap (rebuilds in-memory catalog only; no data movement). Always prefer over drop+recreate when SQLCUBE_META is the only change.

## Atomic batch pattern

When the skill writes a new cube it should use a single transaction-like batch:

```sql
-- Pseudo. Exasol's auto-commit means each statement commits independently.
-- Wrap in EXECUTE SCRIPT for atomicity if it matters.

INSERT INTO SQLCUBE_META.MODELS (...) VALUES (...);
INSERT INTO SQLCUBE_META.DIMENSIONS (...) VALUES (...), (...), ...;
INSERT INTO SQLCUBE_META.FACTS (...) VALUES (...);
INSERT INTO SQLCUBE_META.RELATIONSHIPS (...) VALUES (...), (...), ...;
INSERT INTO SQLCUBE_META.MEASURES (...) VALUES (...), (...), ...;
INSERT INTO SQLCUBE_META.COLUMNS (...) VALUES (...), (...), ...;
CREATE VIRTUAL SCHEMA <CUBE_NAME> USING SQLCUBE.ADAPTER WITH MODEL_NAME='<CUBE_NAME>' ...;
```

If `CREATE VIRTUAL SCHEMA` fails (most often: missing PK on a dimension), the META rows are still there. Re-run after fixing the underlying issue — no need to re-insert.

## Inferring the base model from the source schema

Before any META writes, walk the source schema and derive the base structure. Inputs:

- `SYS.EXA_ALL_TABLES` for the table list
- `SYS.EXA_ALL_CONSTRAINTS` + `SYS.EXA_ALL_CONSTRAINT_COLUMNS` for PKs + FKs
- `SYS.EXA_ALL_COLUMNS` for the column inventory
- Optionally the `__migration_manifest.csv` from exasol-migrate for additional context (source-side types, original names)

Classification rules:

```
Suffix-driven (mirrors exasol-optimize logic):
- *_DIM, *_DIMENSION       → DIMENSION
- *_FACT, *_FCT, *_TXN, *_EVT  → FACT
- Everything else          → DIMENSION by default; warn in manifest
```

Override rule: if the user passed `dimensions_overrides` / `measures_overrides` YAMLs (`exasol-semantic-layer` parameters), user assignments win.

## What gets generated automatically

Even without LLM enrichment, the skill produces:

**DIMENSIONS rows** — one per discovered DIM table, with `DIM_NAME` = stem of physical name (`DIM_CUSTOMER` → `Customer`), `PK_COLUMN` from constraints catalog.

**FACTS rows** — one per discovered FACT table.

**RELATIONSHIPS rows** — one per declared FK, with `RELATIONSHIP_NAME` = the FK column stem (`CUSTOMER_ID` on the fact → relationship named `Customer`). Role-playing dates (multiple FK columns to DIM_DATE) get role-suffixed names (`OrderDate`, `ShipDate`).

**MEASURES rows** — one per numeric column on each FACT, with default aggregations:

| Column hint | Default measure |
|---|---|
| Column name contains `AMOUNT` / `PRICE` / `REVENUE` / `COST` | `SUM(...)` |
| Column name contains `QUANTITY` / `QTY` / `COUNT` | `SUM(...)` |
| Column name contains `RATE` / `PERCENT` / `RATIO` | `AVG(...)` |
| Boolean column on fact | `SUM(CASE WHEN col THEN 1 ELSE 0 END)` (count of true) |
| FK columns (`_ID`) | Skip — never aggregate FKs |

These defaults are deliberately conservative; LLM enrichment refines them with business-specific formulas (e.g., gross-vs-net revenue, FX conversion).

**COLUMNS rows** — one per non-FK / non-PK column. Display names default to title-case of physical (`first_name` → `First Name`), descriptions empty unless LLM enriches.

## Verification — the provides contract

After CREATE, the skill must verify three things:

```sql
-- 1. Virtual schema exists
SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = '<CUBE_NAME>';

-- 2. Model registered
SELECT 1 FROM SQLCUBE_META.MODELS WHERE MODEL_NAME = '<CUBE_NAME>';

-- 3. SELECT round-trip actually works
SELECT * FROM "<CUBE_NAME>"."<ANY_TABLE>" LIMIT 1;
```

Pick `<ANY_TABLE>` from the largest fact table — that's the path most exercised by downstream dashboards. If all three pass → `cube_live` provides satisfied.

If (3) fails, the cube is structurally present but operationally broken. Common cause: FK on the fact references a table the adapter couldn't resolve (mis-cased name, wrong schema). Defer to `troubleshooting.md`.

## Naming gotchas

- **Case sensitivity.** Exasol identifiers are case-folded to uppercase unless quoted. SQLCUBE_META stores them uppercase. When the adapter rewrites, it quotes physical names. Don't INSERT mixed-case into SQLCUBE_META — store uppercase, let the display logic handle the casing for humans.
- **Reserved words.** A few business names collide with Exasol keywords (`ORDER`, `GROUP`, `USER`). Display name is fine, but use quoted identifiers in EXPRESSION strings.
- **Schema-qualified names in EXPRESSION.** Measure expressions reference physical columns. Use unqualified column names if it's unambiguous in the fact; qualify when joining (`DIM_CUSTOMER.SEGMENT`). The adapter will qualify with the source schema during rewrite.
