# Administration

The console's own account, its sessions, what it is protecting itself with, and
who has been signing in.

## Administrator account

Shows the username, when the account was created, when the password last
changed, and the last successful sign-in.

**Change password** requires the current one and a new one of at least 12
characters. Changing it **signs out every session for the account, including the
one you are using** — a stolen session must not outlive the credential it came
from.

!!! warning "The first-boot password is a token, not a credential"
    The password generated at first boot appears in your instance console
    output, which anyone with the right cloud permissions can read. That is
    intentional — it is how you retrieve it without SSH — but it means it is a
    bootstrap token.

    The console requires you to change it at first sign-in, and this page warns
    if an account is still using one.

This release has a single administrator account. Directory-backed console
sign-in is on the [roadmap](../about/roadmap.md).

## Application

Versions and identity, in one place: Proxy Manager version, Squid version,
operating system, kernel, hostname, cloud platform and region, and uptime.

Worth having when raising a support case — see [Support](../support.md).

## Active sessions

Every session signed in with your account: where it came from, which browser,
when it started, when it was last used and when it expires. The one you are
using is marked.

**Revoke** ends a session immediately. Revoking your own returns you to the
sign-in page.

Sessions end after 8 hours, or 60 minutes idle. Both are shown on the page and
configurable in the [configuration file](../reference/configuration.md).

!!! info "Session tokens are never shown"
    Only a short reference is displayed. The appliance stores only a hash of
    each session token, so a read of its database — by a backup, a support
    bundle or a leak — hands nobody a usable session. Showing the token here
    would undo that.

## Security

### The certificate

Details of the certificate the console is serving: who it was issued to and by,
validity dates and days remaining, which hostnames and addresses it covers, its
signature algorithm, and its **SHA-256 fingerprint**.

The fingerprint is there so you can confirm the browser warning you clicked
through was about a self-signed certificate rather than about something between
you and the appliance. Compare it with what your browser shows.

An expiring certificate is flagged at 30 days, and an expired one is reported as
a problem — browsers will refuse the console entirely.

### Replacing the certificate

**Install certificate**, on this page. Two files, both PEM:

- **Certificate** — yours, or a chain with the leaf first followed by any
  intermediates.
- **Private key** — unencrypted, and belonging to that certificate.

The pair is checked before anything is written. A key belonging to a different
certificate, or one that has expired, is refused with the reason and nothing
changes. What passes takes effect on the next connection: no restart, and your
session stays signed in. Connections already open keep the previous certificate
until they are reopened, so reload the page to see the new one.

!!! tip "Include the intermediates"
    A public authority gives you a leaf and one or two intermediate
    certificates. Upload only the leaf and desktop browsers often still work,
    because they cache intermediates from elsewhere, while phones and
    command-line clients fail. Put them in one file, leaf first.

**Generate a new self-signed certificate** replaces whatever is installed with
one created here, covering this appliance's own hostname and addresses,
including its public address. Browsers warn about it, but it always matches the
appliance, so the console stays reachable.

That is the way back. A certificate issued for a name this appliance cannot be
reached by would otherwise leave nobody able to sign in and correct it.

### Removing the browser warning

Two things must both be true before a browser stops warning: the certificate
must come from an authority the browser already trusts, and the name in the
address bar must appear in the certificate.

The certificate generated on the appliance fails both. It is self-signed, and it
covers the appliance's own hostname and addresses, which is rarely what anyone
types.

1. **Give the appliance a DNS name**, such as `proxy.example.com`. This is the
   step people skip, and without it the rest cannot work — public authorities do
   not issue certificates for IP addresses.
2. **Get a certificate for that name**, from a public authority or from your own
   internal one. An internal authority is usually already trusted on managed
   machines, and is the natural choice for a console only reachable internally.
3. **Install it** here.
4. **Reach the console by that name**, not by its address. A certificate for
   `proxy.example.com` still warns if you browse to the IP.

!!! note "Renewal is manual"
    There is no ACME client on the appliance. A certificate from Let's Encrypt
    expires every 90 days and has to be uploaded again each time. A longer-lived
    certificate from an internal authority is far less trouble for an appliance.

!!! tip "For a handful of administrators"
    You do not need an authority at all. Export the appliance's own certificate
    and add it to those machines' trust stores. The fingerprint above is how you
    confirm you are trusting the right one.

### Protections

What the console is actually doing, reported rather than claimed:

| | |
|---|---|
| **Password hashing** | Argon2id, 64 MiB per verification |
| **Session cookies** | `__Host-` prefixed, HttpOnly, SameSite=Strict, HTTPS only |
| **CSRF protection** | Double-submit token on every state-changing request |
| **Rate limiting** | Per-address limits, with account lockout |
| **Privilege separation** | The console runs unprivileged; root work goes through a closed set of named operations |
| **Audit trail** | Authentication and every configuration change recorded |

[:material-arrow-right: The full security model](../about/security-model.md)

## Login history

Successful **and failed** sign-ins, with the source address, kept for 30 days.

Failures are the point. A page showing only successes cannot answer the question
worth asking, which is whether anybody has been trying. The count of failures in
the last 24 hours is called out.

If you see failures you cannot account for, tighten the security group on port
8443 first — the console defends itself, but it should not have to.

## Audit trail

A separate page recording every administrative action: sign-ins and failures,
password changes, rule and list edits, settings changes, configuration applies
and rollbacks, backups created and restored, and **reads** of traffic data —
log searches and analytics exports.

Reads are included deliberately. The trail exists to answer *who looked at
what*, not only *who changed what*, and proxy logs contain your users' browsing
history.

Each entry records who, when, what, the result and the source address. Entries
are written before an action is attempted and updated with its outcome, so an
action that crashes the process still leaves a trace.

## Data & storage

How long traffic history is kept, and what it costs.

| Setting | Default | Range |
|---|---|---|
| Full request detail | 3 days | 1–30 |
| Per-minute history | 1 week | 1–90 |
| Hourly history | 90 days | 7–730 |

The page shows current usage against the size cap, how many requests are stored,
and a measured estimate of daily growth — taken from actual stored history
rather than assumed, so it is meaningful from the first few hours.

Shortening a window does **not** delete anything immediately. History outside
the new window is removed at the next scheduled tidy-up, which gives you time to
change your mind.

The tiers summarise each other, so finer detail cannot be kept longer than
coarser: full detail cannot outlast per-minute history, which cannot outlast
hourly. The console explains any rejection in those terms.

!!! info "There is a hard size cap as well"
    Age-based retention cannot stop a traffic spike filling a volume inside the
    window, so a size cap runs alongside it, discarding the oldest detail first.
    Analytics degrading is always preferable to a full disk stopping the proxy.

## Related

- [Security model](../about/security-model.md) — the design behind the protections
- [Backup & restore](backup-restore.md) — what a backup does and does not contain
- [Configuration file](../reference/configuration.md) — session timeouts and paths
