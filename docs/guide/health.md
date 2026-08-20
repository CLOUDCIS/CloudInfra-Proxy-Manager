# Health

The Health page answers one question — *is this appliance working?* — and, when
the answer is no, says exactly what is wrong and what to do about it.

## The score

A number out of 100. What makes it useful is that it is **arithmetic you can
follow**: fourteen named checks, each with a fixed published weight. Every
deduction is shown against the check that caused it, so the score can always be
accounted for.

| Band | Meaning |
|---|---|
| 90–100 | <span class="pill-allow">Healthy</span> |
| 70–89 | <span class="pill-neutral">Warning</span> |
| Below 70 | <span class="pill-block">Critical</span> |

One rule overrides the arithmetic: **any critical check makes the whole
appliance critical**, regardless of the score. A stopped proxy must never
present as a warning because the numbers happened to land above a threshold.

## The checks

Grouped as the console groups them, with the most a check can deduct.

### Proxy

| Check | Weight | Critical? |
|---|---|---|
| Proxy service running | 100 | Yes |
| Proxy port listening | 40 | Yes |
| Configuration valid | 25 | Yes |
| Cache directory writable | 20 | Yes |
| Last configuration change | 25 | Warning |
| Service stability | 30 | Warning |

**Proxy service** is worth everything, because nothing else matters if Squid is
not running.

**Proxy port** is separate on purpose: a running process that never finished
binding is a distinct failure, and clients see it as a timeout rather than an
error.

**Configuration valid** runs Squid's own parser against the live file. Invalid
means the proxy is running but would not survive a restart — a trap that only
springs at the worst moment.

**Last configuration change** flags a change that was
[rolled back](applying-changes.md). Only the most recent apply is considered: an
older rollback since superseded by a success is history, not a current problem.

**Service stability** counts restarts Squid did not ask for. This is the check
no point-in-time probe would catch — a proxy that keeps dying and being restarted
looks healthy every single time you look at it. Each restart costs 15, capped at
30.

### Network

| Check | Weight | Critical? |
|---|---|---|
| DNS resolution | 15 | Yes |
| Outbound route | 15 | Warning |

**DNS** is critical because a proxy that cannot resolve names fails almost every
request. Testing it requires sending a query, so the appliance resolves one
name: `example.com`, which IANA runs for exactly this purpose. It is served by
root infrastructure rather than a company, so the check neither advertises your
appliance to a vendor nor depends on a commercial service staying up. The page
names what it resolved, so there is no mystery about it.

**Outbound route** reads the kernel routing table and sends nothing. It is a
warning rather than critical because an appliance in a deliberately isolated
subnet is doing its job — calling that critical would train you to ignore the
page.

### System

| Check | Weight | Critical? |
|---|---|---|
| Disk space | 20 at ≥90%, 10 at ≥80% | Critical at 90% |
| Memory | 10 at ≥90% | Warning |

Disk is measured on the **cache filesystem**, because that is where Squid runs
out of room first — long before the root filesystem notices.

Memory uses available rather than free memory. Linux counts reclaimable page
cache as used, so free memory on a busy proxy reads near zero and would make
every appliance look unhealthy.

### Maintenance

| Check | Weight | Critical? |
|---|---|---|
| Traffic ingestion | 10 | Warning |
| Log rotation | 5 | Warning |
| Time synchronisation | 5 | Warning |
| Security updates | 5 | Warning |

**Traffic ingestion** reports gaps in traffic history. Proxying is unaffected,
but analytics are incomplete for those periods — and a silently under-reported
dashboard is worse than one that admits a gap.

**Log rotation** allows for an appliance that has not been up long enough to have
rotated anything, so a new instance is not warned about it an hour after launch.

**Security updates** counts pending updates. A pending reboot is reported
alongside it but is not scored — it does not make a proxy unhealthy, and an
administrator planning a maintenance window simply needs to know.

## Unknown is not healthy

If the appliance cannot determine something, the check reads
<span class="pill-neutral">Unknown</span> and **deducts nothing**. An unreadable
fact is not a failure.

But it is never reported as healthy either. Silently scoring an appliance you
cannot see as working would be worse than reporting nothing at all. Several
checks read as unknown when the privileged helper is not answering, which is
itself the thing to investigate.

## Advice, not just diagnosis

Every failing check carries a line saying what to do:

> **Cache directory** — The cache directory is not writable by the proxy.
> *Squid cannot store objects and may refuse to start. Check ownership of the
> cache directory.*

A health page that reports a problem without saying what to do about it is a
slower way of reading a log.

## Resources

Below the checks, the same figures as numbers you can watch rather than
pass/fail: memory, CPU load, each filesystem, and traffic-history storage.

CPU load is shown **per core** as well as raw, because the raw figure invites the
wrong conclusion — a load of 4 is idle on a 16-core instance and desperate on a
single-core one.

Filesystems are listed separately for the system volume, the proxy cache and the
management data, and deduplicated when they are the same volume. Three identical
bars read as a bug rather than as reassurance.

## On the dashboard

The dashboard carries a compact version showing only what needs attention. When
everything passes it says so in one line — *"All 14 checks passing"* — and if
some checks could not be determined it says that too, because an unknown check
is not a pass.

## Related

- [Logs](logs.md) — what a failing check usually points at
- [Applying changes](applying-changes.md) — where the rollback check comes from
- [Ports & security groups](../reference/ports.md) — what the port check needs
