# Cube enrichment (main prompt)

The big one. Single LLM call per cube. Generates domain description, model labels, attribute display names, additional measures, role-playing dim labels. Output maps directly to `SQLCUBE_REGISTRY` rows.

## When used

`llm_enrichment=true` (default). After structural defaults are computed but before INSERT into the registry.

## System prompt

```
You are designing the business surface of a semantic layer over an Exasol database schema.

Your output populates a metadata catalog (SQLCUBE_REGISTRY) that powers natural-language query
interfaces. Analysts will see your descriptions, business names, and measure proposals. Be
specific, concise, grounded in the actual schema. Never invent tables or columns.

Output strict JSON conforming to the requested schema. No prose, no markdown, no commentary.
```

## User prompt template

```
Source schema: <SOURCE_SCHEMA>
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

=== Discovered FK join graph ===

<FOR_EACH_DECLARED_FK>
- <FACT_SCHEMA>.<FACT_TABLE>.<FK_COL> → <DIM_SCHEMA>.<DIM_TABLE>.<PK_COL>
</FOR_EACH_DECLARED_FK>

=== Models being created ===

<FOR_EACH_MODEL>
- model_id: <MODEL_ID>
  fact_table: <FACT_SCHEMA>.<FACT_TABLE>
  domain_id: <DOMAIN_ID>
  grain_key: <COMMA_SEPARATED_PK_COLS>
</FOR_EACH_MODEL>

=== Structural measures (already generated, do not duplicate) ===

<FOR_EACH_DEFAULT_MEASURE>
- model=<MODEL_ID>, virtual_col=<VIRTUAL_COL>, agg_type=<AGG_TYPE>, physical_expr=<PHYSICAL_EXPR>
</FOR_EACH_DEFAULT_MEASURE>

=== Output JSON schema ===

{
  "domain": {
    "domain_id": "<provided in input, echo>",
    "domain_name": "<readable business name>",
    "description": "<2-4 sentences describing the domain's coverage>"
  },

  "models": [
    {
      "model_id": "<provided in input, echo>",
      "model_label": "<readable business name, no underscores>",
      "grain_description": "<one sentence — what one fact row represents>"
    }
  ],

  "attributes": [
    {
      "physical_table": "<SCHEMA.TABLE>",
      "physical_col": "<COL>",
      "business_name": "<Title Case, no underscores>",
      "description": "<one sentence or empty>",
      "display_type": "text" | "integer" | "decimal" | "currency" | "percent" | "date" | "datetime",
      "display_scale": 0
    }
  ],

  "additional_measures": [
    {
      "model_id": "<MODEL_ID>",
      "virtual_col": "<Business name, Title Case, distinct from defaults>",
      "agg_type": "SUM" | "AVG" | "COUNT" | "COUNT_DISTINCT" | "MIN" | "MAX",
      "physical_expr": "<inner expression using alias 'f' for fact and dim aliases as listed>",
      "filter_expr": "<optional WHERE-style condition, or empty>",
      "data_type": "<Exasol type literal, e.g. DECIMAL(19,4)>",
      "format_mask": "<Excel-style format, e.g. $#,##0.00 or 0.00%>"
    }
  ],

  "role_playing": [
    {
      "model_id": "<MODEL_ID>",
      "dim_table": "<SCHEMA.DIMTABLE>",
      "fact_fk_col": "<column on the fact, e.g. ORDERDATEKEY>",
      "role_label": "<short prefix, e.g. OrderDate, ShipDate>"
    }
  ]
}

=== Rules ===

1. NEVER reference tables / columns that aren't in the schema above. The system will reject
   hallucinated identifiers.
2. grain_description is critical. Say what "one row" means specifically (e.g., "One row per
   order line item, capturing quantity and price").
3. additional_measures is your chance for domain-specific aggregations beyond the structural
   defaults. Examples: gross margin, AOV, on-time rate, return rate. Don't repeat defaults.
4. physical_expr uses alias 'f' for the fact (the adapter generates `FROM <fact> f`). When
   referencing dim columns, use their dim alias from the JOINS structural pass (e.g., 'dim',
   'dim1') — the system supplies these in a follow-up validation pass; LLM doesn't need to
   pick aliases.
5. Use NULLIF for divisors. Always.
6. role_playing entries are required when a fact has two or more FKs to the same dim. The
   structural pass detects the FKs; you provide the role labels. If a fact has only one
   FK to a given dim, omit it from role_playing.
7. Empty descriptions OK when the business_name is self-evident (e.g., "First Name").
8. Cap additional_measures at 8 per model.
9. NEVER include PII guesses. PII columns are redacted before samples reach you.
10. NEVER produce DDL or DELETE / INSERT — only the JSON fields requested.
```

## Validation

Before any registry INSERT:

1. **Identifier resolution.** Every `physical_table` / `physical_col` referenced must exist in `SYS.EXA_ALL_TABLES` / `SYS.EXA_ALL_COLUMNS`. Drop hallucinated entries to `rejected.log`.

2. **Expression dry-run.** For each `additional_measures.physical_expr`, run:
   ```sql
   SELECT <physical_expr> FROM <fact_schema>.<fact_table> f LIMIT 0;
   ```
   Errors → drop the measure, log.

3. **No duplicates.** If LLM proposes a measure with `virtual_col` matching a structural default → drop the LLM version.

4. **Format mask validity.** Quick sanity check (must contain `0`, `#`, or one of `%$€`). Empty string is accepted.

5. **Agg/expr compatibility.** SUM on VARCHAR / BOOLEAN expression → reject. AVG / SUM on COUNT_DISTINCT result → reject.

6. **Role-playing required.** If a fact has multiple FKs to the same dim AND the LLM didn't supply `role_playing` entries, halt with: "Role-playing dimension detected (DIMDATE referenced 2x from FACTINTERNETSALES). Re-run with role labels or supply dimensions_overrides YAML."

## Storage mapping

| Output field | Registry destination |
|---|---|
| `domain.domain_name` | `SQLCUBE_REGISTRY.DOMAINS.DOMAIN_NAME` |
| `domain.description` | `SQLCUBE_REGISTRY.DOMAINS.DESCRIPTION` |
| `models[*].model_label` | `SQLCUBE_REGISTRY.MODELS.MODEL_LABEL` |
| `models[*].grain_description` | `SQLCUBE_REGISTRY.MODELS.GRAIN_KEY` (separate sentence stored in the structural pass as the column list); business description is stored at domain level since MODELS doesn't have a description column |
| `attributes[*].business_name` | `SQLCUBE_REGISTRY.ATTRIBUTES.BUSINESS_NAME` |
| `attributes[*].display_type` / `display_scale` | `SQLCUBE_REGISTRY.ATTRIBUTES.DISPLAY_TYPE` / `DISPLAY_SCALE` |
| `additional_measures[*]` | `SQLCUBE_REGISTRY.MEASURES` (new rows; the structural pass already wrote the defaults) |
| `role_playing[*]` | Influences `SQLCUBE_REGISTRY.JOINS` (extra rows for each role) and `SQLCUBE_REGISTRY.DIMENSIONS` (rows duplicated per role with role-prefixed VIRTUAL_COL) |

User-supplied YAML overrides apply AFTER LLM output and win on conflict (see `references/meta-model.md` YAML format).

## Cost

Token budget scales with schema size. Rough numbers per call:

| Tables | Tokens in | Tokens out | Cost (Haiku) | Cost (Sonnet) |
|---|---|---|---|---|
| 5 | ~3K | ~1K | ~$0.01 | ~$0.05 |
| 10 | ~6K | ~2K | ~$0.02 | ~$0.10 |
| 20 | ~12K | ~4K | ~$0.04 | ~$0.20 |
| 50 | ~30K | ~10K | ~$0.10 | ~$0.50 |

Schemas > 50 tables: split into multiple invocations OR pre-supply overrides.

## Example output (truncated)

For ADVENTUREWORKS with one fact + four dims:

```json
{
  "domain": {
    "domain_id": "internet_sales",
    "domain_name": "Internet Sales",
    "description": "Direct-to-consumer online sales cube. One row per order line item, with attribution to customer, product, date (order and ship), and sales territory."
  },
  "models": [
    {
      "model_id": "factinternetsales",
      "model_label": "Internet Sales",
      "grain_description": "One row per order line item — quantity, unit price, discount, tax, freight for a single product on a single order date."
    }
  ],
  "attributes": [
    {"physical_table": "ADVENTUREWORKS.DIMCUSTOMER", "physical_col": "FIRSTNAME", "business_name": "First Name", "description": "", "display_type": "text", "display_scale": 0},
    {"physical_table": "ADVENTUREWORKS.DIMPRODUCT", "physical_col": "COLOR", "business_name": "Color", "description": "Product color from catalog.", "display_type": "text", "display_scale": 0},
    {"physical_table": "ADVENTUREWORKS.DIMPRODUCT", "physical_col": "STANDARDCOST", "business_name": "Standard Cost", "description": "Cost basis used for gross-margin calculation.", "display_type": "currency", "display_scale": 2}
  ],
  "additional_measures": [
    {
      "model_id": "factinternetsales",
      "virtual_col": "Net Revenue",
      "agg_type": "SUM",
      "physical_expr": "f.SALESAMOUNT - f.TAXAMT - f.FREIGHT",
      "filter_expr": "",
      "data_type": "DECIMAL(19,4)",
      "format_mask": "$#,##0.00"
    },
    {
      "model_id": "factinternetsales",
      "virtual_col": "Gross Margin %",
      "agg_type": "AVG",
      "physical_expr": "(f.UNITPRICE - f.TOTALPRODUCTCOST) / NULLIF(f.UNITPRICE, 0)",
      "filter_expr": "",
      "data_type": "DECIMAL(9,4)",
      "format_mask": "0.00%"
    }
  ],
  "role_playing": [
    {"model_id": "factinternetsales", "dim_table": "ADVENTUREWORKS.DIMDATE", "fact_fk_col": "ORDERDATEKEY", "role_label": "OrderDate"},
    {"model_id": "factinternetsales", "dim_table": "ADVENTUREWORKS.DIMDATE", "fact_fk_col": "SHIPDATEKEY",  "role_label": "ShipDate"}
  ]
}
```
