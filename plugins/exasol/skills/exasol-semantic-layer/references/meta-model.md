# SQLCUBE_META data model

Exact schema and population rules for the metadata tables. Companion to `architecture.md` (which sketches the big picture) and `cube-creation.md` (which orchestrates the writes).

## Schema DDL

The SQLCUBE_META schema is created once at install time. Reference DDL — agents should NOT recreate these tables, only INSERT/UPDATE/DELETE rows.

```sql
CREATE SCHEMA SQLCUBE_META;
OPEN SCHEMA SQLCUBE_META;

CREATE TABLE MODELS (
  MODEL_NAME         VARCHAR(128) NOT NULL,
  SOURCE_SCHEMA      VARCHAR(128) NOT NULL,
  DESCRIPTION        VARCHAR(2000),
  CREATED_AT         TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  UPDATED_AT         TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  OWNER              VARCHAR(128) DEFAULT CURRENT_USER,
  PRIMARY KEY (MODEL_NAME)
);

CREATE TABLE DIMENSIONS (
  MODEL_NAME         VARCHAR(128) NOT NULL,
  DIM_NAME           VARCHAR(128) NOT NULL,
  PHYSICAL_TABLE     VARCHAR(128) NOT NULL,
  PK_COLUMN          VARCHAR(128) NOT NULL,
  DESCRIPTION        VARCHAR(2000),
  PRIMARY KEY (MODEL_NAME, DIM_NAME),
  FOREIGN KEY (MODEL_NAME) REFERENCES MODELS(MODEL_NAME)
);

CREATE TABLE FACTS (
  MODEL_NAME         VARCHAR(128) NOT NULL,
  FACT_NAME          VARCHAR(128) NOT NULL,
  PHYSICAL_TABLE     VARCHAR(128) NOT NULL,
  GRAIN_DESCRIPTION  VARCHAR(2000),
  PRIMARY KEY (MODEL_NAME, FACT_NAME),
  FOREIGN KEY (MODEL_NAME) REFERENCES MODELS(MODEL_NAME)
);

CREATE TABLE MEASURES (
  MODEL_NAME         VARCHAR(128) NOT NULL,
  FACT_NAME          VARCHAR(128) NOT NULL,
  MEASURE_NAME       VARCHAR(128) NOT NULL,
  EXPRESSION         VARCHAR(2000) NOT NULL,
  AGGREGATION        VARCHAR(20)  NOT NULL,
  FORMAT_HINT        VARCHAR(50),
  DESCRIPTION        VARCHAR(2000),
  PRIMARY KEY (MODEL_NAME, FACT_NAME, MEASURE_NAME),
  FOREIGN KEY (MODEL_NAME, FACT_NAME) REFERENCES FACTS(MODEL_NAME, FACT_NAME)
);

CREATE TABLE RELATIONSHIPS (
  MODEL_NAME         VARCHAR(128) NOT NULL,
  RELATIONSHIP_NAME  VARCHAR(128) NOT NULL,
  FACT_NAME          VARCHAR(128) NOT NULL,
  DIM_NAME           VARCHAR(128) NOT NULL,
  FACT_FK_COLUMN     VARCHAR(128) NOT NULL,
  DIM_PK_COLUMN      VARCHAR(128) NOT NULL,
  ROLE_PLAYING_AS    VARCHAR(128),
  PRIMARY KEY (MODEL_NAME, RELATIONSHIP_NAME),
  FOREIGN KEY (MODEL_NAME, FACT_NAME) REFERENCES FACTS(MODEL_NAME, FACT_NAME),
  FOREIGN KEY (MODEL_NAME, DIM_NAME)  REFERENCES DIMENSIONS(MODEL_NAME, DIM_NAME)
);

CREATE TABLE COLUMNS (
  MODEL_NAME         VARCHAR(128) NOT NULL,
  ENTITY_KIND        VARCHAR(10)  NOT NULL,    -- 'DIM' | 'FACT'
  ENTITY_NAME        VARCHAR(128) NOT NULL,    -- DIM_NAME or FACT_NAME
  PHYSICAL_COLUMN    VARCHAR(128) NOT NULL,
  DISPLAY_NAME       VARCHAR(128) NOT NULL,
  DATA_TYPE          VARCHAR(50)  NOT NULL,
  DESCRIPTION        VARCHAR(2000),
  IS_HIDDEN          BOOLEAN      DEFAULT FALSE,
  PRIMARY KEY (MODEL_NAME, ENTITY_KIND, ENTITY_NAME, PHYSICAL_COLUMN)
);
```

If `SELECT 1 FROM SQLCUBE_META.MODELS LIMIT 1` errors with "schema does not exist," route to `install.md`.

## Population by table

### MODELS — one row per cube

```sql
INSERT INTO SQLCUBE_META.MODELS (MODEL_NAME, SOURCE_SCHEMA, DESCRIPTION)
VALUES (
  'ADVENTUREWORKS_CUBE',
  'ADVENTUREWORKS',
  'Sales-and-customer cube derived from the AdventureWorks demo schema. Covers order lines, customer demographics, product hierarchy, and time.'
);
```

Description either LLM-generated or user-supplied. Empty string is fine if `llm_enrichment=false`; analysts see no business summary but the cube still functions.

### DIMENSIONS — one row per DIM table

```sql
INSERT INTO SQLCUBE_META.DIMENSIONS
  (MODEL_NAME, DIM_NAME, PHYSICAL_TABLE, PK_COLUMN, DESCRIPTION)
VALUES
  ('ADVENTUREWORKS_CUBE', 'Customer', 'DIM_CUSTOMER', 'CUSTOMER_ID',
   'Individual and corporate customers, including segment, geography, and credit standing.'),
  ('ADVENTUREWORKS_CUBE', 'Product',  'DIM_PRODUCT',  'PRODUCT_ID',
   'Catalog of finished goods with category, subcategory, brand, and pricing.'),
  ('ADVENTUREWORKS_CUBE', 'Date',     'DIM_DATE',     'DATE_ID',
   'Calendar dimension with year, quarter, month, week, day grain and fiscal year alignment.');
```

`DIM_NAME` is the business name; default to title-case of physical-stem (`DIM_CUSTOMER` → `Customer`). LLM enrichment may rewrite (`DIM_GEO` → `Region` if context suggests).

`PK_COLUMN` comes from the optimize-emitted constraints catalog. Always single-column in v0.1 — composite PKs not yet supported for dimensions.

### FACTS — one row per FACT table

```sql
INSERT INTO SQLCUBE_META.FACTS
  (MODEL_NAME, FACT_NAME, PHYSICAL_TABLE, GRAIN_DESCRIPTION)
VALUES (
  'ADVENTUREWORKS_CUBE',
  'Sales',
  'FACT_SALES',
  'One row per order line item — captures quantity, unit price, discount, tax, and freight per (order, product) pair on the order date.'
);
```

`GRAIN_DESCRIPTION` is the most important free-text field on a fact. Analysts referencing the cube want to know "what does one row mean." LLM should write this from a sample of FACT rows + column inventory. Conservative fallback when LLM absent: `"One row per <FACT_NAME>."`

### MEASURES — one row per business metric

```sql
INSERT INTO SQLCUBE_META.MEASURES
  (MODEL_NAME, FACT_NAME, MEASURE_NAME, EXPRESSION, AGGREGATION, FORMAT_HINT, DESCRIPTION)
VALUES
  ('ADVENTUREWORKS_CUBE', 'Sales', 'Total Revenue',
   'QUANTITY * UNIT_PRICE * (1 - DISCOUNT)', 'SUM', 'currency_usd',
   'Net revenue after line-item discount. Excludes tax and freight.'),

  ('ADVENTUREWORKS_CUBE', 'Sales', 'Order Count',
   'DISTINCT ORDER_ID', 'COUNT', 'integer',
   'Number of distinct orders.'),

  ('ADVENTUREWORKS_CUBE', 'Sales', 'Avg Discount',
   'DISCOUNT', 'AVG', 'percent',
   'Average per-line discount.');
```

Five rules for MEASURES:

1. `EXPRESSION` is the inner expression only — no `SUM(...)` wrapper. The `AGGREGATION` column tells the adapter what to wrap it in. This makes measures composable (you can rewrite `SUM → AVG` without touching the EXPRESSION).
2. `EXPRESSION` references columns in the fact OR joined dimensions. Multi-fact measures (`SUM(SALES.AMOUNT) - SUM(RETURNS.AMOUNT)`) need a v0.2 feature; flag and skip in v0.1.
3. `FORMAT_HINT` is consumed by dashboards (vega-lite axis formatting). Stable values: `currency_usd`, `currency_eur`, `percent`, `integer`, `decimal_2`, `decimal_4`, `bytes`, `duration_seconds`.
4. `AGGREGATION = 'NONE'` is for non-aggregable derived columns (rare). Almost always SUM / AVG / COUNT / MIN / MAX.
5. Measure names are business-facing. Avoid abbreviations. `Total Revenue` not `TotRev`. The model description in MODELS is the only "tone" the user sees up front — measure names carry the rest.

### RELATIONSHIPS — one row per fact↔dim join

```sql
INSERT INTO SQLCUBE_META.RELATIONSHIPS
  (MODEL_NAME, RELATIONSHIP_NAME, FACT_NAME, DIM_NAME, FACT_FK_COLUMN, DIM_PK_COLUMN, ROLE_PLAYING_AS)
VALUES
  ('ADVENTUREWORKS_CUBE', 'Sale by Customer',  'Sales', 'Customer', 'CUSTOMER_ID', 'CUSTOMER_ID', NULL),
  ('ADVENTUREWORKS_CUBE', 'Sale of Product',   'Sales', 'Product',  'PRODUCT_ID',  'PRODUCT_ID',  NULL),
  ('ADVENTUREWORKS_CUBE', 'Order Date',        'Sales', 'Date',     'ORDER_DATE',  'DATE_ID',     'OrderDate'),
  ('ADVENTUREWORKS_CUBE', 'Ship Date',         'Sales', 'Date',     'SHIP_DATE',   'DATE_ID',     'ShipDate'),
  ('ADVENTUREWORKS_CUBE', 'Due Date',          'Sales', 'Date',     'DUE_DATE',    'DATE_ID',     'DueDate');
```

`ROLE_PLAYING_AS` is non-null when the same dimension joins multiple times from one fact (the role-playing-date pattern that `exasol-optimize` discovers). In queries, the analyst references `Sales.ShipDate.year` to disambiguate from `OrderDate`.

`RELATIONSHIP_NAME` should be human-readable. Default: "`<Fact verb> by <Dim>`" or "`<Role> Date`" for date dims. LLM enriches with domain phrasing (`Customer` could become `Customer Who Placed Order`).

### COLUMNS — display metadata per column

```sql
INSERT INTO SQLCUBE_META.COLUMNS
  (MODEL_NAME, ENTITY_KIND, ENTITY_NAME, PHYSICAL_COLUMN, DISPLAY_NAME, DATA_TYPE, DESCRIPTION)
VALUES
  ('ADVENTUREWORKS_CUBE', 'DIM',  'Customer', 'FIRST_NAME',     'First Name',      'VARCHAR(50)', NULL),
  ('ADVENTUREWORKS_CUBE', 'DIM',  'Customer', 'LAST_NAME',      'Last Name',       'VARCHAR(50)', NULL),
  ('ADVENTUREWORKS_CUBE', 'DIM',  'Customer', 'SEGMENT',        'Segment',         'VARCHAR(20)', 'Marketing segment classification.'),
  ('ADVENTUREWORKS_CUBE', 'FACT', 'Sales',    'QUANTITY',       'Quantity',        'DECIMAL(18,2)', NULL),
  ('ADVENTUREWORKS_CUBE', 'FACT', 'Sales',    'UNIT_PRICE',     'Unit Price',      'DECIMAL(18,2)', 'List price at time of order.');
```

Skip FK and PK columns from COLUMNS (they're already encoded in DIMENSIONS/FACTS/RELATIONSHIPS). The adapter knows to hide them at query time.

`IS_HIDDEN = TRUE` for columns that exist in the physical table but shouldn't appear in the cube's surface area (legacy columns, internal flags). Skill never auto-hides; user must opt in via `dimensions_overrides`.

## Upserts vs deletes

The simple semantics: skill DELETEs all rows for a `MODEL_NAME` and re-INSERTs. Why:

- Cube definitions evolve atomically. Partial-update semantics would mean tracking what was previously inferred vs what user overrode — high complexity, low payoff.
- LLM enrichment is non-deterministic. Re-running might rephrase descriptions. Easier to wipe-and-rewrite than to diff.
- User overrides come from YAML files that are merged at write time, not stored separately.

The DROP CASCADE on the virtual schema must happen before the META wipe, since the adapter is reading from these tables.

## YAML override format

When user supplies `measures_overrides=measures.yaml`:

```yaml
measures:
  - fact: Sales
    name: Net Revenue
    expression: QUANTITY * UNIT_PRICE * (1 - DISCOUNT) - TAX
    aggregation: SUM
    format: currency_usd
    description: "Net of tax. Customer-facing definition agreed with Finance."

  - fact: Sales
    name: Gross Margin
    expression: (UNIT_PRICE - DIM_PRODUCT.STANDARD_COST) * QUANTITY
    aggregation: SUM
    format: currency_usd
    description: "Margin contribution before discounts."
```

Merge rule: by `(fact, name)`. User row wins. LLM-generated row with the same name is replaced.

For `dimensions_overrides`:

```yaml
dimensions:
  - physical_table: DIM_CUSTOMER
    display_name: Account
    description: "B2B customers only. Account = corporate buying entity."
    column_overrides:
      - physical_column: SEGMENT
        display_name: Account Tier
        hidden: false
      - physical_column: LEGACY_ID
        hidden: true
```

Most-common use: renaming an LLM hallucination (LLM called it `Region`, business calls it `Territory`).

## Querying SQLCUBE_META directly

When debugging a cube, the meta is its own SQL surface. Useful queries:

```sql
-- All cubes registered:
SELECT MODEL_NAME, SOURCE_SCHEMA, CREATED_AT FROM SQLCUBE_META.MODELS;

-- Measures for a cube:
SELECT FACT_NAME, MEASURE_NAME, EXPRESSION, AGGREGATION
FROM SQLCUBE_META.MEASURES
WHERE MODEL_NAME = 'ADVENTUREWORKS_CUBE'
ORDER BY FACT_NAME, MEASURE_NAME;

-- All relationships from a given fact:
SELECT RELATIONSHIP_NAME, DIM_NAME, FACT_FK_COLUMN, DIM_PK_COLUMN, ROLE_PLAYING_AS
FROM SQLCUBE_META.RELATIONSHIPS
WHERE MODEL_NAME = 'ADVENTUREWORKS_CUBE' AND FACT_NAME = 'Sales';

-- Find any measures pointing at non-existent columns:
SELECT m.MODEL_NAME, m.FACT_NAME, m.MEASURE_NAME, m.EXPRESSION
FROM SQLCUBE_META.MEASURES m
JOIN SQLCUBE_META.FACTS f
  ON m.MODEL_NAME = f.MODEL_NAME AND m.FACT_NAME = f.FACT_NAME
LEFT JOIN SYS.EXA_ALL_COLUMNS c
  ON c.COLUMN_TABLE = f.PHYSICAL_TABLE
WHERE c.COLUMN_TABLE IS NULL;
```

Treat SQLCUBE_META as a queryable, joinable system catalog. That's the whole point.
