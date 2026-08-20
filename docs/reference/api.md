# REST API

The console is a browser client of a REST API on the same port. Everything the
console can do is available programmatically.

**Base:** `https://<appliance>:8443/api/v1`

!!! note "Stability"
    The API exists to serve the console, and is documented so you can automate
    against it. It is versioned in the path, and the `v1` shapes here are stable
    within a release — but it is not a committed public contract in the way a
    published SDK would be. Pin your appliance version if you build on it.

## Authentication

Sign in, keep the cookies, and send the CSRF token on anything that changes
state.

```bash
# Sign in, storing cookies
curl -sk -c jar.txt -X POST https://proxy:8443/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"..."}'

# The CSRF token arrives as a readable cookie
CSRF=$(awk '/csrf/ {print $7}' jar.txt | tail -1)

# Read
curl -sk -b jar.txt https://proxy:8443/api/v1/system/health

# Write — needs the header
curl -sk -b jar.txt -H "X-CloudInfra-CSRF: $CSRF" \
  -H 'Content-Type: application/json' \
  -X POST https://proxy:8443/api/v1/policies -d '{...}'
```

The session cookie is `HttpOnly` and `__Host-` prefixed. The CSRF cookie is
readable by design — it is the double-submit half of the pattern.

There are no API keys or service accounts in this release. Sessions expire after
8 hours, or 60 minutes idle.

## Conventions

| | |
|---|---|
| **Content type** | `application/json` in and out |
| **Errors** | `{"error": "A sentence intended for a person."}` |
| **Validation** | `400` with `{"error":..., "fields":[{"field":..., "message":...}]}` |
| **Times** | RFC 3339 |
| **Caching** | Every response is `no-store` |

Error messages are written for people. Stack traces, database errors and
internal paths go to the appliance's log, never to a client — they help an
attacker far more than they help you.

## Unauthenticated

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/healthz` | Liveness. Reveals nothing else. |
| `POST` | `/auth/login` | Sign in |
| `GET` | `/auth/session` | Whether the caller is signed in |

## Session

| Method | Path |
|---|---|
| `POST` | `/auth/logout` |
| `POST` | `/auth/password` |

## System

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/system/appliance` | Hostname, addresses, versions, uptime, cloud |
| `GET` | `/system/health` | The scored health report |
| `GET` | `/system/metrics` | Memory, load, filesystems, storage |

## Traffic and analytics

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/metrics/summary` | Dashboard totals. `?range=1h\|6h\|24h\|7d\|30d` |
| `GET` | `/metrics/series` | Traffic chart points |
| `GET` | `/metrics/top` | `?dim=domain\|client` |
| `GET` | `/metrics/security` | Blocked-activity counts |
| `GET` | `/traffic/search` | Stored requests. Filters below |
| `GET` | `/traffic/active` | Connections open now |
| `GET` | `/traffic/live` | Server-sent event stream |
| `GET` | `/traffic/{id}` | One request in full |
| `GET` | `/analytics/summary` | Totals and series over an arbitrary window |
| `GET` | `/analytics/top` | `?dim=domain\|client\|user\|rule` |
| `GET` | `/analytics/export` | The same breakdown as CSV |

`/traffic/search` accepts `q`, `client`, `domain`, `method`, `action`, `limit`
and `before` (a keyset cursor — offset paging would rescan an ever-growing
prefix).

Analytics accepts `from` and `to` as epoch seconds or RFC 3339, defaulting to
the last seven days:

```bash
curl -sk -b jar.txt \
  "https://proxy:8443/api/v1/analytics/top?dim=rule&from=1755600000&to=1755686400"
```

## Policy

Every write here **stages** a change. Nothing reaches Squid until you apply.

| Method | Path |
|---|---|
| `GET` `POST` | `/policies` |
| `PUT` `DELETE` | `/policies/{id}` |
| `POST` | `/policies/reorder` |
| `GET` `POST` | `/lists` |
| `PUT` `DELETE` | `/lists/{id}` |

```bash
curl -sk -b jar.txt -H "X-CloudInfra-CSRF: $CSRF" \
  -H 'Content-Type: application/json' \
  -X POST https://proxy:8443/api/v1/policies -d '{
    "name": "Block social media",
    "source":      {"type":"cidr","value":"10.20.0.0/16"},
    "destination": {"type":"domain","value":".facebook.com"},
    "action": "deny",
    "enabled": true
  }'
```

Source types: `any`, `ip`, `cidr`, `network_group`, `user`, `group`.
Destination types: `any`, `domain`, `domain_group`, `cidr`.

Editing a list maintained by a feed returns `409` — those are read-only.

## Settings

| Method | Path |
|---|---|
| `GET` `PATCH` | `/settings/proxy` |
| `GET` `PATCH` | `/settings/directory` |
| `GET` `PATCH` | `/settings/microsoft365` |
| `POST` | `/settings/microsoft365/refresh` |
| `GET` `PATCH` | `/settings/retention` |

!!! info "The directory bind password is write-only"
    `GET /settings/directory` never returns it — only `hasBindPassword`. Sending
    an empty `bindPassword` on `PATCH` keeps the stored one; sending a value
    replaces it.

## Configuration and deployment

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/config/pending` | Staged changes, and whether Apply is available |
| `GET` | `/config/preview` | The `squid.conf` that would be written |
| `POST` | `/config/apply` | Deploy. Streams progress |
| `GET` | `/config/versions` | Deployment history |
| `GET` | `/config/versions/{v}/diff` | Policy-level changes in a version |
| `POST` | `/config/versions/{v}/restore` | Stage an earlier version |

`/config/apply` streams Server-Sent Events — `step` events as each stage runs,
then a final `outcome`:

```bash
curl -Nsk -b jar.txt -H "X-CloudInfra-CSRF: $CSRF" \
  -X POST https://proxy:8443/api/v1/config/apply
```

```text
event: step
data: {"name":"validate","status":"ok","elapsedMs":204}

event: outcome
data: {"result":"applied","version":7,"restarted":false}
```

`result` is `applied`, `rolled_back` or `failed`. A `503` before the stream
starts means deployment is unavailable.

## Backup

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/backup` | Download. Body `{"passphrase":"..."}`, optional |
| `POST` | `/backup/inspect` | Describe an uploaded file. Changes nothing |
| `POST` | `/backup/restore` | Stage the policy from a file |

Inspect and restore take `multipart/form-data` with a `file` part and an
optional `passphrase` field. A protected file with no passphrase returns `401`
with `needPassphrase: true` — a prompt, not a failure.

## Logs

| Method | Path |
|---|---|
| `GET` | `/logs/sources` |
| `GET` | `/logs` |

`/logs` accepts `source` (`proxy-access`, `proxy-cache`, `manager`), `q`,
`level`, `since` (a duration such as `24h`), `limit` and `rotated=1`.

The source is a name from a closed set — a path is rejected.

## Administration

| Method | Path |
|---|---|
| `GET` | `/admin/profile` |
| `GET` | `/admin/sessions` |
| `DELETE` | `/admin/sessions/{ref}` |
| `GET` | `/admin/logins` |
| `GET` | `/admin/security` |
| `GET` | `/audit` |

## Rate limiting

Sign-in attempts are limited per address and per account, returning `429` with a
message saying when to try again. Read endpoints are not rate limited.

## Automating against it

Two things worth knowing:

- **A policy change is two steps.** Write the rule, then `POST /config/apply`. A
  script that only does the first leaves a staged change nobody applied.
- **Watch the outcome.** A `200` from `/config/apply` means the pipeline ran, not
  that the change is live — check `result` in the `outcome` event, since
  `rolled_back` is also a successful run of the pipeline.

## Related

- [Applying changes](../guide/applying-changes.md) — what the pipeline does
- [Configuration file](configuration.md) — the console's own settings
