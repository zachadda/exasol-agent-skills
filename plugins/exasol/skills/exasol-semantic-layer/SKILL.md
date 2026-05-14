---
name: exasol-semantic-layer
description: Build a SQLCube semantic layer on top of an existing Exasol schema. Creates a virtual schema via SQLCUBE.ADAPTER backed by SQLCUBE_META, exposes business-level dimensions / measures / named-relationships, and registers the cube so MCP and downstream clients can query in natural language. Runs after exasol-optimize.

preconditions:
  - schema_optimized:
      doc: "Source schema has PKs, FKs, narrowed types, and a join graph emitted by exasol-optimize."
      check: |
        # SQL:
        # SELECT 1 FROM SYS.EXA_ALL_CONSTRAINTS
        # WHERE CONSTRAINT_SCHEMA = '<SOURCE_SCHEMA>'
        #   AND CONSTRAINT_TYPE = 'PRIMARY KEY'
        # LIMIT 1;
        # Returns >=1 row when satisfied.
      satisfied_by: exasol-optimize
  - sqlcube_installed:
      doc: "SQLCUBE schema + SQLCUBE.ADAPTER UDF + SQLCUBE_META schema all present."
      check: |
        # SQL:
        # SELECT COUNT(*) FROM SYS.EXA_ALL_SCRIPTS
        # WHERE SCRIPT_SCHEMA = 'SQLCUBE' AND SCRIPT_NAME = 'ADAPTER';
        # Returns 1 when satisfied.
      satisfied_by: null   # one-time install; see references/install.md if missing

provides:
  - cube_live:
      doc: "A virtual schema named <cube_name> exists, points at SQLCUBE.ADAPTER, and SELECT against it returns rows from the underlying model. SQLCUBE_META.MODELS has an entry."
      verify: |
        # SQL:
        # SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = '<CUBE_NAME>';
        # SELECT * FROM SQLCUBE_META.MODELS WHERE MODEL_NAME = '<CUBE_NAME>';
        # SELECT * FROM <CUBE_NAME>.<ANY_TABLE> LIMIT 1;

parameters:
  required:
    - source_schema:
        doc: "Underlying Exasol schema (already optimized) the cube reads from."
    - cube_name:
        doc: "Virtual schema name. Convention: `<SOURCE>_CUBE` (e.g., ADVENTUREWORKS → ADVENTUREWORKS_CUBE)."
  optional:
    - llm_enrichment:
        default: true
        doc: "If true, agent generates business descriptions, measure definitions, and named-relationship labels via LLM and stores in SQLCUBE_META. If false, ships a structural cube only — analysts will see raw column names."
    - measures_overrides:
        default: null
        doc: "Path to a YAML file with hand-written measure definitions. Merged with LLM output; user definitions win on conflict."
    - dimensions_overrides:
        default: null
        doc: "Path to a YAML file with hand-written dimension role / hierarchy hints."
    - drop_existing:
        default: false
        doc: "If true, DROP VIRTUAL SCHEMA <cube_name> first. Destructive — agent must confirm before setting true."
    - register_in_mcp:
        default: true
        doc: "If true, write an MCP registration manifest so the cube becomes discoverable via the agent's exasol_db MCP server without manual restart."

estimated_impact:
  v01_full_run: "30 sec – 3 min depending on table count and llm_enrichment. No source-schema modifications; cube is read-only."
  llm_token_cost: "~5–15K tokens per 10 tables with default Haiku model."
---

# Exasol Semantic Layer Skill

Trigger when the user asks to **build a cube**, **create a semantic layer**, **make this queryable in natural language**, **expose this for the BI layer**, **wrap this for MCP**, or **make SQLCube models**. Also when an orchestrator skill declares a `cube_live` precondition.

## What this skill is

The SQLCube semantic layer is an Exasol-native virtual schema that:

- Mounts a physical schema as a queryable virtual one
- Adds business-level metadata (descriptions, measures, named relationships, hierarchies) stored in `SQLCUBE_META`
- Routes queries through `SQLCUBE.ADAPTER` — an Exasol Virtual Schema adapter UDF that rewrites natural query patterns into SQL against the physical tables
- Registers with MCP so `mcp__exasol_db__*` tools see the cube as a discoverable model

What it is NOT:

- Not a Kimball-style aggregate store. Queries hit physical tables, no pre-materialization.
- Not OLAP cubes (SSAS/Mondrian). No MDX, no pre-aggregation, no caching layer in v0.1.
- Not a federation layer for external sources — that's `exasol-migrate`'s job upstream.

## Routing algorithm

| Phase | Reference | When |
|---|---|---|
| Architecture overview | `references/architecture.md` | Read first; explains SQLCUBE schema, ADAPTER UDF, META tables |
| Cube creation | `references/cube-creation.md` | Always — the CREATE VIRTUAL SCHEMA step |
| Meta-model schema | `references/meta-model.md` | When populating SQLCUBE_META.MODELS / DIMENSIONS / MEASURES / RELATIONSHIPS |
| LLM enrichment | `references/llm-enrichment.md` | When `llm_enrichment=true` (default) |
| Query patterns | `references/query-patterns.md` | Once cube is live — canonical SELECT shapes |
| Troubleshooting | `references/troubleshooting.md` | When CREATE VIRTUAL SCHEMA or query fails |

## Pipeline

```
1. Verify preconditions       (schema_optimized AND sqlcube_installed)
2. Read join graph + DDL from source schema (Exasol catalog + optimize manifest)
3. Generate base model        (tables → dimensions/facts via _DIM/_FACT suffix detection)
4. LLM enrichment (optional)  (business names, descriptions, measure formulas)
5. Merge user overrides       (measures_overrides + dimensions_overrides YAMLs)
6. Write SQLCUBE_META rows    (MODELS, DIMENSIONS, MEASURES, RELATIONSHIPS, COLUMNS)
7. CREATE VIRTUAL SCHEMA       (mounting the model)
8. Verify cube_live            (provides contract — 3-step SQL check)
9. Register with MCP (optional) (write manifest, signal hot-reload)
10. Hand back to caller
```

## Step 0: Preconditions

Two prerequisites:

1. **Schema is optimized.** `exasol-optimize` must have run — at minimum, PKs declared and FK join graph emitted. The semantic layer's relationship inference is built on `SYS.EXA_ALL_CONSTRAINTS`, not raw column matching. If FKs aren't declared, the cube will work but every join becomes a CROSS JOIN — silently wrong results.

2. **SQLCUBE infrastructure installed.** SQLCUBE schema, SQLCUBE.ADAPTER UDF, SQLCUBE_META schema and tables. One-time install per Exasol cluster — see `references/install.md`. The `exanano-sqlcube` container has this pre-installed.

If either fails the precondition check → route to satisfying skill before proceeding.

## Conventions

- **One cube per source schema.** Multi-source cubes are possible but they're a v0.2+ feature — keep v0.1 simple.
- **Cube name convention: `<SOURCE>_CUBE`.** Easy to back-trace, hard to confuse with a physical schema.
- **Virtual schemas are read-only.** SELECT only. No INSERT / UPDATE / DELETE through the cube. Writes must go to the underlying physical schema.
- **Measures are SQL expressions, not stored values.** `SUM(LINEITEM.QUANTITY * LINEITEM.UNITPRICE * (1 - LINEITEM.DISCOUNT))` is a measure — Exasol computes per-query.
- **Named relationships use the optimize-discovered join graph.** Don't re-invent join inference here; trust the upstream.

## When to invoke

- `/oneshot-tour` orchestrator declares `cube_live` precondition after `schema_optimized`.
- Developer asks "expose this schema as a semantic layer."
- Analyst wants to query the warehouse in natural language via MCP / Claude Desktop.
- Refresh: cube exists but user has added new tables to the source — re-run to incorporate.

## Related skills

- **exasol-optimize** — required predecessor. Provides PKs / FKs / join graph that the cube's relationship layer reads.
- **exasol-migrate** — typical predecessor of `exasol-optimize`. Brings external data into the schema this cube will model.
- **exasol-dashboard** — natural successor. Builds an HTML+Vega-Lite dashboard against the live cube.
- **oneshot-tour** — orchestrator that chains migrate → optimize → semantic-layer → dashboard.

## Limitations

- **Single-source.** v0.1 mounts one Exasol schema per cube. Multi-source federation = v0.2.
- **No row-level security.** All queries against the cube execute as the connecting user. RLS needs to be declared on the physical tables.
- **LLM enrichment is non-deterministic.** Re-running with the same source schema can produce slightly different descriptions / measure names. The `measures_overrides` YAML is how you lock in what you want.
- **MCP registration requires the MCP server to support hot-reload.** Without it, agent must restart for new cubes to appear. exanano-sqlcube's MCP server does hot-reload; remote Exasol Personal does not (yet).
