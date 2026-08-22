# Pointing Clients at the Proxy

The proxy is a standard HTTP forward proxy on port **3128**. Anything that
speaks proxy settings can use it — there is no agent and nothing to install on
the client.

## Before you start: is the client allowed?

The appliance detects your client network at first boot from the cloud metadata
service and permits traffic from it. A client outside that range gets
`403 Forbidden` — the proxy working correctly, not a fault.

Check the detected range in **Proxy Settings**. If your clients are elsewhere,
add an access rule permitting their network before configuring them.

!!! note "Why the proxy does not simply allow everyone"
    An open proxy is abused within hours of being reachable. The appliance
    ships denying everything except the network it belongs to, and widening
    that is a deliberate act you perform, not a default you inherit.

## Choosing an approach

| Approach | Good for | Trade-off |
|---|---|---|
| **System proxy** | Most environments | Applies to everything on the machine |
| **PAC file** | Mixed traffic, split tunnelling | Needs somewhere to host the file |
| **Environment variables** | Servers, containers, CI | Only applies to tools that read them |
| **Group Policy** | Windows fleets | Windows only |

## System proxy

=== "Windows"

    **Settings → Network & Internet → Proxy → Manual proxy setup**

    - Address: `10.20.1.4`
    - Port: `3128`
    - Tick *Don't use the proxy server for local addresses*

    Or with PowerShell:

    ```powershell
    $key = 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings'
    Set-ItemProperty -Path $key -Name ProxyServer -Value '10.20.1.4:3128'
    Set-ItemProperty -Path $key -Name ProxyEnable -Value 1
    Set-ItemProperty -Path $key -Name ProxyOverride -Value '<local>'
    ```

=== "macOS"

    **System Settings → Network → your interface → Details → Proxies**

    Enable *Web Proxy (HTTP)* and *Secure Web Proxy (HTTPS)*, both pointing at
    `10.20.1.4` port `3128`.

    Or from the terminal:

    ```bash
    networksetup -setwebproxy "Wi-Fi" 10.20.1.4 3128
    networksetup -setsecurewebproxy "Wi-Fi" 10.20.1.4 3128
    ```

=== "Linux desktop"

    **Settings → Network → Network Proxy → Manual**, with the same host and
    port for HTTP and HTTPS.

=== "Firefox"

    Firefox defaults to **Use system proxy settings**, so on Windows it follows
    the settings above and usually needs nothing done to it. Verified against a
    current Firefox: a domain blocked at the proxy was blocked in Firefox
    without touching its configuration.

    It does keep its own settings, though, and they win where they differ. If
    Firefox is not using the proxy when everything else is, check
    **Settings → Network Settings** — someone may have set *No proxy* or a
    manual configuration that points elsewhere.

    One real difference on Windows: *Use system proxy settings* reads the
    Internet Options settings, not the WinHTTP ones. A proxy configured only
    with `netsh winhttp` will not reach Firefox.

    To point it somewhere else deliberately: **Settings → Network Settings →
    Manual proxy configuration**, then tick *Also use this proxy for HTTPS*.

## Environment variables

The convention every command-line tool follows. Note that `https_proxy` takes an
`http://` URL — the proxy is reached over HTTP, and it establishes the TLS
tunnel on your behalf.

```bash
export http_proxy="http://10.20.1.4:3128"
export https_proxy="http://10.20.1.4:3128"
export no_proxy="localhost,127.0.0.1,169.254.169.254,.internal"
```

!!! warning "Always exclude the metadata service"
    `169.254.169.254` is the cloud metadata endpoint. Sending it through a
    proxy breaks instance credentials and role assumption in ways that are
    unpleasant to debug. Put it in `no_proxy` on every cloud instance.

!!! warning "Exclude the appliance's own address too"
    Once a machine is configured to use the proxy, the management console is
    reached *through* it as well — and the console runs on 8443, which the proxy
    refuses to tunnel. `SSL_ports` permits 443 only, so `CONNECT` to any other
    port is denied.

    The symptom is a console that half works: pages already loaded keep
    rendering, and the next action fails with a bare **"Failed to fetch"** and
    nothing in the console's own log, because the request never reached it.

    Add the appliance to the bypass list on any machine you administer it from:

    ```bash
    export no_proxy="localhost,127.0.0.1,169.254.169.254,.internal,10.20.1.4"
    ```

    On Windows, add it to the bypass list rather than relying on
    *Don't use the proxy server for local addresses* — that setting exempts
    names without dots, not private IP addresses.

    This is worth doing on principle rather than as a workaround. Management
    traffic should not depend on the thing it manages: route the console through
    Squid and you lose the console whenever Squid is unhealthy, which is exactly
    when you need it.

Make it permanent in `/etc/environment`, or for one service:

```ini title="/etc/systemd/system/myapp.service.d/proxy.conf"
[Service]
Environment="http_proxy=http://10.20.1.4:3128"
Environment="https_proxy=http://10.20.1.4:3128"
Environment="no_proxy=localhost,127.0.0.1,169.254.169.254,.internal"
```

### Package managers

These do not read the environment reliably, so configure them directly.

=== "apt"

    ```ini title="/etc/apt/apt.conf.d/95proxy"
    Acquire::http::Proxy "http://10.20.1.4:3128";
    Acquire::https::Proxy "http://10.20.1.4:3128";
    ```

=== "yum / dnf"

    ```ini title="/etc/dnf/dnf.conf"
    proxy=http://10.20.1.4:3128
    ```

=== "Docker daemon"

    ```ini title="/etc/systemd/system/docker.service.d/proxy.conf"
    [Service]
    Environment="HTTP_PROXY=http://10.20.1.4:3128"
    Environment="HTTPS_PROXY=http://10.20.1.4:3128"
    Environment="NO_PROXY=localhost,127.0.0.1,169.254.169.254"
    ```

    Then `systemctl daemon-reload && systemctl restart docker`.

## PAC file

A PAC file lets clients decide per destination — useful when internal traffic
should bypass the proxy entirely.

```javascript title="proxy.pac"
function FindProxyForURL(url, host) {
  // Never proxy internal names or the metadata service.
  if (isPlainHostName(host) ||
      dnsDomainIs(host, ".internal") ||
      host === "169.254.169.254" ||
      isInNet(dnsResolve(host), "10.0.0.0", "255.0.0.0")) {
    return "DIRECT";
  }
  return "PROXY 10.20.1.4:3128";
}
```

Host it anywhere clients can reach it and serve it as
`application/x-ns-proxy-autoconfig`. Point clients at the URL, or publish it
through DHCP option 252 or a `wpad` DNS record for automatic discovery.

## Group Policy

**Computer Configuration → Administrative Templates → Windows Components →
Internet Explorer → Proxy Settings**, or the equivalent Edge policies. Modern
Windows also honours a system-wide proxy set by Group Policy for most
applications.

## Confirming it works

=== "Windows"

    ```powershell
    curl.exe -x http://10.20.1.4:3128 -I https://example.com
    ```

    **`curl.exe`, not `curl`.** Windows ships a real curl, but PowerShell
    aliases the name `curl` to its own `Invoke-WebRequest`, which does not
    understand `-x` and fails with a parameter error. It reads like the proxy
    refused the request when nothing ever reached it. The `.exe` is what makes
    PowerShell run the program rather than the alias.

    ```powershell
    Test-NetConnection 10.20.1.4 -Port 3128
    ```

    is the equivalent of `nc -zv` if you want to check the port on its own.

=== "Linux, macOS"

    ```bash
    curl -x http://10.20.1.4:3128 -I https://example.com
    ```

Then check **Live Traffic** in the console. If the request is not there, it did
not reach the proxy — look at the network path rather than at the proxy.

## When authentication is on

If you enable [directory authentication](../guide/directory-authentication.md),
every client must send credentials. Tools take them in the proxy URL:

```bash
export https_proxy="http://alice:password@10.20.1.4:3128"
```

```bash
curl -x http://10.20.1.4:3128 --proxy-user alice:password -I https://example.com
```

Browsers prompt for them and remember them for the session.

!!! danger "Turning authentication on stops clients that are not ready for it"
    Every client must be configured to send credentials before you apply the
    change. Clients that are not will simply stop reaching the internet.

    The automatic rollback protects you from a proxy that fails to start. It
    cannot tell that your clients cannot sign in — from the appliance's point of
    view, refusing an unauthenticated request is correct behaviour.

    Configure the clients first, then enable it.

## Troubleshooting

| Symptom | Usually means |
|---|---|
| Connection times out | Security group or routing — the request never arrived |
| `403 Forbidden` | Client is outside the allowed network, or a rule denied it |
| `407 Proxy Authentication Required` | Authentication is on and no credentials were sent |
| `503 Service Unavailable` | The proxy could not reach the destination — check DNS on the [Health](../guide/health.md) page |
| Works for HTTP, fails for HTTPS | `https_proxy` missing, or set to an `https://` URL instead of `http://` |

For anything that reached the proxy, **Live Traffic** and the request detail
panel will tell you what happened and which rule decided it.
