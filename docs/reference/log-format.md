# Traffic Log Format

The appliance writes a **second** access log in a structured format that the
console reads. Your ordinary `access.log` is left in Squid's native format and
untouched, so existing tooling pointed at it keeps working.

| | |
|---|---|
| **Path** | `/var/log/squid/cloudinfra-events.log` |
| **Format name** | `cloudinfra` |
| **Separator** | Tab |
| **Fields** | 18 |
| **Ownership** | `squid:squid`, mode `0640` |

## Why a second log

Squid's native format is designed to be human-readable, which makes it ambiguous
to parse: fields are space-separated, and URLs, user agents and usernames can all
contain spaces. Anything parsing it has to guess.

The structured log removes the guessing. It is tab-separated with a fixed field
count, and every free-text field is percent-encoded by Squid itself.

## The fields

| # | Squid token | Contents |
|---|---|---|
| 1 | `%ts.%03tu` | Timestamp, epoch seconds with milliseconds |
| 2 | `%tr` | Response time, milliseconds |
| 3 | `%>a` | Client address |
| 4 | `%>p` | Client port |
| 5 | `%[un` | Authenticated username, `-` if none |
| 6 | `%Ss` | Squid result code, e.g. `TCP_MISS`, `TCP_DENIED` |
| 7 | `%03>Hs` | HTTP status sent to the client |
| 8 | `%<st` | Bytes received from the destination |
| 9 | `%>st` | Bytes received from the client |
| 10 | `%rm` | Request method |
| 11 | `%[ru` | Request URL |
| 12 | `%[mt` | MIME type |
| 13 | `%Sh` | Hierarchy code — how the request was forwarded |
| 14 | `%<a` | Destination address |
| 15 | `%err_code` | Squid error code, if any |
| 16 | `%err_detail` | Error detail, if any |
| 17 | `%{ci_rule}note` | **The rule that decided this request** |
| 18 | `%[{User-Agent}>h` | Client user agent |

## The `%[` modifier

Fields 5, 11, 12 and 18 use Squid's `%[` prefix, which percent-encodes the value.

This is what makes tab framing safe. A URL containing a tab or a newline arrives
as `%09` or `%0A` and cannot break the record — so a hostile request cannot forge
a log entry or split one into two.

Any consumer must percent-decode these fields before displaying them, and must
**not** decode before parsing structure. Decoding first destroys the framing the
encoding exists to protect.

## Field 17: rule attribution

Squid has no native field recording which `http_access` line decided a request.
This one is created by the configuration generator.

Each generated rule declares an annotation ACL that always matches and exists
only for its side effect, placed **last** on its `http_access` line:

```squid
acl ci_src_7 src 10.20.0.0/16
acl ci_dst_7 dstdomain .facebook.com
acl ci_mark_7 annotate_transaction ci_rule=7
http_access deny ci_src_7 ci_dst_7 ci_mark_7
```

Because it is last, it only stamps the transaction when every preceding ACL on
that line matched — that is, when this rule is the one that decided. Placed
first, it would stamp every request the line merely considered, and attribution
would be meaningless.

| Value | Means |
|---|---|
| A number | The rule with that identifier decided the request |
| `0` | The appliance's own built-in policy decided it |
| `-` | No annotation — an allow that matched no managed rule |

This works only because the appliance owns both the configuration generator and
the log reader.

## A sample record

Shown with tabs as `→` for readability, and wrapped:

```text
1755600123.456→142→10.20.1.55→51234→-→TCP_DENIED→403→3521→412→GET→
http%3A%2F%2Fwww.facebook.com%2F→text%2Fhtml→HIER_NONE→-→
ERR_ACCESS_DENIED→-→7→Mozilla%2F5.0%20(Windows%20NT%2010.0)
```

Read as: a request from `10.20.1.55` to `www.facebook.com` was denied with 403 by
**rule 7**, taking 142 ms.

## Reading it yourself

The format is deliberately easy to consume.

```bash
# Every denied request, with the rule that denied it
awk -F'\t' '$6 ~ /DENIED/ { print $3, $11, "rule=" $17 }' \
    /var/log/squid/cloudinfra-events.log
```

```bash
# Bandwidth by client
awk -F'\t' '{ b[$3] += $8 } END { for (c in b) print b[c], c }' \
    /var/log/squid/cloudinfra-events.log | sort -rn | head
```

Remember to percent-decode field 11 before displaying URLs.

!!! warning "The field order is a contract"
    The format string and the console's parser are two halves of one agreement.
    If you change the `logformat` line, the console stops being able to read its
    own traffic log.

    If you need a different format for your own tooling, add a **third**
    `access_log` line in `/etc/squid/local.conf` rather than altering this one.

## Rotation

The log rotates daily and is kept for 14 days, with `squid -k rotate` issued so
Squid reopens its files rather than continuing to write to a renamed one. The
[Health page](../guide/health.md) warns if rotation stops running.

The console's [log viewer](../guide/logs.md) reads rotated and compressed
archives as well as the current file.

## Related

- [Live traffic](../guide/live-traffic.md) — the parsed view
- [Access rules](../guide/access-rules.md) — where rule identifiers come from
- [The generated squid.conf](generated-config.md) — where the format is declared
