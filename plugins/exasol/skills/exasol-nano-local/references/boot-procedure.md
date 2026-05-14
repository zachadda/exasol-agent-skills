# Boot procedure

Brings the local Docker Nano up from cold-stop to SQL-reachable. Assumes Docker is running (see SKILL.md Step 0) and the `exanano-sqlcube` container already exists with the Java workaround applied.

## Happy path

```bash
docker start exanano-sqlcube
```

That's the entire boot in normal conditions. Container starts in ~5 seconds. The Exasol process inside takes another ~10–15 seconds to accept SQL connections.

## Verify

After `docker start`, poll the SQL port until it responds:

```bash
for i in {1..30}; do
  exapump sql -p sqlcube-nano "SELECT CURRENT_USER" 2>/dev/null \
    && echo "ok at attempt $i" && break
  sleep 1
done
```

Success looks like:

```
[1/1] SELECT CURRENT_USER 1 rows
CURRENT_USER
SYS
1 statement executed, 0 failed
ok at attempt 6
```

If 30 retries pass without success, see `troubleshooting.md`.

## Status check

These commands answer "is the Nano up?" with decreasing reliability:

| Command | Reliable? | Why |
|---|---|---|
| `exapump sql -p sqlcube-nano "SELECT 1"` | **Always trust this** | Round-trip through the SQL listener is the only true health signal |
| `docker ps --filter name=exanano-sqlcube` | Yes for container alive, not for SQL listening | Container can be `Up` while Exasol process is still initializing |
| `exanano status` | Unreliable in debug-mode setups | Doesn't track debug-mode VMs; documented quirk |

Always check via `exapump sql` for truth.

## Cold stop and restart

Stop the Nano cleanly:

```bash
docker stop exanano-sqlcube
```

Wait for graceful shutdown (~10 sec). Then `docker start` to bring it back. State persists in the `exanano_sqlcube` volume.

If you need to force-kill (rare):

```bash
docker kill exanano-sqlcube
```

Then `docker start`. The Exasol process recovers from its own consistency log on next start. Acceptable for dev, never appropriate for production data.

## What boot does not do

- Does NOT install JDBC drivers — see `jdbc-drivers.md`
- Does NOT create / drop database schemas — that's downstream skill work
- Does NOT touch the `exapump` profile config — see `connect.md` if the profile doesn't exist

## When boot fails

Common patterns and the right next move:

| Symptom | Likely cause | Where to look |
|---|---|---|
| `docker start` exits 0 but SQL port never responds | Java stack not provisioned in volume | `troubleshooting.md` — Java workaround section |
| `docker start` returns "No such container: exanano-sqlcube" | First-time setup never ran | `initial-provision.md` |
| `docker start` returns "port already allocated" | Another process owns 8564 (often a stale container) | `troubleshooting.md` — port conflict |
| SQL responds with auth failure | `EXA_PASSWORD` mismatch | `connect.md` — credential defaults |
| `exapump` reports profile not found | `~/.exapump/config.toml` missing the `sqlcube-nano` profile | `connect.md` — profile setup |

## Idempotence

`docker start` on an already-running container is a no-op (exit 0, no output). Safe to call at the top of every skill that needs the Nano. The verify SELECT is what proves the state, not the start command.
