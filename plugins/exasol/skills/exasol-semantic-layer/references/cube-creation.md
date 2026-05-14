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
    MODEL_IDS = 'internet_sales,reseller_sales,internet_sales_by_month';
```

Each comma-separated id in `MODEL_IDS` becomes a virtual table inside the virtual schema (the studio deployer sets `MODEL_ID = DOMAIN_ID` — see `meta-model.md` conventions). The adapter resolves each at query time by reading the registry's MODELS row.

### Single-model (legacy, `mode=single_model`)

```sql
DROP VIRTUAL SCHEMA IF EXISTS "SQLCUBE_FACTINTERNETSALES" CASCADE;
CREATE VIRTUAL SCHEMA "SQLCUBE_FACTINTERNETSALES"
  USING SQLCUBE.ADAPTER
  WITH
    IS_LOCAL = 'true'
    MODEL_ID = 'internet_sales';
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
   - domain_id    = stable business slug (LLM-named, e.g. `internet_sales`)
   - model_id     = **same string as domain_id** — `deployer.py:430` always writes `MODEL_ID = DOMAIN_ID`
   - dim_alias    = first 3 alphanum lowercase chars of dim table + collision suffix (e.g. DIMPRODUCT → `dim`, DIMCUSTOMER → `dim1`, CALENDAR → `cal`)
   - virtual_schema = 'SQLCUBE_' + sanitize(layer_id)

5. Generate registry rows. **Structural defaults in this codebase come from `services/claude.py` LLM enrichment** — there is no deterministic per-numeric-column rule in the deployer. The LLM produces:
   - DOMAINS:        one per fact / business grouping
   - MODELS:         one per fact-table grain (MODEL_ID == DOMAIN_ID; GRAIN_KEY stays NULL)
   - DIMENSIONS:     for each FK-reachable dim, exposed columns with VIRTUAL_COL in Title Case with spaces
   - MEASURES:       proposed per numeric / amount column on the fact (SUM, COUNT_DISTINCT, AVG)
   - JOINS:          one row per declared FK, alias derived from dim-table name
   - ATTRIBUTES:     one per visible dim column at domain level (display labels)
   - JOIN_PATHS:     one per declared FK, mirror of JOINS at domain level

6. (Already inside step 5 — the LLM enrichment IS the structural pass in this codebase.) The patterns below ("Default measure derivation", "Default DIMENSIONS derivation") describe the **target shape** the LLM is prompted to produce, not deterministic code paths in `deployer.py`. Lock outputs with overrides if you need determinism.

7. Merge user overrides
   - measures_overrides.yaml supplements / replaces measures by (model_id, virtual_col)
   - dimensions_overrides.yaml flips IS_VISIBLE, renames virtual_col, sets sort_order

8. Upsert registry (two-pass DELETE — live-verified from `deployer.py`)
   - **First DELETE pass** (`_build_runtime_registry_sql` lines 400–412): purge by `(FACT_SCHEMA, FACT_TABLE)` pair to clear stale `MODEL_ID` slugs from earlier deploys (handles slug renames).
   - **Second DELETE pass** (per domain): DERIVED_MEASURES → JOINS → MEASURES → DIMENSIONS → MODELS → JOIN_PATHS → ATTRIBUTES → DOMAINS.
   - INSERT order in actual deployer: **DOMAINS → ATTRIBUTES → JOIN_PATHS → MODELS → JOINS → DIMENSIONS → MEASURES → DERIVED_MEASURES**. JOINS lands before DIMENSIONS in `_build_runtime_registry_sql` because DIMENSIONS rows reference alias values JOINS rows define.
   - The registry has **no enforced FK constraints** in `build_registry_ddl`, so order is for human readability only — Exasol does not validate it.

9. CREATE OR REPLACE VIRTUAL SCHEMA
   - DROP IF EXISTS first
   - Build the WITH clause per `mode`
   - Execute the CREATE

10. Verify provides (cube_live)
    - Three checks per SKILL.md
```

## Default measure derivation (LLM-target shape, not deterministic code)

The studio backend does NOT contain a deterministic per-numeric-column structural pass. The shape below is the **target** that `services/claude.py` is prompted to produce. Run with `llm_enrichment=true` (default) for the LLM to attempt it, or supply `measures_overrides.yaml` for full control.

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
   - `VIRTUAL_COL` is Title Case with spaces in live registries (`FIRSTNAME` → `'First Name'`, `ENGLISHPRODUCTNAME` → `'English Product Name'`) — produced by LLM enrichment, not a code rule
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
  (1, internet_sales, ADVENTUREWORKS.DIMDATE, dim5, DATEKEY, ORDERDATEKEY, INNER, 70)
  (2, internet_sales, ADVENTUREWORKS.DIMDATE, dim6, DATEKEY, SHIPDATEKEY,  INNER, 80)
  (3, internet_sales, ADVENTUREWORKS.DIMDATE, dim7, DATEKEY, DUEDATEKEY,   INNER, 90)
```

(Live aliases from `SQLCUBE_ADVENTUREWORKS.internet_sales` on `exanano-sqlcube` 2026-05-13 — the first DIMDATE join gets `dim5` because `dim/dim1/dim2/dim3/dim4` were taken by DIMPRODUCT-class tables earlier in JOIN_ORDER.)

`JOIN_TYPE='INNER'` here because both date FKs are typically declared NOT NULL on the fact. See "Join type auto-resolution" below.

Then DIMENSIONS rows duplicate per role with role-prefixed `VIRTUAL_COL`:

```
DIMENSIONS:
  (..., internet_sales, 'OrderDate Year', 'ADVENTUREWORKS.DIMDATE', 'CALENDARYEAR', 'dim5', ...)
  (..., internet_sales, 'ShipDate Year',  'ADVENTUREWORKS.DIMDATE', 'CALENDARYEAR', 'dim6', ...)
```

The `OrderDate ` prefix is convention; the actual prefix comes from the LLM-derived or override-supplied role label.

Without role labels, the structural pass produces just one dim row for the first role found and skips the other — silently wrong. The skill SHOULD detect role-playing dims and require either LLM enrichment OR a `dimensions_overrides` YAML before proceeding.

## Join type auto-resolution

The DDL column default for `SQLCUBE_REGISTRY.JOINS.JOIN_TYPE` is `'LEFT'`, but the studio backend's `_resolve_join_type()` (in `studio/backend/services/deployer.py`) overrides per-row at deploy time. Logic:

```
if user supplied JOIN_TYPE explicitly (INNER or LEFT) → keep as-is
else if fact's FK column is declared NOT NULL → use INNER
else → use LEFT
```

Why: a NOT NULL FK guarantees every fact row matches a dim row, so INNER and LEFT return identical row counts — but INNER lets Exasol's optimizer pick hash-join shapes more aggressively than it would for a safety-margin LEFT.

Practical implication for the skill:

- The skill should NOT manually pick LEFT for every JOIN row. Let the resolver handle it.
- After `exasol-optimize` runs and ALTERs FKs to NOT NULL (when data permits), re-deploying the cube will flip LEFT → INNER automatically on those joins. This is desirable.
- If a JOINS row needs a specific type regardless of nullability (e.g., a deliberate outer join for orphan-row analysis), set `JOIN_PATHS.JOIN_TYPE = 'LEFT'` explicitly; the resolver respects explicit values and skips the auto-detect.

`_is_fk_not_null()` queries `SYS.EXA_ALL_COLUMNS.COLUMN_IS_NULLABLE` at deploy time. The check happens once per JOIN_PATH per deploy; not a hot path.

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
