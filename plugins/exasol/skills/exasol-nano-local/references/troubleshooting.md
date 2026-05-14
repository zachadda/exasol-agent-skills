# Troubleshooting the local Nano

Failure modes seen on the `exanano-sqlcube` dev container, in rough order of frequency.

## Decision tree

Start here. Pick the symptom matching what you observed:

```
docker start succeeds, SQL port never opens   → §1 Java workaround
docker start: "port already allocated"        → §2 Port conflict
docker start: "No such container"             → see initial-provision.md
SQL connects but auth fails                   → §3 Credentials
exanano status says "stopped" but SQL works   → §4 Debug-mode quirk
Container OOMs / killed by Docker             → §5 Resource limits
Volume corrupted / refuses to mount           → §6 Volume reset (destructive)
IMPORT FROM JDBC fails after driver placed    → §7 JDBC runtime issues
```

## §1 Java workaround (most common cold-start failure)

**Symptom:** `docker start exanano-sqlcube` exits 0, container shows `Up` in `docker ps`, but `exapump sql` keeps returning connection-refused past the 30-second verify loop.

**Cause:** Exasol's CCE (Cluster Control Environment) self-extracts its Java stack on first container start. The pre-provisioned `exanano-sqlcube` volume has this stack baked in, but if the volume was created from a clean Exasol image without applying the workaround, the JVM fails silently and the SQL listener never opens.

**Diagnose:**

```bash
docker logs exanano-sqlcube 2>&1 | tail -50 | grep -iE 'jvm|java|cce|self-extract'
```

Look for `JVM not found` or `cce: self-extract failed`. If present → workaround needed.

**Fix:** The workaround is documented in `exasol-factory-foundation/docs/ops/exanano-local-runtime.md`. Summary:

```bash
# 1. Stop the container
docker stop exanano-sqlcube

# 2. Pull the Java stack from a working source and copy into the volume
#    (the factory-foundation doc has the exact tarball location)
docker run --rm -v exanano_sqlcube:/exa \
  exasol/docker-db:8.34.0 \
  bash -c 'cp -r /exa/EXAClusterOS/java /exa/java && chmod -R 755 /exa/java'

# 3. Re-start
docker start exanano-sqlcube
```

Re-run the boot verify loop. Should connect within 15 seconds now.

If the user's container was created some other way and you can't get this workaround to land cleanly, route to `initial-provision.md` and re-create the container from the canonical recipe.

## §2 Port conflict

**Symptom:** `docker start exanano-sqlcube` returns:

```
Error response from daemon: driver failed programming external connectivity on endpoint exanano-sqlcube: Bind for 0.0.0.0:8564 failed: port is already allocated
```

**Diagnose what owns 8564:**

```bash
lsof -nP -iTCP:8564 -sTCP:LISTEN
```

Common owners:

- **Another stopped-but-not-removed Exasol container:** `docker ps -a | grep 8564`. Solution: `docker rm <stale-container>`.
- **Native `exanano` binary (legacy):** `pgrep -fl exanano`. Solution: `exanano stop`, then retry docker start.
- **Random dev process bound to 8564 (rare):** rebind the Nano to a different host port. Requires recreating the container with `-p 8565:8563` mapping — see `initial-provision.md`.

If you change the host port, remember to update `~/.exapump/config.toml` profile `port = ...` and any upstream skill parameters.

## §3 Credentials

**Symptom:**

```
Authentication failed for user 'SYS' (R3)
```

**Diagnose:**

```bash
docker exec exanano-sqlcube env | grep -E 'EXA_PASSWORD|EXA_USER'
```

Compare to what's in `~/.exapump/config.toml` under `[profiles.sqlcube-nano]`.

**Fix:**

- If the container env says `EXA_PASSWORD=exasol` and your profile has something else → fix the profile.
- If the container env shows a custom password (someone hardened it) → update the profile to match.
- Never try to ALTER USER SYS from a broken auth session. If credentials are truly unknown, re-create the container (see `initial-provision.md`) — the volume data layer survives and re-attaches.

## §4 Debug-mode quirk

**Symptom:** Container is genuinely running and SQL works, but `exanano status` reports `stopped`.

**Cause:** The `exanano` CLI tracks state via a pidfile/socket that the debug-mode boot path doesn't touch. The CLI is unreliable in this configuration; the SQL port is the only source of truth.

**Fix:** Don't use `exanano status` for liveness. Use:

```bash
exapump sql -p sqlcube-nano "SELECT 1"
```

A successful SELECT means the Nano is live regardless of what `exanano status` says.

## §5 Resource limits

**Symptom:** Container started, then died after a minute or two of inactivity (or under load). `docker ps` shows it as `Exited (137)` or `Exited (139)`.

**Cause:** Docker Desktop default RAM (4 GB on macOS) is below Exasol's minimum. Exasol allocates a hugepages-style heap on start; if the kernel can't satisfy it, the OOM killer fires.

**Fix:** Raise Docker Desktop RAM to 8 GB minimum, 12 GB recommended. Settings → Resources → Memory. Restart Docker Desktop, then `docker start exanano-sqlcube`.

If you can't raise RAM (corporate-managed laptop), see `initial-provision.md` for the lean-mode container variant that uses smaller buffer pools — slower but fits in 4 GB.

## §6 Volume reset (destructive — confirm with user first)

**Symptom:** Container starts, Exasol process boots, but every query returns:

```
ERROR: Internal: database files inconsistent (R7)
```

or:

```
Could not open data file '/exa/data/storage/...': checksum mismatch
```

**Cause:** The volume got into a state Exasol can't recover from. Most commonly: host force-shutdown mid-write, Docker Desktop disk image corruption, or filesystem-level damage in `~/.docker/Desktop/vms/...`.

**Fix (DESTRUCTIVE — all data in the volume is lost):**

Before doing this, confirm with the user that the volume contents are throwaway (fresh dev environment). If the user has loaded fixtures, schemas, or work-in-progress data into the Nano, do not nuke without explicit go-ahead.

```bash
docker stop exanano-sqlcube
docker rm exanano-sqlcube
docker volume rm exanano_sqlcube
```

Then re-create via `initial-provision.md`.

## §7 JDBC runtime issues

**Symptom:** Driver jar is in `/exa/jdbc/<VENDOR>/`, container was restarted, but `IMPORT FROM JDBC` still fails.

Sub-cases:

| Error | Likely cause | Fix |
|---|---|---|
| `No suitable driver found for jdbc:...` | Container wasn't restarted after dropping the jar | `docker restart exanano-sqlcube` + re-verify |
| `Connection timed out` | Container can't reach the external host | `docker exec exanano-sqlcube curl -fI https://<host>` to test |
| `PKIX path building failed` | Corporate MITM TLS cert not in JVM truststore | Add cert to `/exa/etc/jdbc-truststore.jks` and restart |
| `SnowflakeSQLException: Private key file is missing` | Snowflake key-pair auth: PEM path is host-side, not in container | `docker cp` the PEM to `/exa/jdbc-keys/` first |
| `ClassNotFoundException: com.snowflake.client.jdbc.SnowflakeDriver` | Jar is wrong vendor or corrupt | Re-download from Maven Central, check `unzip -l <jar> | grep Driver` |

## Colima fallback (macOS Docker Desktop alternative)

If Docker Desktop is unusable on the user's machine (corporate license, performance, license-cost-induced ban), Colima is a drop-in replacement:

```bash
brew install colima docker
colima start --cpu 4 --memory 8 --disk 50
# now `docker` CLI talks to Colima's daemon
docker start exanano-sqlcube
```

Everything else in this skill works identically — Colima exposes a Docker-compatible socket.

## Total bricking — last resort

If nothing above recovers the Nano and the user is willing to accept data loss:

```bash
docker stop exanano-sqlcube
docker rm exanano-sqlcube
docker volume rm exanano_sqlcube
```

Then re-run the full initial provision (see `initial-provision.md`). Takes 5–10 minutes. Always offer this only as a last resort after telling the user it's destructive.
