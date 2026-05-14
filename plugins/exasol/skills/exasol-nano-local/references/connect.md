# Connecting to the local Nano

Covers `exapump` profile setup, credentials, port map, and client connection strings (pyexasol, JDBC, ODBC, Web UI).

## Defaults for `exanano-sqlcube`

| Field | Value |
|---|---|
| Host | `localhost` |
| SQL port | `8564` |
| Web UI port | `8444` (https, self-signed) |
| User | `sys` |
| Password | `exasol` |
| Default schema | (none — set per-session) |
| Fingerprint check | Off for localhost dev |

Production Exasol clusters use port `8563`. The local Nano deliberately uses `8564` to avoid colliding with the legacy native `exanano` binary, which still binds 8563 on some dev laptops.

## `exapump` profile

The canonical headless client. Profile lives in `~/.exapump/config.toml`:

```toml
[profiles.sqlcube-nano]
host = "localhost"
port = 8564
user = "sys"
password = "exasol"
# optional:
# schema = "SQLCUBE"
# encryption = false
```

Test the profile:

```bash
exapump sql -p sqlcube-nano "SELECT CURRENT_USER"
```

Expected output:

```
[1/1] SELECT CURRENT_USER 1 rows
CURRENT_USER
SYS
1 statement executed, 0 failed
```

If profile missing → `exapump: profile 'sqlcube-nano' not found`. Append the block above to `~/.exapump/config.toml`. If file doesn't exist, create it with that block as the only content.

### Alternative: env-var / DSN-flag

`exapump` also accepts environment variables OR an explicit DSN URI flag. Useful for ephemeral sessions, CI jobs, or when a user doesn't have a `~/.exapump/config.toml`:

```bash
# Env-var form (matches what factory-foundation scripts/benchmarks use):
EXA_HOST=localhost:8564 \
EXA_USER=sys \
EXA_PASSWORD=exasol \
exapump sql "SELECT CURRENT_USER"

# DSN-flag form:
exapump sql -d 'exasol://sys:exasol@localhost:8564?tls=true&validateservercertificate=false' \
  "SELECT CURRENT_USER"
```

The studio backend uses env-var-driven connections internally (see `studio/backend/config.py`); the TOML profile is a user-level convenience layer. Either path works against the same Nano.

## pyexasol

```python
import pyexasol

c = pyexasol.connect(
    dsn="localhost:8564",
    user="sys",
    password="exasol",
    compression=True,
)
print(c.execute("SELECT CURRENT_USER").fetchone())
```

No fingerprint validation needed for localhost. For remote Exasol you'd set `websocket_sslopt={"cert_reqs": ssl.CERT_NONE}` or pin the fingerprint — irrelevant here.

## JDBC

Connection string:

```
jdbc:exa:localhost:8564;schema=SQLCUBE;validateservercertificate=0
```

Driver jar location depends on caller. If you're running JDBC from inside the container (e.g., `IMPORT FROM JDBC`), see `jdbc-drivers.md` — the driver must live under `/exa/jdbc/EXASOL/`. If running JDBC from the host (a JVM client app), the driver lives in your local maven cache or classpath, not the container.

## ODBC

DSN block for `~/.odbc.ini`:

```
[ExasolNano]
Driver=/Library/exasol/EXASolution_ODBC/lib/libexaodbc-uo2214lv2.dylib
EXAHOST=localhost:8564
EXAUID=sys
EXAPWD=exasol
```

Driver path varies by OS. On macOS this is the default path after `brew install --cask exasol-odbc-driver`. On Linux: `/opt/exasol/EXASolution_ODBC/lib/libexaodbc-uo2214lv2.so`.

## Web UI

```
https://localhost:8444
```

Self-signed cert → browser warning. Accept-once is fine for dev. Login `sys` / `exasol`. Use this for visual cluster inspection, not as the agent's SQL channel.

## When connection fails

| Symptom | Cause | Fix |
|---|---|---|
| `Connection refused` on 8564 | Container not running OR Exasol process still booting | `docker ps` to confirm, then wait — see `boot-procedure.md` verify loop |
| `Authentication failed for user 'SYS'` | Wrong password in profile | Check `EXA_PASSWORD` in container env. Default is `exasol` |
| `No such profile 'sqlcube-nano'` | `~/.exapump/config.toml` missing the block | Add profile (see above) |
| Connects but every query returns `Schema X does not exist` | No default schema set | Add `schema = ...` to profile, or `USE SCHEMA ...` per session |
| `port already allocated` from Docker | Stale container OR host process owns 8564 | See `troubleshooting.md` — port conflict |

## Credential override

The defaults match the pre-provisioned dev container. If a user has changed the SYS password (rare for local dev, common if the volume was cloned from a hardened image):

```bash
docker exec exanano-sqlcube env | grep EXA_PASSWORD
```

Update the profile to match. Don't try to reset SYS password from outside — re-create the container instead (see `initial-provision.md`).
