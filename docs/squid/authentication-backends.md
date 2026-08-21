# Authentication Backends

The console manages authentication against Active Directory, Entra Domain
Services and OpenLDAP — see
[Directory Authentication](../guide/directory-authentication.md). That is the
supported route, and the one to use if your directory is one of those.

Squid also ships helpers for RADIUS and for a local password file. They are
installed and working on the image; the console does not manage them. This page
covers configuring them by hand, and the limits of doing so.

## What is installed

All helpers live in `/usr/lib/squid/`.

| Backend | Helper | Console support |
|---|---|---|
| LDAP / Active Directory | `basic_ldap_auth` | **Yes** — [use the console](../guide/directory-authentication.md) |
| Directory groups | `ext_ldap_group_acl` | **Yes** |
| RADIUS | `basic_radius_auth` | No — planned |
| Local password file | `basic_ncsa_auth` | No — planned |
| PAM, SASL, getpwnam | `basic_pam_auth`, `basic_sasl_auth`, `basic_getpwnam_auth` | No |
| Kerberos / Negotiate | `negotiate_kerberos_auth` | No |

!!! tip "If a helper appears not to respond, check the path first"
    Older Squid packages installed helpers in `/usr/lib/squid3/`, and that path
    is still in many tutorials and wiki pages. **It does not exist on this
    image.** Running a program that is not there produces silence, which looks
    exactly like a backend timeout.

    ```bash
    ls /usr/lib/squid/
    ```

## Read this before you start

**Squid accepts only one Basic authentication backend.** There can be a single
`auth_param basic program` line. If directory authentication is switched on in
the console, turn it off before configuring RADIUS or a password file by hand —
otherwise two helpers are declared and the result is undefined.

**Authentication does not work with transparent interception.** An intercepted
client does not know it is talking to a proxy, so it never sends credentials.

**Basic authentication is base64, not encryption.** Between client and proxy on
a private network that is normally acceptable; across an untrusted one it is
not.

!!! warning "The access rule cannot live in `local.conf`"
    Every authentication setup needs two things: the helper, and an
    `http_access` rule that requires it. The helper goes in `local.conf` quite
    happily. **The access rule does not** — `local.conf` is included after the
    default `http_access deny all`, so a rule there is never reached. See
    [Configuring Squid Directly](index.md).

    On an appliance running the console this is a real limit: the rule would
    have to go in `squid.conf`, which the console regenerates on every apply.
    A hand-configured backend therefore cannot be wired up permanently while
    the console is managing configuration.

    **On an image without the console**, `squid.conf` is yours and everything
    below works normally.

    If you need RADIUS on a console-managed appliance,
    [tell us](../support.md) — it is on the roadmap and knowing who wants it is
    what moves it up.

---

## RADIUS

### 1. Create the helper's configuration file

Put the shared secret in a file rather than on the command line. Helper
arguments are visible to every user on the instance through `ps`.

```bash
sudo tee /etc/squid/radius.conf >/dev/null <<'CONF'
server radius.example.com
secret your-shared-secret
port 1812
identifier squid-proxy
CONF

sudo chown root:squid /etc/squid/radius.conf
sudo chmod 640 /etc/squid/radius.conf
```

Add the proxy's address as a client on the RADIUS server, with the same shared
secret. `identifier` is the NAS-Identifier the server will see.

### 2. Test before touching Squid

```bash
sudo /usr/lib/squid/basic_radius_auth -f /etc/squid/radius.conf
```

Type `username password` and press Enter. You should get `OK` or `ERR` within a
second or two. Ctrl-D to exit.

| Result | Meaning |
|---|---|
| `OK` or `ERR` | Working — the helper reached the server |
| Nothing at all | The helper is not running. Check the path |
| Hangs | Server unreachable. Check UDP 1812 outbound and that the proxy is a registered client |

### 3. Configure Squid

```
auth_param basic program /usr/lib/squid/basic_radius_auth -f /etc/squid/radius.conf
auth_param basic children 20 startup=0 idle=1
auth_param basic realm CloudInfra Proxy
auth_param basic credentialsttl 1 hour
auth_param basic casesensitive off

acl authenticated proxy_auth REQUIRED
http_access allow authenticated
```

`startup=0` means Squid starts even when the backend is unavailable and spawns
helpers on first use. A proxy that refuses to start because a RADIUS server is
rebooting is worse than one that fails sign-in until it returns.

### Groups are not available with RADIUS

`basic_radius_auth` returns only success or failure. RADIUS attributes such as
`Filter-Id` and `Class` are discarded, so group-based rules cannot work. Write
rules against individual usernames, or use
[directory authentication](../guide/directory-authentication.md), where group
membership is a first-class feature.

---

## Local password file

Best for a small, fixed set of users with no directory behind them.

### 1. Create the file

`htpasswd` comes from `apache2-utils` on Ubuntu and Debian.

```bash
sudo apt-get install -y apache2-utils

sudo htpasswd -c /etc/squid/passwd alice    # -c creates the file
sudo htpasswd    /etc/squid/passwd bob      # omit -c to add more

sudo chown root:squid /etc/squid/passwd
sudo chmod 640 /etc/squid/passwd
```

`-c` overwrites an existing file. Use it once.

### 2. Configure Squid

```
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 20 startup=0 idle=1
auth_param basic realm CloudInfra Proxy
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive off

acl authenticated proxy_auth REQUIRED
http_access allow authenticated
```

### 3. Test

```bash
sudo /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
```

Type `alice yourpassword` and press Enter. `OK` means the file and hash are
good.

---

## Verifying from a client

```bash
curl -x http://alice:password@10.0.1.20:3128 -I https://example.com
```

!!! tip "On Windows, run `curl.exe`"
    PowerShell aliases the bare name `curl` to `Invoke-WebRequest`, which
    does not understand `-x` and fails with a parameter error rather than a
    proxy error. See [Confirming it works](../getting-started/clients.md#confirming-it-works).

Without credentials you should get `407 Proxy Authentication Required`. The
username then appears in the traffic log in place of the `-` shown for anonymous
requests — see [Logs](../guide/logs.md).

## Troubleshooting

| Symptom | Cause |
|---|---|
| Helper produces no output at all | Wrong path — check `ls /usr/lib/squid/` |
| Everyone gets 407 no matter what | The `http_access allow` is below `http_access deny all` |
| Nobody is ever prompted | An earlier rule allows by address; it matches first |
| Works on the command line, fails in Squid | The helper cannot read its credential file — group `squid`, mode 0640 |
| Sign-in prompt on every request | `credentialsttl` too low, or the client is not caching |
| Intermittent failures under load | Raise `children`; each concurrent sign-in needs a helper process |
| Two backends declared | Directory authentication is still enabled in the console |
