# Cube creation

The full deploy flow: write SQLCUBE_REGISTRY rows then create / refresh the virtual schema. Canonical reference implementation: `factory-foundation/studio/backend/services/sqlcube_builder.py` and `services/deployer.py:deploy()`.

## CREATE VIRTUAL SCHEMA statement shape

Two modes — pick based on `mode` parameter.

### Multi-domain (default, `mode=multi_domain`)

```sql
DROP VIRTUAL SCHEMA IF EXISTS "SQLCUBE_ADVENTUREWORKS" CASCADE;
CREATE VIRTUAL SCHEMA "SQLCUBE_ADVENTUREWORKS"
  USING SQLCUBE.ADAPTER
  WITH
    IS_LOCAL = 'true'
    LAYER_ID = 'ADVENTUREWORKS'
    MODEL_IDS = 'factinternetsales,factresellersales,factinternetsales_by_month';
```

Each comma-separated id in `MODEL_IDS` becomes a virtual table inside the virtual schema. The adapter resolves each at query time by reading the registry's MODELS row.

### Single-model (legacy, `mode=single_model`)

```sql
DROP VIRTUAL SCHEMA IF EXISTS "SQLCUBE_FACTINTERNETSALES" CASCADE;
CREATE VIRTUAL SCHEMA "SQLCUBE_FACTINTERNETSALES"
  USING SQLCUBE.ADAPTER
  WITH
    IS_LOCAL = 'true'
    MODEL_ID = 'factinternetsales';
```

Only the one model. Virtual schema is named after the model rather than the layer. Live `exanano-sqlcube` ships with several legacy schemas in this shape (`SQLCUBE_FACTINTERNETSALES`, `SQLCUBE_FACTINTERNETSALES_BY_MONTH`, etc.) — keep using `mode=single_model` when interacting with them.

### What every property does

| Property | Purpose |
|---|---|
| `IS_LOCAL = 'true'` | Tells the adapter to read from the same Exasol cluster, not via JDBC. Always TRUE for SQLCube. |
| `MODEL_ID` (single mode) | The one MODELS row to mount. Becomes a virtual table named `lowercase(model_id)`. |
| `LAYER_ID` (multi mode) | The semantic-layer identifier. Becomes the virtual schema name as `SQLCUBE_<LAYER_ID>`. |
| `MODEL_IDS` (multi mode) | Comma-separated MODEL_IDs to expose. Each is a separate virtual table inside the schema. |

Order of operations matters: properties referenced by the adapter at `set_capabilities` time must already resolve. The DELETE + INSERT pass in the registry must land BEFORE `CREATE VIRTUAL SCHEMA`. The adapter's runtime read isn't deferred — if the registry is empty when CREATE runs, the schema is created but every query against it will error.

## Full deploy sequence

The studio backend's `deployer.py:deploy()` runs this order. Agents following the skill should match:

```
1. Verify preconditions
   - SYS.EXA_ALL_CONSTRAINTS has PKs/FKs for source_schema (exasol-optimize ran)
   - SYS.EXA_ALL_SCRIPTS has SQLCUBE.ADAPTER
   - SYS.EXA_ALL_TABLES has <registry_schema>.MODELS

2. Read source schema + join graph
   - SYS.EXA_ALL_TABLES         → table list
   - SYS.EXA_ALL_COLUMNS        → per-table columns + types
   - SYS.EXA_ALL_CONSTRAINTS    → PKs and FKs
   - SYS.EXA_ALL_CONSTRAINT_COLUMNS → which columns participate

3. Classify each table (FACT vs DIM)
   - Suffix rules: FACT_*, FCT_*, *_FACT, *_FCT, *_TXN, *_EVT → FACT
                   DIM_*, *_DIM, *_DIMENSION → DIM
   - Fallback: tables that are referenced by FK from a FACT but never reference others → DIM
   - Tables with neither outbound nor inbound FKs are skipped (orphan)

4. Plan IDs
   - layer_id     = sanitize(source_schema)             # e.g., ADVENTUREWORKS
   - domain_id    = lowercase(fact_table)               # default; LLM may rename
   - model_id     = lowercase(fact_table)               # default; same as domain in 1-fact case
   - dim_alias    = 'dim', 'dim1', 'dim2', ... assigned in JOIN_ORDER
   - virtual_schema = 'SQLCUBE_' + sanitize(layer_id)

5. Generate registry rows (structural)
   - DOMAINS:        one per fact
   - MODELS:         one per fact
   - DIMENSIONS:     for each FK-reachable dim, expose columns matching the classifier rules
   - MEASURES:       one per numeric column on the fact (SUM default; see step 6 for overrides)
   - JOINS:          one row per declared FK, ordered by FK column ordinal
   - ATTRIBUTES:     one per visible dim column at domain level (mirror DIMENSIONS naming)
   - JOIN_PATHS:     one per declared FK, mirror of JOINS

6. LLM enrichment (optional, when llm_enrichment=true)
   - Add measure proposals
   - Rename DIMENSIONS/ATTRIBUTES virtual_col to business-friendly names
   - Set MODELS.MODEL_LABEL, MODELS.DOMAIN_ID display tweaks
   - See `references/llm-enrichment.md`

7. Merge user overrides
   - measures_overrides.yaml supplements / replaces measures by (model_id, virtual_col)
   - dimensions_overrides.yaml flips IS_VISIBLE, renames virtual_col, sets sort_order

8. Upsert registry (per domain)
   - DELETE in FK-correct order (DERIVED_MEASURES, JOINS, MEASURES, DIMENSIONS, MODELS, JOIN_PATHS, ATTRIBUTES, DOMAINS)
   - INSERT in the opposite order (DOMAINS first, then attributes/join_paths, then MODELS, then per-model: DIMENSIONS, MEASURES, DERIVED_MEASURES, JOINS)

9. CREATE OR REPLACE VIRTUAL SCHEMA
   - DROP IF EXISTS first
   - Build the WITH clause per `mode`
   - Execute the CREATE

10. Verify provides (cube_live)
    - Three checks per SKILL.md
```

## Default measure derivation

When no LLM enrichment and no overrides, the structural pass produces a default measure per numeric column:

| Fact column type | Default measure |
|---|---|
| `DECIMAL(p,s)` p>0 | `AGG_TYPE=SUM`, `PHYSICAL_EXPR='f.<COL>'`, `FORMAT_MASK='#,##0.00'` |
| `INTEGER` / `DECIMAL(p,0)` | `AGG_TYPE=SUM`, `PHYSICAL_EXPR='f.<COL>'`, `FORMAT_MASK='#,##0'` |
| `DOUBLE` | `AGG_TYPE=SUM`, `PHYSICAL_EXPR='f.<COL>'`, `FORMAT_MASK='#,##0.00'` |
| FK column (`*KEY`, `*_ID`) | **Skipped** — never aggregate FKs |
| Boolean | `AGG_TYPE=SUM`, `PHYSICAL_EXPR='CASE WHEN f.<COL> THEN 1 ELSE 0 END'` — true-count |
| Currency-named (`AMOUNT`, `PRICE`, `COST`, `REVENUE`) | Override `FORMAT_MASK='$#,##0.00'` |
| Percent-named (`RATE`, `PERCENT`, `RATIO`) | `AGG_TYPE=AVG`, `FORMAT_MASK='0.00%'` |

Plus three measures regardless of column inventory:

- `Order Count` → `AGG_TYPE=COUNT_DISTINCT`, `PHYSICAL_EXPR='f.<grain_key>'`
- `Order Line Count` → `AGG_TYPE=COUNT`, `PHYSICAL_EXPR='*'`
- `Distinct Customer Count` → if a CUSTOMER_KEY or similar FK exists, `AGG_TYPE=COUNT_DISTINCT`

LLM enrichment usually adds business-specific composites (gross margin, AOV, etc.). See `llm-enrichment.md`.

## Default DIMENSIONS derivation

For each declared FK from FACT to DIM:

1. Add a JOINS row with `DIM_ALIAS` = next available (`dim`, `dim1`, `dim2`, ...)
2. For each non-PK column on the dim that survives the visibility filter:
   - Add a DIMENSIONS row tying `(MODEL_ID, virtual_col)` to `(PHYSICAL_TABLE=<DIM_TABLE_QUALIFIED>, PHYSICAL_COL=<col>, DIM_TABLE_ALIAS=<alias>)`
   - `VIRTUAL_COL` defaults to title-case of `PHYSICAL_COL` (`FIRST_NAME` → `Firstname`)
   - `IS_VISIBLE` defaults TRUE except for known-noisy columns (audit fields, technical-key fields)

Visibility filter rules:

| Column hint | Default IS_VISIBLE |
|---|---|
| Ends in `KEY` and is the PK | FALSE (already in JOINS) |
| Ends in `KEY` and is a snowflake FK | TRUE — analysts may need to drill |
| Starts with `_` or contains `INTERNAL` | FALSE |
| Audit columns (`CREATED_AT`, `UPDATED_AT`, `MODIFIED_BY`) | FALSE |
| Anything else | TRUE |

Override via `dimensions_overrides.yaml`.

## Role-playing dimensions

A fact with two FKs to the same dim (e.g., `ORDERDATEKEY` and `SHIPDATEKEY` both → `DIMDATE.DATEKEY`) gets two JOINS rows with distinct `DIM_ALIAS` values:

```
JOINS:
  (1, factinternetsales, ADVENTUREWORKS.DIMDATE, dim2, DATEKEY, ORDERDATEKEY, LEFT, 30)
  (2, factinternetsales, ADVENTUREWORKS.DIMDATE, dim3, DATEKEY, SHIPDATEKEY,  LEFT, 31)
```

Then DIMENSIONS rows duplicate per role with role-prefixed `VIRTUAL_COL`:

```
DIMENSIONS:
  (..., factinternetsales, 'OrderDate Year',  'ADVENTUREWORKS.DIMDATE', 'CALENDARYEAR', 'dim2', ...)
  (..., factinternetsales, 'ShipDate Year',   'ADVENTUREWORKS.DIMDATE', 'CALENDARYEAR', 'dim3', ...)
```

The `OrderDate ` prefix is convention; the actual prefix comes from the LLM-derived or override-supplied role label.

Without role labels, the structural pass produces just one dim row for the first role found and skips the other — silently wrong. The skill SHOULD detect role-playing dims and require either LLM enrichment OR a `dimensions_overrides` YAML before proceeding.

## Refresh vs recreate

| Situation | Operation |
|---|---|
| Measure added / renamed (registry change only) | Upsert registry rows. Adapter picks up at next query — no virtual-schema action needed. |
| New model added to existing virtual schema | Upsert registry, then `DROP + CREATE VIRTUAL SCHEMA` to update MODEL_IDS in WITH clause. |
| Source table dropped or PK changed | Upsert registry (will remove orphan rows), then `DROP + CREATE`. |
| Cube definition broken | DROP + recreate registry rows + DROP + CREATE virtual schema. |

The adapter has no `REFRESH` semantics — it's stateless and re-reads the registry each query. Whenever the WITH-clause inputs (LAYER_ID / MODEL_ID / MODEL_IDS) change, drop+recreate is required because Exasol caches the WITH properties.

## Verification — provides contract

Three SQL checks after CREATE:

```sql
-- 1. Virtual schema exists with the adapter wired
SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS
WHERE SCHEMA_NAME = 'SQLCUBE_<LAYER_ID>'
  AND ADAPTER_SCRIPT_SCHEMA = 'SQLCUBE'
  AND ADAPTER_SCRIPT_NAME = 'ADAPTER';

-- 2. Registry has at least one model for the domain(s) deployed
SELECT 1 FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID IN (<domains>);

-- 3. Round-trip SELECT actually works
SELECT * FROM "SQLCUBE_<LAYER_ID>"."<lowercase_model_id>" LIMIT 1;
```

All three pass → `cube_live` provides satisfied. Any one fails → see `troubleshooting.md`.

## Atomic batch pattern

Studio backend wraps everything in a single deployment per domain. Agents should match. Pseudo-Python:

```python
sql_batch = (
    build_registry_ddl()           # idempotent CREATE IF NOT EXISTS
  + build_registry_sql(model)      # DELETE + INSERT per domain
  + [build_virtual_schema_sql(layer_id, model_ids)]
)
for stmt in sql_batch:
    conn.execute(stmt)
```

Each statement is independent (Exasol auto-commits DDL). On failure mid-batch: rollback isn't automatic; the partially-written registry is OK because the next run upserts the whole domain again. The virtual schema may be in a bad state if CREATE failed — `DROP VIRTUAL SCHEMA IF EXISTS` at the next attempt clears it.

## Naming gotchas

- **Identifier casing.** Exasol folds unquoted identifiers to uppercase. Inside the registry, everything is uppercase by convention (table/column names) except `MODEL_ID` and `VIRTUAL_COL` which are stored as the canonical lowercase / mixed-case form the adapter uses. Use double-quotes when querying mixed-case identifiers: `SELECT * FROM "SQLCUBE_X"."model_id"`.
- **Schema-qualified physical references.** `DIMENSIONS.PHYSICAL_TABLE` and `JOINS.DIM_TABLE` must be `SCHEMA.TABLE` qualified. The adapter doesn't infer.
- **Alias collisions.** Two DIMENSIONS rows for the same model with the same `DIM_TABLE_ALIAS` but different `PHYSICAL_TABLE` are a bug — the JOIN graph becomes ambiguous. Skill should validate during structural-pass.
- **Reserved words in PHYSICAL_EXPR.** `f.ORDER` collides with Exasol's ORDER keyword in some contexts. Quote: `f."ORDER"`. Skill should auto-quote identifiers it knows are reserved.

## Performance notes

Registry size is tiny (KBs per cube). Adapter reads are cheap. The bottleneck is the underlying physical SQL — same as any Exasol query.

When deploying multiple cubes in sequence, batch the registry upserts and run a single `CREATE VIRTUAL SCHEMA` at the end. Each CREATE triggers `set_capabilities` which is a non-trivial registry scan — avoid firing it N times when one will do.
