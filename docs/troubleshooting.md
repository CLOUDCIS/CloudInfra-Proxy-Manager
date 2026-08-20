# Troubleshooting

Work down this page in order. The first three checks resolve most cases.

## 1. Is the proxy running?

Open the console and look at [Health](guide/health.md). It reports the proxy
service, the cache directory, disk and memory, and explains each deduction.

From a shell:

```bash
systemctl status squid --no-pager
sudo ss -lntp | grep 3128
```

If Squid is not running:

```bash
sudo squid -k parse                    # syntax check - silent if the file is good
sudo tail -50 /var/log/squid/cache.log
sudo journalctl -u squid -n 50 --no-pager
```

A configuration error is the usual cause, and `squid -k parse` names the file
and line. If you have just applied a change from the console and the proxy did
not come up, it will already have rolled itself back — see
[Applying Changes](guide/applying-changes.md).

## 2. Can the proxy reach the internet?

```bash
curl -I https://example.com
```

If this fails, the problem is the instance's own connectivity — routing, NAT,
firewall — and not the proxy. See
[AWS](getting-started/networking-aws.md) ·
[Azure](getting-started/networking-azure.md) ·
[Google Cloud](getting-started/networking-gcp.md).

## 3. Can the client reach the port?

From the client:

```bash
nc -zv 10.0.1.20 3128
```

| Result | Meaning |
|---|---|
| `succeeded` | The network path is fine — the problem is above the network layer |
| `timed out` | Blocked by a security group, NSG or firewall rule |
| `refused` | Reached the host, but nothing is listening — back to step 1 |

## Reading the result codes

[Logs](guide/logs.md) and [Live Traffic](guide/live-traffic.md) show these in the
console. The result code is the field that tells you what happened.

| Code | Meaning |
|---|---|
| `TCP_MISS/200` | Fetched from the origin. Normal |
| `TCP_HIT/200` | Served from cache. Normal |
| `TCP_TUNNEL/200` | An HTTPS tunnel, logged when it closed. Normal |
| `TCP_DENIED/403` | **Blocked by an access rule** |
| `TCP_DENIED/407` | Authentication required, none supplied |
| `TCP_MISS/503` | Could not reach the destination |
| `TCP_MISS/504` | The destination timed out |

If a request does not appear at all, it never reached the proxy. Back to step 3.

## Common problems

### A client gets 403 Forbidden

Its address is not in the configured client networks. This is the most common
single cause, and it is what you get when a subnet was added to the cloud
firewall but not to the appliance.

Find the client address in the traffic log, then check it against **Proxy
Settings → client networks**. An IPv6 client address here almost always means
[IPv6](squid/ipv6.md).

### HTTPS does not work but HTTP does

Nearly always the client. Check that `https_proxy` is an `http://` URL:

```bash
export https_proxy="http://10.0.1.20:3128"      # correct
export https_proxy="https://10.0.1.20:3128"     # wrong - will fail
```

The scheme describes how the client talks to the proxy, not what the proxy
fetches. See [HTTPS and CONNECT](squid/https-and-connect.md).

### An access rule has no effect

Check in this order:

1. **Has it been applied?** Rules do nothing until you apply them — see
   [Applying Changes](guide/applying-changes.md).
2. **Is an earlier rule matching first?** Evaluation stops at the first match.
   [Access Rules](guide/access-rules.md) shows the order.
3. **Is the rule in `local.conf`?** Access rules do not work there. See
   [Configuring Squid Directly](squid/index.md).
4. **Does the domain have a leading dot?** `example.com` matches only that exact
   name; `.example.com` matches subdomains too.

### An HTTPS site cannot be blocked by path

Expected, and not fixable by configuration. The path is inside the encrypted
session and never sent to the proxy. See
[HTTPS and CONNECT](squid/https-and-connect.md#what-this-means-for-filtering).

### An authentication helper appears not to respond

Check the path. Older Squid packages used `/usr/lib/squid3/`, which appears in
many tutorials but does not exist on this image.

```bash
ls /usr/lib/squid/
```

Running a program that is not there produces silence, which looks exactly like a
backend timeout. See [Authentication Backends](squid/authentication-backends.md).

### Everyone is prompted to sign in, and nothing works

The rule requiring authentication is being evaluated after a deny. On a
console-managed appliance, check the rule order in
[Access Rules](guide/access-rules.md).

### An instance loses its cloud identity

Metadata requests are being sent through the proxy. Add the metadata address to
`no_proxy` on every client:

```bash
export no_proxy="localhost,127.0.0.1,169.254.169.254,.internal"
```

This breaks IAM roles on AWS, managed identities on Azure and service accounts
on Google Cloud, usually some time after the change that caused it.

### The disk is filling up

[Health](guide/health.md) reports disk use and will warn before it becomes a
problem.

```bash
df -h /var
du -sh /var/log/squid /var/spool/squid
```

Logs rotate daily with 14 days retained. If they are not rotating:

```bash
sudo logrotate --debug /etc/logrotate.d/squid
```

Traffic data retention is separate and configurable — see
[Administration](guide/administration.md).

### The proxy is slow

Squid handles requests on a single thread, so a saturated core is a real limit
and extra vCPUs will not help. File descriptor exhaustion is the other common
ceiling; [Health](guide/health.md) reports both.

Check DNS as well — slow resolution looks exactly like a slow proxy:

```bash
time nslookup example.com
```

See [High Availability](squid/high-availability.md) for scaling out.

## Useful commands

```bash
sudo squid -k parse                    # validate configuration
sudo squid -k reconfigure              # apply without dropping connections
sudo squid -k rotate                   # rotate logs now
squid -v | head -1                     # version and build options

sudo tail -f /var/log/squid/access.log
sudo tail -f /var/log/squid/cache.log  # Squid's own diagnostics
```

### Reading Cache Manager by hand

Cache Manager carries the runtime statistics — connection counts, hit ratios and
service times — that the [Dashboard](guide/dashboard.md) and
[Health](guide/health.md) are built on.

!!! warning "`squidclient` is not on this image"
    It was removed in Squid 7. Tutorials that reach for `squidclient mgr:info`
    predate that; use `curl` through the proxy instead.

```bash
curl -s -x 127.0.0.1:3128 \
  "http://$(awk '$1=="visible_hostname"{print $2}' /etc/squid/squid.conf):3128/squid-internal-mgr/info"
```

The URL has to name the proxy's own `visible_hostname` and port. Squid treats a
`/squid-internal-mgr/` path as a management request only when it is addressed to
*this* proxy. Ask for `http://127.0.0.1/squid-internal-mgr/info` instead and the
request is allowed, then forwarded as an ordinary request to `127.0.0.1:80`,
where nothing is listening — so you get a Squid error page and it looks like a
broken cache manager.

Replace `info` with `active_requests`, `counters` or `5min` for other reports.

## What to send us

```bash
squid -v | head -1
sudo squid -k parse
systemctl status squid --no-pager
sudo tail -50 /var/log/squid/cache.log
```

Plus the client's IP address and what you expected to happen. That set answers
most of the first round of questions. See [Support](support.md).
