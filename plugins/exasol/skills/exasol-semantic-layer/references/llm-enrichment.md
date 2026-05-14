# LLM enrichment

Generates business descriptions, measure proposals, attribute display names. Without this step the cube is structurally sound but cosmetically sparse — analysts see raw physical column names.

**Three ways to get LLM enrichment**, in order of preference:

1. `POST /sqlcube/generate { llm_enrichment: true }` — studio backend handles the prompt via `services/claude.py` and returns an enriched `SqlcubeModel` ready for `/deploy/execute`. Default path. See `deploy-flow.md` step 5.
2. Agent-side loop using the prompts in `../prompts/` — when you want to drive enrichment yourself (e.g. richer measure proposals than the studio default, custom domain framing). Output JSON merges into the model before `/sqlcube/validate`.
3. Offline pack — GUI users download a ZIP, hand it to an external LLM, upload the returned `.md`. Same shape as #2 but with a human in the loop. Agents skip this.

Target schema: `SQLCUBE_REGISTRY` (configurable via `registry_schema` parameter).

## What gets enriched

| Target | Enrichment | Where stored |
|---|---|---|
| DOMAINS.DOMAIN_NAME | Business label for the domain | `SQLCUBE_REGISTRY.DOMAINS.DOMAIN_NAME` |
| DOMAINS.DESCRIPTION | One-paragraph cube summary | `SQLCUBE_REGISTRY.DOMAINS.DESCRIPTION` |
| MODELS.MODEL_LABEL | Business name for the model (default = title-cased fact table) | `SQLCUBE_REGISTRY.MODELS.MODEL_LABEL` |
| ATTRIBUTES.BUSINESS_NAME | Domain-level display label per physical column | `SQLCUBE_REGISTRY.ATTRIBUTES.BUSINESS_NAME` |
| ATTRIBUTES.DISPLAY_TYPE / DISPLAY_SCALE | UI rendering hints | `SQLCUBE_REGISTRY.ATTRIBUTES` |
| DIMENSIONS.VIRTUAL_COL | Per-model display name (often inherits from ATTRIBUTES) | `SQLCUBE_REGISTRY.DIMENSIONS.VIRTUAL_COL` |
| MEASURES (new rows) | Business-specific measure proposals beyond the structural defaults | `SQLCUBE_REGISTRY.MEASURES` |
| MEASURES.VIRTUAL_COL / FORMAT_MASK | Display name + format mask per measure | `SQLCUBE_REGISTRY.MEASURES` |
| JOIN_PATHS | Domain-level alternative join graph (renames + roles) | `SQLCUBE_REGISTRY.JOIN_PATHS` |

## Inputs to the LLM

For each cube, build a context blob with:

1. **Source schema name + table names** (e.g., "ADVENTUREWORKS: DIMCUSTOMER, DIMPRODUCT, DIMDATE, FACTINTERNETSALES")
2. **DDL per table** — `SELECT * FROM SYS.EXA_DBA_COLUMNS WHERE COLUMN_SCHEMA = '<S>' AND COLUMN_TABLE = '<T>'`
3. **Sample rows per table** — `SELECT * FROM <S>.<T> SAMPLE 10` (Exasol's SAMPLE clause is cheap)
4. **Optimize-emitted join graph** — JOINS / JOIN_PATHS structure already populated by the structural pass from declared FKs
5. **Domain hint** if available — source name (`adventureworks` → retail), or user-passed `domain_hint` parameter

Build the prompt around domain hint + schema + samples. Avoid sending row data that contains PII — sample with `SELECT col1, col2 ... FROM ... SAMPLE 10` and strip columns named `EMAIL`, `SSN`, `PASSWORD`, `PHONE`, `TAX_ID` before serialization.

## Prompt shape (per cube)

Single request per cube, batched output:

```
You are designing the business surface of a semantic layer over a database schema.

Schema name: ADVENTUREWORKS
Domain hint: Retail / e-commerce (inferred from name)

Tables and structure:
[DDL + 5–10 sample rows per table, joined into one blob]

Existing join graph (from FK discovery):
[JOINS + JOIN_PATHS rows as a table]

Existing structural measures (default aggregations):
[MEASURES rows as a table]

Output JSON conforming to this schema:
{
  "model_description": "...",
  "dimensions": [
    {"physical_table": "...", "business_name": "...", "description": "...", "columns": [{"physical_column": "...", "display_name": "...", "description": "..."}]}
  ],
  "facts": [
    {"physical_table": "...", "business_name": "...", "grain": "...", "columns": [...]}
  ],
  "additional_measures": [
    {"fact": "...", "name": "...", "expression": "...", "aggregation": "...", "format": "...", "description": "..."}
  ],
  "relationship_renames": [
    {"existing_name": "...", "new_name": "..."}
  ]
}

Guidelines:
- Be conservative with `additional_measures` — only suggest aggregations that are clearly meaningful in this domain.
- Don't invent columns or tables.
- `expression` must reference only columns visible in the listed DDL.
- Don't restate the structural defaults — only ADD measures that are business-specific.
- Keep descriptions concise (one sentence each).
- Business names should be domain-appropriate, not generic.
```

## Model selection

Default: Haiku-class for cost efficiency. The enrichment is structurally simple — the LLM is consuming DDL + samples and producing JSON, not deep reasoning. Sonnet only if `domain_hint = complex` or the user passes `enrichment_model = sonnet` (parameter, not in v0.1).

Token budget: ~5–15K tokens total per cube of 10 tables. Roughly $0.02–$0.05 per cube on Haiku, $0.10–$0.30 on Sonnet.

## Validation

Before writing LLM output to SQLCUBE_REGISTRY:

1. **Reject hallucinated tables.** Every `physical_table` in the JSON must exist in `SYS.EXA_ALL_TABLES` for the source schema. If not → drop that entry, log warning.
2. **Reject hallucinated columns.** Every `physical_column` and every column referenced in a measure `expression` must exist in `SYS.EXA_ALL_COLUMNS`. Parse the expression for identifiers and cross-check.
3. **Reject malformed expressions.** Dry-run each measure expression as `SELECT <expr> FROM <fact> LIMIT 0`. If Exasol errors → log and drop the measure.
4. **Cap measure count.** If LLM returns >20 additional measures per fact → trim to top 10 by perceived business value (or just take the first 10). Too many measures clutter the cube.
5. **Lint descriptions.** Replace empty / nonsense strings (single-letter, "TODO", placeholder text) with NULL.

Validation results go into the manifest:

```
ADVENTUREWORKS_CUBE.enrichment.log
- accepted: <count> measures, <count> dim descriptions, <count> column descriptions
- rejected_table: <list>
- rejected_column: <list>
- malformed_expression: <list>
```

## User-override flow

User-supplied `measures_overrides` / `dimensions_overrides` YAMLs win over LLM output. Merge order:

```
1. Structural defaults (from cube-creation.md)
2. LLM enrichment (this skill)
3. User overrides (parameter YAMLs)
```

Later layers replace earlier ones on key match. Key for measures = `(fact, name)`. Key for dimensions = `physical_table`. Key for columns = `(entity, physical_column)`.

## Disable path

Skip the entire LLM call when:

- `llm_enrichment=false` parameter
- `ANTHROPIC_API_KEY` not set in env (and no alternate provider configured)
- Source schema has >100 tables (LLM context blows; trigger the "too large to enrich naively" warning and tell user to set `llm_enrichment=false` or paginate)

When disabled, cube ships with structural defaults only. Manifest notes `enrichment: skipped`.

## Reproducibility

LLM output varies per call. Two options for reproducibility-sensitive callers:

- **Lock with overrides YAML.** Run once, capture the output as YAML, commit to repo, future runs read the YAML and skip enrichment.
- **Pin seed (provider-dependent).** Anthropic API doesn't currently expose deterministic mode; OpenAI does via `seed`. For deterministic CI: don't rely on LLM, use YAML overrides.

## Cost notice

The skill SHOULD log token consumption at the end:

```
LLM enrichment: 12 tables, 1 cube, ~8400 tokens, ~$0.03 (Haiku).
```

Show this to the user. Cost transparency is non-negotiable for an agent-driven workflow.
