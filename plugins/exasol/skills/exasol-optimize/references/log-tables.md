# EXA_OPTIMIZE_LOG companion tables

Optional persistence layer for `llm_enrichment` outputs and downstream skill hand-offs. Lives in the `EXA_OPTIMIZE_LOG` schema (which exists on `exanano-sqlcube` but ships empty — these tables are not auto-created by the optimize UDFs).

When the skill writes LLM-enrichment results, it can either:

- **Persist** to these tables (recommended for re-use across sessions and by downstream skills like `exasol-semantic-layer` and `exasol-dashboard`)
- **Surface inline** in plan-presentation output (fallback when the schema doesn't exist or the user lacks DDL privileges)

If `SELECT 1 FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA='EXA_OPTIMIZE_LOG' AND TABLE_NAME='ENRICHMENT_NOTES'` returns 0, the skill falls back to inline-only.

## DDL

Idempotent — agent runs once per cluster if missing:

```sql
CREATE SCHEMA IF NOT EXISTS EXA_OPTIMIZE_LOG;

-- Column-naming hints from prompts/column-naming-hints.md
CREATE TABLE IF NOT EXISTS EXA_OPTIMIZE_LOG.ENRICHMENT_NOTES (
    SOURCE_SCHEMA   VARCHAR(128) NOT NULL,
    TABLE_NAME      VARCHAR(128) NOT NULL,
    COLUMN_NAME     VARCHAR(128) NOT NULL,
    NOTE_TYPE       VARCHAR(50)  NOT NULL,   -- e.g. column_name_hint, table_purpose
    SUGGESTED_NAME  VARCHAR(200),
    DESCRIPTION     VARCHAR(2000),
    CONFIDENCE      VARCHAR(10),             -- high / medium / low
    RECORDED_AT     TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (SOURCE_SCHEMA, TABLE_NAME, COLUMN_NAME, NOTE_TYPE)
);

-- Self-FK classifications from prompts/hierarchy-detection.md
CREATE TABLE IF NOT EXISTS EXA_OPTIMIZE_LOG.HIERARCHY_FLAGS (
    SOURCE_SCHEMA   VARCHAR(128) NOT NULL,
    TABLE_NAME      VARCHAR(128) NOT NULL,
    FK_COLUMN       VARCHAR(128) NOT NULL,
    PK_COLUMN       VARCHAR(128) NOT NULL,
    HIERARCHY_KIND  VARCHAR(50)  NOT NULL,   -- org_chart | bom | category_tree | account_parent | message_thread | geographic | other_hierarchy | false_positive
    CONFIDENCE      VARCHAR(10)  NOT NULL,
    REASONING       VARCHAR(2000),
    AUTO_PROMOTED   BOOLEAN      DEFAULT FALSE,  -- true when REJECTED → ADD_FK based on this classification
    RECORDED_AT     TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (SOURCE_SCHEMA, TABLE_NAME, FK_COLUMN, PK_COLUMN)
);

-- Domain inference from semantic-layer's prompts/domain-detection.md
CREATE TABLE IF NOT EXISTS EXA_OPTIMIZE_LOG.DOMAIN_HINTS (
    SOURCE_SCHEMA   VARCHAR(128) NOT NULL,
    DOMAIN          VARCHAR(50)  NOT NULL,
    SUB_DOMAIN      VARCHAR(100),
    CONFIDENCE      VARCHAR(10)  NOT NULL,
    REASONING       VARCHAR(1000),
    DETECTED_AT     TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (SOURCE_SCHEMA, DETECTED_AT)
);

-- Measure priority hints from semantic-layer's prompts/measure-business-context.md
CREATE TABLE IF NOT EXISTS EXA_OPTIMIZE_LOG.MEASURE_PRIORITY (
    MODEL_ID            VARCHAR(100) NOT NULL,
    VIRTUAL_COL         VARCHAR(100) NOT NULL,
    BUSINESS_PRIORITY   VARCHAR(10)  NOT NULL,   -- high | medium | low
    RECORDED_AT         TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (MODEL_ID, VIRTUAL_COL)
);
```

These are not part of the canonical `factory-foundation/sqlcube/meta/` install. They're skill-specific scratch space.

## Who reads what

| Table | Written by | Read by |
|---|---|---|
| `ENRICHMENT_NOTES` | `exasol-optimize` LLM step | `exasol-optimize` plan-presentation, downstream review tools |
| `HIERARCHY_FLAGS` | `exasol-optimize` LLM step | `exasol-optimize` plan-presentation (auto-promote logic) |
| `DOMAIN_HINTS` | `exasol-semantic-layer` LLM step (domain detection) | `exasol-semantic-layer` cube-enrichment prompt (injects `<DOMAIN_HINT>`) |
| `MEASURE_PRIORITY` | `exasol-semantic-layer` LLM step (measure proposal) | `exasol-dashboard` panel-planning (KPI tile selection) |

## Upsert semantics

All four tables: DELETE-then-INSERT keyed on the PK. Re-running enrichment for the same `(source_schema, table_name, column_name)` replaces the prior note.

```sql
DELETE FROM EXA_OPTIMIZE_LOG.ENRICHMENT_NOTES
WHERE SOURCE_SCHEMA = ? AND TABLE_NAME = ?;

-- then INSERT the new batch
```

## When to skip persistence

- **No DDL privilege.** Skill detects via `CREATE SCHEMA IF NOT EXISTS` error, falls back to inline.
- **Read-only dev environment.** Same — inline-only.
- **One-off invocations.** Skill can be invoked with `persist_enrichment=false` (parameter not yet exposed in v0.1 SKILL.md; add if needed).

In all skip cases, the LLM output is still surfaced to the user in plan-presentation. Persistence is just for cross-session and cross-skill carry-forward.

## Naming convention

Schema name `EXA_OPTIMIZE_LOG` mirrors Exasol's own audit schemas (`EXA_DBA_*`, `EXA_USER_*`, `EXA_OPTIMIZE`). Stays out of business-data schemas. Don't rename without coordinating with downstream skills.

If the eventual SQLCUBE_REGISTRY rename includes these too, update in lockstep.
