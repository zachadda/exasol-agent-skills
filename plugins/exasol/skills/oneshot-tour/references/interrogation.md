# Phase 1 — Interrogation

The orchestrator must fill every `parameters.required` field before plan presentation. This file is the agent's playbook for the Q&A.

## Goal

Collect the structured parameter set defined in `SKILL.md` frontmatter. Use free-form natural language. Confirm each parameter back to the user. Validate the answer is plausible before moving on.

## Parameter checklist

Every field below must be filled before the agent leaves Phase 1.

| Field | How to ask (LLM choice) | How to validate |
|---|---|---|
| `source_kind` | "What kind of source are we loading from?" — enumerate options if user looks stuck | Must be one of `snowflake`, `s3`, `acme_demo_fixture`, `other_jdbc` |
| `source_connection_name` | "Which Exasol connection should I use?" — offer to list existing | Run `SELECT CONNECTION_NAME FROM SYS.EXA_DBA_CONNECTIONS` and surface choices; if user names one that doesn't exist, offer to create |
| `source_schema` | "Which schema in {source_kind}?" | Non-empty string. If known, offer to list: e.g. `IMPORT FROM JDBC AT <conn> STATEMENT 'SHOW SCHEMAS'` |
| `source_table_filter` | "Any specific tables, or all of them?" | Defaults to `%`. Accept wildcards |
| `target_stage_schema` | "Where should the landed tables go? Default: `STAGE_<source_schema_upper>`" | Valid Exasol identifier; offer to create if absent |
| `target_cube_name` | "What should I name the cube?" | Valid Exasol identifier; reject if a virtual schema by that name already exists |
| `llm_enrichment` | "Should I let LLM polish the names + groupings during optimize and cube create? (default: yes)" | Boolean |
| `approval_mode` | "Run all six steps automatically, or pause for approval at each? (auto / step)" | Enum |

## Q&A patterns

### Pattern A — fast path (user knows what they want)

```
agent: What do you want to do?
user:  Load ACME_DEMO.adventureworks from snowflake and query it via the cube.
agent: Got it. Confirmed:
       - source_kind:                snowflake
       - source_schema:              ACME_DEMO
       - source_table_filter:        adventureworks (treating as schema-name suffix)

       Which Snowflake connection?

user:  snowflake_demo_conn

agent: ✓ Connection `snowflake_demo_conn` exists.
       Defaults for target schemas:
       - STAGE_ACME_ADVENTUREWORKS
       - CUBE_ACME_ADVENTUREWORKS
       OK? (press enter to accept)

user:  (enter)
agent: Default target schemas accepted.
       Auto-accept all six steps, or step-by-step? (auto / step)
user:  auto
agent: Auto-accept on. Moving to plan presentation.
```

### Pattern B — explorer (user not sure of inputs)

```
agent: What do you want to do?
user:  Build a cube from some Snowflake data.
agent: I can help. A few decisions first:
       - Which Snowflake connection? (I see these configured: snowflake_demo_conn, snowflake_prod_conn)
user:  snowflake_demo_conn
agent: ✓. And which schema there?
       (running: IMPORT FROM JDBC AT snowflake_demo_conn STATEMENT 'SHOW SCHEMAS')
       Options: ACME_DEMO, PUBLIC, INFORMATION_SCHEMA.
user:  ACME_DEMO
agent: ✓. ACME_DEMO has the following sub-schemas / table prefixes:
       - adventureworks (dim + fact tables for AdventureWorksDW)
       - hospitality (Hilton-style sample)
       - inventory (suffix-named dim/fact)
       Pick one or specify a table filter:
user:  adventureworks
agent: ✓.
       Defaults: STAGE_ACME_ADVENTUREWORKS, CUBE_ACME_ADVENTUREWORKS — OK?
user:  yes
agent: LLM enrichment on naming and groupings — on (default) or off?
user:  on
agent: Approval mode — auto or step-by-step?
user:  step
agent: Step-by-step on.
```

### Pattern C — fallback (user gives no info)

```
agent: What do you want to do?
user:  go
agent: I need a few details:
       1. Which source? (snowflake / s3 / acme_demo_fixture / other JDBC)
       2. Which connection or fixture name?
       3. Which schema there?
       4. Anything you'd like me to name the result?

       If you'd like the bundled walkthrough against the local
       `acme_demo_fixture`, say "fixture" and I'll fill in sensible defaults.
user:  fixture
agent: Using the acme_demo_fixture path:
       - source_kind:           acme_demo_fixture
       - source_schema:         ACME_DEMO
       - target_stage_schema:   STAGE_ACME_DEMO
       - target_cube_name:      CUBE_ACME_DEMO
       - llm_enrichment:        on
       Approval mode? (auto / step)
```

## Validation rules

- Connection-name supplied by user: verify exists via `SYS.EXA_DBA_CONNECTIONS`. If missing, offer to create with credentials prompt.
- Schema name supplied: confirm it exists on the SOURCE side via the connection (run a remote `SHOW SCHEMAS`).
- Target identifier: must conform to Exasol identifier rules. Reject reserved keywords without explicit quoting.
- Virtual-schema name: must NOT already exist. If it does, ask whether to drop + recreate or pick a new name.

## When to use defaults vs ask

Default fields silently:

- `source_table_filter` → `%`
- `target_stage_schema` → `STAGE_<UPPER(source_schema)>` (replace dots with underscores)
- `target_cube_name` → `CUBE_<UPPER(source_schema)>`
- `llm_enrichment` → `true`

Always ask:

- `source_kind`
- `source_connection_name`
- `source_schema`
- `approval_mode`

## Confirmation block

Before leaving Phase 1, print the full parameter set back to the user and let them override anything:

```
agent: Confirmed parameters:
       - source_kind:               snowflake
       - source_connection_name:    snowflake_demo_conn
       - source_schema:             ACME_DEMO
       - source_table_filter:       adventureworks
       - target_stage_schema:       STAGE_ACME_ADVENTUREWORKS
       - target_cube_name:          CUBE_ACME_ADVENTUREWORKS
       - llm_enrichment:            true
       - approval_mode:             auto_accept

       Any changes? (no / specify a field to change)
user:  no
agent: Moving to plan presentation.
```

## Edge cases

| Situation | Action |
|---|---|
| User aborts mid-interrogation | Stop. No work attempted. Acknowledge and exit. |
| User picks `acme_demo_fixture` | Skip `source_connection_name` — it's a local fixture. Other parameters still required. |
| User picks `s3` | Switch to `exasol-import-s3` skill for steps 2-3 (not Snowflake) |
| User specifies a virtual schema name that already exists | Offer to: (a) drop + recreate, (b) pick new name. Default suggestion: (b) |
| LLM enrichment is `false` | Skill chain still runs, just skips the LLM-enrichment hooks in optimize + cube create |
| User refuses to pick `approval_mode` | Default to `step_by_step` for safety |
