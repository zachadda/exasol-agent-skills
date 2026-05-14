# Phase 5 — Dashboard + handoff

Final phase. The cube is verified live in Phase 4. Now the orchestrator renders a Vega-Lite dashboard from the verification sample, surfaces a local URL to the user, and pivots into open-ended MCP query mode.

## Goal

End the tour with something the user can *see* (the chart) plus something they can *do* (ask follow-up questions in natural language and get more queries / charts back). The session does not terminate — it transitions.

## What's produced

- One single-file HTML page rendered locally
- Served on a free local port via `python3 -m http.server` (no infra dependency)
- URL surfaced to user
- Agent remains in the session, ready to answer follow-up business questions against the cube

## Dashboard generation

The `exasol-dashboard` skill owns the actual rendering. Phase 5 just passes parameters and surfaces the URL.

Input to the skill (from `tour_state.verification_sample`):
- columns: `["Calendar Year", "Sales Amount"]`
- rows: `[[2010, 43421.04], ...]`
- chart hint: derived from column types (dim = year → x-axis, measure = numeric → y-axis, type → bar)

Output:
- Path to single-file HTML (e.g., `/tmp/oneshot-2026-05-13-201500.html`)
- URL once server is up (e.g., `http://localhost:8765/oneshot-2026-05-13-201500.html`)

## Chart type selection (deterministic)

| Dim attribute type | Measure type | Chart |
|---|---|---|
| date / datetime | numeric | line |
| ordinal (year, quarter) | numeric | bar |
| text (≤ 12 distinct values) | numeric | bar |
| text (> 12 distinct values) | numeric | horizontal bar (top 10) |
| geographic | numeric | choropleth (deferred to v0.2) |

Default fallback if uncertain: vertical bar.

## Narration

```
agent:
  ### Step 6 of 6 — Dashboard + handoff

    tool: exasol-dashboard skill
      → Vega-Lite spec generated (vertical bar, 4 rows)
      → HTML written to /tmp/oneshot-2026-05-13-201500.html
      → Server started on http://localhost:8765/

  ✓ Dashboard live at http://localhost:8765/oneshot-2026-05-13-201500.html

  ---

  ## Tour complete

  - 14 tables imported
  - 21 constraints applied (2 deferred for manual review)
  - Virtual schema `SQLCUBE_ADVENTUREWORKS` live with 1 model, 23 dimensions, 11 measures
  - Sample dashboard rendered

  You're now in query mode. Ask anything against the cube — I'll translate to
  SQL via MCP and add new charts to the dashboard.

  Examples:
  - "What's average order value by quarter?"
  - "Top reseller by sales territory"
  - "Monthly sales for 2013 as a line chart"
```

## MCP query handoff

After the dashboard renders, the agent enters a passive listening mode. Any further user input is treated as a natural-language analytics question.

Flow per follow-up question:

1. User asks a business question in natural language.
2. Agent uses cube metadata (`SQLCUBE_REGISTRY.MEASURES` + `SQLCUBE_REGISTRY.DIMENSIONS`, keyed by `MODEL_ID`) to ground attribute / measure names. Virtual-schema table = `MODEL_ID`; virtual-schema columns = `VIRTUAL_COL` (quoted, case-preserving).
3. Agent emits SQL against the virtual schema via `mcp__exasol_db__query`.
4. Agent receives result rows.
5. Agent decides: table-only response, or table + new chart added to dashboard?
6. If chart: hand back to `exasol-dashboard` skill, which appends a new tab/section to the existing HTML, server stays up.
7. User keeps asking.

The agent does NOT need to re-trigger `/oneshot-tour` for each follow-up. The skill stays loaded; only the per-query SQL changes.

## How long the session lives

The orchestrator state persists until:

- User explicitly says "done" / "exit" / "stop"
- User triggers a different orchestrator skill (e.g., `/diagnose-slow-query`)
- The agent terminal closes

When session ends, the orchestrator should:
1. Stop the local HTTP server gracefully.
2. Print a summary line: "Stopped dashboard server. Cube SQLCUBE_ADVENTUREWORKS remains queryable."
3. Mark session-end.

## What the user can do next without the orchestrator

Once the tour is complete, the cube is a permanent (until dropped) Exasol object. The user can:

- Query it via any JDBC/ODBC client (Tableau, Power BI, pyexasol, dbt)
- Re-render new dashboards by invoking `exasol-dashboard` directly
- Optimize / iterate via `exasol-optimize` against either STAGE or TARGET schemas
- Drop everything via `DROP VIRTUAL SCHEMA CASCADE`

The tour is not the lock-in — it's the bootstrap.

## Edge cases

| Situation | Action |
|---|---|
| Port 8765 is taken | Try 8766, 8767, ..., 8780 — first free wins. Surface the chosen port. |
| User's machine has no `python3` | Fall back to `python -m http.server`. If still no python, write the HTML and let user open it directly via `file://` |
| User asks a question that doesn't map to any cube attribute | "I don't recognize `<attribute>`. Available dims: ... measures: ..." — list and reprompt |
| User asks for a chart type the skill doesn't support yet | Render whatever's available + tell user what's missing |
| MCP query returns 0 rows | Show empty table + suggest broadening filter; don't render an empty chart |
| User says "render that as a line chart instead" mid-session | Re-render with new chart type, replace the tab in the HTML; server stays up |

## What the orchestrator stops doing after Phase 5

- Interrogation (Phase 1) — done
- Plan presentation (Phase 2) — done
- Execution sequencing (Phase 3) — done
- Verification (Phase 4) — done
- Dashboard render — done

What remains active: MCP query interpretation + dashboard append + session cleanup.

This is the agent's "post-tour" steady state. It's the most useful mode the user is in — they have a live cube and an analyst-on-tap.
