# SQLCUBE_REGISTRY data model

Exact schema and population rules for the registry tables. Companion to `architecture.md` (big picture) and `cube-creation.md` (write orchestration).

Canonical source: `factory-foundation/sqlcube/meta/01_create_tables.sql` and `factory-foundation/studio/backend/services/sqlcube_builder.py:build_registry_ddl()`. Treat those as ground truth; this file mirrors them.

## Schema DDL

Created once at install time. Agents do NOT recreate — only INSERT / UPDATE / DELETE rows. The schema name is `SQLCUBE_REGISTRY` by default, overridable via `settings.exa_registry_schema` in the studio backend.

```sql
CREATE SCHEMA IF NOT EXISTS SQLCUBE_REGISTRY;

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.DOMAINS (
    DOMAIN_ID       VARCHAR(100)  NOT NULL,
    DOMAIN_NAME     VARCHAR(200),
    DESCRIPTION     VARCHAR(2000),
    PRIMARY KEY (DOMAIN_ID)
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.MODELS (
    MODEL_ID          VARCHAR(100)  NOT NULL,
    DOMAIN_ID         VARCHAR(100),
    MODEL_LABEL       VARCHAR(200),
    FACT_SCHEMA       VARCHAR(100)  NOT NULL,
    FACT_TABLE        VARCHAR(100)  NOT NULL,
    GRAIN_KEY         VARCHAR(500),
    OWNER_NAME        VARCHAR(200),
    OWNER_TYPE        VARCHAR(50),
    IS_SYSTEM_MANAGED BOOLEAN       DEFAULT FALSE,
    IS_ACTIVE         BOOLEAN       DEFAULT TRUE,
    VERSION           INTEGER       DEFAULT 1,
    PRIMARY KEY (MODEL_ID),
    FOREIGN KEY (DOMAIN_ID) REFERENCES SQLCUBE_REGISTRY.DOMAINS (DOMAIN_ID)
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.DIMENSIONS (
    DIM_ID          INTEGER       NOT NULL,
    MODEL_ID        VARCHAR(100)  NOT NULL,
    VIRTUAL_COL     VARCHAR(100)  NOT NULL,
    PHYSICAL_TABLE  VARCHAR(200)  NOT NULL,
    PHYSICAL_COL    VARCHAR(100)  NOT NULL,
    DIM_TABLE_ALIAS VARCHAR(20)   NOT NULL,
    DATA_TYPE       VARCHAR(50),
    IS_VISIBLE      BOOLEAN       DEFAULT TRUE,
    SORT_ORDER      INTEGER       DEFAULT 0,
    PRIMARY KEY (MODEL_ID, VIRTUAL_COL),
    FOREIGN KEY (MODEL_ID) REFERENCES SQLCUBE_REGISTRY.MODELS (MODEL_ID)
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.MEASURES (
    MEASURE_ID      INTEGER       NOT NULL,
    MODEL_ID        VARCHAR(100)  NOT NULL,
    VIRTUAL_COL     VARCHAR(100)  NOT NULL,
    AGG_TYPE        VARCHAR(20)   NOT NULL,
    PHYSICAL_EXPR   VARCHAR(2000) NOT NULL,
    FILTER_EXPR     VARCHAR(2000),
    DATA_TYPE       VARCHAR(50),
    FORMAT_MASK     VARCHAR(50),
    IS_VISIBLE      BOOLEAN       DEFAULT TRUE,
    PRIMARY KEY (MODEL_ID, VIRTUAL_COL),
    FOREIGN KEY (MODEL_ID) REFERENCES SQLCUBE_REGISTRY.MODELS (MODEL_ID)
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.DERIVED_MEASURES (
    DERIVED_ID      INTEGER       NOT NULL,
    MODEL_ID        VARCHAR(100)  NOT NULL,
    VIRTUAL_COL     VARCHAR(100)  NOT NULL,
    FORMULA         VARCHAR(2000) NOT NULL,
    DEPENDS_ON      VARCHAR(2000) NOT NULL,
    DATA_TYPE       VARCHAR(50),
    FORMAT_MASK     VARCHAR(50),
    IS_VISIBLE      BOOLEAN       DEFAULT TRUE,
    PRIMARY KEY (MODEL_ID, VIRTUAL_COL),
    FOREIGN KEY (MODEL_ID) REFERENCES SQLCUBE_REGISTRY.MODELS (MODEL_ID)
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.JOINS (
    JOIN_ID         INTEGER       NOT NULL,
    MODEL_ID        VARCHAR(100)  NOT NULL,
    DIM_TABLE       VARCHAR(200)  NOT NULL,
    DIM_ALIAS       VARCHAR(20)   NOT NULL,
    DIM_KEY         VARCHAR(100)  NOT NULL,
    FACT_FK         VARCHAR(100)  NOT NULL,
    JOIN_TYPE       VARCHAR(10)   DEFAULT 'LEFT',
    JOIN_ORDER      INTEGER       NOT NULL,
    PRIMARY KEY (MODEL_ID, JOIN_ORDER),
    FOREIGN KEY (MODEL_ID) REFERENCES SQLCUBE_REGISTRY.MODELS (MODEL_ID)
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.ATTRIBUTES (
    ATTR_ID         VARCHAR(100)  NOT NULL,
    DOMAIN_ID       VARCHAR(100)  NOT NULL,
    BUSINESS_NAME   VARCHAR(200),
    ATTR_TYPE       VARCHAR(20),
    PHYSICAL_TABLE  VARCHAR(200),
    PHYSICAL_COL    VARCHAR(200),
    AGG_FUNCTION    VARCHAR(50),
    FORMULA         VARCHAR(2000),
    DATA_TYPE       VARCHAR(50),
    DISPLAY_TYPE    VARCHAR(20),
    DISPLAY_SCALE   DECIMAL(9,0),
    IS_VISIBLE      BOOLEAN       DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.JOIN_PATHS (
    JOIN_ID         VARCHAR(100)  NOT NULL,
    DOMAIN_ID       VARCHAR(100)  NOT NULL,
    FACT_TABLE      VARCHAR(200),
    DIM_TABLE       VARCHAR(200),
    FACT_KEY_COL    VARCHAR(200),
    DIM_KEY_COL     VARCHAR(200),
    JOIN_TYPE       VARCHAR(10)   DEFAULT 'LEFT'
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.RCLS_ROW_POLICIES (
    POLICY_ID         INTEGER       NOT NULL,
    MODEL_ID          VARCHAR(100)  NOT NULL,
    SUBJECT_TYPE      VARCHAR(20)   NOT NULL,
    SUBJECT_NAME      VARCHAR(200)  NOT NULL,
    EFFECT            VARCHAR(10)   DEFAULT 'ALLOW',
    ROW_PREDICATE_SQL VARCHAR(4000) NOT NULL,
    PRIORITY          INTEGER       DEFAULT 100,
    IS_ACTIVE         BOOLEAN       DEFAULT TRUE,
    PRIMARY KEY (MODEL_ID, POLICY_ID),
    FOREIGN KEY (MODEL_ID) REFERENCES SQLCUBE_REGISTRY.MODELS (MODEL_ID)
);

CREATE TABLE IF NOT EXISTS SQLCUBE_REGISTRY.RCLS_ATTRIBUTE_POLICIES (
    POLICY_ID         INTEGER       NOT NULL,
    MODEL_ID          VARCHAR(100)  NOT NULL,
    VIRTUAL_COL       VARCHAR(100)  NOT NULL,
    SUBJECT_TYPE      VARCHAR(20)   NOT NULL,
    SUBJECT_NAME      VARCHAR(200)  NOT NULL,
    EFFECT            VARCHAR(10)   DEFAULT 'DENY',
    IS_ACTIVE         BOOLEAN       DEFAULT TRUE,
    PRIMARY KEY (MODEL_ID, POLICY_ID),
    FOREIGN KEY (MODEL_ID) REFERENCES SQLCUBE_REGISTRY.MODELS (MODEL_ID)
);
```

If a check returns `object SQLCUBE_REGISTRY.X not found` → route to `install.md`.

## Population semantics

### Upsert by `DOMAIN_ID`

The skill never tries to diff. It deletes everything for a domain (in FK-correct order) then re-inserts. Reference: `sqlcube_builder.py:build_registry_sql()`.

Delete order (each scoped to a single `DOMAIN_ID`):

```sql
DELETE FROM SQLCUBE_REGISTRY.DERIVED_MEASURES WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<dom>');
DELETE FROM SQLCUBE_REGISTRY.JOINS            WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<dom>');
DELETE FROM SQLCUBE_REGISTRY.MEASURES         WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<dom>');
DELETE FROM SQLCUBE_REGISTRY.DIMENSIONS       WHERE MODEL_ID IN (SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS WHERE DOMAIN_ID = '<dom>');
DELETE FROM SQLCUBE_REGISTRY.MODELS           WHERE DOMAIN_ID = '<dom>';
DELETE FROM SQLCUBE_REGISTRY.JOIN_PATHS       WHERE DOMAIN_ID = '<dom>';
DELETE FROM SQLCUBE_REGISTRY.ATTRIBUTES       WHERE DOMAIN_ID = '<dom>';
DELETE FROM SQLCUBE_REGISTRY.DOMAINS          WHERE DOMAIN_ID = '<dom>';
```

Then INSERT order:

```
DOMAINS → ATTRIBUTES → JOIN_PATHS → MODELS → DIMENSIONS → MEASURES → DERIVED_MEASURES → JOINS
```

RCLS_* tables are not touched by this skill. They're populated by a separate security skill (planned).

### Population by table

#### DOMAINS — one row per business grouping

```sql
INSERT INTO SQLCUBE_REGISTRY.DOMAINS (DOMAIN_ID, DOMAIN_NAME, DESCRIPTION)
VALUES (
  'internet_sales',
  'Internet Sales',
  'Direct-to-consumer online sales cube. One row per order line; supports product, customer, geography, time, and promotion attribution.'
);
```

`DOMAIN_ID` is the canonical primary key — keep it short and stable. Tooling joins on it constantly.

#### MODELS — one row per fact-table grain

```sql
INSERT INTO SQLCUBE_REGISTRY.MODELS
  (MODEL_ID, DOMAIN_ID, MODEL_LABEL, FACT_SCHEMA, FACT_TABLE, GRAIN_KEY,
   OWNER_NAME, OWNER_TYPE, IS_SYSTEM_MANAGED, IS_ACTIVE, VERSION)
VALUES (
  'factinternetsales',
  'internet_sales',
  'Internet Sales',
  'ADVENTUREWORKS',
  'FACTINTERNETSALES',
  'SALESORDERNUMBER,SALESORDERLINENUMBER',
  NULL, NULL, FALSE, TRUE, 1
);
```

- `MODEL_ID`: short slug, lowercase. Becomes the virtual table name inside the virtual schema.
- `DOMAIN_ID`: FK into DOMAINS.
- `MODEL_LABEL`: human display name (shown in tooling).
- `FACT_SCHEMA` + `FACT_TABLE`: fully qualified physical reference.
- `GRAIN_KEY`: comma-separated column list that uniquely identifies a row in the fact. Used by the adapter for sanity-checking and for `COUNT_DISTINCT` shortcuts.
- `IS_SYSTEM_MANAGED`: TRUE for system-generated rollup models, FALSE for user-defined. Default FALSE.
- `IS_ACTIVE`: FALSE soft-disables the model without deleting metadata.

#### DIMENSIONS — one row per virtual column from a dim

```sql
INSERT INTO SQLCUBE_REGISTRY.DIMENSIONS
  (DIM_ID, MODEL_ID, VIRTUAL_COL, PHYSICAL_TABLE, PHYSICAL_COL,
   DIM_TABLE_ALIAS, DATA_TYPE, IS_VISIBLE, SORT_ORDER)
VALUES
  (1, 'factinternetsales', 'Englishproductname', 'ADVENTUREWORKS.DIMPRODUCT',  'ENGLISHPRODUCTNAME', 'dim',  'VARCHAR(50) UTF8',  TRUE, 10),
  (2, 'factinternetsales', 'Maritalstatus',      'ADVENTUREWORKS.DIMCUSTOMER', 'MARITALSTATUS',      'dim1', 'VARCHAR(1) UTF8',   FALSE, 160),
  (3, 'factinternetsales', 'Calendaryear',       'ADVENTUREWORKS.DIMDATE',     'CALENDARYEAR',       'dim2', 'DECIMAL(4,0)',      TRUE, 20);
```

Critical fields:

- `VIRTUAL_COL`: how analysts and MCP see the column. Title-case-with-trailing-lowercase by convention (e.g., `Englishproductname`). Display layer can rename via ATTRIBUTES.
- `PHYSICAL_TABLE` is `<SCHEMA>.<TABLE>` fully qualified.
- `DIM_TABLE_ALIAS` is the alias the adapter uses in the generated SQL — must match a `JOINS.DIM_ALIAS` for the same model. `dim` / `dim1` / `dim2` is the convention; the adapter doesn't care about the exact name as long as alias-graph consistency holds.
- `IS_VISIBLE = FALSE` keeps the column in the registry but hides it from query surfaces (the adapter filters by this flag during `load_model_meta()`).
- `SORT_ORDER` controls display ordering in tooling. Has no effect on query results.

#### MEASURES — one row per aggregable column

```sql
INSERT INTO SQLCUBE_REGISTRY.MEASURES
  (MEASURE_ID, MODEL_ID, VIRTUAL_COL, AGG_TYPE,
   PHYSICAL_EXPR, FILTER_EXPR, DATA_TYPE, FORMAT_MASK, IS_VISIBLE)
VALUES
  (1, 'factinternetsales', 'Extended Amount',  'SUM',            'f.EXTENDEDAMOUNT',    NULL, 'DECIMAL(19,4)',   '$#,##0.00', TRUE),
  (2, 'factinternetsales', 'Order Count',      'COUNT_DISTINCT', 'f.SALESORDERNUMBER',  NULL, 'VARCHAR(20) UTF8', '#,##0',     TRUE),
  (3, 'factinternetsales', 'Gross Margin %',   'AVG',            '(f.UNITPRICE - f.TOTALPRODUCTCOST) / NULLIF(f.UNITPRICE,0)', NULL, 'DECIMAL(9,4)', '0.00%', TRUE);
```

Key conventions:

- `PHYSICAL_EXPR` is the INNER expression. The adapter wraps it in `AGG_TYPE(...)` at query time. Don't include `SUM(...)` in `PHYSICAL_EXPR`.
- Use the alias `f` for the fact table. The adapter generates `FROM <FACT_SCHEMA>.<FACT_TABLE> f`. Dim columns can be referenced via their `DIM_TABLE_ALIAS` from DIMENSIONS rows.
- `AGG_TYPE` allowed values: `SUM`, `COUNT`, `COUNT_DISTINCT`, `AVG`, `MIN`, `MAX`. `NONE` is reserved for non-aggregable derived expressions (rare).
- `FILTER_EXPR` is an optional WHERE-style condition applied only when this measure is requested. e.g. `f.STATUS = 'COMPLETED'`.
- `FORMAT_MASK` is a display hint for tooling (Excel-style format codes). The adapter doesn't apply it server-side.

#### DERIVED_MEASURES — formulas over other measures

```sql
INSERT INTO SQLCUBE_REGISTRY.DERIVED_MEASURES
  (DERIVED_ID, MODEL_ID, VIRTUAL_COL, FORMULA, DEPENDS_ON,
   DATA_TYPE, FORMAT_MASK, IS_VISIBLE)
VALUES (
  1, 'factinternetsales', 'Gross Margin',
  '"Extended Amount" - "Total Product Cost"',
  'Extended Amount,Total Product Cost',
  'DECIMAL(19,4)', '$#,##0.00', TRUE
);
```

- `FORMULA` references measure VIRTUAL_COLs by their display name. Quote with double quotes.
- `DEPENDS_ON` is the comma-separated list of measure names the formula needs. Used for query planning and dependency validation.
- v0.1 of this skill writes DERIVED_MEASURES only when explicitly supplied via overrides — auto-generation is out of scope.

#### JOINS — ordered join graph per model

```sql
INSERT INTO SQLCUBE_REGISTRY.JOINS
  (JOIN_ID, MODEL_ID, DIM_TABLE, DIM_ALIAS, DIM_KEY, FACT_FK, JOIN_TYPE, JOIN_ORDER)
VALUES
  (1, 'factinternetsales', 'ADVENTUREWORKS.DIMPRODUCT',           'dim',  'PRODUCTKEY',         'PRODUCTKEY',         'LEFT', 10),
  (2, 'factinternetsales', 'ADVENTUREWORKS.DIMCUSTOMER',          'dim1', 'CUSTOMERKEY',        'CUSTOMERKEY',        'LEFT', 20),
  (3, 'factinternetsales', 'ADVENTUREWORKS.DIMDATE',              'dim2', 'DATEKEY',            'ORDERDATEKEY',       'LEFT', 30),
  (4, 'factinternetsales', 'ADVENTUREWORKS.DIMDATE',              'dim3', 'DATEKEY',            'SHIPDATEKEY',        'LEFT', 31),
  (5, 'factinternetsales', 'ADVENTUREWORKS.DIMSALESTERRITORY',    'dim4', 'SALESTERRITORYKEY',  'SALESTERRITORYKEY',  'LEFT', 40);
```

Critical:

- `DIM_ALIAS` must match a DIMENSIONS row's `DIM_TABLE_ALIAS` for any column you want to expose from that join. Aliases are globally per-model: a dim joined twice (role-playing dates) gets two JOINS rows with distinct aliases (`dim2` for order date, `dim3` for ship date).
- `DIM_KEY` is the PK column on the dim; `FACT_FK` is the FK column on the fact.
- `JOIN_TYPE`: `LEFT` (default, keeps unmatched fact rows), `INNER` (drops them), or `RIGHT` / `FULL` (rare). The adapter emits the exact JOIN_TYPE.
- `JOIN_ORDER`: lower numbers first. Order matters when joins depend on intermediate dims (snowflakes). For flat star schemas, order is cosmetic.

#### ATTRIBUTES — domain-level display labels

```sql
INSERT INTO SQLCUBE_REGISTRY.ATTRIBUTES
  (ATTR_ID, DOMAIN_ID, BUSINESS_NAME, ATTR_TYPE,
   PHYSICAL_TABLE, PHYSICAL_COL, AGG_FUNCTION, FORMULA,
   DATA_TYPE, DISPLAY_TYPE, DISPLAY_SCALE, IS_VISIBLE)
VALUES
  ('dimcustomer_firstname', 'internet_sales', 'First Name', 'dimension',
   'ADVENTUREWORKS.DIMCUSTOMER', 'FIRSTNAME', NULL, NULL,
   'VARCHAR(50) UTF8', 'text', 0, TRUE),
  ('dimproduct_color', 'internet_sales', 'Color', 'dimension',
   'ADVENTUREWORKS.DIMPRODUCT', 'COLOR', NULL, NULL,
   'VARCHAR(15) UTF8', 'text', 0, TRUE);
```

Purpose: domain-level naming layer that sits above DIMENSIONS. Lets you share display names across models (every model in the domain that exposes `DIMCUSTOMER.FIRSTNAME` picks up the "First Name" label).

- `ATTR_TYPE`: `dimension` for browsable values, `measure` for aggregable physicals (less common at this layer).
- `DISPLAY_TYPE`: `text`, `integer`, `decimal`, `currency`, `percent`, `date`, `datetime`. Hint for UI rendering.
- `DISPLAY_SCALE`: decimal places for numeric types.

DIMENSIONS rows still need to be inserted per-model to actually surface the column inside a model's query result. ATTRIBUTES alone is not enough; it's the naming layer, not the inclusion layer.

#### JOIN_PATHS — domain-level alternative graph

```sql
INSERT INTO SQLCUBE_REGISTRY.JOIN_PATHS
  (JOIN_ID, DOMAIN_ID, FACT_TABLE, DIM_TABLE, FACT_KEY_COL, DIM_KEY_COL, JOIN_TYPE)
VALUES
  ('factinternetsales_dimdate_orderdatekey', 'internet_sales',
   'ADVENTUREWORKS.FACTINTERNETSALES', 'ADVENTUREWORKS.DIMDATE',
   'ORDERDATEKEY', 'DATEKEY', 'LEFT'),
  ('factinternetsales_dimdate_shipdatekey', 'internet_sales',
   'ADVENTUREWORKS.FACTINTERNETSALES', 'ADVENTUREWORKS.DIMDATE',
   'SHIPDATEKEY', 'DATEKEY', 'LEFT');
```

Same idea as JOINS but at the domain level rather than the model level. Used by the adapter / studio backend for "join discoverability" (UI suggests possible relationships for a fact across all models). The skill writes these alongside MODELS.JOINS for consistency.

JOIN_ID values are string-prefixed (not integer) — convention is `<fact_table_lower>_<dim_table_lower>_<fact_fk_lower>`.

## Querying the registry directly

Treat the registry as a queryable system catalog:

```sql
-- All cubes:
SELECT MODEL_ID, MODEL_LABEL, DOMAIN_ID, FACT_SCHEMA || '.' || FACT_TABLE AS fact
FROM SQLCUBE_REGISTRY.MODELS WHERE IS_ACTIVE;

-- All measures for a model:
SELECT VIRTUAL_COL, AGG_TYPE, PHYSICAL_EXPR, FORMAT_MASK
FROM SQLCUBE_REGISTRY.MEASURES
WHERE MODEL_ID = 'factinternetsales' AND IS_VISIBLE
ORDER BY MEASURE_ID;

-- Join graph for a model:
SELECT JOIN_ORDER, DIM_TABLE, DIM_ALIAS, DIM_KEY, FACT_FK, JOIN_TYPE
FROM SQLCUBE_REGISTRY.JOINS
WHERE MODEL_ID = 'factinternetsales'
ORDER BY JOIN_ORDER;

-- All dims exposed by a model, with their aliases:
SELECT D.VIRTUAL_COL, D.PHYSICAL_TABLE, D.PHYSICAL_COL, D.DIM_TABLE_ALIAS
FROM SQLCUBE_REGISTRY.DIMENSIONS D
WHERE D.MODEL_ID = 'factinternetsales' AND D.IS_VISIBLE
ORDER BY D.SORT_ORDER;

-- Measure dependency sanity:
SELECT M.MODEL_ID, M.MEASURE_ID, M.VIRTUAL_COL, M.PHYSICAL_EXPR
FROM SQLCUBE_REGISTRY.MEASURES M
JOIN SQLCUBE_REGISTRY.MODELS MD ON M.MODEL_ID = MD.MODEL_ID
LEFT JOIN SYS.EXA_ALL_COLUMNS C
  ON C.COLUMN_SCHEMA = MD.FACT_SCHEMA
 AND C.COLUMN_TABLE  = MD.FACT_TABLE
 AND POSITION(C.COLUMN_NAME IN UPPER(M.PHYSICAL_EXPR)) > 0
WHERE C.COLUMN_NAME IS NULL  -- measure references something we couldn't verify
ORDER BY M.MODEL_ID;
```

That last one is approximate — it doesn't recursively resolve dim aliases. For full validation, dry-run each `PHYSICAL_EXPR` per `cube-creation.md` validation steps.

## YAML override shapes

`measures_overrides.yaml`:

```yaml
measures:
  - model: factinternetsales
    virtual_col: "Net Margin"
    agg_type: SUM
    physical_expr: "(f.UNITPRICE - f.TOTALPRODUCTCOST) * f.ORDERQUANTITY"
    data_type: "DECIMAL(19,4)"
    format_mask: "$#,##0.00"
    visible: true
```

`dimensions_overrides.yaml`:

```yaml
dimensions:
  - model: factinternetsales
    virtual_col: "Englishproductname"
    visible: true
    sort_order: 5
attributes:
  - attr_id: dimproduct_color
    business_name: "Bike Color"
    display_type: text
```

Override merge order (later wins on key collision):

```
1. Structural defaults (from optimize join graph)
2. LLM enrichment output
3. User-supplied YAML overrides
```

Key for measures: `(model, virtual_col)`. Key for dimensions: `(model, virtual_col)`. Key for attributes: `attr_id`.
