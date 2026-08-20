# Directory Authentication

By default the proxy identifies clients by address. Directory authentication
makes it identify *people*: users sign in with their existing directory
credentials, rules can name users and groups, and the traffic log records who
made each request.

!!! danger "This will stop clients that are not ready for it"
    Turning authentication on requires **every** client to send proxy
    credentials. Any client not configured for it stops reaching the internet
    the moment you apply the change.

    The [automatic rollback](applying-changes.md) protects you from a proxy that
    fails to start. It cannot tell that your clients cannot sign in — from the
    appliance's point of view, refusing an unauthenticated request is correct
    behaviour.

    [Configure the clients first](../getting-started/clients.md), then enable it.

## How it works

The appliance holds no directory client of its own. Squid ships helper programs
for exactly this, and the console configures them:

- `basic_ldap_auth` binds as the user to check their password.
- `ext_ldap_group_acl` answers "is this user a member of that group?"

Three consequences worth knowing:

- **No passwords are stored here.** The appliance never sees a user password
  except in transit to your directory, and stores none of them.
- **A directory outage degrades to failed proxy authentication**, not a broken
  console. You can still sign in and change policy.
- **No additional software** is installed to make this work.

## Supported directories

| Directory | Notes |
|---|---|
| **Active Directory** | On-premises AD, or anything sharing its schema |
| **Microsoft Entra Domain Services** | The managed domain that exposes LDAP for Microsoft 365 identities |
| **OpenLDAP** | And its derivatives |

Squid also ships helpers for **RADIUS** and for a **local password file**. The
console does not manage those, but they are installed and can be configured by
hand — see [Authentication Backends](../squid/authentication-backends.md), which
also explains why only one backend can be active at a time.

### Microsoft 365 and Entra ID

This is the point that causes the most confusion, so it is worth stating
directly.

!!! warning "Entra ID has no LDAP endpoint"
    Microsoft Entra ID (formerly Azure AD) speaks Microsoft Graph and OIDC. It
    does **not** offer LDAP, so Squid cannot authenticate against it directly.

    The supported path is **Microsoft Entra Domain Services** — a managed domain
    Microsoft runs alongside your tenant, which exposes those same Microsoft 365
    identities over LDAPS. From the proxy's point of view it is ordinary LDAP
    over TLS against an Active Directory schema.

    Entra Domain Services is a separate Azure resource with its own cost. If you
    do not have it, you cannot authenticate Microsoft 365 users at the proxy.

When using it:

- Point **Server** at the LDAPS name of your managed domain.
- The **bind account** must be a member of *AAD DC Administrators*.
- Base DN follows your managed domain name, for example
  `DC=example,DC=onmicrosoft,DC=com`.

## Setting it up

**Proxy Settings → Directory Authentication.**

Fill the form in before you tick *Require sign-in* — the fields stay editable so
you can prepare the configuration and enable it as a separate, deliberate step.

### Server and transport

| Field | Notes |
|---|---|
| **Server** | One or more hostnames, space-separated. Squid tries each in turn — this is the only failover available. |
| **Port** | 636 with TLS, 389 without. |
| **Require TLS** | Leave this on. |

!!! danger "Do not turn TLS off"
    Without TLS the service account password crosses your network in clear text
    on every lookup.

    With TLS on, the console passes Squid's strict flag, which **fails** if TLS
    cannot be established rather than silently continuing unencrypted. That
    difference is the whole point of the setting.

### Finding users

| Field | Example |
|---|---|
| **Base DN** | `DC=example,DC=com` |
| **Bind account** | `CN=squid,OU=Service Accounts,DC=example,DC=com` |
| **Bind password** | The service account's password |
| **User filter** | Leave empty for the directory default |

The bind account only needs **read** access to look users up. Do not use a
privileged account.

Default user filters, applied automatically per directory type:

| Directory | Filter |
|---|---|
| Active Directory / Entra DS | `(sAMAccountName=%s)` |
| OpenLDAP | `(uid=%s)` |

`%s` is replaced with the username being checked. Override it if your directory
identifies users differently — `(userPrincipalName=%s)` is common where people
sign in with their email address.

!!! info "Anonymous bind"
    If your directory permits anonymous searches, leave the bind account empty
    and no credential is stored at all. A bind account with no password is
    rejected, because it is always a mistake.

### Where the bind password lives

Not in `squid.conf`. That file is world-readable and its contents are shown in
the console's configuration preview.

The password is written to `/etc/squid/auth-bind.secret`, readable only by root
and the `squid` group, and passed to the helper by file reference. Passing it as
a command-line argument would also expose it in the process list to any local
user running `ps`.

It travels with the configuration, so rolling back to an earlier version
restores the credential that version was working with. The console never returns
it to the browser — the form shows only whether one is set.

### Groups

Tick **Group lookup** to allow rules that name directory groups.

| Field | Example |
|---|---|
| **Group base DN** | Defaults to the base DN above |
| **Group filter** | Leave empty for the directory default |

| Directory | Filter |
|---|---|
| Active Directory / Entra DS | `(&(objectClass=group)(cn=%g)(member=%u))` |
| OpenLDAP | `(&(objectClass=posixGroup)(cn=%g)(memberUid=%u))` |

`%g` is the group name, `%u` the signed-in user.

!!! warning "Both placeholders are required"
    A group filter without `%u` matches the group regardless of who is asking —
    every group rule would then allow everybody. The console refuses to save a
    filter missing either placeholder rather than generating a policy that
    silently does the wrong thing.

Group membership is cached for five minutes. Without a cache, every request from
every user becomes an LDAP query, which takes the directory out long before it
troubles the proxy.

## Enabling it

1. Configure and save the settings with *Require sign-in* **off**.
2. Configure your clients to send credentials.
3. Come back, tick *Require sign-in*, save.
4. Review the pending change, then **Apply**.

The console warns before you apply, and again in the change summary, because
this is the one setting that can take an entire network offline.

## Writing rules that name people

Once enabled, [access rules](access-rules.md) can use:

- **Directory user** — matches one signed-in account.
- **Directory group** — matches members of a directory group.

> **Allow** members of `Sales` to reach `dropbox.com` at all times.

Usernames and group names cannot contain brackets, asterisks, backslashes,
quotes or line breaks. Those characters have meaning in LDAP filters, and a
group called `Sales(EU)` would build a filter that does not parse — so the
console refuses them rather than generating a rule that matches nobody.

## What you get in return

Once usernames are flowing:

- [Live Traffic](live-traffic.md) shows the user for each request.
- [Analytics](analytics.md) gains a **Users** tab — top users by requests and
  bandwidth, over any period.
- Blocked-request investigation names a person rather than an address that may
  have moved desk since.

## Turning it off

Untick *Require sign-in*, save and apply. Clients stop being asked for
credentials immediately.

Rules that name users or groups will then refuse to generate, because they
cannot work — you will see a clear error at apply time naming the rule. Disable
or delete those rules first.

## Troubleshooting

| Symptom | Usually means |
|---|---|
| Every user fails to authenticate | User filter does not match your schema — try `(userPrincipalName=%s)` |
| Some users work, others do not | Those users are outside the base DN |
| Group rules never match | Group filter is missing `%u`, or the group base DN is wrong |
| Authentication worked, then stopped | Service account password expired, or the directory is unreachable |
| Everything fails after enabling TLS | Squid does not trust your directory's certificate chain |

The proxy error log records what the helper reported. Read it in the console
under **Logs → Proxy error log**, filtered to warnings and errors — no SSH
needed.

[:material-arrow-right: Reading the logs](logs.md)

## Related

- [Pointing clients at the proxy](../getting-started/clients.md) — sending credentials
- [Access rules](access-rules.md) — rules that name users and groups
- [Security model](../about/security-model.md) — how the credential is protected
