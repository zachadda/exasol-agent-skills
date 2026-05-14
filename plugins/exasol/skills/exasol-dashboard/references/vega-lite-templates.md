# Vega-Lite templates

Canonical Vega-Lite v5 specs the agent emits, one per chart type. Data is embedded inline (`values:`) per the snapshot model. Replace placeholders (`<...>`) with concrete values at HTML generation time.

## Spec wrapper

Every chart in the HTML is rendered by a small embed call:

```html
<div id="panel-<N>" class="panel"></div>
<script>
vegaEmbed('#panel-<N>', <SPEC>, {actions: false, theme: '<THEME>'});
</script>
```

`actions: false` hides Vega's debug menu. `theme` is `vox` for light, `dark` for dark.

The `<SPEC>` is the JSON object below, stringified into the HTML.

## Horizontal bar

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"d": "...", "m": 0}, ...]},
  "mark": {"type": "bar", "color": "#1f77b4"},
  "encoding": {
    "y": {"field": "d", "type": "nominal", "sort": "-x", "title": "<D_TITLE>"},
    "x": {"field": "m", "type": "quantitative", "axis": {"format": "<FORMAT>"}, "title": "<M_TITLE>"},
    "tooltip": [
      {"field": "d", "type": "nominal", "title": "<D_TITLE>"},
      {"field": "m", "type": "quantitative", "format": "<FORMAT>", "title": "<M_TITLE>"}
    ]
  },
  "width": "container",
  "height": {"step": 22}
}
```

`height: {"step": 22}` gives ~22px per bar — comfortable for up to 25 categories without scroll.

## Vertical bar (when category labels are short)

Same as horizontal, swap x and y. Use only when categories are ≤10 chars wide; otherwise labels overlap.

## Line chart (time series)

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"t": "2024-01-01", "m": 0}, ...]},
  "mark": {"type": "line", "color": "#1f77b4", "point": false},
  "encoding": {
    "x": {"field": "t", "type": "temporal", "title": "<T_TITLE>"},
    "y": {"field": "m", "type": "quantitative", "axis": {"format": "<FORMAT>"}, "title": "<M_TITLE>"},
    "tooltip": [
      {"field": "t", "type": "temporal", "title": "<T_TITLE>"},
      {"field": "m", "type": "quantitative", "format": "<FORMAT>", "title": "<M_TITLE>"}
    ]
  },
  "width": "container",
  "height": 300
}
```

Temporal axis auto-formats based on tick density. `point: false` keeps the line clean; toggle `true` for sparse series (<30 points).

## Multi-line (line with categorical color)

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"t": "2024-01-01", "m": 0, "g": "..."}, ...]},
  "mark": {"type": "line", "point": false},
  "encoding": {
    "x": {"field": "t", "type": "temporal", "title": "<T_TITLE>"},
    "y": {"field": "m", "type": "quantitative", "axis": {"format": "<FORMAT>"}, "title": "<M_TITLE>"},
    "color": {"field": "g", "type": "nominal", "scale": {"scheme": "tableau10"}, "title": "<G_TITLE>"},
    "tooltip": [
      {"field": "g", "type": "nominal", "title": "<G_TITLE>"},
      {"field": "t", "type": "temporal", "title": "<T_TITLE>"},
      {"field": "m", "type": "quantitative", "format": "<FORMAT>", "title": "<M_TITLE>"}
    ]
  },
  "width": "container",
  "height": 320
}
```

Cap series count at 8. Beyond that, color discrimination fails — switch to small-multiples (faceted) instead.

## Scatter

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"x": 0, "y": 0, "label": "..."}, ...]},
  "mark": {"type": "circle", "size": 60, "opacity": 0.7, "color": "#1f77b4"},
  "encoding": {
    "x": {"field": "x", "type": "quantitative", "axis": {"format": "<FORMAT_X>"}, "title": "<X_TITLE>"},
    "y": {"field": "y", "type": "quantitative", "axis": {"format": "<FORMAT_Y>"}, "title": "<Y_TITLE>"},
    "tooltip": [
      {"field": "label", "type": "nominal", "title": "<LABEL_TITLE>"},
      {"field": "x", "type": "quantitative", "format": "<FORMAT_X>", "title": "<X_TITLE>"},
      {"field": "y", "type": "quantitative", "format": "<FORMAT_Y>", "title": "<Y_TITLE>"}
    ]
  },
  "width": "container",
  "height": 320
}
```

Opacity 0.7 to soften overplotting. For ≥1000 points, drop opacity to 0.4 and shrink size to 30.

## Heatmap

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"d1": "...", "d2": "...", "m": 0}, ...]},
  "mark": {"type": "rect"},
  "encoding": {
    "y": {"field": "d1", "type": "nominal", "sort": "ascending", "title": "<D1_TITLE>"},
    "x": {"field": "d2", "type": "nominal", "sort": "ascending", "title": "<D2_TITLE>"},
    "color": {"field": "m", "type": "quantitative", "scale": {"scheme": "blues"}, "legend": {"format": "<FORMAT>"}, "title": "<M_TITLE>"},
    "tooltip": [
      {"field": "d1", "type": "nominal", "title": "<D1_TITLE>"},
      {"field": "d2", "type": "nominal", "title": "<D2_TITLE>"},
      {"field": "m", "type": "quantitative", "format": "<FORMAT>", "title": "<M_TITLE>"}
    ]
  },
  "width": "container",
  "height": {"step": 22}
}
```

For dark theme, swap `scheme: blues` for `scheme: viridis`.

## Histogram

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"m": 0}, ...]},
  "mark": {"type": "bar", "color": "#1f77b4"},
  "encoding": {
    "x": {"field": "m", "type": "quantitative", "bin": true, "axis": {"format": "<FORMAT>"}, "title": "<M_TITLE>"},
    "y": {"aggregate": "count", "type": "quantitative", "title": "Frequency"}
  },
  "width": "container",
  "height": 280
}
```

Vega-Lite picks bin count automatically. Override only if the data shape demands it.

## Stacked bar (2D + M)

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"d1": "...", "d2": "...", "m": 0}, ...]},
  "mark": "bar",
  "encoding": {
    "x": {"field": "d1", "type": "nominal", "sort": "-y", "title": "<D1_TITLE>"},
    "y": {"field": "m", "type": "quantitative", "axis": {"format": "<FORMAT>"}, "title": "<M_TITLE>"},
    "color": {"field": "d2", "type": "nominal", "scale": {"scheme": "tableau10"}, "title": "<D2_TITLE>"},
    "tooltip": [
      {"field": "d1", "type": "nominal", "title": "<D1_TITLE>"},
      {"field": "d2", "type": "nominal", "title": "<D2_TITLE>"},
      {"field": "m", "type": "quantitative", "format": "<FORMAT>", "title": "<M_TITLE>"}
    ]
  },
  "width": "container",
  "height": 320
}
```

## Small multiples (faceted line)

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"values": [{"t": "2024-01-01", "m": 0, "facet": "...", "color": "..."}, ...]},
  "mark": {"type": "line", "point": false},
  "encoding": {
    "x": {"field": "t", "type": "temporal", "title": null},
    "y": {"field": "m", "type": "quantitative", "axis": {"format": "<FORMAT>"}, "title": null},
    "color": {"field": "color", "type": "nominal", "scale": {"scheme": "tableau10"}}
  },
  "facet": {"field": "facet", "type": "nominal", "columns": 3, "title": "<FACET_TITLE>"},
  "spec": {"width": 220, "height": 140}
}
```

3-column grid. For >9 facets, fall back to a multi-line plot or top-N filter.

## KPI tile (not Vega-Lite)

Single-row, single-measure result. Rendered as HTML directly:

```html
<div class="kpi-tile">
  <div class="kpi-title"><M_TITLE></div>
  <div class="kpi-value"><FORMATTED_VALUE></div>
  <div class="kpi-context"><OPTIONAL_CAPTION></div>
</div>
```

CSS keeps the tile uniform with chart panels:

```css
.kpi-tile {
  padding: 24px;
  text-align: center;
}
.kpi-value {
  font-size: 48px;
  font-weight: 600;
  color: #1f77b4;       /* var(--accent) in dark theme */
}
.kpi-title {
  font-size: 14px;
  color: var(--muted);
  margin-bottom: 8px;
}
.kpi-context {
  font-size: 12px;
  color: var(--muted);
  margin-top: 4px;
}
```

Number formatting uses the same FORMAT codes as Vega-Lite axes (formatted server-side, not via vega — formatter is a Python helper).

## Table (fallback)

```html
<table class="data-table">
  <thead>
    <tr><th>Col1</th><th>Col2</th>...</tr>
  </thead>
  <tbody>
    <tr><td>v1</td><td>v2</td>...</tr>
    ...
  </tbody>
</table>
```

Plain `<table>`. No DataTables, no sorting, no pagination. Snapshot.

Styling:

```css
.data-table {
  border-collapse: collapse;
  width: 100%;
  font-size: 13px;
}
.data-table th, .data-table td {
  text-align: left;
  padding: 6px 10px;
  border-bottom: 1px solid var(--row-border);
}
.data-table th {
  background: var(--th-bg);
  font-weight: 600;
}
```

## CDN dependencies

The HTML loads three scripts from jsDelivr CDN:

```html
<script src="https://cdn.jsdelivr.net/npm/vega@5"></script>
<script src="https://cdn.jsdelivr.net/npm/vega-lite@5"></script>
<script src="https://cdn.jsdelivr.net/npm/vega-embed@6"></script>
```

Pinned to major versions. Total transfer: ~600 KB gzipped, cached after first load.

For air-gapped environments, bundle Vega locally — but that's out of v0.1 scope. Skill emits a warning if `host_dir` is on a network share, suggesting the user check connectivity.
