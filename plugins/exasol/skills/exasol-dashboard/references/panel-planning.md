# Panel planning

Decides what panels show up on the dashboard. Two paths: `panels=auto` (LLM-driven, default) or `panels=<yaml-path>` (explicit).

## Path A: auto (LLM-driven)

Used when the user hasn't supplied a panels YAML. Default for `/oneshot-tour` and ad-hoc visualization.

### Inputs

For a SQLCube source — read from `SQLCUBE_REGISTRY` (the cube definition schema, **not** `SQLCUBE_META` which is studio app state + lineage):

- `SQLCUBE_REGISTRY.DOMAINS` (`DOMAIN_ID`, `DOMAIN_NAME`, `DESCRIPTION`) — business grouping above models
- `SQLCUBE_REGISTRY.MODELS` (`MODEL_ID`, `MODEL_LABEL`, `FACT_SCHEMA`, `FACT_TABLE`) — one row per fact-table grain. `MODEL_ID` is the lowercase virtual-table name and **equals `DOMAIN_ID` in this codebase** (deployer.py assigns them in lockstep).
- `SQLCUBE_REGISTRY.DIMENSIONS` (`MODEL_ID`, `VIRTUAL_COL`, `PHYSICAL_TABLE`, `PHYSICAL_COL`, `DIM_TABLE_ALIAS`, `IS_VISIBLE`) — column-level virtual surface. Filter `IS_VISIBLE = TRUE` to match what the adapter projects onto the virtual schema.
- `SQLCUBE_REGISTRY.MEASURES` (`MODEL_ID`, `VIRTUAL_COL`, `AGG_TYPE`, `PHYSICAL_EXPR`, `FORMAT_MASK`, `IS_VISIBLE`) — aggregable virtual columns. `VIRTUAL_COL` is Title Case with spaces (`"Sales Amount"`, `"Order Count"`).
- `SQLCUBE_REGISTRY.JOINS` (`MODEL_ID`, `DIM_TABLE`, `DIM_ALIAS`, `DIM_KEY`, `FACT_FK`, `JOIN_TYPE`, `JOIN_ORDER`) — physical join graph. `JOIN_TYPE` is auto-resolved to `INNER` for NOT NULL FKs, `LEFT` otherwise by `deployer._resolve_join_type()`.
- `SQLCUBE_REGISTRY.ATTRIBUTES` (`DOMAIN_ID`, `BUSINESS_NAME`, `DISPLAY_TYPE`) — domain-level display labels useful for axis titles.
- (Optional) Sample rows: `SELECT * FROM "<virtual_schema>"."<model_id>" LIMIT 20` (note both identifiers must be quoted; `model_id` is lowercase).

See `exasol-semantic-layer/references/meta-model.md` for the full registry DDL + the schema-vs-app-state distinction.

For a raw schema source:

- `SYS.EXA_ALL_TABLES` + `SYS.EXA_ALL_COLUMNS` for the schema
- Sample rows per table (5–10 each)
- LLM has to do more work — no measure definitions, no dimensional roles

### LLM prompt shape

```
You are designing a dashboard for the following cube/schema.

[Cube metadata or raw schema inventory]

Goal: produce <llm_panel_count> panels that tell the most interesting story about this data.
Mix of: 1–2 KPI tiles, 2–3 bars/lines/heatmaps, optionally 1 distribution or scatter.

For each panel output JSON:
{
  "title": "...",
  "intent": "kpi" | "bar" | "line" | "scatter" | "heatmap" | "histogram" | "table",
  "rationale": "one sentence",
  "sql": "SELECT ... FROM <cube_or_schema> ...",
  "row_limit": 5000
}

Hard rules:
- SQL must be valid Exasol SQL against the schema/cube.
- Use measures by their MEASURE_NAME when querying a cube.
- Reference dimensions as <DimName>.<Column> for cubes.
- No subqueries that return >5000 rows.
- Don't repeat the same dim+measure combination across panels.
- Aim for variety — don't make all 6 panels bar charts.
- For time series, use the most natural date role (OrderDate over ShipDate when both exist).
```

Return: JSON array of <llm_panel_count> entries. Skill validates each (SQL parses, references resolve) and discards bad ones.

### Defaults the LLM should hit

Without explicit guidance, the LLM should produce a reasonable mix. A 6-panel auto dashboard typically looks like:

1. KPI: total of the primary measure (e.g., Total Revenue)
2. KPI: row count or distinct entity count (e.g., Order Count)
3. Time series: primary measure over time (line chart, OrderDate.Year or .Month)
4. Top categories: primary measure grouped by a primary dimension (bar)
5. Cross-cut: heatmap of two dimensions × one measure
6. Distribution: histogram or scatter exploring spread

Skill doesn't enforce this shape — LLM has latitude — but flagged in the prompt as "good defaults if nothing else is interesting."

### Validation steps after LLM returns

For each entry:

1. **Parse the SQL.** Wrap in `SELECT * FROM (...) WHERE 1=0 LIMIT 0` and execute. If errors → drop the panel, log to manifest.
2. **Check row count.** Run with `LIMIT <row_limit + 1>`. If result has >row_limit rows → annotate "Result truncated, showing top N." Keep the panel.
3. **Classify the shape.** Pull DDL of the result, apply `chart-selection.md` rules. Compare against `intent`. If matrix disagrees → matrix wins, annotate.
4. **De-dupe.** If two panels return structurally identical data shapes for the same dimension/measure pair → keep one.

Manifest tracks: how many panels LLM proposed, how many survived validation, what was dropped and why.

### Cost notes

Single LLM call per dashboard. ~3–8K tokens in, ~1–3K tokens out. ~$0.01–0.02 per dashboard on Haiku. Cheap. Surface the cost at end:

```
Dashboard planning: 1 LLM call, ~5400 tokens, ~$0.015 (Haiku).
```

## Path B: explicit YAML

For reproducible dashboards (CI smoke tests, demos that need to look the same every run), accept a YAML path:

```yaml
title: "Q4 Sales Dashboard"
panels:
  - title: "Total Revenue YTD"
    intent: kpi
    sql: |
      SELECT "Total Revenue"
      FROM ADVENTUREWORKS_CUBE.Sales
      WHERE OrderDate.Year = 2024;
    format_hint: currency_usd

  - title: "Revenue by Segment"
    intent: bar
    sql: |
      SELECT Customer.Segment, "Total Revenue"
      FROM ADVENTUREWORKS_CUBE.Sales
      WHERE OrderDate.Year = 2024
      GROUP BY Customer.Segment
      ORDER BY "Total Revenue" DESC;

  - title: "Monthly Revenue Trend"
    intent: line
    sql: |
      SELECT OrderDate.Year, OrderDate.Month, "Total Revenue"
      FROM ADVENTUREWORKS_CUBE.Sales
      WHERE OrderDate.Year IN (2023, 2024)
      GROUP BY OrderDate.Year, OrderDate.Month
      ORDER BY 1, 2;

  - title: "Revenue Heatmap (Segment × Category)"
    intent: heatmap
    sql: |
      SELECT Customer.Segment, Product.Category, "Total Revenue"
      FROM ADVENTUREWORKS_CUBE.Sales
      WHERE OrderDate.Year = 2024
      GROUP BY Customer.Segment, Product.Category;
```

YAML wins. Skill skips LLM call entirely. Validation runs the same (parse SQL, cap rows, classify shape vs intent).

If a panel's SQL fails validation, the YAML path FAILS LOUDLY — agent halts and tells user "panel X has bad SQL." Unlike auto-mode (where dropping a bad panel is acceptable), explicit YAML implies the user wants exactly these panels.

## Mixed mode (future, not v0.1)

Hybrid: user supplies 2 must-have panels in YAML, skill fills remaining 4 via LLM. Not implemented in v0.1 — single path per invocation.

## Heuristics for fact / dim choice

When the LLM is auto-planning and has multiple facts to choose from:

- **Pick the fact with most measures** as the primary. Likely the analytics-rich one.
- **Pick the dimension with most relationships** as the primary categorical breakout. It cross-cuts most facts cleanly.
- **Use the dimension with explicit hierarchy hints** (year > quarter > month) for time-series rollups.

When raw schema source (no cube metadata):

- Largest table (by row count) → assumed fact, use for KPI tiles
- Numeric columns on the largest table → measures (SUM by default)
- VARCHAR columns with <100 distinct values → dimensions
- DATE / TIMESTAMP columns → time axes

Less reliable than cube-driven planning. Encourage the user to run `exasol-semantic-layer` first for better dashboards.
