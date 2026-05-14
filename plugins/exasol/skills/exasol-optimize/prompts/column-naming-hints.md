# Column-naming hints

Prompt template — annotates cryptic / abbreviated column names with business names and descriptions.

## When used

Triggered when `llm_enrichment=true` AND the schema has columns where the cryptic-name heuristic fires:

- All-uppercase ≤ 6 chars (e.g., `CUSTID`, `ORDDT`, `QTY`)
- Single-word abbreviations matching common patterns (`AMT`, `QTY`, `DT`, `TS`, `NUM`, `CD`, `TY`, `STS`)
- Snake_case with a single-letter prefix (`p_name`, `c_id`)
- camelCase with consecutive caps (`cstmrId`, `prdSku`)

Columns that already look human-readable (`CUSTOMER_NAME`, `ORDER_DATE`, `SHIPPING_ADDRESS`) skip this prompt — no need to annotate the obvious.

## System prompt

```
You are annotating database column names for a data warehouse team. Your job is to suggest a
readable business name and a brief description for cryptic / abbreviated column names. You are
NOT renaming columns — you are providing hints that a human reviewer will see in a UI.

Be conservative. If a column is genuinely ambiguous given the samples and table context, say so
and mark confidence as "low". Never fabricate a meaning to fill space.

Output JSON only. No prose, no markdown, no commentary.
```

## User prompt template

```
Schema: <SOURCE_SCHEMA_NAME>
Domain hint: <DOMAIN_HINT_OR_unspecified>

Tables with cryptic columns to annotate:

<FOR_EACH_TABLE>
Table: <TABLE_NAME>
Sample rows (5):
<SAMPLE_ROWS_AS_PIPE_DELIMITED_TABLE>

Cryptic columns:
<FOR_EACH_CRYPTIC_COLUMN>
- <COL_NAME> (<DATA_TYPE>): distinct values seen: <UP_TO_5_DISTINCT_VALUES_OR_RANGE>
</FOR_EACH_CRYPTIC_COLUMN>
</FOR_EACH_TABLE>

Output JSON:
{
  "annotations": [
    {
      "table": "<TABLE_NAME>",
      "column": "<COL_NAME>",
      "suggested_name": "<readable name, Title Case>",
      "description": "<one sentence>",
      "confidence": "high" | "medium" | "low"
    }
  ]
}

Rules:
- One entry per cryptic column listed. Don't skip; mark low-confidence ones with confidence:low.
- suggested_name uses Title Case with spaces ("Customer ID" not "customer_id" not "CustomerID").
- description is one sentence, no more.
- Don't suggest type changes, PK assignments, or DDL — only naming hints.
- If a column is truly opaque (no samples, no related-column context), set confidence:low and
  description: "Insufficient context for naming."
```

## Validation rules

After receiving response:

1. **Schema match.** Every `(table, column)` in annotations must appear in the cryptic list sent to the LLM. Drop hallucinated rows.
2. **Casing.** `suggested_name` must contain at least one space OR be a single-word noun. Reject `customerID`, `customer_id`. Force Title Case.
3. **Description length.** > 200 chars → truncate at first sentence boundary.
4. **No DDL leakage.** Reject if `suggested_name` or `description` contains SQL keywords (`ALTER`, `INSERT`, `SELECT`). LLM occasionally tries to "help" with DDL.

## Output handling

Surviving hints are passed back to the skill's plan-presentation phase as
JSON (in-memory, not persisted). The phase renders each hint alongside the
DDL it relates to:

```
ALTER TABLE FACT_SALES MODIFY COLUMN AMT DECIMAL(18,2);
  hint: AMT → "Sale Amount" (medium confidence)
  description: Likely the monetary value of the sale line.
```

User decides whether to act on the hint (renames are out-of-skill — optimize
doesn't rename columns in v0.1).

No Exasol-side persistence in v0.1. If a future iteration adds cross-session
memory, the appropriate home is a new table in `SQLCUBE_META` (where studio
backend already stores app-state and lineage metadata via `bootstrap.sql`),
not a separately-managed schema.

## Cost

Roughly 200–500 tokens in per cryptic column, ~50 tokens out. Cap per-run at 100 columns to keep latency bounded — if a schema has more, top-100 by table importance (largest tables first).

## Disable path

Set `llm_enrichment=false` to skip entirely. Skill still runs, just doesn't write to ENRICHMENT_NOTES.
