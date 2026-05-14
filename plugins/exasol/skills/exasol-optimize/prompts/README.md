# Optimize LLM prompts

Prompt templates the skill substitutes and sends to the LLM during the
`llm_enrichment` phase of `exasol-optimize`. Each file declares its inputs,
expected output shape, and validation rules.

## When the skill uses these

The optimize pipeline is **deterministic** for PK / FK discovery, type
narrowing, and DDL emission. The LLM never decides what to ALTER. It only
**annotates** the analysis with hints a human reviewer would find useful.

Hints are returned inline as JSON to the skill caller. The skill renders
them alongside the DRY_RUN_PLAN output so the user sees both the structural
change AND the LLM's read on what it means.

No persistence layer in v0.1. Hint outputs live for the duration of the
plan-presentation phase and are not stored back to Exasol. If a downstream
skill (semantic-layer enrichment, dashboard panel-planning) needs the same
LLM-derived metadata, the orchestrator re-invokes the prompt or passes the
JSON through. Cross-session persistence would need its own design (likely a
new table in `SQLCUBE_META` rather than a separately-managed schema, since
that's the existing app-metadata home — see studio backend's
`bootstrap.sql`).

## Prompts in this directory

| File | Purpose | Inputs | Output |
|---|---|---|---|
| `column-naming-hints.md` | Suggest business names + descriptions for cryptic columns | DDL + 5–10 sample rows per table | JSON `{table, column, suggested_name, description, confidence}` per column |
| `hierarchy-detection.md` | Flag self-FK candidates as hierarchical relationships | self-FK list + parent/child sample paths | JSON `{table, fk_column, hierarchy_kind, confidence, reasoning}` per candidate |

## Contract

All prompts in this directory:

1. **Take JSON in, return JSON out.** No prose responses. Skill parses and
   validates.
2. **Reference real columns only.** Schema is supplied as part of the prompt
   context. Any hallucinated identifier is rejected during validation.
3. **Include a `confidence` field** (`high` / `medium` / `low`). Skill
   surfaces low-confidence hints with a "review" tag instead of silently
   incorporating.
4. **Strip PII at prompt-build time, not at LLM time.** Skill scans column
   names against the patterns in `pii-patterns.md` (shared across plugin)
   and drops sample values before serialization.

## Model selection

Default: Haiku. Optimize enrichment is structurally simple — the LLM is
reading samples and suggesting names. Sonnet only when explicitly requested
via `enrichment_model=sonnet` (parameter not yet exposed in v0.1).

Token budget: ~3–8K total per schema of 20 tables. ~$0.01–0.02 on Haiku.
