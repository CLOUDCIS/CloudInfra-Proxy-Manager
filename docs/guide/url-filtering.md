# URL Filtering

Lists are named, reusable sets of domains or networks that
[access rules](access-rules.md) refer to. You maintain *"Blocked Domains"* once
rather than repeating the same entries across five rules.

## Two kinds

| Kind | Holds | Used as |
|---|---|---|
| **Domain list** | Hostnames and domain suffixes | A rule's destination |
| **Network list** | CIDR ranges and addresses | A rule's source or destination |

## Creating one

**URL Filtering → Add list.** Give it a name, choose the kind, and paste entries
one per line.

```text
facebook.com
.instagram.com
.tiktok.com
x.com
```

Entries are normalised as they are saved, so what you see stored is exactly what
the proxy will match on.

### Domain matching

The same rule as everywhere in Squid, and the one thing worth being careful
about:

| Entry | Matches | Does not match |
|---|---|---|
| `example.com` | `example.com` only | `www.example.com` |
| `.example.com` | `example.com` and all subdomains | — |

If you mean the whole site, use the leading dot. A list of bare hostnames will
quietly fail to block the `www.` version, which is the most common way a
blocklist appears not to work.

!!! info "Domains, not paths"
    Lists match on the hostname. Blocking `example.com/one-section` is not
    possible for HTTPS sites, because the path is inside the encrypted session
    and never reaches the proxy — see
    [HTTPS and CONNECT](../squid/https-and-connect.md#what-this-means-for-filtering).
    This is the second most common reason a list appears not to work.

### Network entries

```text
10.20.0.0/16
192.168.1.0/24
203.0.113.45/32
```

A bare address is accepted and treated as a single host.

## Using a list

Lists do nothing on their own — they take effect when a rule refers to one:

> **Deny** clients in `10.20.0.0/16` to reach *Social Media* at all times.

A list can be referenced by any number of rules.

## Enabling and disabling

A list can be disabled without deleting it. A disabled list is not written into
the configuration at all, so rules referring to it stop matching — useful for
temporarily lifting a block without unpicking your rules.

## Deleting

Deleting a list that a rule still refers to is **refused**, naming the rule. A
dangling reference would fail the next apply, and discovering that at deployment
time is far worse than being told now.

Remove or repoint the rule first.

## Large lists

Lists are written to files on disk rather than inline in the configuration.
Squid reads large sets from files far more efficiently, so a list of several
thousand domains is entirely reasonable.

The practical limit is memory rather than the console: every entry is loaded by
Squid at startup.

## Automatic lists

Some lists are maintained by the appliance rather than by you. They are marked
<span class="pill-neutral">AUTOMATIC</span> and offer **View** instead of
**Edit**.

The [Microsoft 365 endpoints](microsoft-365.md) feature creates lists this way.
Their contents come from a published feed and are rewritten at each refresh, so
an edit would be discarded — the console refuses it rather than silently
reverting your work later.

To change what an automatic list contains, change the settings of the feed that
owns it.

## Nothing applies until you apply it

Creating, editing or deleting a list **stages** the change like any other.

[:material-arrow-right: How applying works](applying-changes.md)

## A practical structure

Lists that tend to earn their keep:

| List | Kind | Purpose |
|---|---|---|
| **Allowed Domains** | Domains | Sites permitted regardless of other rules |
| **Blocked Domains** | Domains | The organisation's blocklist |
| **Office Networks** | Networks | Client ranges permitted to use the proxy |
| **Server Networks** | Networks | Servers with a different, narrower policy |

Combined with rule ordering, that covers most policies without a rule per site.

## Related

- [Access rules](access-rules.md) — where lists are used
- [Microsoft 365 endpoints](microsoft-365.md) — lists that maintain themselves
- [Applying changes](applying-changes.md) — getting them live
