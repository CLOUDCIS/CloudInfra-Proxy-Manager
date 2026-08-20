# Azure Networking

Most support cases we see are network configuration rather than the proxy. This
page covers the Azure side of a working deployment.

For the ports themselves, see [Ports & Security Groups](../reference/ports.md).

## Does the proxy interfere with inbound traffic?

**No.** This is asked often enough to answer first.

This is a *forward* proxy. It only handles traffic that a client has been
configured to send it. It is not in the path of anything else.

If you run an Application Gateway, Front Door or a Load Balancer in front of web
servers, that inbound traffic reaches them exactly as it does today. The two
paths are independent:

```
internet ──► Application Gateway ──► web servers      (inbound, untouched)

web servers ──► proxy ──► internet                     (outbound, filtered)
```

You do not need to exclude the Application Gateway from anything, or change your
inbound NSG rules. Configure the proxy for outbound traffic and leave the
inbound path alone.

The one thing to check is that where a web server is *also* an outbound client
of the proxy, its `no_proxy` list excludes what it must reach directly —
internal services, Azure metadata, and any Azure API endpoints it uses.

## Placement

Put the appliance in its own subnet within the VNet your clients use, and reach
it over its private address.

The appliance does not need a public IP address to give clients internet access.

## Network security groups

Inbound, on the appliance's subnet or NIC:

| Priority | Port | Source | Action |
|---|---|---|---|
| 100 | 3128 | Your client subnet prefixes | Allow |
| 110 | 8443 | Your administrator addresses only | Allow |
| 120 | 22 | Your administrator addresses, if at all | Allow |
| 4096 | Any | Any | Deny |

Outbound: 80 and 443 to the `Internet` service tag, plus TCP 389 or 636 to your
domain controllers if
[directory authentication](../guide/directory-authentication.md) is configured.

On the **client** subnets, allow outbound TCP 3128 to the appliance's subnet
prefix.

Service tags are convenient but blunt: `VirtualNetwork` as a source allows every
address in the VNet. Name your client prefixes explicitly if the proxy should be
restricted to a subset.

!!! danger "Never allow 3128 from the Internet service tag"
    An open proxy is found by scanners within hours and used to relay traffic
    attributed to your subscription.

## Azure Firewall alongside the proxy

If you also run Azure Firewall, decide which does what rather than filtering
twice in the same path. A common split is Azure Firewall for network-layer
control and non-HTTP protocols, and this appliance for HTTP and HTTPS policy,
user attribution and request visibility.

Make sure the firewall permits the appliance's own outbound 80 and 443.

## Egress addresses

Destinations that allowlist your IP need the appliance's outbound address, not
the VM's private one.

Attach a **NAT gateway** to the appliance's subnet with a static public IP.
Without one, Azure uses default outbound access, which gives no stable address
and is being retired.

```bash
az network public-ip show -g myGroup -n proxy-nat-ip --query ipAddress -o tsv
```

## User-defined routes

For this deployment model you normally need **no** UDRs. Clients address the
proxy directly, so ordinary VNet routing applies.

UDRs are only relevant if you want traffic forced through the proxy without
configuring clients — see the [roadmap](../about/roadmap.md#transparent-interception).

## Peered VNets and hub-and-spoke

Clients in peered VNets can use the appliance if:

1. peering is established both ways with forwarded traffic allowed
2. NSGs permit 3128 between the address spaces
3. the remote address spaces are configured as **client networks** on the
   appliance

The third is easy to miss. Peering and NSGs can both be correct while the proxy
still denies the request — check the traffic log for `TCP_DENIED/403`.

## Metadata service

Clients must reach `169.254.169.254` **directly**, never through the proxy.
Proxying metadata breaks managed identities and several agents.

```bash
export no_proxy="localhost,127.0.0.1,169.254.169.254,.internal.cloudapp.net"
```

## Private endpoints

Where you use private endpoints for Storage, Key Vault or SQL, exclude those
hostnames from the proxy so traffic stays on the Azure backbone rather than
taking a round trip out and back.

## IPv6

Add the VNet's IPv6 address space as a client network on the appliance. It is
not automatic, and it is the usual reason IPv6 clients are denied. See
[IPv6](../squid/ipv6.md).

NSG rules are separate per address family.

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
| Connection times out | NSG rules, peering, routing |
| Connection refused | The proxy is not running — check [Health](../guide/health.md) |
| 403 from the proxy | Client prefix missing from the client networks |
| Works for IPv4, not IPv6 | IPv6 prefix missing, or a v6 NSG rule |
| Managed identity stops working | Metadata is being proxied — fix `no_proxy` |

More in [Troubleshooting](../troubleshooting.md).
