# Caching

Squid is a caching proxy, and the appliance caches by default. This page covers
what the console controls, what actually benefits from caching on the modern
web, and how to tune it for the case that pays best — a fleet of machines
downloading the same packages and updates.

## What the console controls

**Proxy Settings** gives you the size and shape of the cache:

| Setting | What it does |
|---|---|
| Caching enabled | Turns the cache on or off entirely |
| Cache size | Disk space the cache may use, in MB |
| Memory cache | RAM held for the hottest objects |
| Maximum object size | Largest single object that will be stored |

What it does **not** yet control is *which* content is cached and for how long.
That is `refresh_pattern`, and today it is the same on every appliance. Tuning it
means editing [`local.conf`](index.md) by hand, which is what the rest of this
page describes. A content-caching section in the console is on the
[roadmap](../about/roadmap.md).

## Be realistic about what caches

A proxy that does not decrypt traffic cannot cache encrypted responses — it
never sees them. Since most of the web is now HTTPS, **general browsing sees
very little cache benefit**, and a low hit rate is normal rather than a sign of
misconfiguration.

Where caching still earns its place is repetition, and cloud fleets are full of
it:

- **Operating system updates and package repositories.** Fifty instances running
  `apt upgrade` fetch the same files. Debian and Ubuntu archives are plain HTTP
  by default, so they cache well.
- **Container and image layers** served over HTTP.
- **Agent and software rollouts** across many machines.
- **Build and CI environments** pulling the same dependencies repeatedly.

## What the shipped configuration already does

Three refresh patterns ship, the Squid recommended minimum:

```squid
refresh_pattern ^ftp:             1440  20%  10080
refresh_pattern -i (/cgi-bin/|\?) 0     0%   0
refresh_pattern .                 0     20%  4320
```

The last line is the one that matters. It means an object with no explicit expiry
is considered fresh for 20% of its age since it was last modified, capped at
4320 minutes — three days.

That is better for package caching than it first looks. After three days Squid
does not re-download the file; it **revalidates** it with an
`If-Modified-Since` request. If the file has not changed the origin answers
`304 Not Modified` and the body is still served from cache. You pay one small
request, not the download.

So fleet package caching works out of the box. Tuning changes how often that
revalidation happens, not whether caching happens at all.

## Raising the maximum object size

The default maximum object size is **50 MB**. That covers most `.deb` and `.rpm`
packages, but anything larger is silently not stored — container layers, ISOs,
large installers.

If you are caching those, raise it in **Proxy Settings**, and size the cache to
match: a 3 GB cache holding 2 GB objects will thrash.

## Tuning for package repositories

Add patterns to `/etc/squid/local.conf`, which the console never overwrites.
`refresh_pattern` is not order-sensitive in the way access rules are, so it works
there.

```squid
# Package files change under a new name rather than in place, so they can be
# held for a long time without serving anything stale.
refresh_pattern -i \.(deb|udeb|rpm|drpm)$   129600 100% 129600
refresh_pattern -i \.(tar\.gz|tgz|whl|jar)$ 129600 100% 129600

# Repository metadata must stay current or clients install the wrong versions.
# Short freshness here is correct, not a compromise.
refresh_pattern -i /(Release|InRelease|Packages|Sources)(\.gz|\.bz2|\.xz)?$ 0 0% 0
refresh_pattern -i /repodata/.*\.xml(\.gz)?$ 0 0% 0
```

Then check and reload:

```bash
sudo squid -k parse && sudo squid -k reconfigure
```

!!! warning "Most apt-caching guides will not work on this appliance"
    Nearly every Squid package-caching guide online uses `refresh_pattern`
    options such as `override-expire`, `override-lastmod`, `ignore-no-store`,
    `ignore-private` or `ignore-reload`.

    **Those require Squid to be compiled with `--enable-http-violations`, and
    this image is not.** They make Squid ignore the caching instructions an
    origin server sent, which is a deliberate protocol violation — useful in a
    closed lab, and not something we ship on an appliance that carries other
    people's traffic.

    Copy one of those guides and Squid will complain about the option rather
    than apply it. Stick to the `min percent max` tuning above, which needs no
    such build flag and is enough for package caching because repositories
    already send sensible caching headers.

## Excluding things from the cache

To stop something being cached at all, deny it:

```squid
acl no_cache_sites dstdomain .internal.example.com .dynamic.example.com
cache deny no_cache_sites
```

Worth doing for internal applications that return per-user content over plain
HTTP, where a cached response could reach the wrong person.

Anything over HTTPS is not cached regardless, so this only concerns HTTP.

## Checking whether it is working

The console reports the cache hit rate on the Dashboard, calculated **over
cacheable requests only** — encrypted tunnels that could never have been cached
are excluded from the denominator, so the figure is not diluted into
meaninglessness.

From the appliance you can see the same thing in more detail:

```bash
curl -s -x 127.0.0.1:3128 \
  "http://$(awk '$1=="visible_hostname"{print $2}' /etc/squid/squid.conf):3128/squid-internal-mgr/info" \
  | grep -iE "hit ratio|Storage Swap size|Storage Mem size"
```

In the access log, `TCP_HIT` and `TCP_MEM_HIT` are served from cache,
`TCP_REFRESH_UNMODIFIED` is a cheap revalidation, and `TCP_MISS` went to the
origin.

If the hit rate stays near zero on a workload you expected to cache, check the
maximum object size before anything else — it is the most common cause.

## Sizing the cache

The shipped default is 3000 MB on disk and 100 MB in memory. Reasonable starting
points:

| Workload | Disk cache |
|---|---|
| A handful of servers, occasional updates | 3 GB (the default) |
| A fleet patching on a schedule | 10–20 GB |
| Container or CI workloads pulling large layers | 30 GB or more, with the object size raised |

Squid needs roughly 10 MB of RAM per GB of disk cache for its index, on top of
the memory cache. A 30 GB cache therefore wants around 300 MB of RAM before
`cache_mem` is counted — worth checking against the instance size.

Changing the cache directory or its size requires a **restart** rather than a
reload, which drops established connections. The console tells you before it
applies.
