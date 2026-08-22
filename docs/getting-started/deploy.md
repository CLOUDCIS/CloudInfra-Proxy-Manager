# Deploy the Image

CloudInfra Proxy Manager ships as a pre-built marketplace image. There is
nothing to install: launch it, open the console, and the proxy is already
running.

[Deploy Solution :material-cloud-download:](https://cloudinfrastructureservices.co.uk/squid-proxy-server/){ .md-button .md-button--primary target="_blank" rel="noopener" }

The product page carries the marketplace links for each cloud. The rest of this
page is what to settle before you launch — instance size, where to put it, and
which ports to open — and what the appliance does for itself on first boot.

## What you are launching

| | |
|---|---|
| **Operating system** | Ubuntu 24.04 LTS |
| **Proxy engine** | Squid 7, built from source |
| **Management console** | HTTPS on port 8443 |
| **Proxy service** | HTTP on port 3128 |

## Sizing

Squid is far more sensitive to memory and disk than to CPU, and the management
layer is capped so it can never take resources from the proxy.

| Users | Instance | Disk | Notes |
|---|---|---|---|
| Up to 100 | 2 vCPU, 4 GB | 30 GB | Comfortable for a small office |
| 100–500 | 2 vCPU, 8 GB | 60 GB | The common case |
| 500–2000 | 4 vCPU, 16 GB | 100 GB+ | Raise the cache size to match |

Disk matters more than it looks. The volume holds the object cache, Squid's own
logs and the traffic history the console reports on. All three are bounded — the
cache by its configured size, logs by rotation, analytics by a retention window
and a hard size cap — but they need room to reach those bounds.

!!! tip "Start smaller than you think"
    The console reports actual memory, disk and cache usage on the
    [Health](../guide/health.md) page. It is easier to resize up after a week of
    real traffic than to guess correctly now.

## Network placement

The proxy needs to be reachable by your clients, and it needs to reach the
internet on their behalf. In practice:

- Put it in a **private subnet** with a NAT gateway, or a public subnet with an
  Elastic IP — either works.
- Clients reach it on **port 3128**, so they must have a route to it.
- It reaches the internet on **80** and **443** outbound.

## Security groups

This is the part worth getting right, because the defaults are deliberately
permissive about *where the service listens* and rely on you to restrict *who
can reach it*.

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 3128 | TCP | Your client subnets | Proxy traffic |
| 8443 | TCP | Your administrators only | Management console |
| 22 | TCP | Your administrators only, or nothing | SSH — not needed for normal use |

!!! danger "Do not expose port 8443 to the internet"
    The console binds to all interfaces so that it is reachable from your
    network without an SSH tunnel on first launch. That means the security
    group is the only thing restricting access to it.

    Restrict 8443 to the addresses your administrators actually come from. The
    console defends itself — HTTPS, Argon2id password hashing, CSRF protection,
    rate limiting and account lockout — but none of that is a reason to publish
    it.

Port 22 is not required. Everything you need is in the console, including
[reading the logs](../guide/logs.md). If you never open it, that is one fewer
thing to secure.

[:material-arrow-right: Full port and security group reference](../reference/ports.md)

## What happens at first boot

The image is sealed with no credentials in it, so the first boot generates what
this instance needs:

1. **A TLS certificate** for the console, generated on the instance. It is
   self-signed, so your browser warns once — see
   [Your First 10 Minutes](quickstart.md) for how to confirm you are talking to
   the right appliance.
2. **An administrator password**, random and unique to this instance. It is
   written to the instance console output, which you can read through your
   cloud provider without SSH.
3. **Your client network**, detected from the cloud metadata service so the
   proxy accepts traffic from your VPC rather than from a range baked in at
   build time.

None of this exists in the image. Two instances launched from the same image
have different certificates and different passwords.

!!! info "Give it a minute"
    First boot generates a certificate and creates the administrator account
    before the console starts listening. On a small instance this takes under a
    minute. If the console does not answer immediately, wait rather than
    relaunching.

## Next

[:material-arrow-right: Your first 10 minutes](quickstart.md)
