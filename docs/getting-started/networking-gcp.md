# Google Cloud Networking

Most support cases we see are network configuration rather than the proxy. This
page covers the Google Cloud side of a working deployment.

For the ports themselves, see [Ports & Security Groups](../reference/ports.md).

## Do I need `--can-ip-forward`?

**No — not for this deployment model.** This trips people up often enough to
answer first, because a deployment that fails to set it is usually not the
actual problem.

IP forwarding lets an instance handle packets addressed to somewhere else. That
is what a NAT instance or a *transparent* proxy does. This appliance does not
forward packets: the client opens a connection **to the proxy**, and the proxy
opens its own separate connection onward. Every packet is addressed to a machine
that wants it.

So if you cannot enable `canIpForward`, or your deployment template does not
expose it, proceed without it and configure clients as described in
[Pointing Clients at the Proxy](clients.md).

!!! note "It cannot be changed later"
    `canIpForward` can only be set when an instance is **created**. If you
    genuinely need it, the instance has to be recreated — another reason to be
    sure you need it first.

## Placement

Put the appliance in a subnet your clients can reach, and reach it over its
internal address.

With Cloud NAT configured for its subnet, an instance with no external address
still reaches the internet — and cannot be reached from it. That is the
arrangement to aim for.

## Firewall rules

Google Cloud firewall rules apply by network tag, which is more maintainable
than address ranges. Tag the appliance `squid-proxy`.

```bash
gcloud compute firewall-rules create allow-proxy-clients \
    --network=my-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:3128 \
    --source-ranges=10.0.0.0/8 \
    --target-tags=squid-proxy
```

Replace the source range with your actual client subnets. Add a separate rule
for TCP 8443 restricted to your administrator addresses, and one for TCP 22 only
if you need it.

Outbound, the default egress rule allows everything. If you have replaced it,
allow TCP 80 and 443, plus 389 or 636 to your domain controllers where
[directory authentication](../guide/directory-authentication.md) is configured.

!!! danger "Never set --source-ranges=0.0.0.0/0 on the 3128 rule"
    An open proxy is found by scanners within hours and used to relay traffic
    attributed to your project.

## Egress addresses

Destinations that allowlist your IP need Cloud NAT's address, not the instance's
internal one. Reserve a static address so it does not change:

```bash
gcloud compute addresses create proxy-nat-ip --region=us-east1

gcloud compute routers nats create proxy-nat \
    --router=my-router \
    --region=us-east1 \
    --nat-external-ip-pool=proxy-nat-ip \
    --nat-custom-subnet-ip-ranges=my-subnet
```

## Shared VPC

Where the appliance runs in a service project attached to a host project:

- firewall rules are defined in the **host** project
- the service project needs `compute.networkUser` on the subnet
- clients in other service projects reach the appliance over the shared network

As always, also add those client ranges as **client networks** on the appliance.
Firewall rules being correct does not mean the proxy will accept the request.

## Private Service Connect

You can publish the appliance as a PSC service so consumers reach it without VPC
peering. One consequence is worth knowing before you build on it.

!!! warning "PSC hides the original client address"
    Connections arrive translated, so the proxy sees the PSC endpoint's address
    rather than the client's. Every entry in the traffic log shows the same
    source, [Top Clients](../guide/analytics.md) becomes meaningless, and
    client-based access rules cannot distinguish consumers.

If you need real client addresses through PSC, enable **PROXY protocol** on the
service attachment and configure Squid to trust it, in
`/etc/squid/local.conf`:

```
http_port 3128 require-proxy-header
proxy_protocol_access allow localnet
```

Both ends must agree. A proxy expecting the header when the attachment is not
sending it will reject every connection, and the reverse fails too. **We do not
currently test this configuration** — treat it as a starting point, verify it in
a lab, and [tell us](../support.md) how it goes.

## Metadata service

Clients must reach `metadata.google.internal` and `169.254.169.254`
**directly**, never through the proxy. Proxying metadata breaks service account
credentials and several agents.

```bash
export no_proxy="localhost,127.0.0.1,169.254.169.254,metadata.google.internal,.internal"
```

## Private Google Access

Where Private Google Access or PSC endpoints for Google APIs are configured,
exclude `.googleapis.com` from the proxy so that traffic stays on Google's
network rather than taking a round trip out and back.

## IPv6

The subnet must be dual-stack with external IPv6 access configured for egress.
Then add the subnet's IPv6 range as a client network on the appliance — it is
not automatic, and it is the usual reason IPv6 clients are denied. See
[IPv6](../squid/ipv6.md).

Firewall rules are separate per address family.

## Checking it works

```bash
# From a client - is the port reachable?
nc -zv 10.0.1.20 3128

# Does the proxy work?
curl -x http://10.0.1.20:3128 -I https://example.com

# From the appliance - does it have internet access?
curl -I https://example.com
```

| Symptom | Look at |
|---|---|
| Connection times out | Firewall rules, network tags, routing |
| Connection refused | The proxy is not running — check [Health](../guide/health.md) |
| 403 from the proxy | Client range missing from the client networks |
| All traffic shows one client address | Private Service Connect — see above |
| Works for IPv4, not IPv6 | IPv6 range missing, or a v6 firewall rule |
| Service account credentials fail | Metadata is being proxied — fix `no_proxy` |

More in [Troubleshooting](../troubleshooting.md).
