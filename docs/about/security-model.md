# Security Model

This appliance sits in the path of your users' web traffic and holds a record of
where they went. That makes it worth attacking, and the design assumes so.

This page describes what protects what, and — as importantly — what it does
**not** protect against.

## Two processes, deliberately

```text
┌────────────────────────┐        ┌──────────────────────────┐
│ cloudinfra-proxyd      │        │ cloudinfra-privhelper    │
│ runs as: cloudinfra    │◄──────►│ runs as: root            │
│ no capabilities at all │  Unix  │ closed set of operations │
│ parses logs, serves UI │ socket │ nothing else             │
└────────────────────────┘        └──────────────────────────┘
```

The process with the attack surface — the one parsing proxy logs and handling
administrator input over HTTP — holds **no privileges whatsoever**. It cannot
write `/etc/squid`, cannot restart services, cannot read the cache directory.

Anything privileged goes to a second, much smaller process over a local socket.

### The closed verb set

The console asks for one of a fixed list of named operations. It never passes a
file path, a command fragment or a configuration body across that boundary.

| Operation | Does |
|---|---|
| `ping` | Is the helper alive |
| `service.status` | Read Squid's unit state |
| `service.reload` | `squid -k reconfigure` |
| `service.restart` | Restart Squid |
| `squid.version` | Read the engine version |
| `config.validate` | Parse `live` or `candidate` |
| `config.install` | Move the staged tree into place |
| `config.snapshot` | Copy the live tree aside |
| `config.restore` | Put a snapshot back |
| `system.facts` | Read-only maintenance facts |

Ten operations. Every path they touch is compiled in. Every argument is a typed
value — an enum, or an integer that is range-checked — and the helper
re-validates everything even though the caller is our own process.

The usual shortcut here is a sudoers entry with a wildcard. That is how
appliances get compromised, because any argument reaching a shell is an injection
point. Nothing here reaches a shell: commands are executed as an argument array
with a minimal environment.

A test enforces the size of that list. Adding an operation fails the build unless
somebody also writes down why it has to run as root.

### Confinement

The console's systemd unit removes what it does not need: no capabilities, a
read-only filesystem except three specific paths, private `/tmp`, no new
privileges, a system-call filter, and hard limits of 512 MB memory and 50% CPU
so a runaway query is killed rather than competing with the proxy.

## Authentication

| | |
|---|---|
| **Passwords** | Argon2id, 64 MiB per verification |
| **Session tokens** | 256-bit random; only a SHA-256 hash is stored |
| **Cookies** | `__Host-` prefixed, `HttpOnly`, `SameSite=Strict`, HTTPS only |
| **CSRF** | Double-submit token required on every state-changing request |
| **Rate limiting** | Per address, plus per-account lockout |
| **Sessions** | 8 hours absolute, 60 minutes idle |

Only the **hash** of each session token is stored, so a read of the database — by
a backup, a support bundle or a leak — hands nobody a usable session. The console
never displays a token or its hash.

Concurrent password verifications are bounded, because each allocates 64 MiB and
the service is capped at 512 MB. Unbounded concurrency there is an
out-of-memory condition waiting to be triggered.

## Secrets

| Secret | Where it lives | Protection |
|---|---|---|
| Administrator password | `cloudinfra.db` | Argon2id hash |
| Session tokens | `cloudinfra.db` | SHA-256 hash only |
| TLS private key | `/etc/cloudinfra/tls/key.pem` | Generated per instance, root-owned |
| Directory bind password | `/etc/squid/auth-bind.secret` | `0640 root:squid` |

The directory bind password is the interesting one, because it must be readable
by a program Squid spawns. It is deliberately **not** in `squid.conf`, which is
world-readable and is displayed in the console's configuration preview, and
deliberately **not** a command-line argument, which would appear in the process
list to any local user running `ps`.

## Nothing is shared between instances

The image is sealed with no credentials in it. First boot generates:

- A TLS certificate and key, unique to that instance
- An administrator password, random per instance
- The client network, detected from cloud metadata

A pre-seal check fails the build if a certificate, database or credential file is
found in the image. Two instances launched from the same image share nothing.

The initial password is written to the instance console output so you can
retrieve it by proving control of the instance rather than by SSH — and the
console forces you to change it at first sign-in, because that output is a
bootstrap token rather than a credential.

## Untrusted input

Three things arrive from outside and become configuration or display:

**Proxy logs.** Written by Squid from data attackers control — URLs, user agents.
Free-text fields are percent-encoded by Squid, so a tab or newline cannot forge
or split a record. The parser rejects malformed lines rather than guessing.

**The Microsoft 365 feed.** Somebody else's JSON over the network becoming proxy
rules. Every hostname and range is validated; anything that would not make a
valid ACL entry is dropped **and counted**, so a change in the feed's shape shows
up rather than quietly shrinking your lists.

**Administrator input.** Rules, lists and directory settings become Squid
directives. Values that could break out of the directive they are rendered into
— quotes, newlines, LDAP metacharacters — are **refused rather than escaped**.
Nothing legitimate contains them, and refusing stays correct as the rendering
changes around it.

## The audit trail

Every administrative action is recorded with who, when, what, the result and the
source address. Entries are written *before* an action is attempted and updated
with the outcome, so an action that crashes the process still leaves a trace.

Reads are audited as well as writes — log searches and analytics exports —
because the trail exists to answer *who looked at what*, not only *who changed
what*, and proxy logs contain your users' browsing history.

## Supply chain

The console is a single static Go binary with **three** third-party libraries:

| Library | For |
|---|---|
| `golang.org/x/crypto` | Argon2id |
| `golang.org/x/net` | Public suffix list |
| `modernc.org/sqlite` | Pure-Go SQLite |

Everything else is the standard library. There is no JavaScript framework, no
charting library, no icon font and no web assets fetched at runtime — the console
is served entirely from the binary.

This is a deliberate cost. A security appliance's own dependency tree is part of
its attack surface, and it is what a marketplace vulnerability scan reports
against.

Notably, [directory authentication](../guide/directory-authentication.md) added
**no** dependency: Squid's own LDAP helpers do that work.

## What this does not protect against

Stated plainly, because a security page that only lists strengths is marketing.

**It does not decrypt HTTPS.** No TLS interception, so no visibility into paths
or content, and no scanning of encrypted traffic. That is a deliberate position —
interception means installing a CA on every client and holding a key that can
impersonate any site.

**It is not a malware scanner or a web categorisation service.** It blocks what
you tell it to block.

**Anyone who can reach port 8443 can attempt to sign in.** The console defends
itself, but the security group is your first control and should be tight.

**Root on the instance is game over.** Anyone with root can read the databases
and the logs. Standard for an appliance, worth stating.

**A backup contains your full access policy.** No credentials, so it is safe to
store in version control — but it is not public information either. Use a
passphrase if it leaves your control.

## Related

- [Architecture](architecture.md) — how the components fit together
- [Ports & security groups](../reference/ports.md) — the first line of defence
- [Administration](../guide/administration.md) — sessions, login history, the audit trail
