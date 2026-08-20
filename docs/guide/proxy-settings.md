# Proxy Settings

The engine-level settings: what port the proxy listens on, how it caches, and
how it resolves names. Policy lives in [access rules](access-rules.md); this is
everything else.

## Network

| Setting | Default | Notes |
|---|---|---|
| **Proxy port** | 3128 | Changing it requires a **restart**, which drops every active connection |
| **Visible hostname** | `CloudInfra-Proxy` | Appears in Squid's error pages |
| **DNS servers** | Inherited from the instance | Override only if you need to |
| **Connection timeout** | 60 seconds | How long to wait for a destination |

!!! warning "Changing the port needs more than this page"
    Your security group must allow the new port, and every client must be
    reconfigured. The proxy will happily listen somewhere nothing is connecting.

    The [rollback](applying-changes.md) cannot save you here: listening on the
    new port is a complete success from the appliance's point of view.

DNS is worth leaving alone unless you have a reason. The instance's resolver
usually handles internal names correctly, and overriding it is a common way to
break resolution of your own domains.

## Cache

| Setting | Default | Notes |
|---|---|---|
| **Enable cache** | On | Disk caching of cacheable objects |
| **Cache directory** | `/var/spool/squid` | Changing it requires a **restart** |
| **Cache size** | 3000 MB | Keep well under the free space on that volume |
| **Memory cache** | 100 MB | Hot objects held in RAM |
| **Maximum object size** | 50 MB | Larger objects are not cached |

### How much does caching actually help?

Less than it used to, and the console is honest about why.

Almost all traffic is HTTPS, which arrives as an encrypted tunnel this appliance
does not decrypt — so it cannot be cached by anything. A healthy proxy on a
modern network sees a low cache hit rate against total traffic and there is
nothing wrong with that.

The [dashboard](dashboard.md) reports the hit rate against **cacheable** traffic
and states what share was cacheable, so the figure means something.

Caching still earns its place for operating system and package updates,
container image layers, and internal HTTP services — where the same large object
is fetched by many machines.

!!! tip "Sizing the cache"
    Squid needs roughly 10 MB of RAM per GB of disk cache for its index, on top
    of the memory cache. A 20 GB disk cache wants about 200 MB of RAM before
    anything else.

    Leave headroom on the volume. A full cache filesystem stops Squid caching
    and can stop it logging, and the [Health page](health.md) will tell you at
    80%.

Turning the cache off entirely is reasonable for a proxy used purely for access
control and visibility. It removes the disk sizing question and the cache
directory as a failure mode.

## Logging

**Access logging** controls whether Squid writes its native access log. Leave it
on unless you have a specific reason.

!!! danger "This does not disable the console's own traffic log"
    The appliance writes a second, structured log that the dashboard, live
    traffic and analytics all read. Turning off access logging stops the native
    `access.log` your other tooling might read — it does not stop the console
    seeing traffic.

    Disabling it removes the log a great many investigations start from, for a
    disk saving that log rotation already manages.

## Directory authentication

Configured on this page but documented separately — it changes who can use the
proxy at all.

[:material-arrow-right: Directory authentication](directory-authentication.md)

## The generated configuration

At the bottom, read-only, is exactly what would be written to
`/etc/squid/squid.conf`, with a checksum.

You never need to read it for normal administration. It is there because a
product that generates your configuration should let you see what it generated,
and because it is useful when comparing against Squid documentation.

If you need directives the console does not manage, put them in
`/etc/squid/local.conf` — never generated, never overwritten.

[:material-arrow-right: What the generator produces](../reference/generated-config.md)

## Nothing applies until you apply it

Every setting here **stages** a change. The proxy carries on with its current
configuration until you press **Apply changes**.

Settings requiring a restart are flagged before you apply, so an interruption is
never a surprise.

[:material-arrow-right: How applying works](applying-changes.md)

## Related

- [Access rules](access-rules.md) — policy, as opposed to engine settings
- [Health](health.md) — where cache and disk problems show up
- [Configuration file](../reference/configuration.md) — settings for the console itself
