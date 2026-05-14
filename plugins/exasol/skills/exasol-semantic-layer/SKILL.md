---
name: exasol-semantic-layer
description: Build a SQLCube semantic layer on top of an existing Exasol schema. Populates the SQLCUBE_REGISTRY metadata schema (Domains, Models, Dimensions, Measures, Joins, Attributes) and registers a virtual schema via SQLCUBE.ADAPTER so the data is queryable as a wide denormalized cube. Runs after exasol-optimize.

preconditions:
  - schema_optimized:
      doc: "Source schema has PKs, FKs, narrowed types declared by exasol-optimize. The skill reads these from SYS.EXA_ALL_CONSTRAINTS to build the JOIN graph."
      check: |
        # SQL:
        # SELECT 1 FROM SYS.EXA_ALL_CONSTRAINTS
        # WHERE CONSTRAINT_SCHEMA = '<SOURCE_SCHEMA>'
        #   AND CONSTRAINT_TYPE = 'PRIMARY KEY'
        # LIMIT 1;
        # Returns >=1 row when satisfied.
      satisfied_by: exasol-optimize
  - sqlcube_installed:
      doc: "SQLCUBE.ADAPTER UDF and the registry schema (default SQLCUBE_REGISTRY) are deployed. Pre-shipped on exanano-sqlcube; one-time install on other Exasol clusters."
      check: |
        # SQL — both must pass:
        # SELECT 1 FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA='SQLCUBE' AND SCRIPT_NAME='ADAPTER';
        # SELECT 1 FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA='<registry_schema>' AND TABLE_NAME='MODELS';
      satisfied_by: null   # one-time install; see references/install.md if missing
  - exa_optimize_udfs_installed:
      doc: "EXA_OPTIMIZE schema holds the 10 deploy-flow UDFs (ANALYZE_CONSTRAINTS, INFER_JOIN_PATHS, DRY_RUN_PLAN, etc.) the skill uses for constraint discovery and join-graph inference."
      check: |
        # SQL:
        # SELECT COUNT(*) FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA='EXA_OPTIMIZE';
        # Returns >= 5 when exasol-optimize has installed the bundle.
      satisfied_by: exasol-optimize
  - studio_backend_reachable:
      doc: "Studio FastAPI backend reachable over HTTP (default http://localhost:8001). Enables the preferred /sqlcube/generate → /validate → /deploy/execute flow. When unreachable the skill degrades to direct UDF + hand-crafted SQL — see references/deploy-flow.md path B/C."
      check: |
        # Shell:
        # curl -sf http://localhost:8001/api/health >/dev/null && echo OK
        # OR: factory-foundation/scripts/agent_env_check.sh
      satisfied_by: null   # operator-managed; degrade gracefully if absent

provides:
  - cube_live:
      doc: "A virtual schema named SQLCUBE_<layer_id> exists, is wired to SQLCUBE.ADAPTER, has at least one MODELS row in the registry, and a SELECT round-trip against its lowercase virtual table returns rows."
      verify: |
        # SQL — all three must pass:
        # SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = 'SQLCUBE_<LAYER_ID>';
        # SELECT 1 FROM <registry_schema>.MODELS WHERE DOMAIN_ID = '<one of the deployed domains>';
        # SELECT * FROM SQLCUBE_<LAYER_ID>."<lowercase_model_id_or_domain_id>" LIMIT 1;

parameters:
  required:
    - source_schema:
        doc: "Underlying Exasol schema the cube reads from (e.g., ADVENTUREWORKS). Sanitized to form the virtual schema name: SQLCUBE_<source_schema_upper>."
  optional:
    - layer_id:
        default: null
        doc: "Override the auto-derived semantic-layer id. Default: sanitized source_schema. Determines the virtual schema name (SQLCUBE_<layer_id>) and groups multiple domains under one virtual schema."
    - registry_schema:
        default: SQLCUBE_REGISTRY
        doc: "Schema holding the cube metadata tables. Default matches the studio backend convention; the name is configurable in case the convention changes (settings.exa_registry_schema)."
    - adapter_script:
        default: SQLCUBE.ADAPTER
        doc: "Adapter script name. Almost never overridden — the studio deploys a single canonical adapter."
    - llm_enrichment:
        default: true
        doc: "When true, agent fills business labels, descriptions, additional measures via LLM and writes to MODELS / MEASURES / ATTRIBUTES. When false, only structural defaults from the optimize join graph land."
    - measures_overrides:
        default: null
        doc: "Path to YAML file with hand-written measure definitions. User-supplied wins over LLM and over structural defaults."
    - dimensions_overrides:
        default: null
        doc: "Path to YAML file with attribute renames, IS_VISIBLE flags, sort-order hints."
    - drop_existing:
        default: false
        doc: "If true, DROP VIRTUAL SCHEMA <name> CASCADE before recreating. Skill DELETEs registry rows by domain_id regardless (upsert pattern). Setting drop_existing=true also removes the virtual schema wrapper."
    - mode:
        default: multi_domain
        doc: "`multi_domain` uses LAYER_ID + MODEL_IDS properties so one virtual schema exposes multiple domain tables. `single_model` (legacy) uses MODEL_ID only — kept for backward-compatible flows."

estimated_impact:
  v01_full_run: "30 sec – 3 min depending on table count and llm_enrichment. No source-schema modifications; cube layer is read-only. Registry INSERTs are idempotent (DELETE-then-INSERT by domain_id)."
  llm_token_cost: "~5–15K tokens per 10 tables with default Haiku model."
---

# Exasol Semantic Layer Skill

Trigger when the user asks to **build a cube**, **create a semantic layer**, **make this queryable in natural language**, **expose this for the BI layer**, **wrap this for MCP**, or **make SQLCube models**. Also when an orchestrator skill declares a `cube_live` precondition.

## What this skill is

The SQLCube semantic layer is an Exasol-native cube built on three pieces:

- **`SQLCUBE.ADAPTER`** — a Lua adapter UDF that intercepts queries against a virtual schema and rewrites them into a wide denormalized SELECT against physical tables. Source lives in `exasol-factory-foundation/sqlcube/adapter/`.
- **`SQLCUBE_REGISTRY`** — the metadata schema (DOMAINS, MODELS, DIMENSIONS, MEASURES, DERIVED_MEASURES, JOINS, JOIN_PATHS, ATTRIBUTES, RCLS_*). One row in MODELS per fact-table grain; child rows describe which dim columns are exposed and how joins resolve. Schema name configurable via `settings.exa_registry_schema` — defaults to `SQLCUBE_REGISTRY`.
- **`SQLCUBE_<LAYER_ID>` virtual schemas** — what analysts and MCP clients see. Each contains one virtual table per model (lowercase identifier). SELECTs against them return a flat row of (chosen dim attributes + aggregated measures) per group.

What it is NOT:

- Not a Kimball star-schema query surface. Querying the cube returns one wide denormalized table, not a join graph the user assembles themselves.
- Not OLAP / MDX. No pre-aggregation, no cache layer. Adapter rewrites query → underlying physical SQL each call.
- Not a federation layer. Source data must already be in Exasol (see `exasol-migrate`).

## Routing algorithm

| Phase | Reference | When |
|---|---|---|
| Architecture overview | `references/architecture.md` | Read first; explains Models vs Domains, virtual-schema naming, adapter runtime |
| **Deploy flow** | **`references/deploy-flow.md`** | **Primary** — studio backend endpoints + `EXA_OPTIMIZE` UDFs do every step. Default agent path. |
| LLM enrichment | `references/llm-enrichment.md` | When `/sqlcube/generate llm_enrichment=true` won't suffice or you're filling proposals after the fact |
| Query patterns | `references/query-patterns.md` | After cube is live — wide-denormalized SELECT shapes |
| Registry metadata model | `references/meta-model.md` | Reference for the registry table shapes the deployer writes. Read when debugging, not when authoring SQL by hand. |
| Cube creation (fallback) | `references/cube-creation.md` | **Fallback only** — pure hand-crafted SQL for clusters with no studio backend reachable. Don't default here. |
| Install | `references/install.md` | When `sqlcube_installed` precondition fails |
| Troubleshooting | `references/troubleshooting.md` | When CREATE VIRTUAL SCHEMA or query fails |

## Pipeline (preferred — studio-driven; see `deploy-flow.md`)

```
1. Verify preconditions             (schema_optimized AND sqlcube_installed; EXA_OPTIMIZE UDFs present)
2. Declare constraints              (EXA_OPTIMIZE.ANALYZE_CONSTRAINTS → /optimize/apply-review-proposal OR DRY_RUN_PLAN — owned by exasol-optimize skill)
3. Infer join graph                 (EXECUTE SCRIPT EXA_OPTIMIZE.INFER_JOIN_PATHS)
4. Parse source DDL                 (GET /api/sqlcube/introspect-sql?schema=<S>)
5. Generate SqlcubeModel            (POST /api/sqlcube/generate llm_enrichment=true)
6. (Optional) Run skill prompts     (prompts/cube-enrichment.md if /generate's deterministic draft needs richer LLM proposals)
7. Validate                         (POST /api/sqlcube/validate)
8. Pre-flight conflict check        (POST /api/deploy/check)
9. Execute deploy                   (POST /api/deploy/execute → services.deployer.deploy() does registry + adapter + virtual schema)
10. Verify cube_live                (provides contract — 3-step SQL check)
11. Hand back to caller
```

Steps 2–9 are HTTP / UDF calls. The agent never hand-writes `INSERT INTO SQLCUBE_REGISTRY.*` SQL in this path. Only fall back to the hand-crafted batch in `cube-creation.md` when the studio backend is unreachable.

## Step 0: Preconditions

Four prerequisites:

1. **Schema is optimized.** `exasol-optimize` must have run so `SYS.EXA_ALL_CONSTRAINTS` has PKs and FKs for the source schema. The studio `/sqlcube/generate` endpoint reads these to build join paths; `EXECUTE SCRIPT EXA_OPTIMIZE.INFER_JOIN_PATHS('<S>')` falls back to name-stem matching when declared FKs are missing.
2. **SQLCUBE infrastructure installed.** Three things: `SQLCUBE` schema with `ADAPTER` UDF, the registry schema (default `SQLCUBE_REGISTRY`) with all required tables, and the adapter Lua bundle uploaded to `/buckets/bfsdefault/default/sqlcube_adapter.lua` (or equivalent). `exanano-sqlcube` ships this pre-installed. For others see `references/install.md`.
3. **`EXA_OPTIMIZE` UDFs installed.** The 10-script bundle (`ANALYZE_CONSTRAINTS`, `INFER_JOIN_PATHS`, `DRY_RUN_PLAN`, etc.) — installed by the `exasol-optimize` skill or directly via `studio/backend/services/scripts_library.py:install_optimize_scripts()`. Without these the constraint-discovery + join-inference steps in `deploy-flow.md` have no surface to call.
4. **Studio backend reachable (preferred).** Enables `/sqlcube/generate` + `/sqlcube/validate` + `/deploy/execute` — the primary deploy flow. When the backend isn't reachable, fall back to `deploy-flow.md` path B (UDF-only) or C (pure hand-crafted SQL).

If any of #1–#3 fails → route to the satisfying skill before proceeding. If #4 fails → degrade to fallback path; do not break.

## Conventions

- **Virtual schema naming.** `SQLCUBE_<LAYER_ID>` where layer_id = sanitized source_schema. ADVENTUREWORKS → `SQLCUBE_ADVENTUREWORKS`. Sanitizer (`_safe_identifier_token`) uppercases, replaces non-alnum with underscore, collapses repeats, truncates to 120 chars.
- **One MODELS row per fact-table grain.** `MODEL_ID` is a short stable identifier (e.g., `internet_sales`); `FACT_SCHEMA` + `FACT_TABLE` qualify the underlying table. The studio deployer always assigns `MODEL_ID == DOMAIN_ID` — see `references/meta-model.md` conventions.
- **Multiple models per source schema = multi-domain.** All share one virtual schema. Each model becomes a virtual table inside (lowercase model_id).
- **DIMENSIONS are column-level, not table-level.** Each row exposes one virtual column from one physical dim. `DIM_TABLE_ALIAS` (e.g., `dim`, `dim1`) ties back to the JOINS row that brings the dim in.
- **MEASURES are inner expressions + AGG_TYPE.** `PHYSICAL_EXPR='f.EXTENDEDAMOUNT'` and `AGG_TYPE='SUM'` are stored separately so the adapter can compose `SUM(f.EXTENDEDAMOUNT)` at query time. Use `f` as the fact alias by convention.
- **Adapter is runtime-driven.** It reads the registry tables on every query (`load_model_meta()` in `metadata_registry.lua`). No `REFRESH VIRTUAL SCHEMA` required after registry updates — the next query picks them up.
- **Upsert by domain_id.** The skill DELETEs all rows for a domain (in FK-correct order) then re-INSERTs. Don't try partial updates — diff complexity > rewrite cost.

## When to invoke

- `/oneshot-tour` orchestrator declares `cube_live` precondition after `schema_optimized`.
- Developer asks "expose this schema as a semantic layer."
- Analyst wants natural-language access via MCP / Claude Desktop.
- Source schema gained new tables — re-run to incorporate.

## Related skills

- **exasol-optimize** — required predecessor. PKs / FKs feed JOINS row generation.
- **exasol-migrate** — typical predecessor of `exasol-optimize`. Brings external data into the schema this cube models.
- **exasol-dashboard** — natural successor. Renders Vega-Lite panels against the cube.
- **oneshot-tour** — orchestrator that chains migrate → optimize → semantic-layer → dashboard.

## Limitations

- **Single source schema per layer.** v0.1 reads one Exasol schema. Cross-schema cubes are blocked by the adapter's current source resolution.
- **No row-level security from this skill.** RCLS_ROW_POLICIES / RCLS_ATTRIBUTE_POLICIES tables exist in the registry but v0.1 of this skill does not populate them. Configure manually if needed.
- **LLM enrichment is non-deterministic.** Lock outputs by capturing the resulting registry rows or supply `measures_overrides` YAMLs.
- **Registry schema name is hardcoded to default.** While the skill exposes `registry_schema` as a parameter, the adapter UDF currently reads from `settings.exa_registry_schema` baked into the studio backend. Overriding the parameter without redeploying the adapter will break query-time resolution.
- **No DDL emission for the underlying tables.** Cube is metadata only. Type changes / column renames on source tables require optimize re-run + this skill re-run.
