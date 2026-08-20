# Custom Block Page

When a request is denied, the user sees Squid's built-in error page. You can
replace it with your own, or redirect the browser to a page you host.

Read the HTTPS caveat before choosing — it affects most modern traffic, and it
is the reason this is less useful than it first appears.

## Redirecting to your own page

```
acl blocked_sites dstdomain .example-blocked.com

deny_info 302:https://intranet.example.com/blocked?url=%u blocked_sites
http_access deny blocked_sites
```

`deny_info` must name the same ACL as the `http_access deny` line it belongs to.
That is how Squid knows which denial gets which page.

Because the `http_access` line is involved, this cannot be done entirely in
`local.conf` on a console-managed appliance — see
[Configuring Squid Directly](index.md). The `deny_info` and `acl` lines are fine
there; the deny rule itself belongs in the console.

### Available macros

| Macro | Value |
|---|---|
| `%u` | The full requested URL |
| `%U` | The requested URL without the query string |
| `%i` | Client IP address |
| `%a` | Username, if authenticated |
| `%M` | Request method |
| `%T` | Timestamp |
| `%o` | Message returned by an external ACL |

Your page can read these from its query string to show the user what they tried
to reach and who to ask about it.

!!! warning "Allow the block page itself"
    If the page is hosted somewhere the proxy also filters, the redirect is
    blocked too and the user sees nothing useful. Add an allow rule for its
    domain, above the deny.

## A custom error page served by the proxy

Serve the page from the proxy itself, with no dependency on another host.

```bash
sudo cp /usr/share/squid/errors/en/ERR_ACCESS_DENIED \
        /usr/share/squid/errors/en/ERR_CUSTOM_BLOCKED
sudo nano /usr/share/squid/errors/en/ERR_CUSTOM_BLOCKED
```

The file is HTML and takes the same macros. Then:

```
deny_info ERR_CUSTOM_BLOCKED blocked_sites
```

Files under `/usr/share/squid/errors/` are replaced when the proxy engine is
upgraded. Keep your copy elsewhere, or point `error_directory` at a directory of
your own:

```
error_directory /etc/squid/errors
```

Setting `error_directory` disables Squid's automatic language negotiation — every
user gets that one directory regardless of their browser language.

## Closing the connection instead

```
deny_info TCP_RESET malware_sites
```

Useful against automated clients, which retry a 403 but usually give up on a
reset. Less useful for people, who get an unexplained failure.

## The HTTPS caveat

!!! danger "A custom block page usually does not appear for HTTPS sites"
    When the proxy denies an HTTPS request, it is denying a `CONNECT` before any
    tunnel exists. There is no HTTP session for a redirect to act on, so the
    browser does not follow the 302 — it reports that the connection failed.

    The user sees a generic browser error such as
    `ERR_TUNNEL_CONNECTION_FAILED`, not your page.

This is inherent to how HTTPS is proxied — see
[HTTPS and CONNECT](https-and-connect.md) — and applies to every proxy that is
not decrypting traffic.

| Traffic | Result |
|---|---|
| HTTP site blocked | Your page is shown |
| HTTPS site blocked | Browser connection error |
| HTTP request that would have redirected to HTTPS | Your page is shown |

Because most sites are HTTPS, treat a custom block page as a useful addition for
the cases where it works rather than as your main way of telling users what
happened. Pair it with an internal note explaining what is filtered and who to
contact.

The [Blocked Requests](../guide/analytics.md) view shows you what was denied
regardless of what the user saw.

## Testing

```bash
# HTTP - expect 302 with your Location header
curl -x http://10.0.1.20:3128 -I http://blocked.example.com

# HTTPS - expect 403 on the CONNECT, and no usable page
curl -x http://10.0.1.20:3128 -I https://blocked.example.com
```

## Common mistakes

| Symptom | Cause |
|---|---|
| Built-in page still shown | `deny_info` names a different ACL from the `http_access deny` |
| Redirect fails or loops | The block page's own domain is filtered — allow it |
| Nothing shown for HTTPS | Expected. See the caveat above |
| Custom page lost after an update | Files under `/usr/share/squid/errors/` are replaced on upgrade |
