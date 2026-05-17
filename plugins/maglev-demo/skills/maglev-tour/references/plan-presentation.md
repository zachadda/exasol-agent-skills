# Phase 2 — Plan presentation

Render the 8-step plan as a checklist before executing. Boss sees what's about to happen + signs off on approval mode.

## Render template

```
agent:
  ### Maglev Tour plan — 8 phases

  | # | Phase | Skill | What lands |
  |---|---|---|---|
  | 1 | Target           | exasol-nano-local      | Exasol session active (localhost:8568) |
  | 2 | Source           | exasol-migrate         | Snowflake JDBC reachable via SNOWFLAKE_CONNECTION |
  | 3 | Schema           | exasol-migrate         | Source = ACME_DEMO.ADVENTUREWORKS (9 tables) |
  | 4 | Migrate          | exasol-migrate         | ACME_ADVENTUREWORKS schema on Exasol, ~84k rows, ~15s |
  | 5 | Optimize         | exasol-optimize        | 2-pass: PKs + FKs declared, DIM_DATE upgraded |
  | 6 | Cube             | exasol-semantic-layer  | LATTICE_ACME_ADVENTUREWORKS virtual schema with 4 domains + ADW metric pack |
  | 7 | Security         | exasol-rcls            | RCLS_CANADA / RCLS_EUROPE / RCLS_EXEC users + 4 row policies |
  | 8 | Query (showcase) | exasol-query-personas  | Side-by-side persona query: same SQL, four row scopes |

  Estimated total: 3-5 minutes against demo-tour. Cube re-deploy is idempotent; RCLS upsert; nothing destructive.

  Approve `auto_accept` or `step_by_step`?
```

## Decision tree

- User picks `auto_accept` → Phase 3 runs continuously.
- User picks `step_by_step` → Phase 3 pauses after each step with `Continue to Step N+1? (yes/no/inspect)`.
- User says "skip to step N" → fast-forward by running verify-only on 1..N-1, halt if any precondition fails.
- User says "what's in state X already?" → run the warm-state probe from `interrogation.md` and report.

## When NOT to present the full plan

If interrogation detected `warm` state (everything already deployed), shorten to:

```
agent:
  Cube live + RCLS seeded already. Skipping Phases 1-7 (verify-only).
  Running Phase 8 (persona showcase) directly.
```

Boss sees the result in 5 seconds. Re-running the full pipeline against a warm fixture wastes ~3 minutes and risks a transient TLS handshake error spooking them.
