# Dashboard

The page you land on. It answers *is this working, and what is it doing right
now?*

## The appliance header

Across the top, the facts you need before anything else:

```text
proxy-01 · 10.20.1.4 · Squid 7.6 · Healthy · Up 18d 7h
```

Plus operational chips: the port the proxy is actually listening on, whether
caching is enabled and at what size, how many rules are active, and whether
there are pending changes waiting to be applied.

The port shown is the one the proxy is **configured** for, not an assumed 3128 —
so the header never claims a port the proxy is not on.

## Key figures

Across the selected period:

| Figure | Notes |
|---|---|
| **Requests** | Total handled, with the allowed and blocked split |
| **Bandwidth** | Downloaded and uploaded |
| **Blocked** | Count and share of total |
| **Active connections** | Open right now |

Each shows movement against the previous equivalent period, so a spike is
visible without comparing two screens.

!!! info "Active connections come from Squid, not the log"
    The access log records transactions that have **finished**, so a count of
    what is open right now has no log-derived equivalent. That figure is sampled
    from Squid's Cache Manager every ten seconds.

## Traffic chart

Requests over time, with allowed and blocked distinguished, over the selected
range. The time selector — 1 hour through 30 days — drives the whole page.

## Panels

**Top Destinations** and **Top Clients** — the busiest of each, with requests
and bandwidth. Clicking through filters the traffic view.

**Recent Blocked Requests** — the most recent refusals, each naming the rule
responsible. Usually the fastest route to *"why can't I reach this?"*

**Proxy Health** — a compact summary showing only what needs attention. When
everything passes it says so in one line, and if some checks could not be
determined it says that too, because an unknown check is not a pass.

[:material-arrow-right: Full health page](health.md)

## Cache effectiveness

Reported as the share of **cacheable** requests served from cache, alongside what
proportion of traffic was cacheable at all.

Measured against all traffic, a perfectly healthy proxy reports 2–5% and looks
broken — because almost everything is HTTPS tunnels, which nothing can cache
without decrypting them. Reporting it that way would be technically true and
practically useless.

## HTTP and HTTPS split

The share of traffic that is encrypted. On a typical network this is 90% or
more, and it is the single best explanation for why a proxy's cache hit rate and
URL visibility are lower than somebody expects.

## When traffic history is unavailable

If the analytics store cannot be opened, the dashboard says so and the rest of
the console keeps working — the appliance header, health, policy editing and
applying changes all continue.

An analytics failure must never take the proxy or the management interface with
it.

## Related

- [Live traffic](live-traffic.md) — individual requests as they happen
- [Analytics](analytics.md) — arbitrary periods and deeper breakdowns
- [Health](health.md) — the checks behind the pill
