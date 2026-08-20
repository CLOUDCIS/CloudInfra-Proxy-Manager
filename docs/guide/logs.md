# Logs

Read the appliance's logs from the console. Diagnosing a proxy problem should
not require an SSH session, and with this it does not.

## What you can read

| Log | Contents |
|---|---|
| **Proxy access log** | Every request the proxy handled, in Squid's own format |
| **Proxy error log** | Squid's warnings and errors — the first place to look when the proxy misbehaves |
| **Management log** | This console: sign-ins, configuration changes and its own errors |

A log that does not exist yet is still listed, and says so. Omitting it would
leave you wondering whether the console had forgotten about it.

The viewer opens on a log that actually has content, so a fresh appliance whose
proxy has not run yet does not present an empty page that reads as broken.

## Filtering

| Control | Does |
|---|---|
| **Search** | Case-insensitive substring match |
| **Severity** | All, warnings and errors, or errors only |
| **Time range** | Last 15 minutes through last 7 days |
| **Lines** | 200 to 2000 |
| **Include archived files** | Extends the search into rotated logs |

Severity is derived per log. Squid's error log carries its own `WARNING:` and
`ERROR:` markers; the management log uses structured levels. In the access log,
a **denied** request is not an error — it is the proxy working — so denials stay
at information level while `5xx` results are raised.

## Reading the status line

Under the controls, the viewer states what it actually did:

> 200 lines · read 15 KB · from access.log · more lines match than are shown

Two different statements can appear, and they mean different things:

- **"more lines match than are shown"** — ordinary. Your line limit was reached;
  raise it or narrow the search.
- **"stopped after reading the newest portion of the log"** — important. The
  read gave up before reaching the end of the file, so there may be older
  matches nobody has looked at. Narrow the time range or search a smaller log.

A search that quietly stops early is a search that lies, so the second case is
always stated.

## How it reads large files

Access logs reach gigabytes. The viewer never loads one whole: it scans backwards
from the end in chunks and stops as soon as it has enough matching lines. Asking
for the last 200 lines of a two-gigabyte log reads a few hundred kilobytes.

There is a cap on how far back a single search will read, which is where the
"stopped after reading the newest portion" message comes from.

## Archived logs

**Include archived files** extends the search into rotated logs, including
compressed ones. Lines from an archive are marked, so you always know whether
you are looking at current or historical output.

Rotation is configured on the image: the access log rotates daily and is kept
for 14 days; the error log rotates weekly or at 32 MB and is kept for 8 weeks.
The [Health page](health.md) warns if rotation stops running, because logs that
grow unchecked will eventually fill the disk.

## Auto-refresh

Re-runs the current search every ten seconds. It stops when you leave the page —
an auto-refresh that keeps reading a gigabyte log while nobody is watching costs
the appliance real work.

## Searches are audited

Searching a proxy log is a read of your users' browsing records, so it is
recorded in the [audit trail](administration.md#audit-trail) with who searched,
when, and for what.

The audit trail exists to answer *who looked at what*, not only *who changed
what*.

## Where the files are

Listed for reference; you do not need them for normal use.

| Log | Path |
|---|---|
| Proxy access log | `/var/log/squid/access.log` |
| Proxy error log | `/var/log/squid/cache.log` |
| Structured traffic log | `/var/log/squid/cloudinfra-events.log` |
| Management log | `/var/log/cloudinfra/manager.log` |

The structured traffic log is the one the console's analytics read. Your ordinary
`access.log` is left untouched in Squid's native format, so any existing tooling
you point at it keeps working.

[:material-arrow-right: Traffic log format](../reference/log-format.md)

## Related

- [Health](health.md) — what a failing check usually points at
- [Live traffic](live-traffic.md) — the parsed view of the same requests
- [Traffic log format](../reference/log-format.md) — the field-by-field reference
