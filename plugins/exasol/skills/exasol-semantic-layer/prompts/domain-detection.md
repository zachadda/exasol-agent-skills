# Domain detection

Prompt template — infers the business domain from schema name and table inventory. Result feeds into subsequent prompts as `<DOMAIN_HINT>`.

## When used

- Caller did NOT pass `domain_hint` parameter
- Schema has > 5 tables (smaller schemas don't need domain context)

If `domain_hint` is supplied, skip this prompt entirely.

## System prompt

```
You are identifying the business domain of a database schema based on table names and sample data.
Return JSON only.
```

## User prompt template

```
Schema name: <SOURCE_SCHEMA_NAME>

Tables:
<FOR_EACH_TABLE>
- <TABLE_NAME>: <ROW_COUNT_OR_unknown> rows, columns: <COLUMN_NAME_LIST_FIRST_8_OR_ALL>
</FOR_EACH_TABLE>

Sample data from the largest 3 tables:
<FOR_EACH_TOP_3_TABLES>
<TABLE_NAME> (5 rows):
<PIPE_DELIMITED_SAMPLE>
</FOR_EACH_TOP_3_TABLES>

Output JSON:
{
  "domain": "<one-or-two-word domain like 'retail', 'logistics', 'healthcare', 'finance', 'manufacturing', 'marketing', 'hr', 'sales', 'fintech', 'gaming', 'telecom', 'energy', 'public_sector', 'unknown'>",
  "sub_domain": "<more specific, e.g. 'e-commerce', 'b2b-sales', 'banking-core', 'claims-processing', 'supply-chain'>",
  "confidence": "high" | "medium" | "low",
  "reasoning": "<one sentence — which signal made the call>"
}
```

## Validation

1. `domain` must be from the allowlist in the prompt. Reject novel domains and route to `unknown`.
2. `reasoning` < 10 chars → demote confidence to `low`.
3. If `confidence: low` AND domain is anything but `unknown` → keep but tag for plan-presentation review.

## Output handling

The classification is returned inline as JSON to the skill — no Exasol-side
persistence in v0.1. The skill holds it in memory and passes the
`<DOMAIN_HINT>` and `<SUB_DOMAIN>` strings into the subsequent
`cube-enrichment.md` prompt template.

Re-running the skill re-detects. If the user wants stable domain assignment
across runs, supply `domain_hint` parameter explicitly and skip this prompt
entirely.

## Cost

~500 tokens in, ~100 out. Negligible.

## Examples (for prompt testing)

Schema `ADVENTUREWORKS` with tables `DIM_CUSTOMER`, `DIM_PRODUCT`, `FACT_SALES`, `FACT_RETURNS` → expected:

```json
{
  "domain": "retail",
  "sub_domain": "e-commerce",
  "confidence": "high",
  "reasoning": "Customer/product/sales/returns is the canonical retail fact-dim shape"
}
```

Schema `TPCH_SF1` → expected `manufacturing` + `supply-chain`. Schema `CLAIMS_2024` → `healthcare` + `claims-processing`. Schema `UNKNOWN_X` with three generic tables → `unknown` + low confidence.
