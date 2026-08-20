# The Generated squid.conf

`/etc/squid/squid.conf` is a build artefact. It is rewritten from your policy on
every [apply](../guide/applying-changes.md), and any edit you make to it is lost
at the next one.

You never need to read it for normal administration. It is documented because a
product that generates your configuration should let you see what it generated,
and because it is useful when comparing against Squid's own documentation.

View it read-only at the bottom of [Proxy Settings](../guide/proxy-settings.md).

## Structure

The generator emits sections in a fixed order, because Squid requires an ACL to
be declared before the `http_access` line that uses it.

1. Header — when it was generated, and by which version
2. Client networks (an include)
3. Listener — port, hostname, DNS, timeouts
4. Port safety ACLs
5. Authentication, if enabled
6. Built-in safety rules
7. Your rules, in order
8. The default deny
9. Cache
10. Logging
11. Your `local.conf` (an include)

## Header

```squid
# =============================================================================
# CloudInfra Squid Proxy - GENERATED CONFIGURATION - DO NOT EDIT
# =============================================================================
# Generated: 2026-08-20T09:14:22Z
# Generator: CloudInfra Proxy Manager 1.0.0
#
# This file is rewritten from policy objects every time changes are applied.
# Any edit made here is lost on the next apply, and the console will warn that
# the file no longer matches what it generated.
#
# To add directives the console does not manage, edit /etc/squid/local.conf
# instead. That file is never generated and never overwritten.
# =============================================================================
```

The console stores a checksum with each version, so an edit made outside the
console can be detected and reported rather than silently overwritten.

## Client networks

```squid
include /etc/squid/conf.d/10-localnet.conf
```

Written at first boot from the cloud metadata service, on top of an RFC 1918
baseline. Detection **widens** the shipped baseline and never narrows it.

Included by explicit path rather than a wildcard, so a missing file is a hard
parse error rather than a proxy that silently denies everyone.

## Port safety

```squid
acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
...
```

!!! note "What is not declared here"
    `CONNECT`, `all`, `manager`, `localhost`, `to_localhost` and `to_linklocal`
    are **predefined in Squid 7** and must not be redeclared. Configurations
    carried over from Squid 3 often declare `acl CONNECT method CONNECT`, which
    is now an error.

## Authentication

Absent unless [directory authentication](../guide/directory-authentication.md)
is enabled — and when it is off, the file says so rather than leaving a silent
gap:

```squid
# --- Authentication ----------------------------------------------------------
# Proxy authentication is off. Clients are identified by address only, and the
# username field in the traffic log stays empty.
```

When enabled:

```squid
auth_param basic program /usr/lib/squid/basic_ldap_auth -v 3 -ZZ \
    -b "DC=example,DC=com" -f "(sAMAccountName=%s)" \
    -D "CN=squid,OU=Service Accounts,DC=example,DC=com" \
    -W /etc/squid/auth-bind.secret -h "dc1.example.com" -p 636
auth_param basic children 20 startup=0 idle=1
auth_param basic realm CloudInfra Proxy
auth_param basic credentialsttl 60 minutes
auth_param basic casesensitive off

acl ci_authenticated proxy_auth REQUIRED
```

Three details worth noting:

- **`-ZZ`, not `-Z`.** The single form falls back to plaintext if TLS cannot be
  established, which would put the service account password on the wire. The
  double form fails instead.
- **`-W`, not the password.** The credential is read from a file that is
  `0640 root:squid`. An argument would be world-readable here and would also
  appear in the process list.
- **`startup=0`.** Squid does not spawn helpers until the first authenticated
  request, so it starts even when the directory is unreachable. An appliance
  that refuses to boot because a domain controller is rebooting would be worse
  than one that fails authentication until it returns.

Group lookup adds:

```squid
external_acl_type ci_group ttl=300 negative_ttl=60 children-max=10 %LOGIN \
    /usr/lib/squid/ext_ldap_group_acl -v 3 -ZZ -b "DC=example,DC=com" \
    -f "(&(objectClass=group)(cn=%g)(member=%u))" ...
```

## Built-in rules

```squid
acl ci_builtin annotate_transaction ci_rule=0
http_access deny !Safe_ports ci_builtin
http_access deny CONNECT !SSL_ports ci_builtin
```

Rule id `0` marks a denial no managed rule caused, so the console can
distinguish *"your policy blocked this"* from *"the proxy refused it"*.

## Your rules

Each rule becomes its ACL declarations followed by one `http_access` line:

```squid
# rule 7: Block social media
# Deny clients in 10.20.0.0/16 to reach .facebook.com at all times.
acl ci_src_7 src 10.20.0.0/16
acl ci_dst_7 dstdomain .facebook.com
acl ci_mark_7 annotate_transaction ci_rule=7
http_access deny ci_src_7 ci_dst_7 ci_mark_7
```

The comment is the same sentence the console showed you before you saved it, so
the file is readable by anyone who knows Squid but not this product.

**The annotation is always last.** It always matches and exists only for its side
effect, so placing it last means it stamps the transaction only when everything
before it matched — that is, when this rule is the one that decided. Placed
first, it would stamp every request the line merely considered.

A rule naming a directory group renders with the authentication requirement
first, because the group helper needs a username to look up:

```squid
acl ci_src_9 external ci_group "Sales"
acl ci_mark_9 annotate_transaction ci_rule=9
http_access allow ci_authenticated ci_src_9 ci_dst_9 ci_mark_9
```

Named lists are written to files under `/etc/squid/lists/` and referenced by
path, because Squid reads large sets from disk far more efficiently than from
inline ACL lines.

## The default

```squid
http_access allow localhost
http_access deny all ci_builtin
```

**The default is deny.** A request reaches the internet because a rule you wrote
allowed it, not because nothing stopped it.

## Logging

```squid
logformat cloudinfra %ts.%03tu\t%tr\t%>a\t...
access_log stdio:/var/log/squid/access.log squid
access_log stdio:/var/log/squid/cloudinfra-events.log cloudinfra
buffered_logs off
```

Two logs: your native `access.log` untouched for existing tooling, and the
structured one the console reads.

`buffered_logs off` means entries reach disk immediately, so live traffic is
actually live rather than appearing in batches.

[:material-arrow-right: Traffic log format](log-format.md)

## Your own directives

```squid
include /etc/squid/local.conf
```

Last, so your directives can override anything above. Never generated, never
overwritten, and never restored by a rollback — so undoing the console's
configuration cannot undo your edits.

!!! warning "It is parsed with everything else"
    A mistake in `local.conf` fails the validation step of your **next** apply,
    which makes the failure appear attached to an unrelated change. If
    validation fails on something innocuous, check `local.conf` first.

## Related

- [Access rules](../guide/access-rules.md) — what generates most of this
- [Applying changes](../guide/applying-changes.md) — how it reaches Squid
- [Traffic log format](log-format.md) — the logging section in detail
