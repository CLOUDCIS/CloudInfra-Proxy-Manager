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

=== "Windows"

    ```powershell
    Test-NetConnection 10.0.1.20 -Port 3128
    ```

    Read `TcpTestSucceeded` in the output.

=== "Linux, macOS"

    ```bash
    nc -zv 10.0.1.20 3128
    ```

| Result | Meaning |
|---|---|
| `succeeded` | The network path is fine — the problem is above the network layer |
| `timed out` | Blocked by a security group, NSG or firewall rule |
| `refused` | Reached the host, but nothing is listening — back to step 1 |

!!! warning "In PowerShell, write `curl.exe` rather than `curl`"
    Every `curl` command on this page works on Windows, but PowerShell aliases
    `curl` to `Invoke-WebRequest`, which does not understand `-x` and fails with
    a parameter error rather than a proxy error. Diagnosing a proxy with the
    wrong tool wastes a lot of time, because the failure looks like the thing
    you are investigating.

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

### Every client gets 403, and the client networks look right

A different fault, and the giveaway is *every*. One client being refused is
usually its address; the whole network being refused on an appliance that was
working is not.

It affects appliances built before this was fixed, and only after you have
applied a change from the console. The image ships with the client-network
allow in place, so a newly launched appliance proxies correctly — it is the
first apply, replacing that configuration, that removes it.

Check whether the generated configuration permits your networks:

```bash
sudo grep -c 'http_access allow localnet' /etc/squid/squid.conf
```

`1` means this is not your problem. `0` means the configuration defines your
client networks but never permits them, so every request falls through to the
default deny.

Restore access with the override file, which is never generated and never
overwritten:

```bash
echo 'http_access allow localnet' | sudo tee -a /etc/squid/local.conf
sudo squid -k parse && sudo squid -k reconfigure
```

That survives later applies, which is what makes it a usable stopgap. Upgrade
when a newer image is available: the fix is in the configuration generator.

!!! danger "Remove the override once you have upgraded"
    It is not harmless to leave behind. `local.conf` is included *above* the
    generated access rules, so this line allows the whole client network before
    any of them are considered — including the rule that requires users to sign
    in. On an upgraded appliance with directory authentication switched on, an
    override left in place lets every client through **without credentials**,
    and nothing in the console shows it.

    ```bash
    sudo sed -i '/^http_access allow localnet$/d' /etc/squid/local.conf
    sudo squid -k parse && sudo squid -k reconfigure
    ```

    Then confirm the generated configuration carries the allow itself:

    ```bash
    grep -c 'http_access allow localnet' /etc/squid/squid.conf   # expect 1
    ```

!!! note "Why this was not obvious sooner"
    The same appliances have a second fault that conceals the first: the
    post-apply health probe cannot work out the appliance's own address, so
    every apply fails verification and reverts. While that is happening the
    broken configuration never reaches a running proxy, and what people report
    is "my rule did not take effect" rather than a denial.

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

### An apply is refused before anything changes

The **Validate** step runs Squid's own parser over the staged configuration, so
a refusal here means the proxy is untouched and still serving the previous
configuration. Nothing is broken and nothing needs restoring.

Read the message: where the fault comes from one of your rules it names that
rule, so go to it, correct it, and apply again. Until it is corrected no other
change can be applied either, because every apply stages the whole
configuration — so fix it rather than working around it.

See [Applying Changes](guide/applying-changes.md) for what each step does.

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

### Traffic stopped appearing, and the log files are empty

Check whether the logs rotated recently:

```bash
sudo ls -l /var/log/squid/
```

If `access.log` is zero bytes while `access.log.1` is large and both share a
timestamp, the files rotated but Squid did not reopen them — it is still writing
into the renamed file, which nothing reads. The console shows no traffic, and
the disk keeps filling behind a log that looks rotated.

```bash
sudo squid -k rotate
```

That reattaches it immediately, and traffic reappears. On images built before
21 August 2026 this recurs at every rotation; a scheduled `squid -k rotate` is a
reasonable stopgap until the appliance is replaced.

### `NONE_NONE/000` entries from 127.0.0.1 in the access log

```
1787313003.942  0 127.0.0.1 NONE_NONE/000 0 - error:transaction-end-before-headers
```

This is the appliance checking itself. The health check opens a connection to
the proxy port to confirm something is listening, then closes it without sending
a request, and Squid logs that like any other transaction.

Harmless, and expected at the health polling interval. Alongside it you will see
successful `GET` requests to `squid-internal-mgr/info`, which is the console
reading Cache Manager for the connection counts the access log cannot provide.

Both stay in the native access log deliberately — when diagnosing the proxy they
are exactly what you want to see — and neither is counted as customer traffic in
the console's own analytics.

### `WARNING: Ignoring 172.31.0.0/16 because it is already covered`

Seen in `cache.log` on every start and reload, on AWS instances in a default
VPC. It is Squid noting that first boot added your VPC's range to the client
list when the shipped RFC1918 baseline already covered it.

Harmless: the range is permitted either way, and detection deliberately widens
the list rather than replacing it, so that an unreachable metadata service
degrades to a working proxy rather than one that denies everyone.

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
