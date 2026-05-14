# Measure business context (incremental)

Targeted prompt for proposing additional measures on a fact table. Used when the user adds a new fact, or asks "what measures would make sense here?" without running full cube enrichment.

## When used

- User invokes the skill with `target_action=measures_only` (parameter not yet exposed in v0.1)
- A new fact table has been added to the source schema and the user wants measure proposals without re-running the whole cube
- An existing cube exists but the user dropped a fact's measures and wants suggestions

## System prompt

```
You are proposing aggregated business measures for a single fact table in a semantic layer.

Output JSON only. Measures must be SQL-valid against the listed columns. Reference dimension
columns when joined via the listed relationships.
```

## User prompt template

```
Fact table: <FACT_TABLE>
Domain: <DOMAIN>
Grain: <GRAIN_DESCRIPTION_FROM_FACTS_OR_unspecified>

Columns:
<FOR_EACH_COLUMN>
- <COL_NAME> (<DATA_TYPE>)<IF_FK> → FK to <DIM>.<PK></IF_FK>
</FOR_EACH_COLUMN>

Joined dimensions (via existing relationships):
<FOR_EACH_RELATIONSHIP>
- <DIM_BUSINESS_NAME> (<DIM_PHYSICAL_TABLE>) — columns: <COMMA_LIST_OF_DIM_COLUMNS>
</FOR_EACH_RELATIONSHIP>

Sample rows (5):
<PIPE_DELIMITED_SAMPLE>

Existing structural measures (default aggregations):
<FOR_EACH_EXISTING_MEASURE>
- <MEASURE_NAME>: <EXPRESSION> [<AGGREGATION>]
</FOR_EACH_EXISTING_MEASURE>

Output JSON:
{
  "measures": [
    {
      "name": "<Title Case business name>",
      "expression": "<SQL expression, no outer aggregation wrapper>",
      "aggregation": "SUM" | "AVG" | "COUNT" | "COUNT_DISTINCT" | "MIN" | "MAX",
      "format": "currency_usd" | "currency_eur" | "percent" | "integer" | "decimal_2" | "decimal_4" | "bytes" | "duration_seconds",
      "description": "<one sentence>",
      "business_priority": "high" | "medium" | "low"
    }
  ]
}

Rules:
- Propose 3–8 measures. Quality over quantity.
- Don't repeat existing structural measures. Look at what's already there and ADD value.
- Use NULLIF(divisor, 0) defensively in ratio expressions.
- Measures can reference dim columns (e.g., DIM_PRODUCT.STANDARD_COST) — they're auto-joined.
- business_priority reflects how often analysts would ask for this in your judgment.
- Description should clarify the business meaning, not restate the SQL.
- Examples to consider depending on domain:
  - retail: gross margin, AOV, return rate, repeat customer rate
  - logistics: on-time-rate, average dwell time, exception rate
  - finance: yield, NIM, charge-off rate, fee-to-loan ratio
  - healthcare: claim approval rate, average length of stay, readmission rate
  - manufacturing: yield rate, scrap rate, OEE, downtime hours
  - saas: MRR, ARR, NRR, churn rate, expansion rate, LTV/CAC
```

## Validation

Same checks as `cube-enrichment.md`:

1. Expression dry-runs against the fact (with potential joins to listed dims).
2. No duplicate names vs existing measures.
3. Format from allowlist.
4. Aggregation matches expression shape (no `SUM` on VARCHAR).

Additional:

5. **Business priority sanity.** Skill caps `high` priority at 3 per call — LLMs over-mark importance. After receiving response, take top 3 `high` by order of appearance, demote remainder to `medium`.

## Storage

Approved measures go into `SQLCUBE_REGISTRY.MEASURES` (one row per measure, keyed by `(MODEL_ID, VIRTUAL_COL)`). The LLM's `business_priority` field is not part of the registry schema — it's stored in a side log for downstream consumers:

```sql
-- Side table for priority hints, consumed by dashboard panel-planning
INSERT INTO EXA_OPTIMIZE_LOG.MEASURE_PRIORITY (MODEL_ID, VIRTUAL_COL, BUSINESS_PRIORITY)
VALUES (?, ?, ?);
```

Downstream `exasol-dashboard` reads this when picking which measures to feature in KPI tiles.

## Cost

~2–5K tokens in (one table's worth of context), ~1–2K tokens out. ~$0.005–0.01 on Haiku per call.

## Integration with cube-enrichment.md

The two prompts share output shape for measures. If the skill is in full-cube mode, `cube-enrichment.md` covers everything. If incremental, `measure-business-context.md` is the surgical option.

User-supplied `measures_overrides.yaml` still wins over LLM output in both cases.
