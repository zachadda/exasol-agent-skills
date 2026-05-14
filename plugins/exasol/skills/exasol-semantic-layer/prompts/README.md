# Semantic-layer LLM prompts

Prompt templates for the agent-side enrichment loop. **Equivalent to the GUI "offline pack" workflow** in `factory-foundation/studio/frontend/src/lib/offlinePacks.js` — where a human downloads a ZIP, hands the bundled `prompt.md` + `cube_context.json` to Claude/GPT externally, then uploads the returned `sqlcube_metric_proposals.md` / `sqlcube_optimization_review.md` back to Studio. The agent loop collapses both halves: agent IS the LLM, so the round trip disappears.

Use these when:

1. Calling `POST /sqlcube/generate llm_enrichment=true` isn't enough — typically because you want richer measure proposals or a domain-specific narration on top of the deterministic draft.
2. The studio backend isn't reachable but you still want enrichment locally.
3. You're explicitly running the offline-pack agent loop (skill prompt = pack prompt content, agent IS the LLM, agent merges results back via `/sqlcube/validate` + `/deploy/execute`).

The studio backend has its own LLM path (`services/claude.py`); these skill prompts are what the agent runs when driving the loop itself.

See `references/deploy-flow.md` (primary flow) and `references/llm-enrichment.md` (merge mechanics) for where the prompt output lands in the pipeline.

## Prompts in this directory

| File | Purpose | Output | Cost |
|---|---|---|---|
| `cube-enrichment.md` | Full cube enrichment in one call | JSON: model description + dim/fact business names + measure suggestions + relationship renames + column descriptions | ~5–15K tokens |
| `measure-business-context.md` | Domain-specific measure proposal beyond structural defaults | JSON: list of `{fact, name, expression, aggregation, format, description}` | ~3–8K tokens (subset of above) |
| `domain-detection.md` | Infer the business domain from schema name + samples (used when caller doesn't supply `domain_hint`) | JSON: `{domain, sub_domain, confidence}` | ~500 tokens |

## Order of calls

For a fresh cube with no user hints:

```
1. domain-detection.md       (one call, 500 tokens)
2. cube-enrichment.md         (one call, 5–15K tokens, domain from #1 injected)
```

For incremental updates (user adds a new measure):

```
1. measure-business-context.md   (one call, targeted)
```

For reproducibility-sensitive flows: skip prompts entirely, supply `measures_overrides` + `dimensions_overrides` YAMLs. The skill merges YAML over structural defaults, no LLM round-trips.

## Contract

All prompts:

1. **JSON in, JSON out.** No prose. Validation runs after.
2. **Schema-grounded.** Every identifier in the LLM output must resolve to a
   real table/column in the source schema. The validator drops hallucinated
   rows.
3. **Conservative defaults.** Don't invent measures the data can't support.
   Don't rename dimensions to something the LLM thinks is "better" unless
   the original name is genuinely cryptic.
4. **Domain hint propagation.** Once detected (or supplied), it's injected
   into every subsequent prompt so the LLM stays in domain.
5. **PII scrubbing at prompt-build.** Sample rows passed to the LLM are
   stripped of columns matching `EMAIL`, `SSN`, `PHONE`, `TAX_ID`,
   `PASSWORD`, `CREDIT_CARD`, `DOB` (case-insensitive). See
   `pii-patterns.md` (plugin-level shared file, if it exists).

## Model selection

Default: Haiku for everything except `cube-enrichment.md` on large schemas
(>30 tables) where Sonnet is worth it. The skill auto-routes:

```
if table_count <= 30 OR enrichment_model_override is set: Haiku
else: Sonnet
```

User can override via `enrichment_model` parameter (not yet exposed in v0.1
SKILL.md frontmatter — add when needed).

## Disable

`llm_enrichment=false` in skill parameters → skip all prompts in this
directory. Cube ships with structural defaults only.
