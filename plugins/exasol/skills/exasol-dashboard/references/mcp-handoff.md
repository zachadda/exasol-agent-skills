# MCP query handoff

After the dashboard URL is printed, control of the conversation returns to the agent — and natural follow-ups from the user (drill-downs, new questions, "show me X by Y instead") should NOT regenerate the dashboard each time. They should hit MCP directly against the underlying cube.

This reference describes the hand-off protocol.

## The split

| Question type | Path |
|---|---|
| "Show me a dashboard" | exasol-dashboard skill |
| "What's the total revenue for Q4?" | MCP query against cube |
| "Compare segment A vs segment B" | MCP query |
| "Refresh the dashboard with this filter" | exasol-dashboard skill with `panels=<yaml>` |
| "Update the dashboard to include shipping data" | exasol-dashboard skill (regenerate auto) |
| "Add a new measure: gross margin" | exasol-semantic-layer skill (META update) + dashboard regen |

Dashboard skill is for STRUCTURED VISUAL artifacts. MCP is for AD-HOC TEXTUAL queries with maybe inline tables. Drawing this line keeps the dashboard snapshot honest (regen → fresh artifact) and keeps MCP queries fast (no HTML generation overhead).

## What MCP needs to know about the cube

`mcp__exasol_db__list_exasol_schemas` returns Exasol schemas the agent can query. The SQLCube virtual schema appears in this list once `CREATE VIRTUAL SCHEMA` is committed (assuming the MCP server doesn't filter virtual schemas — see `exasol-semantic-layer/troubleshooting.md` §7).

After dashboard generation, prompt the user once:

```
Dashboard is up. For follow-up questions, query the cube directly via MCP:

  Available cube: <CUBE_NAME>
  Measures: <list from SQLCUBE_META.MEASURES>
  Dimensions: <list from SQLCUBE_META.DIMENSIONS>

Ask anything ("top customers by revenue in 2024", "compare Q3 vs Q4 by segment", etc.) — I'll route to the cube.
```

This single onboarding message orients the user. After that, the agent uses MCP tools when conversational queries land.

## When does the agent regenerate the dashboard?

Three triggers:

1. **User explicitly asks** — "make a new dashboard" / "refresh the dashboard" / "redo the dashboard with these filters."
2. **Cube structure has changed** — new measures added, dimensions renamed (the prior dashboard is now stale).
3. **Data has changed materially** — fresh migration ran, new fact rows.

Otherwise → MCP path. The dashboard sits there as a reference; users can keep referring back to it visually while asking textual questions.

## Re-using the dashboard URL across the session

The HTML file persists at `<host_dir>/<source_name>.html`. The HTTP server keeps running until killed. So the URL is stable across the rest of the session:

```
http://localhost:8765/<source_name>.html
```

When the user asks for a new chart specifically (not a full dashboard refresh):

- **Option A**: re-run the skill with `panels=<yaml>` containing only the new panel → new dashboard at same URL (overwrites prior HTML, server still serves).
- **Option B**: MCP query, return the result inline as a table or markdown. No HTML.

Option B is the default for ad-hoc. Option A only when the user wants the chart in their browser.

## What MCP tools to prefer

For querying the cube:

- `mcp__exasol_db__find_exasol_tables_and_views` — discover what's queryable in the cube
- `mcp__exasol_db__describe_exasol_table_or_view` — get column lists for a virtual table
- Free-form SQL via a generic SQL tool (skill registry varies by MCP server config)

Avoid:

- `mcp__exasol_db__describe_exasol_built_in_function` — irrelevant for cube queries
- `mcp__exasol_db__find_exasol_custom_functions` — only useful when debugging SQLCUBE.ADAPTER itself

## Caveat: when the dashboard data and MCP data diverge

Dashboard is a snapshot. If the user runs a migration in the middle of the session, the dashboard data is stale but MCP returns fresh data. The agent should call this out:

```
Note: dashboard was generated at 14:32. Underlying data has changed since.
Asking ad-hoc gives you live numbers; the visual panels do not.
```

Don't silently let the user trust the dashboard for newly-updated questions. Either re-run the dashboard or be explicit.

## Caveat: MCP can't always render the cube

Some MCP server configurations filter `SYS.EXA_VIRTUAL_SCHEMAS` from their schema discovery. If the user's MCP doesn't surface the cube, agent has two options:

1. **Query the physical schema directly.** Doesn't get measures or named relationships, but tables work.
2. **Tell the user to reconfigure their MCP server.** Point at `exasol-semantic-layer/troubleshooting.md` §7.

For the `/oneshot-tour` use case, the exanano-sqlcube MCP server is pre-configured to expose virtual schemas — so this caveat doesn't bite for the demo path. It can bite for users on other Exasol clusters.

## When to suggest re-running upstream skills

If the user's follow-up question reveals a structural gap:

- "Show me Net Revenue (after tax)" → measure doesn't exist → suggest `exasol-semantic-layer` with a `measures_overrides` YAML adding it.
- "I need a Region dimension" → not in source schema → `exasol-migrate` to bring it in.
- "Why is this number wrong?" → could be type mismatch / FK miss → re-run `exasol-optimize`.

These are out-of-band — not part of the dashboard skill's job. But the dashboard skill SHOULD recognize when the question is asking for something structural and route the user back upstream.

## Closing the dashboard cleanly

When the session ends, or the user moves on to other work, the http.server should be killed:

```bash
if [ -f /tmp/exasol-dashboard.pid ]; then
  kill $(cat /tmp/exasol-dashboard.pid) 2>/dev/null
  rm /tmp/exasol-dashboard.pid
fi
```

Skill SHOULD register this cleanup with the session-end hook if one is available. Otherwise, the orphaned `python3 -m http.server` process is harmless (it sits there until the user reboots), but offends tidiness.

## v0.2 ideas (out of scope for v0.1)

Things the handoff would gain from:

- **Live cube subscription.** Dashboard auto-refreshes when SQLCUBE_META changes. Needs WebSocket + cache invalidation. Significant build.
- **Inline drill-down.** Click a bar → open a side panel with the rows. Needs JS event handling beyond Vega's default.
- **Conversational MCP overlay.** Embed a chat widget in the HTML that connects to the same MCP server. The dashboard becomes a working surface, not just a snapshot.
- **Export.** PNG / PDF / Excel export from the dashboard.

None of this in v0.1. The snapshot-plus-MCP split is intentionally simple and lets us ship.
