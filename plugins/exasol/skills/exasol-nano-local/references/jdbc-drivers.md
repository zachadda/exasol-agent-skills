# JDBC drivers inside the Nano

How to place a JDBC driver so `IMPORT FROM JDBC` and federated queries reach external sources (Snowflake, Postgres, MySQL, Redshift, BigQuery via JDBC wrapper, etc.).

## Where drivers live

Inside the container:

```
/exa/jdbc/<VENDOR_UPPER>/
```

One subdirectory per vendor. Examples:

```
/exa/jdbc/SNOWFLAKE/snowflake-jdbc-3.14.5.jar
/exa/jdbc/POSTGRES/postgresql-42.7.3.jar
/exa/jdbc/MYSQL/mysql-connector-j-8.4.0.jar
```

`<VENDOR_UPPER>` is uppercase by convention and matches the value the agent will reference in `IMPORT FROM JDBC AT '...' SOURCE_TYPE = 'JDBC' ...` calls. The Exasol JVM scans `/exa/jdbc/*/*.jar` at process startup. New jars need a container restart.

## Why this path

`/exa` is the Docker volume mount (`exanano_sqlcube`). Anything dropped there persists across `docker stop` / `docker start`. The Exasol java classloader is configured (in the volume's CCE Java stack) to walk `/exa/jdbc/` at boot. Other paths inside the container are tmpfs or container-local and get wiped.

## Placing a driver

Two paths. Pick based on whether you have the jar locally already.

### From host filesystem

```bash
docker cp ./snowflake-jdbc-3.14.5.jar \
  exanano-sqlcube:/exa/jdbc/SNOWFLAKE/snowflake-jdbc-3.14.5.jar
```

If the vendor directory doesn't exist:

```bash
docker exec exanano-sqlcube mkdir -p /exa/jdbc/SNOWFLAKE
docker cp ./snowflake-jdbc-3.14.5.jar \
  exanano-sqlcube:/exa/jdbc/SNOWFLAKE/snowflake-jdbc-3.14.5.jar
```

### Download inside the container

```bash
docker exec exanano-sqlcube bash -c '
  mkdir -p /exa/jdbc/SNOWFLAKE
  cd /exa/jdbc/SNOWFLAKE
  curl -fSL -o snowflake-jdbc-3.14.5.jar \
    https://repo1.maven.org/maven2/net/snowflake/snowflake-jdbc/3.14.5/snowflake-jdbc-3.14.5.jar
'
```

Maven Central mirrors are stable. Vendor download pages occasionally rotate URLs — prefer Maven Central when possible.

## Restart to pick up new jars

```bash
docker restart exanano-sqlcube
```

Then re-verify SQL (see `boot-procedure.md` verify loop). The JVM only scans `/exa/jdbc/` at process startup.

## Verify a driver is loaded

After restart, test from `exapump`:

```sql
-- Snowflake example. Replace creds + account.
IMPORT INTO (X DECIMAL(18))
FROM JDBC
  DRIVER  = 'SNOWFLAKE'
  AT      'jdbc:snowflake://YOUR_ACCT.snowflakecomputing.com/?warehouse=COMPUTE_WH&db=SNOWFLAKE_SAMPLE_DATA'
  USER    'USER'
  IDENTIFIED BY 'PASS'
  STATEMENT 'SELECT 1';
```

Success → 1 row. Driver-class errors look like:

```
ERROR: [42000] No suitable driver found for jdbc:snowflake://...
```

That means the jar isn't on the classpath. Re-check `/exa/jdbc/SNOWFLAKE/*.jar` exists and that you restarted the container.

## Vendor → driver coordinate cheat sheet

| Vendor | Maven coordinate | Notes |
|---|---|---|
| Snowflake | `net.snowflake:snowflake-jdbc` | Single fat jar, no separate deps |
| Postgres | `org.postgresql:postgresql` | Use latest matching your server major |
| MySQL | `com.mysql:mysql-connector-j` | Old `mysql:mysql-connector-java` artifact deprecated |
| Redshift | `com.amazon.redshift:redshift-jdbc42` | Hosted on Amazon's repo, not Maven Central; pull from S3 download page |
| BigQuery | `com.google.cloud:google-cloud-jdbc` | Pulls dozens of transitive deps — use Simba JDBC zip from Google docs instead |
| SQL Server | `com.microsoft.sqlserver:mssql-jdbc` | Match driver to JDK version (jdk8/jdk11/jdk17 classifier) |

When an upstream skill (e.g., `exasol-migrate-snowflake`) declares a precondition `nano_jdbc_dir` with a vendor parameter, the verify step is:

```bash
docker exec exanano-sqlcube ls /exa/jdbc/<VENDOR_UPPER>/ | grep -c '\.jar$'
```

Returns ≥ 1 → driver present.

## When the driver is present but IMPORT still fails

Most commonly:

- **Network** — the container can reach Maven Central (egress NAT) but cannot reach a corporate Snowflake host on a VPN-only network. Test with `docker exec exanano-sqlcube curl -fI https://YOUR_ACCT.snowflakecomputing.com`.
- **TLS** — corporate MITM cert not trusted by the JVM in the volume. Add the cert to `/exa/etc/jdbc-truststore.jks` and restart, or use a driver-level `insecureMode=true` for dev only.
- **Auth** — Snowflake key-pair auth needs `PRIVATE_KEY_FILE` parameter pointing at a path inside the container, not the host. `docker cp` the PEM first.

See `troubleshooting.md` for the full failure-mode table.
