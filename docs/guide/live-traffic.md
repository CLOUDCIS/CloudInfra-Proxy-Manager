# Live Traffic

A running view of requests as the proxy handles them, and the fastest way to
answer *"why is this site not working?"*

## The stream

Requests appear within a second or two of completing, showing time, client,
destination, method, result, size and duration, with allowed and blocked clearly
distinguished.

**Pause** freezes the view without stopping collection — useful when something
interesting scrolls past. **Clear** empties the display. The search box filters
what is shown as you type.

## Request details

Click any request for the full record:

- The complete URL, method and protocol
- Client address, and username when
  [authentication](directory-authentication.md) is on
- Squid's result code and the HTTP status
- Bytes in and out, and how long it took
- Cache result — hit, miss, or not cacheable
- **The rule that decided it**

That last line is the one that matters. Not *"denied by policy"* but:

> **Blocked by:** Block social media

Squid has no native field recording which `http_access` line decided a request.
The configuration generator adds one, stamping each rule's identity onto the
transactions it decides, and the log reader resolves it back to the rule's name.
This works only because the appliance owns both ends.

Blocks the appliance made itself — an unsafe port, a `CONNECT` to a non-TLS
port, the default deny — are attributed to *"Appliance default"*, so you can
always tell *"your policy blocked this"* from *"the proxy refused it"*.

## HTTPS and what you can see

Almost all traffic is HTTPS, which arrives as a `CONNECT` tunnel. The appliance
does **not** decrypt it, so for an HTTPS request you see:

- The **hostname** the client asked for
- The **bytes** transferred and the **duration**
- Whether it was allowed or blocked, and by which rule

You do **not** see the path, the query string, or anything inside the tunnel.
That is a deliberate design position, not a limitation to be worked around — see
[TLS interception](../about/faq.md#does-it-decrypt-https) in the FAQ.

!!! info "Why a long-running tunnel appears late"
    Squid writes a log entry when a transaction **completes**. A `CONNECT`
    tunnel held open for an hour does not appear in the log until it closes.

    The console fills that gap by polling Squid's Cache Manager for currently
    active requests, so a long-lived connection is visible while it is still
    running rather than only in hindsight.

## Active connections

The dashboard reports connections open right now, and how many distinct clients
are using the proxy. Neither figure exists in the access log at all — the log
records what has finished. They come from Squid's Cache Manager, sampled every
ten seconds.

## When to use this instead of Analytics

| Use Live Traffic for | Use [Analytics](analytics.md) for |
|---|---|
| "Is this working *now*?" | "What happened last Tuesday?" |
| Watching a change take effect | Trends over weeks |
| Finding why one request failed | Top destinations, clients and users |
| Confirming a client is reaching the proxy | Reporting |

## Searching history

The live view holds recent traffic. To search further back, use the search
filters — client, domain, method, action and time range — which query the stored
history rather than the live buffer.

Full request detail is kept for a shorter period than the summarised history, so
very old individual requests may no longer be retrievable even though they are
still counted in Analytics. The retention window is yours to set.

[:material-arrow-right: Retention settings](administration.md#data--storage)

## Related

- [Access rules](access-rules.md) — where the deciding rules come from
- [Analytics](analytics.md) — the same traffic, historically
- [Logs](logs.md) — the raw log behind all of this
