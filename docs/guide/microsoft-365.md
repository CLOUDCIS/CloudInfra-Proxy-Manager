# Microsoft 365 Endpoints

Microsoft 365 resolves to a large and constantly changing set of hostnames and
address ranges. Microsoft publishes them, changes them roughly monthly, and
expects administrators to keep up.

Keeping a proxy's rules in step by hand is miserable, and getting it wrong looks
to your users like *"Teams is broken"*. This turns Microsoft's published feed
into ordinary lists your rules can reference, and keeps them current.

## What it does

When enabled, the appliance fetches Microsoft's endpoint list and maintains up
to three lists:

| List | Contents |
|---|---|
| **Microsoft 365 domains** | Hostnames, converted to Squid's domain-matching form |
| **Microsoft 365 addresses** | IPv4 ranges |
| **Microsoft 365 IPv6 addresses** | IPv6 ranges |

They appear in [URL Filtering](url-filtering.md) alongside your own lists,
marked <span class="pill-neutral">AUTOMATIC</span>, and any
[access rule](access-rules.md) can reference them.

## Setting it up

**URL Filtering → Microsoft 365 Endpoints.**

### Cloud

| Option | For |
|---|---|
| **Worldwide** | The commercial cloud — almost everyone |
| **US Government GCC High** | GCC High tenants |
| **US Government DoD** | DoD tenants |
| **China (21Vianet)** | Tenants operated by 21Vianet |

!!! warning "Sovereign clouds publish completely different addresses"
    Choosing the wrong one allows ranges your tenant never uses and blocks the
    ones it does. If you are unsure, you are on Worldwide.

### Categories

Microsoft's own classification:

| Category | What it is | Recommended |
|---|---|---|
| **Optimize** | The latency-sensitive core — Teams media, Exchange Online, SharePoint | Yes |
| **Allow** | Required for the services to work, and tolerant of proxying | Yes |
| **Default** | The long tail of CDNs and telemetry | Usually not |

Optimize and Allow are selected by default, which is Microsoft's own
recommendation. Default is large and is not required for the services to
function.

### Service areas and required-only

Restrict to particular service areas — Exchange, SharePoint, Skype, Common — if
you only use some of them. Leave everything selected to include whatever
Microsoft publishes, including areas added in future.

**Required only** skips endpoints Microsoft marks optional. It is on by default
and accounts for most of the Default category's bulk.

### Refresh interval

How often to check for a new published version. Daily is generous — Microsoft
changes the list about monthly.

The full list is only downloaded when the published version has actually
changed. The version endpoint is a few bytes; the endpoint list is hundreds of
kilobytes, and downloading it hourly regardless would be rude to a free service
the appliance depends on.

## Using the lists

Create an [access rule](access-rules.md) referring to them. The typical one:

> **Allow** any client to reach *Microsoft 365 domains* at all times.

Place it **above** any broad deny, so Microsoft 365 traffic is permitted before
a general restriction can catch it.

!!! tip "Two lists, two rules"
    Most traffic matches on domain. The address lists matter for clients that
    connect by IP, and for Teams media in particular. If Teams calls fail while
    everything else works, add a rule for the address list too.

## What it does to your traffic

The lists **allow** traffic; they do not bypass the proxy. Microsoft's own
guidance for the Optimize category is to send it directly rather than through a
proxy at all, because proxying adds latency to real-time media.

This product does not implement bypass — it is a forward proxy, and traffic sent
to it goes through it. If you want true bypass for Optimize endpoints, do it in
your network routing or PAC file, using the same address list as a reference.

## Refresh state

The panel reports what actually happened, not just that the feature is on:

- **Published version** and when it was last successfully updated
- **Contents** — how many domains and ranges are in effect
- **Entries skipped** — anything in the feed that could not be used

That last figure is worth watching. Microsoft's feed is somebody else's JSON
arriving over the network and becoming proxy configuration, so anything that
would not make a valid rule is dropped — and counted, so a change in the feed's
shape shows up as a rising skip count rather than as lists that quietly shrink.

**Refresh now** fetches immediately rather than waiting for the schedule.

## Appliances with no internet access

An appliance with no route out is a supported configuration. The refresh simply
fails, and the panel says so:

> The last refresh did not succeed. This appliance may have no route to the
> internet, which is a supported configuration — these lists simply cannot be
> maintained without one.

Lists already in place keep working. The appliance keeps trying, and a
previously successful update stays visible, so you can tell *"never worked"*
from *"worked, but the last attempt failed"*.

The endpoint feed is reached at `endpoints.office.com` over HTTPS, outbound only.
It is the only scheduled outbound connection the appliance makes.

## The lists are read-only

You cannot edit their contents. An edit would be discarded at the next refresh,
and silently reverting somebody's work is worse than refusing it — so the
console shows them read-only, with a **View** action instead of **Edit**.

To change what they contain, change the categories and service areas above.

## Turning it off

Untick *Maintain lists* and save. The lists are removed rather than left stale —
a list nothing updates any more, still referenced by rules, is worse than no
list at all.

If a rule still references one, removal is refused with the rule named. Delete
or repoint the rule first.

## Related

- [URL filtering](url-filtering.md) — your own lists
- [Access rules](access-rules.md) — referencing lists in policy
- [Applying changes](applying-changes.md) — a refresh stages a change like any other
