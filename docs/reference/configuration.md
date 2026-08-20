# Configuration File

Settings for the **management console** itself, as distinct from
[Proxy Settings](../guide/proxy-settings.md), which configure Squid.

```text
/etc/cloudinfra/config.yaml
```

The service runs correctly with this file absent or empty — every value has a
built-in default, and the shipped file is entirely comments. You should rarely
need to touch it.

## Format

One `key: value` per line. `#` starts a comment.

```yaml
listen_addr: ":8443"
session_ttl: "4h"
```

Deliberately a flat subset rather than full YAML: it covers everything the
appliance needs without a parser dependency on a service whose argument is a
minimal supply chain.

Unknown keys are **ignored** rather than fatal, so a file written for a newer
release will not stop an older one from starting.

## Environment variables

Any key can be overridden with `CLOUDINFRA_<KEY>` in upper case. Environment
takes precedence over the file.

```bash
CLOUDINFRA_LISTEN_ADDR=":9443"
```

Useful for containers and for testing without editing the file.

## Settings

### Console

| Key | Default | Notes |
|---|---|---|
| `listen_addr` | `:8443` | Bind address and port |
| `tls_cert` | `/etc/cloudinfra/tls/cert.pem` | Serving certificate |
| `tls_key` | `/etc/cloudinfra/tls/key.pem` | Its private key |

Binding to all interfaces is deliberate — a localhost-only default would force
everyone through an SSH tunnel on first launch. Exposure is controlled by the
[security group](ports.md).

Replace the certificate and key with your own and restart the service; the
console does not care where they came from, and
[Administration → Security](../guide/administration.md#security) will show
yours.

### Sessions

| Key | Default | Notes |
|---|---|---|
| `session_ttl` | `8h` | Absolute lifetime |
| `idle_timeout` | `60m` | Signs out an unattended console |

Shorten both if the console is reachable from a wide range. A non-positive value
falls back to the default rather than disabling the timeout.

### Storage

| Key | Default | Notes |
|---|---|---|
| `data_dir` | `/var/lib/cloudinfra` | Databases and snapshots |
| `log_dir` | `/var/log/cloudinfra` | The console's own log |
| `max_analytics_bytes` | `8589934592` | Hard cap on traffic history (8 GiB) |
| `live_buffer_size` | `5000` | Requests held for the live view |

Setting `data_dir` also moves the databases, the ingest position and the
first-boot marker within it. Setting `log_dir` moves the management log.

Analytics live in their own database file so an oversized or corrupted traffic
store can be removed without touching policies, audit history or configuration
versions.

The size cap is what age-based [retention](../guide/administration.md#data--storage)
cannot provide: a traffic spike can fill a volume well inside the retention
window. Analytics degrading is always preferable to a full disk stopping the
proxy.

### Proxy engine

| Key | Default | Notes |
|---|---|---|
| `squid_conf` | `/etc/squid/squid.conf` | The file the generator writes |
| `squid_access_log` | `/var/log/squid/access.log` | Squid's native log |
| `events_log` | `/var/log/squid/cloudinfra-events.log` | The structured log the console reads |
| `squid_port` | `3128` | Where the console expects the proxy |
| `ingest_state` | *inside `data_dir`* | Position in the traffic log |

!!! warning "`squid_port` here is not the proxy's port"
    This tells the **console** where to find the proxy. The port Squid actually
    listens on is set in [Proxy Settings](../guide/proxy-settings.md) and written
    into the generated configuration.

    Changing one without the other makes the console report a healthy proxy as
    unreachable.

### Privileged helper

| Key | Default |
|---|---|
| `priv_socket` | `/run/cloudinfra/priv.sock` |

Where the console reaches the root helper that performs privileged operations.

[:material-arrow-right: Security model](../about/security-model.md)

## Applying a change

```bash
sudo systemctl restart cloudinfra-proxyd
```

The console reads this file only at startup. Restarting it does **not** affect
Squid — the two services are deliberately independent, and proxying continues
uninterrupted.

## Services

| Unit | Runs as | Purpose |
|---|---|---|
| `cloudinfra-proxyd` | `cloudinfra` | The management console |
| `cloudinfra-privhelper` | `root` | Privileged operations, closed verb set |
| `cloudinfra-firstboot` | `root` | One-shot: certificate, admin account, network detection |
| `squid` | `squid` | The proxy itself |

Neither console service depends on `squid`, in either direction. If the console
stops, Squid keeps proxying. If Squid stops, the console still starts to tell you
why.

## Related

- [Proxy Settings](../guide/proxy-settings.md) — configuring Squid rather than the console
- [Ports & security groups](ports.md) — what to expose
- [Architecture](../about/architecture.md) — how the pieces fit
