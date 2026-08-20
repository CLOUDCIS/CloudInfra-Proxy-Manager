# Roadmap

What is planned, and what is deliberately not. Priorities move with customer
demand — if something here matters to you, [tell us](../support.md), because
that is largely how the order gets decided.

!!! note "No dates"
    We would rather say what is being worked on than commit to dates we would
    then have to defend. Nothing here is a contractual commitment.

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
