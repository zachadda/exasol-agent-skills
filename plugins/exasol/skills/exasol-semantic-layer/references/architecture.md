# SQLCube semantic layer architecture

Read first. Five-minute orientation to the moving parts so the rest of the references make sense.

## The pieces

```
┌──────────────────────────────────────────────────────────┐
│ Physical schema (e.g., ADVENTUREWORKS)                    │
│   FACT_SALES, DIM_CUSTOMER, DIM_PRODUCT, DIM_DATE, ...    │
│   PKs declared, FKs declared (via exasol-optimize)        │
└──────────────────────────────────────────────────────────┘
                          ▲ underlying tables
                          │
┌──────────────────────────────────────────────────────────┐
│ Virtual schema (e.g., ADVENTUREWORKS_CUBE)                │
│   CREATE VIRTUAL SCHEMA ... USING SQLCUBE.ADAPTER WITH    │
│     MODEL_NAME='ADVENTUREWORKS_CUBE'                      │
└──────────────────────────────────────────────────────────┘
                          ▲ queries
                          │
┌──────────────────────────────────────────────────────────┐
│ SQLCUBE.ADAPTER (Exasol Virtual Schema adapter UDF)       │
│   Implements pushdown / capabilities / refresh callbacks  │
│   Reads model definition from SQLCUBE_META                │
│   Rewrites queries → SQL against physical schema          │
└──────────────────────────────────────────────────────────┘
                          ▲ uses
                          │
┌──────────────────────────────────────────────────────────┐
│ SQLCUBE_META schema (system-of-record for cube metadata)  │
│   MODELS         — one row per cube                       │
│   DIMENSIONS     — dimension entities (DIM tables)        │
│   FACTS          — fact entities (FACT tables)            │
│   MEASURES       — aggregates over fact columns           │
│   RELATIONSHIPS  — FK-derived named joins                 │
│   COLUMNS        — display names, descriptions, types     │
└──────────────────────────────────────────────────────────┘
```

The flow is unidirectional. Queries hit the virtual schema, ADAPTER intercepts, ADAPTER reads SQLCUBE_META to figure out what each virtual identifier maps to, ADAPTER rewrites into physical SQL, Exasol's optimizer runs it. No materialization. No caching. No state outside SQLCUBE_META.

## Why this shape

**Q: Why not just use Exasol views?**
A: Views are stateless. They have no description, no relationship metadata, no measure formulas, no business names. SQLCUBE_META is where semantic information lives. Views are a degenerate semantic layer.

**Q: Why a Virtual Schema instead of generated views?**
A: Virtual Schemas have a query-rewrite hook (the ADAPTER UDF). We can do things like:
- Rewrite `SELECT total_revenue FROM cube.sales` into `SELECT SUM(LINEITEM.QUANTITY * LINEITEM.UNITPRICE * (1 - LINEITEM.DISCOUNT)) FROM ...`
- Auto-join based on RELATIONSHIPS metadata when the query references columns from multiple dimensions
- Surface helpful errors when a query references a measure that doesn't apply to the requested grain

None of that is possible with plain views.

**Q: Why store metadata in tables, not files?**
A: Exasol-native. The cube definition is queryable, joinable, versionable via standard SQL. Backup is `EXPORT SCHEMA`. No file system to lose, no YAML to drift from production. Bonus: the LLM enrichment step writes rows; future cube editors are also just SQL.

**Q: How does this differ from dbt's semantic layer?**
A: dbt's semantic layer is YAML → compiled SQL → executed by the warehouse. Ours is SQL → SQLCUBE_META → executed by the warehouse via ADAPTER. Same conceptual layers, different storage / runtime. dbt is more declarative and version-control-friendly; ours is more database-native and live-editable.

## What SQLCUBE_META looks like

Five core tables. Schema lives in `SQLCUBE_META`. Detail in `meta-model.md`.

### MODELS

One row per cube.

| Column | Type | Purpose |
|---|---|---|
| MODEL_NAME | VARCHAR | Primary key. Matches virtual schema name. |
| SOURCE_SCHEMA | VARCHAR | Physical schema this cube reads from. |
| DESCRIPTION | VARCHAR | Business-level description (LLM-generated or user) |
| CREATED_AT | TIMESTAMP | Audit |
| UPDATED_AT | TIMESTAMP | Audit |
| OWNER | VARCHAR | User who created the cube |

### DIMENSIONS

| Column | Type | Purpose |
|---|---|---|
| MODEL_NAME | VARCHAR | FK → MODELS |
| DIM_NAME | VARCHAR | Business name (e.g., "Customer", LLM-derived from DIM_CUSTOMER) |
| PHYSICAL_TABLE | VARCHAR | Underlying DIM table |
| PK_COLUMN | VARCHAR | From exasol-optimize discovery |
| DESCRIPTION | VARCHAR | Business description |

### FACTS

| Column | Type | Purpose |
|---|---|---|
| MODEL_NAME | VARCHAR | FK → MODELS |
| FACT_NAME | VARCHAR | Business name |
| PHYSICAL_TABLE | VARCHAR | Underlying FACT table |
| GRAIN_DESCRIPTION | VARCHAR | What one row represents |

### MEASURES

| Column | Type | Purpose |
|---|---|---|
| MODEL_NAME | VARCHAR | FK → MODELS |
| FACT_NAME | VARCHAR | FK → FACTS within model |
| MEASURE_NAME | VARCHAR | Business name (e.g., "Total Revenue") |
| EXPRESSION | VARCHAR | SQL expression — typically an aggregate |
| AGGREGATION | VARCHAR | SUM / AVG / COUNT / MIN / MAX / NONE |
| FORMAT_HINT | VARCHAR | currency_usd / percent / integer / etc. |
| DESCRIPTION | VARCHAR | Business description |

### RELATIONSHIPS

| Column | Type | Purpose |
|---|---|---|
| MODEL_NAME | VARCHAR | FK → MODELS |
| RELATIONSHIP_NAME | VARCHAR | Business name (e.g., "ordered_by", "shipped_to") |
| FACT_NAME | VARCHAR | One side |
| DIM_NAME | VARCHAR | Other side |
| FACT_FK_COLUMN | VARCHAR | Physical join column on fact |
| DIM_PK_COLUMN | VARCHAR | Physical join column on dim |
| ROLE_PLAYING_AS | VARCHAR | Null for normal joins; non-null for role-playing dates ("order_date", "ship_date") |

## SQLCUBE.ADAPTER UDF

The query rewrite engine. Implements the Exasol Virtual Schema adapter contract:

- `set_capabilities` — declares which SQL operators the adapter can push down (selects, projects, aggregates, joins between mounted tables)
- `refresh` — rebuilds the virtual catalog from SQLCUBE_META; called by `REFRESH VIRTUAL SCHEMA`
- `drop_virtual_schema` — cleanup

At query time, the adapter walks the parse tree from Exasol, maps virtual identifiers to physical identifiers via SQLCUBE_META, expands measures via their EXPRESSION column, and emits the rewritten SQL back to Exasol's optimizer.

Source lives in `exasol-factory-foundation/sqlcube/adapter/` (Lua UDF). Not in this skill's scope to modify — agents should treat the adapter as a black box and only manipulate SQLCUBE_META.

## When the ADAPTER is missing

If `SELECT * FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA = 'SQLCUBE' AND SCRIPT_NAME = 'ADAPTER'` returns 0 rows, the infrastructure isn't installed. The exanano-sqlcube container ships it pre-installed; other Exasol clusters need a one-time install. Defer to `references/install.md` (and the `sqlcube` repo's deployment scripts).

## Mental model

Treat SQLCUBE_META as the source-of-truth and the virtual schema as a computed view of it. Every operation in this skill is:

1. INSERT / UPDATE rows into SQLCUBE_META
2. Run `CREATE VIRTUAL SCHEMA ... USING SQLCUBE.ADAPTER` (or `REFRESH VIRTUAL SCHEMA`) so the adapter picks up the changes

That's the loop. Everything else is detail.
