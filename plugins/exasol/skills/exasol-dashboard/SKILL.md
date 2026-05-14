---
name: exasol-dashboard
description: Build an ephemeral HTML+Vega-Lite dashboard against a live SQLCube cube or any Exasol schema. Generates a single self-contained HTML file, serves it via `python -m http.server`, prints the URL, and hands ongoing query control back to MCP. v0.1 supports KPI tiles, bar/line/area charts, scatter, heatmap, and tables — driven by query shape inference.

preconditions:
  - data_source_live:
      doc: "Either a SQLCube cube_live OR a raw Exasol schema with at least one queryable table. Skill auto-detects which path applies."
      check: |
        # SQL — pick one:
        # SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = '<NAME>';
        # OR
        # SELECT 1 FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA = '<NAME>' LIMIT 1;
      satisfied_by: exasol-semantic-layer   # preferred path — cube provides richer metadata
  - python3_available:
      doc: "Local `python3` binary on the agent host (used for the lightweight HTTP server). 3.7+ is fine. No third-party deps required."
      check: |
        # Bash:
        # python3 --version 2>&1 | grep -qE '3\.(7|8|9|1[0-9])' && echo ok
      satisfied_by: null   # standard on macOS and most Linux dev hosts; if missing, halt

provides:
  - dashboard_url:
      doc: "Local HTTP server is running, serving an HTML+Vega-Lite dashboard. Printing the URL to the user is the hand-off."
      verify: |
        # Bash:
        # curl -fsI "http://localhost:${DASHBOARD_PORT}/" | head -1 | grep -q '200 OK'

parameters:
  required:
    - source_name:
        doc: "Cube name (preferred) OR raw schema name. Skill probes SYS.EXA_VIRTUAL_SCHEMAS first."
  optional:
    - panels:
        default: auto
        doc: "Either `auto` (LLM-driven panel selection from cube structure) or a path to a YAML file with explicit panel definitions."
    - port:
        default: 8765
        doc: "Host port for the http.server. If occupied, skill probes +1 / +2 / +3 / +4 / +5 before halting."
    - host_dir:
        default: /tmp/exasol-dashboards
        doc: "Local directory the HTML lands in. Auto-created. Serves as the http.server root."
    - title:
        default: "<source_name> Dashboard"
        doc: "Browser tab title and H1."
    - theme:
        default: light
        doc: "`light` or `dark`. Vega-Lite config preset only — no per-chart styling."
    - llm_panel_count:
        default: 6
        doc: "Cap on auto-generated panels when `panels=auto`. Keeps the dashboard readable."

estimated_impact:
  v01_full_run: "10–30 sec for HTML generation, plus one query per panel. Server runs until user kills it; no automatic shutdown."
  output_size: "Single HTML, typically 20–80 KB, plus embedded JSON data for each chart (caps at 5000 rows per panel)."
---

# Exasol Dashboard Skill

Trigger when the user asks to **build a dashboard**, **visualize**, **chart this**, **show me a dashboard**, **plot the cube**, **make a quick BI view**, **render charts against the data**, or after a `/oneshot-tour` reaches the `cube_live` provides and routes to dashboard handoff.

## What this skill is

A zero-dependency "dashboard in a tab" generator. Renders one HTML file containing:

- Vega-Lite specs embedded inline (via the CDN-loaded Vega-Lite runtime)
- Data tables embedded inline (no live database connection from the browser)
- Light CSS for a grid layout

Serves it via Python's standard-library HTTP server. Single file, single port, no build step, no framework, no auth.

What this skill is NOT:

- Not a long-lived BI tool. Server is meant to run during the agent session and die when the user closes the terminal.
- Not interactive (filters, slicers, drilldowns). Static charts only in v0.1.
- Not authenticated. Anyone on localhost can read the dashboard. No production posture.
- Not a replacement for Looker / Superset / Metabase. Use those when you need governance, sharing, scheduling.

## Routing algorithm

| Phase | Reference | When |
|---|---|---|
| Chart-type selection | `references/chart-selection.md` | Always — picks bar / line / heatmap / etc. per panel |
| Vega-Lite templates | `references/vega-lite-templates.md` | When emitting specs (one template per chart type) |
| Panel planning | `references/panel-planning.md` | When `panels=auto` (LLM-driven panel inventory) |
| HTTP server lifecycle | `references/http-server.md` | Once HTML is on disk |
| MCP query handoff | `references/mcp-handoff.md` | After the URL is printed |

## Pipeline

```
1. Verify data source       (cube preferred; raw schema acceptable)
2. Plan panel inventory     (LLM-driven from cube metadata, or read YAML)
3. For each panel:
   a. Compose SQL           (against cube or raw schema)
   b. Execute               (via exapump or pyexasol)
   c. Cap result            (5000 rows max; LIMIT or aggregate-down)
   d. Pick chart type       (chart-selection.md rules)
   e. Generate Vega-Lite    (templates + data)
4. Assemble HTML            (grid layout, panel sections)
5. Write to host_dir        (host_dir/<source_name>.html)
6. Start http.server        (in background)
7. Verify server up         (curl HEAD)
8. Print URL to user        (provides contract)
9. Hand control to MCP      (mcp-handoff.md)
```

## Conventions

- **One panel, one chart, one query.** Don't combine into dashboards with too many charts. v0.1 cap: 6 panels by default (`llm_panel_count`).
- **Vega-Lite over D3 / Chart.js.** Declarative, JSON-based, runs from a CDN. Agent emits JSON, the browser does the rest.
- **Embed data, don't query live.** v0.1 dashboards are snapshots. To refresh, re-run the skill. Live queries against the cube happen via MCP, not the dashboard.
- **Server background-launched.** `python3 -m http.server` runs in a background process. Skill captures the PID, prints the URL, exits.
- **Charts on a uniform palette.** Light theme uses an Exasol-blue accent; dark theme uses an off-white. No per-panel custom colors.

## When to invoke

- `/oneshot-tour` orchestrator reaches `cube_live` and routes here for the final hand-off.
- Developer asks "show me a quick chart of X against Y" — skill builds a single-panel dashboard.
- Demo path: spin up the dashboard, walk an audience through KPI tiles, hand to MCP for follow-up.

## Related skills

- **exasol-semantic-layer** — preferred upstream. Cube metadata drives auto-panel selection. Raw schema works but produces less interesting dashboards.
- **exasol-optimize** — indirect dependency (via cube). Schema must have FKs + narrowed types before the cube exists.
- **oneshot-tour** — orchestrator that hands here last.

## Limitations

- **Snapshot data.** No interactive refresh. The whole dashboard regenerates each invocation.
- **5000 row cap per panel.** Hardcoded in v0.1 to keep HTML payload manageable. Larger requires LIMIT / aggregation in the query.
- **Localhost only.** No remote viewing. Users SSHing into a dev box won't see the dashboard without port-forwarding.
- **No PNG / PDF export.** Vega has export but it requires extra Node tooling. Skip for v0.1.
- **No auth on the HTTP server.** Don't run on a public network.
