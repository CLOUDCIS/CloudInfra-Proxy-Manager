# HTTPS and CONNECT

**HTTPS works out of the box. There is nothing to configure.**

This is the question we are asked most often, so it is worth answering before
anything else. The proxy carries HTTPS on the same port as HTTP, with no
additional setup, no certificate and no separate listener.

If HTTPS is not working for you, the cause is almost always the client. Check
that `https_proxy` is set to an `http://` URL — see
[Pointing Clients at the Proxy](../getting-started/clients.md).

The rest of this page explains what the proxy can see of that traffic, because
it determines what filtering is possible and answers several questions that
otherwise look like product limitations.

## How it works

For an HTTP request, the client sends the whole request to the proxy:

```
GET http://example.com/products/index.html HTTP/1.1
```

The proxy sees the host, the path, the query string and the headers. It can
cache the response, filter on any part of the URL, and log all of it.

For an HTTPS request, the client first asks the proxy to open a tunnel:

```
CONNECT example.com:443 HTTP/1.1
```

The proxy opens a TCP connection to `example.com:443` and copies bytes between
the two ends without understanding them. The client and the website negotiate
TLS **through** the tunnel, so the encryption is genuinely end to end. Neither
the proxy nor anyone on the path can read the contents.

## What the proxy can and cannot see

| | HTTP | HTTPS |
|---|---|---|
| Hostname | Yes | **Yes** |
| Port | Yes | Yes |
| Path and query string | Yes | **No** |
| Request and response headers | Yes | No |
| Response body | Yes | No |
| Bytes transferred | Yes | Yes |
| Response caching | Yes | No |

The hostname is visible because the client has to say where to connect.
Everything after it is inside the encrypted session.

## What this means for filtering

You **can** block or allow an HTTPS site by domain. This is what
[Access Rules](../guide/access-rules.md) and [URL Filtering](../guide/url-filtering.md)
do, and they work identically for HTTP and HTTPS.

You **cannot** block one page or path on an HTTPS site.

!!! info "Why a URL pattern does not match HTTPS traffic"
    For a CONNECT request, the entire URL Squid has to match against is
    `example.com:443`. The path is not merely hidden from the rule — it has not
    been sent yet, and will not be until after the encrypted session is
    established.

    There is no ordering or syntax fix. The information is not present.

A common request is to block a site while permitting one path on it — deny
`bitbucket.org` but allow `bitbucket.org/myteam/myrepo`. Over HTTPS this is not
possible without decrypting the traffic. The practical alternatives:

- allow the domain and control access at the service itself, using its own
  permissions
- use the service's IP allowlisting, pointed at your proxy's egress address
- accept domain-level granularity

This is a property of TLS, not of this appliance or of Squid. Any proxy that
filters HTTPS by path is decrypting the traffic.

## Log entries look different

An HTTPS request is logged as a tunnel, and it is logged when it **closes**, not
when it opens:

```
1755678901.234    142 10.0.1.55 TCP_MISS/200      412 GET     http://example.com/  - ...
1755678912.881  30512 10.0.1.55 TCP_TUNNEL/200 184320 CONNECT example.com:443     - ...
```

Two consequences:

- A long-lived HTTPS connection does not appear in the traffic log until it
  ends. A video stream running for an hour shows up an hour late. This is why
  [Live Traffic](../guide/live-traffic.md) supplements the log with a separate
  view of connections currently open.
- The byte count covers the whole tunnel, not one page.

See [Traffic Log Format](../reference/log-format.md) for the full field list.

## Caching

Encrypted responses cannot be cached, because the proxy cannot read them. As the
web is now overwhelmingly HTTPS, expect a low cache hit rate.

The value of a proxy in a modern deployment is centralised egress control,
policy and visibility rather than bandwidth saving. The
[Dashboard](../guide/dashboard.md) reports the hit rate; a low figure is normal
and not a sign of misconfiguration.

## Decrypting HTTPS

Squid can decrypt HTTPS by presenting the client a certificate it generates
itself — "SSL bump", or TLS interception. The engine in this image is built with
OpenSSL support and the certificate generator is installed, so it is technically
possible to configure by hand.

The console does not manage it and we do not currently document it. Before you
go looking, understand what it commits you to:

- **Every client must trust a certificate authority you operate.** On unmanaged
  or personal devices this is impractical.
- **Certificate pinning breaks.** Most mobile apps, many installers, and `git`
  over HTTPS will fail outright, often without a useful error message.
- **You become responsible for decrypted traffic** — its security, and in many
  jurisdictions telling your users it is happening.

If URL-level control over HTTPS is a requirement for you, this is the only route
to it. [Tell us](../support.md) — it is on the roadmap as a decision rather than
a feature, and knowing who needs it is what moves it.
