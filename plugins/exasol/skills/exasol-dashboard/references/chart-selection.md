# Chart-type selection

Picks a Vega-Lite chart type per panel based on the shape of the result set. Deterministic — no LLM call. Inputs are: column count, column types, row count, and whether the columns are dimensions or measures.

## Decision matrix

Columns are classified during query plan:

- **D (categorical dimension)** — VARCHAR / CHAR / small DECIMAL with low distinct count
- **T (temporal)** — DATE / TIMESTAMP, or a year/quarter/month column
- **M (measure / numeric)** — DECIMAL / DOUBLE with high distinct count, typically aggregated

| Shape | Rows | Chart |
|---|---|---|
| 1 M | 1 | **KPI tile** — single big number with format hint |
| 1 D + 1 M | 2–50 | **Horizontal bar** — sorted desc |
| 1 D + 1 M | 51–500 | **Top-N bar** (LIMIT 25) + "showing top 25 of N" caption |
| 1 D + 1 M | >500 | **Histogram** (M distribution, ignore D unless ordinal) |
| 1 T + 1 M | any | **Line chart** — time series |
| 1 T + 1 M + 1 D | any | **Multi-line chart** — color by D (cap distinct D at 8) |
| 1 D + 2 M | 2–50 | **Grouped bar** — two bars per category |
| 1 D + 2 M (ratio implied) | 2–50 | **Scatter** — M1 vs M2, label by D |
| 2 M | any | **Scatter** — M1 vs M2 |
| 2 D + 1 M | <=300 | **Heatmap** — D1 × D2, color by M |
| 2 D + 1 M | >300 | **Stacked bar** — D1 on x, M stacked by D2 |
| 1 T + 2 D + 1 M | any | **Small multiples** — facet by one D, time series colored by other |
| Anything wider (≥ 5 columns) | <=100 | **Table** — fall back; charts get illegible past 4 dims |
| Anything wider | >100 | **Table preview** (first 100 rows) + "use cube directly" hint |

## Classification rules

### D (categorical) detection

- DDL type is `VARCHAR(n)` with `n <= 200` → D
- DDL type is any string AND `COUNT(DISTINCT col) / COUNT(*) < 0.1` AND distinct count <= 1000 → D
- Numeric column with distinct count <= 20 → D (treat as enum)
- Otherwise → not D

### T (temporal) detection

- DDL type is DATE / TIMESTAMP → T
- Column name matches `(?i)(date|year|quarter|month|week|day|time)` AND distinct count <= 366*20 → T (heuristic for year columns)

### M (measure) detection

- Numeric AND not D → M

If the result set has measures from a cube (driven by SQLCUBE_META.MEASURES), trust the AGGREGATION column for hints — `SUM` measures are M, `COUNT DISTINCT` is M with integer format hint.

## Multi-result handling

When `panels=auto`, the LLM produces a list of (title, sql, intended_chart) entries. The skill executes each query, classifies the result, and applies the matrix above. If the LLM's `intended_chart` matches what the matrix would pick → use it. If they differ → matrix wins (LLM doesn't see actual data shape, matrix does).

If the matrix picks a chart that doesn't match the LLM intent meaningfully (e.g., LLM said "line", matrix says "table") → annotate the panel: "Showed as table because [reason]". Don't silently coerce.

## Format hints from cube

When the source is a cube, measures carry `FORMAT_HINT` from SQLCUBE_META.MEASURES:

| Format hint | Vega-Lite axis format |
|---|---|
| `currency_usd` | `$,.0f` |
| `currency_eur` | `€,.0f` |
| `percent` | `.1%` |
| `integer` | `,d` |
| `decimal_2` | `,.2f` |
| `decimal_4` | `,.4f` |
| `bytes` | `.2s` |
| `duration_seconds` | `,.1f` + axis title `(s)` |

Apply to the measure axis automatically. Saves the LLM from re-deriving formatting per panel.

## Color encoding rules

| Situation | Color |
|---|---|
| Single series, light theme | `#1f77b4` (matplotlib blue) |
| Single series, dark theme | `#90caf9` (light blue) |
| Categorical color (multi-line, heatmap, stacked) | Vega-Lite `category` scheme (= tableau10) |
| Sequential color (heatmap measure axis) | `blues` (light) / `viridis` (dark) |

Never use rainbow / red-green for sequential — accessibility.

## Special cases

### "Big number" panels

When the result set is exactly one row and one measure, render as a KPI tile, not a chart. The HTML treats KPI tiles separately (large font, format hint applied, optional sparkline if a time series is available alongside).

KPI tile spec is not Vega-Lite — it's a `<div>` in the HTML. See `vega-lite-templates.md` for the markup.

### Distribution panels

When the LLM asks for a histogram explicitly OR the data shape suggests one (1 D + 1 M with >500 rows where D is high-cardinality), use Vega-Lite's `bin: true` on the M axis. Bucket count auto-defaults via Vega's Sturges' rule.

### Ranked lists with too many categories

>50 categories on a bar chart is unreadable. Truncate to top 25 (or bottom 25 if measure is "lateness" / "errors"). Add caption: "Top 25 by <measure name>".

### Empty result sets

Render a placeholder card: "No data for this panel." Don't error or skip silently. Caption explains the query that returned zero rows so the user can debug.

## When NOT to chart

Skip the chart and render a table when:

- Result has ≥5 columns (charts get illegible)
- All columns are measures (no dimensional axis)
- Result has fewer rows than columns × 2 (sparse, table is clearer)

Tables in v0.1 are just `<table>` HTML, no DataTables, no sorting controls. Static snapshot.
