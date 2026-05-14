# Cube enrichment (main prompt)

The big one. Single LLM call per cube. Generates model description, dimension/fact business names, additional measures, relationship renames, and column display names. Output drives the bulk of SQLCUBE_META rows.

## When used

`llm_enrichment=true` (default). After structural defaults are computed but before SQLCUBE_META INSERTs run.

## System prompt

```
You are designing the business surface of a semantic layer over a database schema.

Your output will be inserted into a metadata catalog (SQLCUBE_META) that powers natural-language
query interfaces. Analysts will see your descriptions and measure names. Be specific, concise,
and grounded in the actual schema. Never invent tables or columns.

Output strict JSON conforming to the requested schema. No prose, no markdown, no commentary.
```

## User prompt template

```
Schema name: <SOURCE_SCHEMA>
Domain: <DOMAIN_FROM_domain-detection.md_OR_user_supplied>
Sub-domain: <SUB_DOMAIN_OR_unspecified>

=== Tables ===

<FOR_EACH_TABLE>
<TABLE_NAME> (classified as <DIM_OR_FACT>):
  PK: <PK_COLUMN_OR_none>
  Columns:
  <FOR_EACH_COLUMN>
    - <COL_NAME> (<DATA_TYPE>)<IF_FK> → FK to <REFERENCED_TABLE>.<REFERENCED_COLUMN></IF_FK>
  </FOR_EACH_COLUMN>
  Sample (5 rows, PII columns redacted):
<PIPE_DELIMITED_SAMPLE_OR_no-samples-available>
</FOR_EACH_TABLE>

=== Discovered relationships (from optimize) ===

<FOR_EACH_RELATIONSHIP>
- <FACT_TABLE>.<FK_COL> → <DIM_TABLE>.<PK_COL>  (role: <ROLE_OR_none>)
</FOR_EACH_RELATIONSHIP>

=== Structural measures (default aggregations already generated) ===

<FOR_EACH_DEFAULT_MEASURE>
- <FACT>.<MEASURE_NAME>: <EXPRESSION> [<AGGREGATION>]
</FOR_EACH_DEFAULT_MEASURE>

=== Output JSON schema ===

{
  "model_description": "<2–4 sentences describing this cube's domain coverage>",

  "dimensions": [
    {
      "physical_table": "<TABLE_NAME>",
      "business_name": "<readable name, Title Case, no underscores>",
      "description": "<one sentence>",
      "columns": [
        {
          "physical_column": "<COL_NAME>",
          "display_name": "<Title Case>",
          "description": "<one sentence or empty string>"
        }
      ]
    }
  ],

  "facts": [
    {
      "physical_table": "<TABLE_NAME>",
      "business_name": "<readable name>",
      "grain": "<one sentence — what does ONE row represent?>",
      "columns": [
        {
          "physical_column": "<COL_NAME>",
          "display_name": "<Title Case>",
          "description": "<one sentence or empty string>"
        }
      ]
    }
  ],

  "additional_measures": [
    {
      "fact": "<FACT_BUSINESS_NAME>",
      "name": "<measure name, Title Case, distinct from defaults>",
      "expression": "<SQL expression, references columns in fact OR joined dims; no SUM()/AVG() wrapper>",
      "aggregation": "SUM" | "AVG" | "COUNT" | "COUNT_DISTINCT" | "MIN" | "MAX",
      "format": "currency_usd" | "currency_eur" | "percent" | "integer" | "decimal_2" | "decimal_4" | "bytes" | "duration_seconds",
      "description": "<one sentence>"
    }
  ],

  "relationship_renames": [
    {
      "fact": "<FACT_PHYSICAL_TABLE>",
      "dim": "<DIM_PHYSICAL_TABLE>",
      "fk_column": "<FACT_FK_COLUMN>",
      "new_name": "<business-friendly relationship name>"
    }
  ]
}

=== Rules ===

1. NEVER reference tables / columns that aren't in the schema above. The system will reject
   hallucinated identifiers.
2. Grain descriptions are critical. For each fact, say what "one row" means with specifics
   (e.g., "One row per order line item, capturing quantity and price").
3. additional_measures is your chance to add business-specific aggregations beyond the structural
   defaults. Examples: gross-margin formulas, customer-acquisition cost, net retention rates.
   Don't repeat the defaults. Don't propose measures the data can't support.
4. expression must be valid Exasol SQL referencing only listed columns. Use NULLIF for divisors.
5. Relationship renames are optional. Only suggest when the default name (which is the FK column
   stem) is misleading. Skip the field entirely when defaults are fine.
6. Empty descriptions are fine for column rows where the column name already speaks for itself
   (e.g., FIRST_NAME → just display_name "First Name", empty description).
7. Cap additional_measures at 5 per fact. Quality over quantity.
8. NEVER include PII guesses. If a column was redacted in samples, don't fabricate what it means.
9. NEVER produce DDL, INSERT statements, or any executable SQL outside `expression` fields.
```

## Validation

The skill runs each LLM output through these checks before any SQLCUBE_META INSERT:

1. **Identifier resolution.** Every `physical_table` and `physical_column` must exist in
   `SYS.EXA_ALL_TABLES` / `SYS.EXA_ALL_COLUMNS` for the source schema. Drop hallucinated entries
   to a `rejected.log` file alongside the manifest.

2. **Expression parses.** Each `additional_measures.expression` is dry-run:
   ```sql
   SELECT <expression> FROM <source_schema>.<physical_fact> LIMIT 0;
   ```
   Errors → drop the measure, log to `rejected.log`.

3. **No duplicate measure names.** If LLM proposed `Total Revenue` and the structural defaults
   already have `Total Revenue` → drop the LLM version.

4. **Format hint allowlist.** Only the 8 listed values. Anything else → default to `integer`
   for SUM/COUNT, `decimal_2` for AVG.

5. **Business name conflict.** Two dimensions / facts cannot share a `business_name`. On
   collision, keep the longer-named entity (more descriptive), demote the other to add a
   disambiguator (`Customer (Account)`).

6. **Description length.** > 500 chars → truncate at first sentence boundary. Empty string
   passes through to SQLCUBE_META as NULL.

## Storage

Validated rows get distributed across SQLCUBE_META tables per `meta-model.md`. The full mapping:

| Output field | SQLCUBE_META target |
|---|---|
| `model_description` | MODELS.DESCRIPTION |
| `dimensions[*].business_name` | DIMENSIONS.DIM_NAME |
| `dimensions[*].description` | DIMENSIONS.DESCRIPTION |
| `facts[*].business_name` | FACTS.FACT_NAME |
| `facts[*].grain` | FACTS.GRAIN_DESCRIPTION |
| `dimensions[*].columns / facts[*].columns` | COLUMNS rows |
| `additional_measures` | MEASURES rows (new, beyond defaults) |
| `relationship_renames` | RELATIONSHIPS.RELATIONSHIP_NAME updates |

User-supplied overrides (`measures_overrides.yaml` / `dimensions_overrides.yaml`) apply AFTER
LLM output and win on conflict.

## Cost

Token budget scales with schema size. Rough numbers per call:

| Tables | Tokens in | Tokens out | Cost (Haiku) | Cost (Sonnet) |
|---|---|---|---|---|
| 5 | ~3K | ~1K | ~$0.01 | ~$0.05 |
| 10 | ~6K | ~2K | ~$0.02 | ~$0.10 |
| 20 | ~12K | ~4K | ~$0.04 | ~$0.20 |
| 50 | ~30K | ~10K | ~$0.10 | ~$0.50 |

Schemas > 50 tables: split into multiple cube-creation invocations OR pre-supply overrides for
the well-understood subset, restricting the LLM to the unknown portion.

## Example output (truncated)

For ADVENTUREWORKS with 4 tables:

```json
{
  "model_description": "AdventureWorks sales cube. Covers customer demographics, product catalog, and order-line revenue. Time dimension supports order-date and ship-date roles.",
  "dimensions": [
    {
      "physical_table": "DIM_CUSTOMER",
      "business_name": "Customer",
      "description": "Individual and corporate buyers, including segment and geography.",
      "columns": [
        {"physical_column": "FIRST_NAME", "display_name": "First Name", "description": ""},
        {"physical_column": "SEGMENT", "display_name": "Segment", "description": "Marketing segment classification (Consumer, Corporate, Home Office)."}
      ]
    },
    ...
  ],
  "facts": [
    {
      "physical_table": "FACT_SALES",
      "business_name": "Sales",
      "grain": "One row per order line item, capturing quantity, unit price, discount, and tax for a specific product on an order date.",
      "columns": [
        {"physical_column": "QUANTITY", "display_name": "Quantity", "description": ""},
        {"physical_column": "UNIT_PRICE", "display_name": "Unit Price", "description": "Catalog price at time of order."}
      ]
    }
  ],
  "additional_measures": [
    {
      "fact": "Sales",
      "name": "Net Revenue",
      "expression": "QUANTITY * UNIT_PRICE * (1 - DISCOUNT) - TAX",
      "aggregation": "SUM",
      "format": "currency_usd",
      "description": "Revenue after discount and tax."
    },
    {
      "fact": "Sales",
      "name": "Avg Discount Rate",
      "expression": "DISCOUNT",
      "aggregation": "AVG",
      "format": "percent",
      "description": "Average per-line discount rate."
    }
  ],
  "relationship_renames": [
    {"fact": "FACT_SALES", "dim": "DIM_DATE", "fk_column": "ORDER_DATE", "new_name": "Order Date"},
    {"fact": "FACT_SALES", "dim": "DIM_DATE", "fk_column": "SHIP_DATE", "new_name": "Ship Date"}
  ]
}
```
