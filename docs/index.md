---
hide:
  - navigation
---

<div class="cis-banner" markdown>
![CloudInfra Proxy Manager](assets/logo.png)
</div>

# CloudInfra Proxy Manager

**A proxy you can actually see into, and change without holding your breath.**

CloudInfra Proxy Manager is a management, analytics and policy layer for
[Squid](https://www.squid-cache.org/) on AWS, Azure and Google Cloud. Squid is
excellent at proxying and offers almost nothing for running it: no traffic
visibility, no readable policy, and a configuration file where one typo stops
the service. This adds the parts that were missing.

Everything runs on your own instance. No traffic data leaves your network, and
there is no cloud service to depend on.

[Just launched? Start here :material-arrow-right:](getting-started/quickstart.md){ .md-button .md-button--primary }
[Deploy Solution :material-cloud-download:](https://cloudinfrastructureservices.co.uk/squid-proxy-server/){ .md-button target="_blank" rel="noopener" }
[How it works :material-sitemap:](about/architecture.md){ .md-button }
[Security model :material-shield-lock:](about/security-model.md){ .md-button }

---

## Where do you want to start?

<div class="grid cards" markdown>

-   :material-rocket-launch: __Just launched the image?__

    ---

    Sign in, confirm the proxy is healthy and send your first request through it.

    [:material-arrow-right: Your first 10 minutes](getting-started/quickstart.md)

-   :material-traffic-light: __Want to control what is reachable?__

    ---

    Write access rules in plain language, and see exactly which rule decided
    each request.

    [:material-arrow-right: Access rules](guide/access-rules.md)

-   :material-account-key: __Want to know who, not just which address?__

    ---

    Authenticate proxy users against Active Directory, Entra ID or OpenLDAP.

    [:material-arrow-right: Directory authentication](guide/directory-authentication.md)

-   :material-microsoft: __Running Microsoft 365 behind the proxy?__

    ---

    Keep Teams, Exchange and SharePoint working without maintaining hundreds
    of endpoints by hand.

    [:material-arrow-right: Microsoft 365 endpoints](guide/microsoft-365.md)

</div>

---

## What it adds to Squid

<div class="grid cards" markdown>

-   :material-eye-outline: __See your traffic__

    ---

    Live request stream, historical analytics, top destinations and clients,
    cache effectiveness and blocked-request reporting — read from Squid's own
    log rather than from an agent.

    [:material-arrow-right: Dashboard](guide/dashboard.md)

-   :material-format-list-checks: __Readable access policy__

    ---

    Rules that read as sentences — *"Deny clients in 10.20.0.0/16 to reach
    facebook.com"* — generated into correct Squid ACLs. You never edit
    `squid.conf` by hand.

    [:material-arrow-right: Access rules](guide/access-rules.md)

-   :material-backup-restore: __Changes that undo themselves__

    ---

    Every change is validated, snapshotted, applied and health-checked. If the
    proxy does not come back healthy, the previous configuration is restored
    automatically.

    [:material-arrow-right: Applying changes](guide/applying-changes.md)

-   :material-target: __Know which rule blocked it__

    ---

    Squid has no native field for which rule decided a request. This stamps
    each one, so a blocked request tells you the policy that blocked it — not
    just that something did.

    [:material-arrow-right: Live traffic](guide/live-traffic.md)

-   :material-heart-pulse: __Health you can account for__

    ---

    Fourteen named checks, each with a fixed published weight. The score is
    arithmetic you can follow, not a number with no explanation.

    [:material-arrow-right: Health](guide/health.md)

-   :material-console-line: __No SSH required__

    ---

    Read Squid's access and error logs, and the manager's own, from the
    console. Search, filter by severity, and look into rotated files.

    [:material-arrow-right: Logs](guide/logs.md)

</div>

---

## Built to be run, not just installed

<div class="stat-cards" markdown>

<div class="stat-card"><div class="stat-num">14</div><div class="stat-label">Scored health checks</div></div>
<div class="stat-card"><div class="stat-num">10</div><div class="stat-label">Privileged operations, closed set</div></div>
<div class="stat-card"><div class="stat-num">3</div><div class="stat-label">Third-party code dependencies</div></div>
<div class="stat-card"><div class="stat-num">0</div><div class="stat-label">Outbound connections by default</div></div>

</div>

The management console runs unprivileged. Anything needing root goes through a
small helper over a local socket that accepts a fixed set of named operations
and nothing else — no paths, no command fragments, no shell. The whole product
is a single static binary with three third-party libraries, because a security
appliance's own dependency tree is part of its attack surface.

[:material-arrow-right: Read the security model](about/security-model.md)

---

## Where it runs

Available as a pre-built image on the AWS, Azure and Google Cloud marketplaces,
built on Ubuntu 24.04 LTS with Squid 7.

The proxy and the management console are deliberately independent: if the
console stops, Squid keeps proxying, and if Squid stops, the console still comes
up to tell you why.

[Deploy the image :material-arrow-right:](getting-started/deploy.md){ .md-button .md-button--primary }
[Deploy Solution :material-cloud-download:](https://cloudinfrastructureservices.co.uk/squid-proxy-server/){ .md-button target="_blank" rel="noopener" }
