# Ports & Security Groups

## Inbound

| Port | Protocol | Who needs it | Notes |
|---|---|---|---|
| **3128** | TCP | Your client subnets | Proxy traffic |
| **8443** | TCP | Your administrators | Management console (HTTPS) |
| **22** | TCP | Nobody, ideally | SSH — not required for normal use |

### Port 3128 — the proxy

Restrict to the networks your clients are actually on. An open proxy is found and
abused within hours of being reachable from the internet.

Note this is defence in depth rather than the only control: the appliance also
denies requests from outside its configured client network at the policy level.
Both should be right.

### Port 8443 — the console

!!! danger "Never expose this to the internet"
    The console binds to all interfaces so it is reachable from your network
    without an SSH tunnel on first launch. **The security group is the only
    thing restricting who can reach it.**

    Restrict it to the addresses your administrators come from — a bastion
    subnet, a VPN range, or your office egress addresses.

The console defends itself: HTTPS only, Argon2id password hashing, `__Host-`
prefixed session cookies, CSRF tokens, rate limiting and account lockout. None
of that is a reason to publish it.

### Port 22 — SSH

Not required. Everything needed for normal administration is in the console,
including [reading the logs](../guide/logs.md), which is the usual reason people
reach for SSH.

Leaving it closed removes an entire class of exposure. Open it only if you need
it, and only from where you need it.

## Outbound

| Destination | Port | Purpose | Required? |
|---|---|---|---|
| Anywhere | 80, 443 | Proxying your clients' traffic | Yes |
| Your DNS resolvers | 53 | Name resolution | Yes |
| `endpoints.office.com` | 443 | [Microsoft 365 endpoint lists](../guide/microsoft-365.md) | Only if enabled |
| Your directory servers | 636 or 389 | [Directory authentication](../guide/directory-authentication.md) | Only if enabled |
| Package repositories | 80, 443 | Operating system updates | Recommended |

The appliance makes **no scheduled outbound connections by default**. The
Microsoft 365 feed is the only one, and only when you enable it.

!!! info "The one diagnostic query"
    The [health page](../guide/health.md) resolves `example.com` to check DNS.
    There is no way to test name resolution without sending a query, and
    `example.com` is reserved by IANA for exactly this purpose — served by root
    infrastructure rather than by a company, so it neither advertises your
    appliance to a vendor nor depends on a commercial service.

    No telemetry, licence check or phone-home exists in this product.

## Internal

Not exposed, listed for completeness:

| Interface | Purpose |
|---|---|
| `/run/cloudinfra/priv.sock` | Console to privileged helper. Unix socket, mode 0660, owner `root:cloudinfra` |
| `127.0.0.1:3128` | The console reads Squid's Cache Manager over loopback |

## Example security group

An appliance serving a VPC on `10.20.0.0/16`, administered from a bastion subnet
on `10.20.99.0/24`:

**Inbound**

| Type | Port | Source |
|---|---|---|
| Custom TCP | 3128 | `10.20.0.0/16` |
| HTTPS | 8443 | `10.20.99.0/24` |

**Outbound**

| Type | Port | Destination |
|---|---|---|
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |
| DNS | 53 | Your resolvers, or `0.0.0.0/0` |

## Changing the proxy port

Set it in [Proxy Settings](../guide/proxy-settings.md). Two things to do first:

1. Allow the new port in the security group.
2. Reconfigure every client.

Changing the port requires a **restart**, which drops every active connection,
and the console tells you so before you apply.

!!! warning "The rollback cannot help you here"
    If you move the port without updating the security group, the proxy starts
    correctly and passes every health check — it simply has nobody able to reach
    it. From the appliance's point of view that is a complete success.

## Related

- [Deploy the image](../getting-started/deploy.md) — placement and sizing
- [Pointing clients at the proxy](../getting-started/clients.md) — the client side
- [Security model](../about/security-model.md) — what is protecting what
