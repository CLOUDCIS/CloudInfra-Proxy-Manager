# AWS Networking

Most support cases we see are network configuration rather than the proxy. This
page covers the AWS side of a working deployment.

For the ports themselves, see [Ports & Security Groups](../reference/ports.md).

## Placement

Put the appliance in a **private subnet** with a route to the internet through a
NAT gateway, and reach it from clients over their private addresses.

```
        clients (private subnets)
                 │
        appliance :3128
                 │
           NAT gateway
                 │
        internet gateway
```

The appliance does not need a public IP address to give clients internet access.
A public IP means the internet can reach port 3128 and the console on 8443,
which is exactly what you do not want.

If you must place it in a public subnet, restrict the security group to your own
CIDR ranges and never to `0.0.0.0/0`.

## Security groups

Inbound to the appliance:

| Type | Port | Source |
|---|---|---|
| Custom TCP | 3128 | Your client subnet CIDRs |
| Custom TCP | 8443 | Your administrator addresses only |
| SSH | 22 | Your administrator addresses, if at all |

Outbound: 80 and 443 to `0.0.0.0/0`, plus anything your clients legitimately
need. If [directory authentication](../guide/directory-authentication.md) is
configured, also allow TCP 389 or 636 to your domain controllers.

On the **clients**, allow outbound TCP 3128 to the appliance's security group.
Referencing the security group rather than a CIDR keeps the rule correct as
addresses change.

!!! danger "Never open 3128 to 0.0.0.0/0"
    An open proxy is found by scanners within hours and used to relay traffic
    attributed to your account. The appliance also denies requests from outside
    its configured client networks at the policy level, but both controls should
    be right.

## Network ACLs

Default network ACLs allow everything and need no change. If you have replaced
them, remember they are **stateless**: allowing 3128 inbound is not enough, you
must also allow the ephemeral range `1024–65535` outbound for replies.

## Egress addresses

Destinations that allowlist your IP need the appliance's outbound address, which
is the NAT gateway's Elastic IP — not the instance's private address.

```bash
aws ec2 describe-nat-gateways \
  --filter Name=vpc-id,Values=vpc-xxxx \
  --query 'NatGateways[].NatGatewayAddresses[].PublicIp'
```

A NAT gateway per availability zone gives a small, stable set of addresses to
hand out. See [High Availability](../squid/high-availability.md) when running
more than one appliance.

## Source/destination checking

**You do not need to disable this.** It only matters when an instance forwards
packets that are not addressed to it — a NAT instance or a transparent proxy.
The appliance terminates the client connection and opens its own, so the check
never applies.

If you are following a guide that says to disable it, that guide is describing
transparent interception, which is a different deployment model. See the
[roadmap](../about/roadmap.md#transparent-interception).

## Metadata service

Clients must reach `169.254.169.254` **directly**, never through the proxy.
Proxying metadata breaks instance identity, IAM role credentials, SSM and the
CloudWatch agent — usually some time after the change that caused it.

```bash
export no_proxy="localhost,127.0.0.1,169.254.169.254,.internal,.amazonaws.com"
```

Including `.amazonaws.com` sends AWS API traffic direct as well, which is
normally what you want — especially with VPC endpoints, where a proxied request
would leave the AWS network and come back.

## Transit Gateway and peering

Clients in other VPCs can use the appliance if:

1. routes exist in both directions
2. the appliance's security group allows the remote CIDRs
3. the remote CIDRs are configured as **client networks** on the appliance

The third is easy to miss. Routing and security groups can both be correct while
the proxy still denies the request — check the traffic log for
`TCP_DENIED/403`.

## IPv6

Two things are needed, and neither is automatic:

**Egress** — attach an egress-only internet gateway and route `::/0` to it from
the appliance's subnet.

**Client networks** — add your VPC's IPv6 CIDR on the appliance. This is the
usual reason IPv6 clients are denied. See [IPv6](../squid/ipv6.md).

Security group rules are separate per address family.

## Checking it works

```bash
# From a client - is the port reachable?
nc -zv 10.0.1.20 3128

# Does the proxy work?
curl -x http://10.0.1.20:3128 -I https://example.com

# From the appliance - does it have internet access?
curl -I https://example.com
```

!!! tip "On Windows, run `curl.exe`"
    PowerShell aliases the bare name `curl` to `Invoke-WebRequest`, which
    does not understand `-x` and fails with a parameter error rather than a
    proxy error. See [Confirming it works](clients.md#confirming-it-works).

| Symptom | Look at |
|---|---|
| Connection times out | Security group, network ACL, routing |
| Connection refused | The proxy is not running — check [Health](../guide/health.md) |
| 403 from the proxy | Client CIDR missing from the client networks |
| Works for IPv4, not IPv6 | IPv6 CIDR missing, or a v6 security group rule |
| Instance loses its IAM role | Metadata is being proxied — fix `no_proxy` |

More in [Troubleshooting](../troubleshooting.md).
