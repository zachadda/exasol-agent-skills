# First-time provision of `exanano-sqlcube`

When `docker start exanano-sqlcube` returns `No such container`, the host has never been set up. This reference covers the one-time creation. Five to ten minutes end-to-end.

This skill **does not automate provision** — there are too many host-specific choices (Docker Desktop vs Colima, RAM allocation, port preference, factory-foundation checkout location). The agent's job here is to surface the steps to the user and answer questions, not run them headlessly.

## Canonical source of truth

The full provisioning recipe lives in:

```
exasol-factory-foundation/docs/ops/exanano-local-runtime.md
```

That doc is the authoritative version, maintained alongside the factory-foundation studio backend it supports. The summary below is for quick orientation; defer to that doc for exact tarball URLs, version pins, and the Java workaround steps.

## Prerequisites

| Prereq | Check | Action if missing |
|---|---|---|
| Docker daemon | `docker info` | Install Docker Desktop or Colima |
| 8 GB+ free RAM | `docker info | grep Memory` | Raise Docker Desktop allocation |
| 10 GB+ free disk | `df -h ~/Library/Containers/com.docker.docker` (macOS) | Free disk or move Docker storage |
| Repo checkout | `ls ~/Documents/GitHub/exasol-factory-foundation` | `git clone …/exasol-factory-foundation` |

## High-level steps

The recipe in `exanano-local-runtime.md` walks through:

1. **Pull the Exasol Docker image** (`exasol/docker-db:<version>`). Version pin is in the doc — currently 8.34.0.
2. **Create the named volume** (`docker volume create exanano_sqlcube`).
3. **Apply the Java workaround** — extract the CCE Java stack into the volume before first container start. This is the part that diverges from upstream Exasol docs and is the reason the container is "pre-provisioned."
4. **Run the container** with the right port + volume bindings:
   ```bash
   docker run -d \
     --name exanano-sqlcube \
     --shm-size=4g \
     -p 8564:8563 \
     -p 8444:8443 \
     -v exanano_sqlcube:/exa \
     exasol/nano:latest \
     --provision-stacks java
   ```
   (Matches the canonical command in `docs/ops/exanano-local-runtime.md`. Image is `exasol/nano:latest`, not the heavier `docker-db`. `--shm-size=4g` is required.)
5. **Wait for first-boot initialization** (~2 minutes the first time — provisions system catalog, license, default schemas).
6. **Verify** via `exapump sql -p sqlcube-nano "SELECT CURRENT_USER"`.
7. **Run the SQLCube schema bootstrap.** Not automatic on container start. Run:
   ```bash
   python3 scripts/deploy_adapter_exanano.py
   ```
   from the `exasol-factory-foundation` checkout. Creates `SQLCUBE`, `SQLCUBE_REGISTRY`, and the AdventureWorks fixture. `SQLCUBE_META` is a separate bootstrap, fired by the studio backend's lineage endpoint or directly via `studio/backend/fixtures/sqlcube-meta/bootstrap.sql`.
8. **(Optional) Drop JDBC drivers** if upstream skills need them (see `jdbc-drivers.md`).

## What the agent should do

If the user hits "No such container":

1. **Stop and surface the situation.** Do not attempt to `docker run` headlessly — the parameters are host-specific.
2. **Point at the canonical doc:**
   ```
   See exasol-factory-foundation/docs/ops/exanano-local-runtime.md.
   It's the one-time setup recipe.
   ```
3. **Offer to walk through it interactively** if the user wants step-by-step help.
4. **After they confirm provision is done,** loop back to `boot-procedure.md` — `docker start` should now succeed.

## Why not automate

Three reasons:

- **Privileged container flag.** `docker run --privileged` is required by Exasol's storage layer. Many orgs ban this. Agent shouldn't trip a policy violation silently.
- **Port choice.** 8564 is the local convention but conflicts with whatever the user already has. Agent can't know which port is free without interactive negotiation that's better in a doc.
- **Volume pre-seeding.** The Java workaround step needs the right CCE tarball, which is version-tied to the image. Mis-pairing silently breaks the JVM and the user gets symptom §1 in troubleshooting.

Automating this would gain us 30 seconds of headless time at the cost of significant tail-risk. Not worth it.

## Bare-metal `exanano` (legacy, no Docker)

The native `exanano` CLI exists on some older dev machines. It binds port 8563 (not 8564) and runs Exasol directly on the host. If you encounter it:

- Don't conflate with the Docker Nano. They are different runtimes.
- If both are present, port 8563 is bare-metal, port 8564 is Docker.
- Migration path: stop bare-metal `exanano stop`, provision Docker via this doc, point profile at 8564.

The skill defaults assume Docker. If a user explicitly wants bare-metal, route to factory-foundation's older docs — but recommend migrating to Docker.
