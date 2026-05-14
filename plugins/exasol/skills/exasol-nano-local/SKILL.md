---
name: exasol-nano-local
description: Local Docker-based Exasol Nano operation — boot, status, JDBC driver placement, troubleshooting, and connection setup for `exanano-sqlcube` and similar local single-node containers. For zero-setup local dev, demos, and CI smoke tests. Distinct from `exasol-setup-personal` (which deploys to AWS).

preconditions:
  - docker_running:
      doc: "Docker daemon is reachable"
      check: |
        # Bash check, not SQL — orchestrator must run:
        # docker info > /dev/null 2>&1 && echo ok
      satisfied_by: null   # User must install / start Docker manually. Skill surfaces the install URL but does not automate it.

provides:
  - nano_running:
      doc: "Local Nano container is up, SQL port accepts connections, the user's exapump profile reaches it"
      verify: |
        # Bash, not SQL — orchestrator runs:
        # EXA_HOST=localhost:<port> EXA_USER=sys EXA_PASSWORD=exasol \
        # exapump sql -p <profile> "SELECT CURRENT_USER"
        # Returns SYS when satisfied.
  - nano_jdbc_dir:
      doc: "JDBC driver directory inside the container is writable and the container's CCE Java stack is provisioned. Used by IMPORT FROM JDBC and BY upstream skills like exasol-migrate-snowflake."
      verify: |
        # Bash:
        # docker exec <container> ls /exa/jdbc 2>/dev/null && echo ok

parameters:
  required: []
  optional:
    - container_name:
        default: exanano-sqlcube
        doc: "Docker container name. The pre-provisioned dev container is `exanano-sqlcube` (created with the Java workaround applied to the volume)."
    - sql_port:
        default: 8564
        doc: "Host port mapped to the container's SQL listener. The pre-provisioned dev container uses 8564; the legacy native `exanano` listened on 8563."
    - web_ui_port:
        default: 8444
        doc: "Host port mapped to the container's Exasol Web UI (https)."
    - data_volume:
        default: exanano_sqlcube
        doc: "Docker volume holding `/exa` (database files + JDBC drivers)."
    - exapump_profile:
        default: sqlcube-nano
        doc: "Profile name in `~/.exapump/config.toml` pointing at this Nano."

estimated_impact:
  start: "5–15 sec, no DB changes"
  cold_install: "5–10 min (one-time per host)"
---

# Exasol Nano Local Skill

Trigger when the user mentions **local Exasol**, **Docker Nano**, **exanano**, **exanano-sqlcube**, **boot Nano**, **start Nano**, **local single-node Exasol**, **Exasol on Docker**, **Exasol on my laptop**, **JDBC driver in Nano**, **Nano won't start**, or asks to **spin up Exasol locally** without using AWS.

Do not trigger for **Exasol Personal on AWS** — that's `exasol-setup-personal`. The two are siblings, not overlaps.

## What this skill is

`exanano-sqlcube` is a local single-node Exasol instance running in a Docker container. It's the zero-setup runtime for development, smoke harnesses, factory tours, and demos. The skill teaches an agent how to:

1. Boot the container (`docker start ...`) and confirm SQL connectivity.
2. Place JDBC drivers in the right path inside the volume so `IMPORT FROM JDBC` and federated queries work.
3. Recover from the most common failure modes (Java workaround, port conflicts, debug-mode quirks).
4. Connect via `exapump`, `pyexasol`, JDBC, or the Web UI.

The skill does NOT cover container provisioning from scratch. The pre-built `exanano-sqlcube` container with the Java workaround applied is assumed. If the container doesn't exist, route to `references/initial-provision.md` (which documents the one-time setup) before retrying boot.

## Step 0: Verify Docker is running

Before any other action:

```bash
docker info > /dev/null 2>&1 && echo "ok" || echo "docker not running"
```

If `docker not running` → halt and tell the user:
- "Start Docker Desktop and re-trigger this skill."
- Optional: explain the Colima fallback (see `references/troubleshooting.md`).

If `ok` → proceed.

## Routing algorithm

| Phase | Reference | When to load |
|---|---|---|
| Boot | `references/boot-procedure.md` | Always — agent's first action |
| Status / verify | `references/connect.md` | After boot, to confirm SQL port is live |
| Driver install | `references/jdbc-drivers.md` | When upstream skill needs JDBC (e.g., `exasol-migrate-snowflake` declares `nano_jdbc_dir` precondition with a specific vendor) |
| Failure recovery | `references/troubleshooting.md` | When boot or verify fails |
| First-time setup | `references/initial-provision.md` | When container does not exist — points to factory-foundation's `docs/ops/exanano-local-runtime.md` and the Java workaround |

## When to invoke

- An orchestrator skill (e.g., `/oneshot-tour`) needs a local Nano to run against.
- A developer asks the agent to "spin up Exasol locally to test something."
- A CI job in `exasol-factory-foundation` needs the Nano up for smoke harness runs.
- A migration skill needs a JDBC driver placed before IMPORT FROM JDBC will work.
- Recovery after a Docker Desktop restart wipes the running state.

## Conventions

- **Container name + port are not magic.** Defaults match the pre-provisioned dev container on Zachary's laptop. Override via parameters when the user's setup differs.
- **`exapump` is the canonical SQL client** for headless invocation. Don't shell into the container for SQL.
- **JDBC drivers go in `/exa/jdbc/<VENDOR_UPPER>/`** inside the container. Restart container after dropping new jars.
- **Debug mode is the recovery path.** When `exanano start` fails (macOS/build-specific self-extract issue), drop into debug mode via screen — see troubleshooting.

## Related skills

- **exasol-setup-personal** — sibling skill for AWS-deployed Exasol Personal. Not interchangeable.
- **exasol-database** — once Nano is up, `exasol-database` covers SQL execution, connections, imports.
- **exasol-bucketfs** — for managing files inside `/exa` volume (UDF jars, fixture data).
- **oneshot-tour** — orchestrator that lists `nano_running` as Step 1 precondition.

## Limitations

- Local Nano is single-node by definition. No HA, no clustering, no large-scale benchmarks. Use AWS Exasol cluster (via `exasol-setup-personal` or factory-foundation infrastructure docs) for those.
- DDL is auto-commit (same as production Exasol). Dropped tables don't come back without backup.
- The `exanano` CLI's status reporting is unreliable in debug mode. Agent should verify via SQL port reachability, not `exanano status`.
- Web UI on `https://localhost:8444` uses a self-signed cert. Browser warnings expected.
