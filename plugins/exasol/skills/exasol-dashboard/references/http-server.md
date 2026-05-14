# HTTP server lifecycle

The dashboard ships as a single HTML file on local disk, served by `python3 -m http.server` so the browser can load Vega-Lite's CDN scripts and render the embedded JSON. This reference covers launch, port-collision recovery, and cleanup.

## Launch

After the HTML is written to `<host_dir>/<source_name>.html`:

```bash
cd <host_dir>
python3 -m http.server <port> --bind 127.0.0.1 > /tmp/exasol-dashboard.log 2>&1 &
echo $! > /tmp/exasol-dashboard.pid
```

Key flags:

- `--bind 127.0.0.1` — localhost only. Never bind 0.0.0.0; that exposes the dashboard to the LAN.
- `> /tmp/exasol-dashboard.log 2>&1` — capture stdout/stderr. Useful for debugging.
- `&` — background. Agent process doesn't block on the server.
- `echo $! > ...pid` — save the PID so cleanup is possible later.

Skill emits this exact pattern (or the equivalent via `subprocess.Popen` if the orchestrator is Python).

## Verify the server is up

Before printing the URL to the user, confirm the server actually responded:

```bash
for i in {1..10}; do
  if curl -fsI "http://localhost:<port>/<source_name>.html" >/dev/null 2>&1; then
    echo "ok"
    break
  fi
  sleep 0.2
done
```

`http.server` opens fast (<200ms typically), but on cold-start Python imports the loop covers it. If 10 attempts fail → the server didn't start; check `/tmp/exasol-dashboard.log` and report.

## Print the URL

Final user-facing output:

```
Dashboard ready: http://localhost:<port>/<source_name>.html
```

That's it. No flashy emoji, no surrounding boilerplate. Treat the URL line as the provides contract — downstream callers grep for it.

## Port collision recovery

`<port>` defaults to 8765. If something else has it:

```bash
lsof -nP -iTCP:<port> -sTCP:LISTEN
```

If owner is another `python3 -m http.server` from a previous dashboard run → kill it (with the user's permission) or just bump to the next port. v0.1 logic:

1. Try requested port.
2. If `lsof` shows it's a previous dashboard PID (from `/tmp/exasol-dashboard.pid`) → `kill <pid>`, retry.
3. Otherwise probe `<port> + 1`, `+2`, `+3`, `+4`, `+5`.
4. If 6 consecutive ports occupied → halt, tell user to free a port.

When the skill bumps the port, log it: `"port 8765 in use (by <process>), using 8766 instead"`.

## Cleanup — when and how

The server is meant to run for the duration of the agent session. Not a daemon, not persistent. Three cleanup triggers:

### Trigger 1: user closes the dashboard panel / says they're done

Run:

```bash
kill $(cat /tmp/exasol-dashboard.pid) 2>/dev/null
rm /tmp/exasol-dashboard.pid
```

Always confirm with the user before killing. Maybe they want it open.

### Trigger 2: re-running the dashboard skill

If `panels=auto` is re-invoked (e.g., to refresh data), the old server is stale. Kill the old one:

```bash
if [ -f /tmp/exasol-dashboard.pid ]; then
  OLD_PID=$(cat /tmp/exasol-dashboard.pid)
  if kill -0 "$OLD_PID" 2>/dev/null; then
    kill "$OLD_PID"
  fi
  rm /tmp/exasol-dashboard.pid
fi
```

Then launch fresh.

### Trigger 3: explicit teardown command

User can ask "kill the dashboard." Skill exposes this implicitly — agent reads `/tmp/exasol-dashboard.pid` and kills it. No formal command needed.

## Multiple dashboards in parallel

Edge case: user spins up two cubes and wants both visualized. The skill SHOULD support this by using a per-source PID file:

```
/tmp/exasol-dashboard-<source_name>.pid
```

And a per-source port from the +1 / +2 probe logic. Two dashboards on 8765 and 8766. Acceptable.

When the user has multiple running, list them on each subsequent invocation:

```
Note: 2 dashboards already running:
  - 8765 / ADVENTUREWORKS_CUBE
  - 8766 / TPCH_CUBE
```

So they can decide which to kill.

## Why python3 -m http.server and not Flask / FastAPI

- **Zero deps.** Python 3 stdlib only. No `pip install`.
- **No build step.** Static HTML, no template engine.
- **Predictable.** Doesn't auto-reload, doesn't watch files, doesn't surprise.
- **Universal.** Every macOS / Linux dev box has it.

The trade-off: no live updates, no WebSocket, no auth. All by design for v0.1.

## Why not just `open <html_file>://`

`file://` URLs don't load Vega-Lite from the CDN cleanly in all browsers (CORS / mixed-protocol issues). Some Chrome configurations refuse to fetch HTTPS resources from `file://` origins. The HTTP server avoids this entirely.

## Browser auto-open (optional)

Skill MAY auto-open the URL in the default browser:

```bash
# macOS:
open "http://localhost:<port>/<source_name>.html"

# Linux:
xdg-open "http://localhost:<port>/<source_name>.html"
```

But only when the agent has terminal access — not via SSH from a server with no display. v0.1 default: don't auto-open, just print the URL. Add `auto_open=true` parameter in v0.2 if requested.

## Security posture

Localhost-bind + 127.0.0.1 means the only attacker is someone already on the machine. That's the same trust boundary as the agent itself.

But: the HTML contains data from the user's database. If the user has `/tmp` mounted on a shared filesystem (rare but possible on some HPC / corporate setups), the JSON in the HTML is readable by other users on that filesystem.

Don't put PII-laden samples in dashboards on shared systems. The skill MAY warn:

```bash
if df -h /tmp | grep -qE 'nfs|cifs|gpfs'; then
  echo "WARNING: /tmp is on a network filesystem. Dashboard data may be readable by other users."
fi
```

Skill should set `host_dir` to a private directory in that case (default `/tmp/exasol-dashboards` becomes `~/.local/exasol-dashboards`).

## Logs

`/tmp/exasol-dashboard.log` captures stdout/stderr from `http.server`. Useful for:

- 404s when the user types a wrong path
- Random connection drops
- Confirming the server logged the user's browser hit

The log grows unbounded — if the user keeps the dashboard up for hours, rotate or truncate manually. v0.1 doesn't auto-rotate.
