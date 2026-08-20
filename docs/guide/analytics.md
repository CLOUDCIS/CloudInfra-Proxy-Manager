# Analytics

The [dashboard](dashboard.md) answers *what is happening now*. Analytics answers
*what happened over a period you choose*.

Both read the same stored records, deliberately — comparing the two over the same
window and finding different numbers would rightly make you distrust both.

## Choosing a period

Presets cover the common cases — last 24 hours, 7 days, 30 days, 90 days. The
**From** and **To** fields take any window, which is what the page exists for:
*"what happened during the outage last Tuesday afternoon"* is not a preset.

How far back you can go depends on your
[retention settings](administration.md#data--storage).

## The tabs

### Traffic

Totals and shape over the period: requests, how many were blocked, bytes
transferred, cache hit rate and the HTTPS share — with a chart of requests over
time and a second line for blocked traffic.

The chart states what one point covers, because a 90-day chart and a one-hour
chart otherwise look identical.

!!! info "Cache hit rate is measured against cacheable traffic"
    The figure is the share of **cacheable** requests that were served from
    cache, and the page says what proportion of traffic that was.

    Measured against all traffic instead, a healthy proxy reports 2–5% and looks
    broken — because almost everything is HTTPS tunnels, which are not cacheable
    by anything that does not decrypt them.

### Destinations

Top domains by requests, with blocked counts and bandwidth for each. This is
where "what is actually using our bandwidth" gets answered.

### Clients

The same breakdown by client address.

### Users

The same breakdown by signed-in user. Only populated when
[directory authentication](directory-authentication.md) is enabled — the proxy
only learns who made a request when users authenticate.

If it is empty, the page says why rather than showing a blank table, because an
empty table reads as broken analytics rather than as a feature that is switched
off.

Once available it is usually more useful than Clients: an address stops
identifying a person the moment somebody moves desk or reconnects.

### Blocks

What is being refused, and by which rule. Rules are named rather than numbered,
and rules that have since been **deleted** are still named — *"a rule that no
longer exists blocked 4,000 requests last month"* is a true and useful
statement.

Blocks by the appliance itself appear as *"Appliance default"*.

## Everything else

The long tail beyond the top entries is grouped as **Everything else** rather
than dropped, so a total that does not obviously add up has its explanation on
screen. The page says when this has happened.

This exists because storage growth would otherwise track the number of distinct
sites your users visit, which is unbounded. Folding the tail makes it a function
of time alone.

## Export

**Export CSV** downloads the current breakdown over the current period — the same
query the table is showing, up to 500 rows.

Values are quoted, so a domain containing a comma cannot shift every column, and
entries beginning `=`, `+`, `-` or `@` are prefixed so a spreadsheet does not
evaluate them as formulas.

!!! note "Exports are audited"
    Exporting traffic records is a read of your users' browsing history, so it
    is recorded in the [audit trail](administration.md#audit-trail) with who,
    when, and what range — the same as searching the logs.

## Retention and what you can ask

Traffic is stored at three levels of detail, each kept for a different length of
time:

| Level | Default | Answers |
|---|---|---|
| Full request detail | 3 days | "Show me that exact request" |
| Per-minute summaries | 7 days | Recent charts, fine detail |
| Hourly summaries | 90 days | Trends over months |

So a 90-day Analytics query works, while retrieving one individual request from
90 days ago does not. Full detail is by far the largest consumer of disk, which
is why it has the shortest window.

All three are adjustable.

[:material-arrow-right: Retention settings](administration.md#data--storage)

## Related

- [Dashboard](dashboard.md) — the same data, right now
- [Live traffic](live-traffic.md) — individual requests
- [Directory authentication](directory-authentication.md) — enabling the Users tab
