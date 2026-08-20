# Access Rules

Access rules decide what the proxy permits. You write them as sentences; the
console turns them into correct Squid ACL directives. You never edit
`squid.conf`.

## The model

A rule has four parts:

| Part | Options |
|---|---|
| **Source** | Any client · An IP address · A network (CIDR) · A named network list · A directory user · A directory group |
| **Destination** | Anywhere · A domain · A network (CIDR) · A named domain list |
| **Action** | Allow · Deny |
| **Schedule** | Always |

Which reads as:

> **Deny** clients in `10.20.0.0/16` to reach `facebook.com` at all times.

The review step of the wizard shows you that sentence before anything is
created. You confirm a decision, not a configuration fragment.

!!! note "Schedules"
    Only *Always* is available in this release. The field exists so that
    time-based policies later need no migration of your existing rules.

## Order matters

Squid evaluates rules top to bottom and **stops at the first match**. So a broad
allow above a narrow deny means the deny never fires.

Rules are listed in evaluation order and can be reordered by dragging. The
priority number shown is the order, not an importance rating.

### Shadowed rules

The console analyses your rule set and warns when one rule can never match
because an earlier one already covers it:

> This rule can never match. Rule 2 *"Allow everyone everywhere"* already allows
> everything it would allow.

The analysis is deliberately conservative — it only reports a rule as shadowed
when it can prove containment, so it will miss some cases rather than cry wolf
about rules that are actually fine.

## Domain matching

This is where Squid's behaviour surprises people, so the console is explicit
about it.

| You write | It matches | It does not match |
|---|---|---|
| `example.com` | `example.com` only | `www.example.com` |
| `.example.com` | `example.com` and every subdomain | — |
| `*.example.com` | Normalised to `.example.com` | — |

If you mean "the whole site", use the leading dot. The wizard shows what your
entry will match before you save it.

## Built-in rules

Before any of your rules, the appliance enforces a small set you cannot reorder
or remove:

- Unsafe destination ports are denied.
- `CONNECT` to anything other than a TLS port is denied.
- The local Cache Manager interface is reachable from the appliance itself, so
  the console can read live statistics.
- Anything not matched by your rules is denied.

That last one is the important one: **the default is deny**. A request reaches
the internet because a rule you wrote allowed it, not because nothing stopped
it.

Blocks caused by these built-ins are attributed to *"Appliance default"* in the
console rather than to one of your rules, so you can always tell "your policy
blocked this" from "the proxy refused it".

## Which rule decided a request

Squid has no native field recording which `http_access` line decided a request.
The generator adds one: each rule stamps its own identifier onto the
transactions it decides, and the traffic log records it.

The practical result is that in [Live Traffic](live-traffic.md) you can click any
request — allowed or blocked — and see the rule responsible by name. Not "denied
by policy", but *"Blocked by: Block social media"*.

This works because the appliance owns both the configuration generator and the
log reader. It is the single most useful thing the product does when somebody
asks why a site is not working.

## Rules that name people

With [directory authentication](directory-authentication.md) enabled, the source
of a rule can be a **user** or a **group** rather than an address:

> **Allow** members of `Sales` to reach `dropbox.com` at all times.

Two things to know:

- A group rule implies authentication. The generated configuration requires a
  signed-in user before the group is looked up, because a group lookup with no
  username is meaningless.
- Creating a user or group rule while authentication is **off** is refused, with
  an explanation. Squid would never learn who made a request, so the rule would
  silently never match — which is the worst possible outcome for an access rule.

## Enabling and disabling

A rule can be disabled without deleting it. A disabled rule is not generated at
all, so it costs nothing and keeps its place in the order for when you turn it
back on. Useful for testing whether a rule is the cause of a problem.

## Nothing applies until you apply it

Creating, editing, reordering, enabling or deleting a rule **stages** the
change. The proxy carries on with its current policy until you press **Apply
changes**.

[:material-arrow-right: How applying works](applying-changes.md)

## A worked example

A common starting policy, in order:

| # | Rule | Reads as |
|---|---|---|
| 1 | Block malware domains | Deny any client to reach *Malware list* |
| 2 | Allow Microsoft 365 | Allow any client to reach *Microsoft 365 domains* |
| 3 | Block social media | Deny clients in `10.20.0.0/16` to reach *Social list* |
| 4 | Allow the office | Allow clients in `10.20.0.0/16` to reach anywhere |

Order is doing real work here. Moving rule 4 to the top would make rules 1–3
unreachable, and the console would tell you so.

## Related

- [URL filtering](url-filtering.md) — building the lists rules refer to
- [Microsoft 365 endpoints](microsoft-365.md) — a list that maintains itself
- [Live traffic](live-traffic.md) — seeing rules take effect
- [The generated squid.conf](../reference/generated-config.md) — what your rules become
