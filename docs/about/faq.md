# Frequently Asked Questions

## About the product

### What does this add to Squid?

Squid is excellent at proxying and offers almost nothing for running it. This
adds traffic visibility, readable access policy, safe configuration deployment
with automatic rollback, health monitoring, log reading, backup and restore, and
directory authentication — all from a browser.

Squid itself is unmodified upstream Squid 7.

### Is Squid modified?

No. It is compiled from upstream source with a standard set of options. The
management layer reads its logs and generates its configuration; it does not
patch it.

### Can I still use Squid directives the console does not manage?

Yes. Put them in `/etc/squid/local.conf`, which is included by the generated
configuration and is never generated, overwritten or restored.

### Does it phone home?

No. There is no telemetry, licence check or usage reporting.

The appliance makes **no scheduled outbound connections by default**. Enabling
[Microsoft 365 endpoints](../guide/microsoft-365.md) adds one, to
`endpoints.office.com`.

The [health page](../guide/health.md) resolves `example.com` to check DNS —
there is no way to test name resolution without sending a query, and that name
is reserved by IANA for exactly this purpose.

## HTTPS

### Does it decrypt HTTPS?

**No.** There is no TLS interception in this product.

For an HTTPS request you can see the hostname, the bytes transferred, the
duration, and whether it was allowed or blocked and by which rule. You cannot see
the path, the query string or the content.

This is a deliberate position. Interception means installing a certificate
authority on every client and holding a private key that can impersonate any
site on the internet — a serious liability, and a large amount of machinery to
secure. Blocking by hostname covers the overwhelming majority of access-control
requirements without it.

### Then why is my cache hit rate so low?

Because almost all traffic is HTTPS, and an encrypted tunnel cannot be cached by
anything that does not decrypt it.

The console reports the hit rate against **cacheable** traffic and states what
share was cacheable, so the figure means something. Measured against total
traffic, a perfectly healthy proxy reports 2–5% and looks broken.

### Can I block a specific URL path?

Only for plain HTTP. For HTTPS the path is inside the tunnel and the proxy never
sees it — block at the hostname level instead.

## Deployment

### How many users can one appliance handle?

Squid scales well; memory and disk matter more than CPU. Roughly:

| Users | Instance | Disk |
|---|---|---|
| Up to 100 | 2 vCPU, 4 GB | 30 GB |
| 100–500 | 2 vCPU, 8 GB | 60 GB |
| 500–2000 | 4 vCPU, 16 GB | 100 GB+ |

Watch the [health page](../guide/health.md) for actual usage rather than
guessing.

### Can I run more than one for high availability?

You can run several behind a load balancer, but each is independent — there is
no shared state or configuration sync in this release. You would configure each
one, or [restore the same backup](../guide/backup-restore.md) onto each, which
works because backups are portable between appliances.

Central management of multiple appliances is on the
[roadmap](roadmap.md).

### Do I need SSH access?

No. Everything for normal administration is in the console, including
[reading the logs](../guide/logs.md). Leaving port 22 closed removes an entire
class of exposure.

### Can I use my own TLS certificate for the console?

Yes. Replace `/etc/cloudinfra/tls/cert.pem` and `key.pem` and restart the
service. The console does not care where a certificate came from, and
[Administration → Security](../guide/administration.md#security) will show
yours.

## Configuration

### Why does nothing take effect until I press Apply?

So that a wrong change is survivable. Everything is staged, and applying runs
validation, takes a snapshot, deploys, then health-checks the result — restoring
the previous configuration automatically if the proxy does not come back
healthy.

[:material-arrow-right: The full pipeline](../guide/applying-changes.md)

### My rule is not matching. Why?

The three usual causes:

1. **Domain form.** `example.com` matches only that exact host. Use
   `.example.com` for the domain and everything under it.
2. **Rule order.** Squid stops at the first match, so a broad allow above your
   rule means yours never runs. The console warns about rules it can prove are
   shadowed.
3. **It is not applied.** Check for a pending-changes banner.

Then look at [Live Traffic](../guide/live-traffic.md) — click the request and it
names the rule that actually decided it.

### Why was my change rolled back?

The proxy failed its health check after the change. The outcome panel says which
of the four checks failed. Common causes are a configuration Squid loads but
cannot serve with, and a port change the appliance itself cannot then reach.

Your previous configuration was restored automatically.

### Can I edit squid.conf directly?

You can, but it is overwritten at the next apply, and the console will report
that the file no longer matches what it generated. Use `local.conf` for anything
the console does not manage.

## Authentication

### Can I authenticate against Microsoft 365?

Yes, but not against Entra ID directly — **Entra ID has no LDAP endpoint**. The
supported path is Microsoft Entra Domain Services, a managed domain that exposes
the same identities over LDAPS.

[:material-arrow-right: Directory authentication](../guide/directory-authentication.md)

### Will enabling authentication break my clients?

Yes, if they are not configured to send credentials. Every client must be ready
before you apply the change.

The automatic rollback covers a proxy that fails to start. It cannot detect that
your clients cannot sign in — refusing an unauthenticated request is correct
behaviour from the appliance's point of view.

### Are user passwords stored on the appliance?

No. The appliance never sees a user password except in transit to your directory
during a bind, and stores none of them. Only the directory **service account**
password is stored, and it is kept out of `squid.conf` and out of the process
list.

## Data

### How long is traffic history kept?

Three tiers with separate retention: full request detail for 3 days, per-minute
summaries for 7 days, hourly summaries for 90 days. All adjustable, with a hard
size cap alongside so a traffic spike cannot fill the disk inside the window.

[:material-arrow-right: Data & storage](../guide/administration.md#data-storage)

### Does traffic data leave the appliance?

No. It is stored on the instance and never sent anywhere. There is no cloud
service behind this product.

### Can I get the data out?

Yes — CSV export from [Analytics](../guide/analytics.md), and the structured
traffic log is a documented tab-separated format you can process with ordinary
tools.

[:material-arrow-right: Traffic log format](../reference/log-format.md)

### Does a backup contain passwords?

No. Backups contain rules, lists and settings — no password hashes, session keys
or certificates. Safe to keep in version control; use a passphrase if it leaves
your control, since it does contain your full access policy.

## Troubleshooting

### The console will not load

Check the security group allows 8443 from where you are, then that the service is
running. First boot generates a certificate and creates the administrator
account before the console listens, which takes up to a minute on a small
instance.

### Clients get 403 Forbidden

Either the client is outside the allowed network, or a rule denied it.
[Live Traffic](../guide/live-traffic.md) will tell you which — and if it is a
rule, which rule.

### Clients get 407

[Authentication](../guide/directory-authentication.md) is enabled and the client
is not sending credentials.

### The health score dropped and I do not know why

The health page shows every check with its deduction and advice on what to do.
Every point removed is attributed to a named check.

[:material-arrow-right: Health](../guide/health.md)

## Still stuck?

[:material-arrow-right: Contact support](../support.md)
