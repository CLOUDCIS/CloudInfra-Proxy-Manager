# IPv6

The proxy supports IPv6 on both sides — it accepts connections from IPv6 clients
and reaches IPv6 destinations. One default needs changing first, and it is the
most common cause of an IPv6 client being denied on an otherwise working proxy.

## The default that catches people out

The listener is not the problem: `http_port 3128` accepts IPv4 and IPv6 already.
The **client network list** is.

The appliance allows these IPv6 ranges out of the box:

| Range | What it is |
|---|---|
| `fc00::/7` | Unique local addresses — the IPv6 equivalent of RFC1918 |
| `fe80::/10` | Link-local addresses |

Neither covers **global unicast**, which is what the public internet uses — and,
importantly, what the major clouds assign to instances. AWS gives instances
addresses from `2600:1f00::/24`. Azure and Google Cloud assign from their own
global ranges.

So an IPv6-enabled client matches none of the allowed ranges, falls through to
the default deny, and is refused:

```
1755678901.234  0 2600:1f18:abcd:1234::5 TCP_DENIED/403 ...
```

!!! tip "How to recognise it"
    A `TCP_DENIED/403` with an IPv6 client address, on a proxy where the
    equivalent IPv4 client works, is almost always this.

## Fixing it

Find the IPv6 range your clients use — the VPC, VNet or subnet prefix your cloud
assigned, not an individual address — and add it as a client network in
**Proxy Settings**. Both address families are accepted in the same place, and
the change goes through the usual validate-and-apply pipeline.

Use the allocated prefix rather than a host address. AWS allocates a `/56` to a
VPC and a `/64` to each subnet; adding a single `/128` works for one machine and
puzzles you when the next one fails.

### Finding your range

```bash
# AWS
aws ec2 describe-vpcs --vpc-ids vpc-xxxx \
  --query 'Vpcs[].Ipv6CidrBlockAssociationSet[].Ipv6CidrBlock'

# Azure
az network vnet show -g myGroup -n myVnet --query addressSpace.addressPrefixes

# Google Cloud
gcloud compute networks subnets describe my-subnet --region us-east1 \
  --format='value(ipv6CidrRange)'
```

Or from a client itself:

```bash
ip -6 addr show scope global
```

## Egress

The proxy also needs its own route to the IPv6 internet. This is cloud
configuration rather than proxy configuration.

=== "AWS"

    Attach an **egress-only internet gateway** to the VPC and route `::/0` to it
    from the proxy's subnet. Egress-only permits outbound connections and their
    replies while blocking inbound, which is the right shape for a proxy that
    should not be reachable from the internet.

    See [AWS networking](../getting-started/networking-aws.md).

=== "Azure"

    Assign the instance an IPv6 address and route `::/0` appropriately for your
    topology.

    See [Azure networking](../getting-started/networking-azure.md).

=== "Google Cloud"

    The subnet must be dual-stack with external IPv6 access configured.

    See [Google Cloud networking](../getting-started/networking-gcp.md).

Verify from the proxy itself:

```bash
curl -6 -I https://ipv6.google.com
```

If that fails, the problem is cloud routing rather than the proxy.

## Address family preference

These go in `/etc/squid/local.conf`. Neither is order-sensitive, so both work
there — see [Configuring Squid Directly](index.md).

Prefer IPv4 where DNS returns both, which is useful while testing or where IPv6
egress is unreliable:

```
dns_v4_first on
```

Firewall and security group rules are **separate per address family** in all
three clouds. Allowing 3128 from an IPv4 range does not allow it from an IPv6
one — another common cause of a client that cannot connect at all.

## Logging

IPv6 addresses appear in the traffic log in full, uncompressed form. When
searching for a client, match on a prefix rather than the exact string your own
tooling shows you, which may have abbreviated it.

## Checklist

1. `curl -6 -I https://ipv6.google.com` from the proxy — proves egress
2. Client's IPv6 prefix added as a client network — proves policy
3. Security group or firewall allows 3128 over IPv6 — proves the network path
4. `curl -6 -x http://[2600:...]:3128 -I https://example.com` from the client
5. Check the traffic log for `TCP_DENIED/403`
