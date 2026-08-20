# High Availability

A forward proxy holds no shared state between requests, so running more than one
is straightforward: put several behind a load balancer and clients fail over
without noticing.

The complications are configuration drift and, if you use it, authentication.

## The shape

```
        clients
           │
   internal load balancer   (TCP / layer 4, port 3128)
        ┌──┴──┐
     proxy-a  proxy-b       (separate availability zones)
        └──┬──┘
        internet
```

Use a **layer 4 / TCP** load balancer — an AWS Network Load Balancer, an Azure
Load Balancer, or a Google Cloud internal TCP load balancer.

!!! warning "Not a layer 7 load balancer"
    A layer 7 balancer tries to interpret the HTTP it sees, and proxy requests
    are not ordinary HTTP requests. `CONNECT` in particular will confuse it.

Point clients at the load balancer's address instead of an individual proxy —
see [Pointing Clients at the Proxy](../getting-started/clients.md).

## Health checks

A TCP check against port 3128 proves the port is open, which is not the same as
proving Squid is healthy. Prefer an HTTP check where your balancer supports one:

| Setting | Value |
|---|---|
| Protocol | HTTP |
| Port | 3128 |
| Path | `/squid-internal-static/icons/SN.png` |
| Expected | 200 |

Squid serves that file itself and needs no upstream connectivity to do it, so
the check distinguishes "Squid is running" from "the internet is reachable".

!!! note "Why a `squid-internal-static` path and not `squid-internal-mgr`"
    Squid rewrites `/squid-internal-static/` requests to its own hostname
    automatically, so a health check that arrives addressed to the instance's
    address is still served. Management reports under `/squid-internal-mgr/` get
    no such rewrite — they must name the proxy's own `visible_hostname` and
    port, or Squid forwards them as ordinary requests and they fail. See
    [Troubleshooting](../troubleshooting.md#reading-cache-manager-by-hand).

Allow the balancer's health-check source ranges to reach port 3128 alongside
your client networks.

Do **not** put the console on 8443 behind the load balancer. Each appliance's
console manages that appliance, and its session cookies are per-instance. Reach
each console directly.

## Session affinity

Not required. Each request is independent, and a client whose connection moves
between proxies will not notice — with one exception.

If you use authentication, each appliance caches credentials separately
(`credentialsttl`). A client moving between proxies may be asked to
re-authenticate. Browsers resend cached credentials automatically so users rarely
see a prompt, but scripted clients that do not cache will authenticate once per
proxy.

## Keeping configuration in step

This is the real work. **Nothing synchronises configuration between
appliances.** Each holds its own database, its own policy and its own version
history, and they will drift.

Multi-appliance management is on the [roadmap](../about/roadmap.md). Until then:

**Configure each appliance identically, by hand.** Practical for two appliances
and a policy that changes rarely. Use
[Backup & Restore](../guide/backup-restore.md) to copy a known-good
configuration from one to the other, which is faster and less error-prone than
repeating the clicks.

**Build a custom image.** Configure one appliance, snapshot it, and launch
replicas from the image. Every change then means a new image and a rolling
replacement — reasonable if your policy is stable.

**Treat one appliance as the source of truth.** Make changes there, export a
backup, restore it onto the others. This keeps the version history meaningful on
at least one of them.

!!! note "Restore copies policy, not identity"
    A restored backup brings the access rules, lists and settings. It does not
    bring the TLS certificate or administrator credentials, which are generated
    per instance at first boot. That is deliberate — see
    [Security Model](../about/security-model.md).

## Sizing

Squid handles requests on a single thread. Beyond roughly 1,000 concurrent
connections per instance, scale out rather than up — extra vCPUs do not help as
much as extra instances.

The [Health](../guide/health.md) view reports file descriptor use, which is
usually the first ceiling a busy proxy reaches.

## Egress addresses

Each proxy has its own outbound address, so a destination that allowlists your
egress IP needs **all** of them. Plan for this before adding a proxy.

| Cloud | Recommended |
|---|---|
| AWS | A NAT gateway per availability zone |
| Azure | A NAT gateway on the proxy subnet |
| Google Cloud | Cloud NAT with reserved external addresses |

Routing egress through a NAT gateway keeps the list short and stable as you
scale, rather than changing every time an instance is replaced.

## What is not covered

- **Automatic configuration sync** — on the roadmap
- **Central management of several appliances** — on the roadmap
- **A shared cache between proxies** — Squid supports cache peering, but with
  most traffic encrypted the benefit rarely justifies the complexity. See
  [HTTPS and CONNECT](https-and-connect.md)
