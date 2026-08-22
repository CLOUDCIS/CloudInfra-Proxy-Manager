# Roadmap

What is planned, and what is deliberately not. If something here matters to
you, [tell us](../support.md) — that is how the order gets decided.

!!! note "No dates, and no false certainty"
    We would rather say what is being worked on than commit to dates we would
    then have to defend. Nothing here is a contractual commitment.

    The ordering below is our judgement about what would be most useful, not a
    measurement of what has been asked for. Telling us we have it wrong is the
    single most useful thing you can do with this page.

## Being considered next

**Guided first-run setup.** A short wizard on first sign-in — administrator
password, proxy port, client network, cache — ending with a confirmed, applied
configuration. The client-network step is the one that matters: it is currently
detected automatically at first boot and never explicitly confirmed by anyone.

**Scheduled policies.** Rules that apply during particular hours or days. The
data model already carries a schedule field for this, so existing rules need no
migration.

**Alerting.** Email, Microsoft Teams, Slack or webhook notification when the
health score drops, a deployment rolls back, or blocked traffic spikes. Today
the appliance reports its state accurately and waits for you to look.

**Scheduled and richer reporting.** PDF alongside CSV, and reports that arrive
by email rather than being fetched.

**Choosing what to cache.** Proxy Settings controls the size and shape of the
cache — on or off, disk, memory, maximum object size — but not which content is
cached or for how long. That is `refresh_pattern`, and it is currently identical
on every appliance.

The case worth tuning for is a fleet downloading the same packages and updates,
where holding package files for weeks and keeping repository metadata fresh are
opposite requirements. Both are a few lines of configuration, so the plan is a
small set of choices in the console rather than a syntax editor.

Until then it is [documented as a manual step](../squid/caching.md), including
the trap that most Squid caching guides online use options this build
deliberately does not have.

**A dedicated security view.** Blocked requests, the clients being blocked most
often, the destinations they are trying to reach, and failed sign-ins — on one
page instead of spread across the dashboard, analytics and administration.

Most of it is assembly: the console already counts blocks by client, domain and
rule. The useful new part is ranking clients by how often they are blocked,
which is how you notice a misconfigured host or somebody probing, and which no
current screen surfaces. Failed sign-ins belong there too — a proxy block and an
authentication failure are usually the same investigation, and today they are in
different parts of the console.

One honest constraint shapes it: individual requests are kept for three days
while the per-client and per-domain totals go back ninety, so such a page shows
recent detail against a longer trend rather than pretending to both at once.

## Further out

**Central management of several appliances.** One console managing a fleet, with
shared policy. Each appliance is independent today; backups are portable between
them, which covers the simple cases.

**Directory sign-in for administrators.** Today the *proxy users* can
authenticate against a directory but the console has a single local
administrator account. Multiple administrators with roles — full access versus
read-only for auditors — belongs with it.

**Managed category feeds.** Subscribing to maintained lists of malware, phishing
and content categories, in the same way
[Microsoft 365 endpoints](../guide/microsoft-365.md) work now.

**SIEM forwarding.** Structured events to Microsoft Sentinel, Splunk or syslog.
The traffic log is already a documented, machine-readable format, so this is
transport rather than a new data model.

**Kerberos and Negotiate authentication.** Single sign-on for domain-joined
clients, so users are not prompted at all. Squid ships the helper; it needs
keytab management the console does not have yet.

## Squid features the console does not manage

Squid can do a great deal that this console does not expose. That is deliberate
— every setting we surface is one we then have to validate, document, roll back
safely and support. But it is worth being explicit about what is available
underneath, because anything Squid supports can be configured by hand in
`/etc/squid/local.conf`, which the console never overwrites.

| Capability | Available on the appliance? | Console support |
|---|---|---|
| **Time-based access rules** | Yes | Planned — see above |
| **Hiding internal client addresses** | Partly | Planned |
| **URL rewriting and redirection** | Yes | Not planned as a generic feature |
| **ICAP / eCAP content adaptation** | **No — needs a rebuild** | Under consideration |
| **Transparent interception** | Partly | Not planned for cloud |
| **Reverse proxy and load balancing** | Yes | Not planned — different product |

### Hiding internal client addresses

Squid adds an `X-Forwarded-For` header carrying the internal address of the
client that made each request, which is then visible to every site your users
visit. That is information disclosure some organisations care about and others
rely on for their own logging.

Squid's `forwarded_for` directive can delete or truncate it, and needs no
special build. A console setting for it is a small piece of work.

Stripping *arbitrary* headers is a different matter: it needs
`--enable-http-violations`, which this image is not built with, and Squid's own
documentation warns that doing so violates the HTTP standard and "could make you
liable for problems which it causes."

### ICAP and eCAP

The protocols for handing traffic to an external malware scanner, DLP system or
content filter before it reaches the user.

This image is **not currently built with ICAP support**, so it cannot be enabled
by configuration alone. We are considering adding the build option at the next
image release — even without console support, that would let anyone with an
existing ICAP appliance wire it up in `local.conf`. If you have one, telling us
would move this along considerably.

### Transparent interception

Intercepting traffic without configuring clients, via TPROXY or WCCP.

The appliance is built with the Linux netfilter support this needs, but the
approach fits cloud deployments poorly: the proxy must become the default route
for your clients, which on AWS means a Gateway Load Balancer or route-table
changes — more work than configuring the clients, not less.

There is a second problem worth knowing about. Intercepted connections carry no
`CONNECT` request, so Squid has no hostname to match on and domain rules match
only IP addresses. Making interception genuinely useful would require inspecting
the TLS handshake to recover the destination name, so the two are one piece of
work rather than two.

### Reverse proxy and load balancing

Squid can sit in front of web servers as an accelerator. It is a capable
feature and an entirely different product from this one: every concept in this
console — client networks, blocked destinations, who browsed where — describes
outbound traffic. A reverse-proxy mode would need its own policy model,
dashboard and health checks, sharing almost nothing with what is here.

If that is what you need, we would rather point you at a tool built for it than
half-build one.

## Not planned

Stated plainly, so nobody waits for something that is not coming.

**TLS interception.** Decrypting HTTPS means installing a certificate authority
on every client and holding a key that can impersonate any site on the internet.
That is a serious liability to take on, and the great majority of access-control
requirements are met by blocking on hostname.

If you need content inspection of encrypted traffic, this is not the product for
that, and we would rather say so than half-build it.

**Antivirus and content scanning.** A different product with a different update
cadence and a different threat model.

**Endpoint agents.** This is a network proxy. Anything requiring software on
every client is out of scope.

**A cloud-hosted management service.** The appliance is single-tenant and your
traffic data never leaves your network. That is a selling point, not a
limitation to be removed.

## Requesting something

Support requests and feature requests go to the same place, and both are read.

If you are asking for something on this page, saying so moves it up. If you are
asking for something that is not, that is more useful still — it is the requests
we have not thought of that change this list.

[:material-arrow-right: Contact us](../support.md)
