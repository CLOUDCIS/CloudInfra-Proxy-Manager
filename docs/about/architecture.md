# Architecture

How the pieces fit, and why they are arranged this way.

## The components

```text
                    ┌─────────────────────────────────────┐
   your clients ───►│  squid            (port 3128)       │───► internet
                    │  runs as: squid                     │
                    └───────┬──────────────────┬──────────┘
                            │ writes           │ mgr: interface
                            ▼                  │ (loopback)
              /var/log/squid/                  │
              cloudinfra-events.log            │
                            │ tails            │ polls
                    ┌───────▼──────────────────▼──────────┐
  administrators ──►│  cloudinfra-proxyd  (port 8443)     │
                    │  runs as: cloudinfra, unprivileged  │
                    └───────┬─────────────────────────────┘
                            │ Unix socket, closed verb set
                    ┌───────▼─────────────────────────────┐
                    │  cloudinfra-privhelper              │
                    │  runs as: root                      │
                    └─────────────────────────────────────┘
```

| Component | User | Job |
|---|---|---|
| `squid` | `squid` | Proxying. Untouched upstream Squid 7 |
| `cloudinfra-proxyd` | `cloudinfra` | Console, API, log ingestion, analytics |
| `cloudinfra-privhelper` | `root` | The ten privileged operations |
| `cloudinfra-firstboot` | `root` | One-shot per-instance setup |

## The independence rule

The proxy and the management layer are deliberately **not** coupled in either
direction.

> If the management layer stops, Squid must keep proxying. If Squid stops, the
> console must still come up to explain why.

So the console's unit does not depend on `squid`, and `squid` does not depend on
the console. A management failure must never become a proxy outage, and a proxy
outage must never take away the tool you would use to diagnose it.

This shows up throughout. If the analytics database cannot be opened, the console
still serves the appliance header, health page and policy editor. If the
privileged helper is not answering, you can still edit policy — it stays staged
until deployment is possible again.

## Policy is the source of truth

Your rules and lists live in SQLite. `squid.conf` is generated from them and
**never parsed back**.

That abstraction is the point. An administrator edits readable rules; the
generator turns them into valid ACL directives. Coupling the UI to lines in a
configuration file would make every later feature — diffing, history, shadow
analysis, rule attribution — either impossible or a text-processing exercise.

It also means the [configuration history](../guide/applying-changes.md#configuration-history)
diff is in terms of *rules and lists*, not lines of a file nobody wrote.

## How traffic becomes a dashboard

```text
squid ──► cloudinfra-events.log ──► tailer ──► parser ──► ring buffer ──► live view
                                                    └───► batched ──► SQLite ──► rollups
```

1. Squid writes a structured, tab-separated line per completed request.
2. A tailer follows the file, surviving rotation by tracking device and inode,
   and carrying partial lines across reads.
3. The parser turns each line into a record, rejecting malformed ones rather
   than guessing.
4. Records go to an in-memory ring for [live traffic](../guide/live-traffic.md)
   and, in batches, to SQLite.
5. Batches are folded into per-minute and per-hour rollups as they are written.

### Why rollups

Storing every request forever is not viable, and querying raw rows for a 90-day
chart is slow. Three tiers, each with its own retention:

| Tier | Default | Answers |
|---|---|---|
| Full detail | 3 days | "Show me that exact request" |
| Per-minute | 7 days | Recent charts |
| Per-hour | 90 days | Trends over months |

Rollups are dimensioned by total, domain, client, user and rule. Beyond a
per-bucket cap the tail folds into an "everything else" key, so storage growth
is a function of **time alone** rather than of how many distinct sites your users
visit. Without that cap, growth would be unbounded.

### Two things the log cannot tell you

**Active connections.** The log records transactions that have *finished*, so
"how many are open right now" has no log-derived equivalent. Sampled from Squid's
Cache Manager every ten seconds.

**Long-lived tunnels.** An HTTPS `CONNECT` held open for an hour does not appear
in the log until it closes. The console polls Cache Manager for active requests
so it is visible while running.

## Which rule decided a request

Squid has no field for this. The generator creates one by stamping each rule's
identity onto the transactions it decides, using an ACL that always matches and
exists only for its side effect, placed last on each `http_access` line.

This is the clearest example of why owning both ends matters: the generator
writes the annotation, and the log reader resolves it back to a rule name.
Neither half works alone.

[:material-arrow-right: Details](../reference/log-format.md#field-17-rule-attribution)

## Deployment as a transaction

An apply is closer to a database transaction than to a file copy: render, stage,
validate, snapshot, install, activate, verify, then commit **or** roll back.

The verification step is what makes the rollback meaningful — it includes a real
HTTP request through the proxy, because a proxy that has started but refuses
everything would pass every simpler check.

[:material-arrow-right: The full pipeline](../guide/applying-changes.md)

## Storage

| Path | Contents |
|---|---|
| `/var/lib/cloudinfra/cloudinfra.db` | Accounts, sessions, policy, audit, versions |
| `/var/lib/cloudinfra/analytics.db` | Traffic history |
| `/var/lib/cloudinfra/snapshots/` | Previous configurations |
| `/etc/squid/` | Generated configuration and lists |
| `/etc/squid/.candidate/` | Staging area — the only path under `/etc/squid` the console may write |
| `/etc/squid/local.conf` | Yours. Never generated |

Analytics are in a separate database file so an oversized or corrupted traffic
store can be deleted without touching policy, audit history or configuration
versions.

Both use SQLite in WAL mode with `STRICT` tables. There is no database server to
run, secure, patch or back up — appropriate for a single-tenant appliance, and
one fewer thing in the CVE surface.

## Technology

| | | Why |
|---|---|---|
| **Go** | Console and helper | Single static binary, no runtime, small dependency tree |
| **SQLite** | Storage | No server, no daemon, one file per concern |
| **Vanilla JavaScript** | Console UI | No framework, no build step, no CDN |
| **MkDocs** | This documentation | — |

The console's JavaScript is deliberately plain — no framework and no bundler.
Every asset is embedded in the binary and served from it, so the browser fetches
nothing from anywhere else. A strict Content-Security-Policy enforces that.

## Related

- [Security model](security-model.md) — the privilege boundary in detail
- [Applying changes](../guide/applying-changes.md) — the deployment pipeline
- [Traffic log format](../reference/log-format.md) — the contract between the two halves
