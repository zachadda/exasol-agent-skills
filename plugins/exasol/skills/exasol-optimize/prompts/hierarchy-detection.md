# Hierarchy detection

Prompt template — classifies self-FK candidates as legitimate hierarchies vs accidental name collisions.

## When used

ANALYZE_CONSTRAINTS discovers self-referential FK candidates — a column on table T that values-match another column on T (typically `MANAGER_ID` → `EMPLOYEE_ID`). These are usually hierarchical (org chart, BOM, category tree, account parent) but occasionally false positives.

The skill flags candidates with `ACTION_NAME='REJECTED_CANDIDATE'` AND `reason='self-fk-needs-review'` and sends each batch to this prompt.

## System prompt

```
You are reviewing self-referential foreign key candidates discovered by automated schema analysis.
For each candidate, decide whether it represents a legitimate hierarchical relationship (org chart,
BOM, category tree, account parent, message threading) or a coincidental column value overlap.

Output JSON only. Be specific in reasoning — name the hierarchy type when you can.
```

## User prompt template

```
Schema: <SOURCE_SCHEMA_NAME>
Domain hint: <DOMAIN_HINT_OR_unspecified>

Self-FK candidates to classify:

<FOR_EACH_CANDIDATE>
Table: <TABLE_NAME>
  Child column: <FK_COL>   (suggested FK — child references parent)
  Parent column: <PK_COL>  (suggested PK — typically the table's primary key)

Table sample (5 rows showing both columns):
<SAMPLE_ROWS_PIPE_DELIMITED>

Column samples:
  <FK_COL> distinct values (up to 10): <VALUES>
  <PK_COL> distinct values (up to 10): <VALUES>

Match statistics:
  Non-null <FK_COL>: <COUNT>
  <FK_COL> values that resolve to <PK_COL>: <COUNT>
  Cycle detection: <"no cycle" | "cycle detected at depth <N>">
</FOR_EACH_CANDIDATE>

Output JSON:
{
  "classifications": [
    {
      "table": "<TABLE_NAME>",
      "fk_column": "<FK_COL>",
      "pk_column": "<PK_COL>",
      "hierarchy_kind": "org_chart" | "bom" | "category_tree" | "account_parent" | "message_thread" | "geographic" | "other_hierarchy" | "false_positive",
      "confidence": "high" | "medium" | "low",
      "reasoning": "<one or two sentences>"
    }
  ]
}

Rules:
- One entry per candidate sent. No skipping.
- For "false_positive", reasoning must explain why the column values overlap without semantic relationship
  (e.g., shared sequence generator, common surrogate-key space).
- "other_hierarchy" only for genuine tree structures that don't fit the listed kinds.
- Cycles in the data mean it's NOT a strict tree — either flag confidence:low or use kind:false_positive
  with reasoning.
```

## Validation rules

After response:

1. **Schema match.** Every `(table, fk_column, pk_column)` must match a sent candidate.
2. **Cycle vs hierarchy.** If the input data reported a cycle AND classification is a non-cyclic hierarchy
   (`org_chart`, `bom`, `category_tree`) → downgrade confidence to `low` and tag for human review.
3. **Confidence floor.** If reasoning is < 20 chars → treat as low-quality, force confidence:low.

## Action

For each surviving classification:

| hierarchy_kind | confidence | Skill action |
|---|---|---|
| any non-false_positive | high | Auto-promote to FK (re-route from REJECTED to ADD_FK action) |
| any non-false_positive | medium | Surface in plan-presentation as "candidate hierarchy — review" |
| any non-false_positive | low | Leave in REJECTED, log note |
| false_positive | any | Leave in REJECTED, log note |

The "auto-promote on high" path is the only LLM influence on the final DDL plan. Even there, the user sees the change in DRY_RUN_PLAN output before commit — no silent action.

## Storage

```sql
INSERT INTO EXA_OPTIMIZE_LOG.HIERARCHY_FLAGS
  (SOURCE_SCHEMA, TABLE_NAME, FK_COLUMN, PK_COLUMN, HIERARCHY_KIND, CONFIDENCE, REASONING, AUTO_PROMOTED)
VALUES
  (?, ?, ?, ?, ?, ?, ?, ?);
```

`AUTO_PROMOTED=TRUE` when the skill upgraded REJECTED → ADD_FK based on this classification. Audit trail for "why is this FK there?"

## Cost

~300–700 tokens in per candidate (samples are the bulk), ~150 tokens out. Schemas typically have ≤ 5 self-FK candidates, so a single batched call per run. ~$0.005 on Haiku.

## Disable path

Set `llm_enrichment=false` to skip. Self-FK candidates remain in REJECTED bucket, surfaced to user for manual review without LLM hints.

## Test fixtures

The `ACME_TORTURE` synthetic schema (factory-foundation) includes:

- `EMPLOYEE.MANAGER_ID → EMPLOYEE.EMPLOYEE_ID` (expected: org_chart, high)
- `CATEGORY.PARENT_CATEGORY_ID → CATEGORY.CATEGORY_ID` (expected: category_tree, high)
- A deliberate false-positive case where two unrelated columns share an ID space (expected: false_positive, high)

Use these to regression-test prompt changes.
