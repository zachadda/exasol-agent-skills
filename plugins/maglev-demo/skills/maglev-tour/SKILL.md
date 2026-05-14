---
name: maglev-tour
description: Agent parody of the Maglev Tour GUI demo. Walks a stakeholder through the 8-step end-to-end pipeline — Target → Source → Schema → Migrate → Optimize → Cube → Security → Query — via the same Maglev Studio HTTP backend the GUI uses, narrating progress via TaskCreate/TaskUpdate instead of ActionLog widgets. Demo-only; not in the public exasol-skills marketplace. Source contract is documented in exasol-zemantic-layer/exasol-zemantic-layer/FACTORY_TOUR_DEMO_PATH.md.

preconditions:
  - studio_backend_reachable:
      doc: "Studio FastAPI backend reachable. Default port for the Maglev Tour demo is 8002 (per exasol-zemantic-layer/FACTORY_TOUR_DEMO_PATH.md). Auth via cookie obtained from POST /api/auth/login {password, persona='admin'}. STUDIO_AUTH_BYPASS=true accepts any password on dev machines."
      check: |
        # Bash:
        # curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8002/api/auth/me
        # Returns 200 when authenticated; 401 means backend up but need login.
      satisfied_by: null   # operator-managed; halt + ask user to start backend if not reachable
  - nano_running:
      doc: "Local Exasol Nano container on port 8568 (demo-tour fixture) reachable."
      check: |
        # SQL via exapump:
        # exapump sql --profile demo-tour "SELECT 1"
      satisfied_by: exasol-nano-local
  - source_connection_seeded:
      doc: "Exasol-side CONNECTION object pointing at the demo source. Default for Maglev Tour: SNOWFLAKE_CONNECTION → qaasyds-bq49773.snowflakecomputing.com / ACME_DEMO / ADVENTUREWORKS. Operator-seeded one-time per exasol-zemantic-layer/FACTORY_TOUR_DEMO_PATH.md §Setup step 3."
      check: |
        # SQL:
        # SELECT 1 FROM SYS.EXA_ALL_CONNECTIONS WHERE CONNECTION_NAME='SNOWFLAKE_CONNECTION';
      satisfied_by: null   # operator-managed
  - migration_scripts_installed:
      doc: "EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL script installed."
      check: |
        # SQL:
        # SELECT 1 FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA='EXA_DB_MIGRATION' AND SCRIPT_NAME='SNOWFLAKE_TO_EXASOL';
      satisfied_by: exasol-migrate
  - optimize_udfs_installed:
      doc: "EXA_OPTIMIZE.* UDFs installed (ANALYZE_CONSTRAINTS, ANALYZE_DIM_DATE, BUILD_UNKNOWN_MEMBER_INSERT, DRY_RUN_PLAN, etc.)"
      check: |
        # SQL:
        # SELECT COUNT(*) FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA='EXA_OPTIMIZE';
        # Returns >= 5.
      satisfied_by: exasol-optimize

provides:
  - cube_live:
      doc: "SQLCUBE_ACME_ADVENTUREWORKS virtual schema returns rows for the canonical demo query (sales by territory country)."
      verify: |
        SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = :target_cube_name;
  - rcls_seeded:
      doc: "Four demo personas (SYS / RCLS_CANADA / RCLS_EUROPE / RCLS_EXEC) exist, four ROW policies wired in SQLCUBE_REGISTRY.RCLS_ROW_POLICIES, virtual schema refreshed."
      verify: |
        SELECT COUNT(*) FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES WHERE MODEL_ID = :model_id;
        -- Expect 4 rows.
  - persona_switch_demonstrable:
      doc: "POST /api/query/execute {execution_user='RCLS_CANADA', sql=<sales-by-country>} returns only Canada rows; same SQL with execution_user='SYS' returns all countries."

parameters:
  required: []
  optional:
    - target_cube_name:
        default: "SQLCUBE_ACME_ADVENTUREWORKS"
        doc: "Virtual schema name created in Step 6."
    - source_connection_name:
        default: "SNOWFLAKE_CONNECTION"
        doc: "Exasol CONNECTION to use as the source."
    - source_db_filter:
        default: "ACME_DEMO"
    - source_schema_filter:
        default: "ADVENTUREWORKS"
    - target_schema:
        default: "ACME_ADVENTUREWORKS"
        doc: "Where the migrated tables land on Exasol."
    - approval_mode:
        default: "auto_accept"
        doc: "`auto_accept` runs phases continuously; `step_by_step` pauses after each one for user confirmation."
    - studio_base_url:
        default: "http://127.0.0.1:8002"
        doc: "Studio FastAPI base URL. Demo default is 8002 (Maglev Tour) — distinct from 8001 (other studio runs)."

estimated_impact:
  full_cold_run: "~60-65 sec end-to-end against demo-tour with Snowflake CONNECTION + migration scripts + EXA_OPTIMIZE UDFs already seeded. Cold-start dress rehearsal 2026-05-14 = 62.22s wall time. Breakdown: Target 1.0s | Source 3.3s (OCSP warmup) | Schema 7.4s | Migrate 28.7s (84k rows, 9 tables, Arrow JDBC parallel) | Optimize 9.0s (two-pass) | Cube 11.3s | Security 1.4s | Persona showcase 0.1s. Boss-demo target of <60s missed by 2s — driven mostly by Snowflake first-call OCSP and JDBC databases/schemas listing."
  warm_run: "~5 sec — warm-state probe detects cube + RCLS in place, skips Phases 1-7 verify-only, runs Phase 8 showcase directly."

internal: true
not_in_marketplace: true
---

# /maglev-tour — agent parody of the GUI Maglev Tour

End-to-end demo orchestrator. Same backend endpoints the GUI hits (`studio/frontend/src/pages/FactoryTour.jsx`); the agent replaces ActionLog widgets with `TaskCreate` / `TaskUpdate` progress. Source of truth: `exasol-zemantic-layer/exasol-zemantic-layer/FACTORY_TOUR_DEMO_PATH.md`.

## When to trigger

- User invokes `/maglev-tour` directly.
- User says "demo the factory tour", "show the boss the agent pipeline", "run the 8-step demo", "do the snowflake demo end to end".

Do **not** trigger for narrower asks. `/oneshot-tour` is the simpler intro flow (no RCLS, no persona switch) and should still be used when the goal is just cube + dashboard.

## What this skill is

Eight phases. Each phase hits a known studio HTTP endpoint, verifies the underlying DB state changed, narrates via TaskCreate/TaskUpdate, and routes to the appropriate domain skill for tool calls. The orchestrator owns sequencing + narration; the domain skills own tool calls.

| # | Phase | Skill | Primary endpoint |
|---|---|---|---|
| 1 | Target | exasol-nano-local + studio | `POST /api/connections/profiles/:id/activate` |
| 2 | Source | exasol-migrate | `POST /api/import/jdbc/test-connection` |
| 3 | Schema | exasol-migrate | `GET /api/import/jdbc/databases` + `/jdbc/schemas` |
| 4 | Migrate | exasol-migrate | `POST /api/import/snowflake/preview-migration` → fan-out `POST /api/import/snowflake/run-sql` |
| 5 | Optimize | exasol-optimize | `POST /api/optimize/analyze` (constraints + date_dim) → `POST /api/optimize/run-sql` per finding (two-pass) |
| 6 | Cube | exasol-semantic-layer | `GET /api/sqlcube/introspect-sql` → run + `POST /api/sqlcube/introspect-from-rows` → `POST /api/sqlcube/generate` → `POST /api/sqlcube/apply-metric-pack` → `POST /api/deploy/execute` |
| 7 | Security | **exasol-rcls** (this plugin) | `POST /api/browser/security/seed-demo` |
| 8 | Query | **exasol-query-personas** (this plugin) | `POST /api/query/execute {execution_user, sql}` for each persona |

## Routing algorithm

Five phases (Maglev Tour GUI matches this — sub-step grouping):

| Phase | Reference | What happens |
|---|---|---|
| 1. Interrogation | `references/interrogation.md` | Free-form Q&A until target_cube_name + source + filters are bound. Defaults from frontmatter usually suffice for the canonical demo. |
| 2. Plan presentation | `references/plan-presentation.md` | Render the 8-step plan as a checklist; ask for approval mode. |
| 3. Execution | `references/execution.md` | Run the 8 steps via the matching domain skill. Narrate via TaskCreate/TaskUpdate. Halt on first verify failure. |
| 4. Verification | `references/verification.md` | Confirm `cube_live` + `rcls_seeded` + `persona_switch_demonstrable` provides. |
| 5. Persona showcase | `references/persona-showcase.md` | The demo punchline. Three side-by-side persona queries (SYS / RCLS_CANADA / RCLS_EUROPE) showing row-count + Sales Amount divergence. |

## Conventions

- **Use the demo backend (port 8002 by default), not the generic studio port.** GUI demo + agent demo share `STUDIO_SESSION_SECRET`, so cookies from one session work in the other.
- **Studio FastAPI must be running.** Skill halts with a clear "boot the backend" message if not — does NOT silently fall back to direct SQL because the demo's metric-pack injection and seed-demo lifecycle only live in the backend.
- **demo-tour container is the canonical fixture.** Port 8568, container `exanano-demo-tour`. Snowflake CONNECTION and migration scripts pre-seeded. If running against a different container, expect Phase 1-3 to require additional setup.
- **Re-runs are idempotent.** `seed-demo` upserts; `/deploy/execute` drop+creates virtual schema; `optimize/run-sql` catches "already exists" errors. Safe to re-invoke the whole tour against demo-tour without state corruption.
- **GUI parity over invention.** When the skill must choose between a "cleaner" agent path and one that mirrors `FactoryTour.jsx`, prefer GUI parity — the boss is watching for "this is the same demo, just driven differently."
- **AI narration is real LLM, not pre-baked. The agent path beats the GUI here.** GUI's `factoryDemoLLM.js` ships static `OPTIMIZE_AI_TRANSCRIPT` + `CUBE_AI_TRANSCRIPT` strings + pre-authored `OPTIMIZE_AI_RECS` / `CUBE_AI_RECS` arrays (Flatten DIMPRODUCT snowflake, Role-play DIMDATE, Junk dim, AVG_ORDER_VALUE, GROSS_MARGIN_PCT, Grain-lock, Snowflake-hop). The GUI does NOT call an LLM in either step — the cards are static. The agent IS the LLM, so the agent's AI narration is real analysis of the actual UDF findings + parsed_ddl. Demo wins on "the AI is really thinking" because — in the agent path — it really is.

## Related skills

- **exasol-nano-local** — Phase 1 precondition.
- **exasol-migrate** — Phases 2-4.
- **exasol-optimize** — Phase 5.
- **exasol-semantic-layer** — Phase 6. See `references/deploy-flow.md` in that skill for the full HTTP contract.
- **exasol-rcls** (this plugin) — Phase 7.
- **exasol-query-personas** (this plugin) — Phase 8.
- **/oneshot-tour** — simpler 6-phase variant ending in dashboard, no RCLS / personas.

## Limitations

- **ADW-curated path.** The metric pack injected at Phase 6 (Gross Profit / Gross Margin Pct / Distinct Customer Count / Distinct Product Count) is auto-applied when the domain_id matches `internet_sales`. For other source schemas the pack is a no-op — the cube still deploys but the boss-friendly measures don't land.
- **RCLS seed is domain-specific.** `seed-demo` writes the four canonical row predicates against a DIMSALESTERRITORY-backed cube. Schemas without a Sales Territory Country attribute will see policies created but no row filtering.
- **No real Phase 4 dry-run flag.** The Snowflake migration script's DEBUG mode still hits the source catalog (live-verified). For a truly source-less Phase 4 (offline demo), use `POST /api/import/jdbc/preview` instead — but the GUI uses snowflake/preview-migration so the agent matches that for parity.
- **demo-tour-only.** This skill assumes the specific Snowflake → ACME_ADVENTUREWORKS path. Re-pointing at a different source (Postgres, BigQuery) requires updating defaults + may break the metric-pack auto-detection.
