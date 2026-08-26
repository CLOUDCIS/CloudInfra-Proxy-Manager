# Applying Changes

Nothing you do in the console reaches Squid immediately. Every edit is
**staged**, and reaches the proxy only when you press **Apply changes** — at
which point it goes through a pipeline designed around a single assumption:
sooner or later, a change will be wrong.

## Why it works this way

Editing `squid.conf` by hand has a well-known failure mode. One typo, the reload
fails, and the proxy is down for everyone until somebody notices and works out
which line broke it. On a remote appliance, "somebody notices" often means a
helpdesk queue.

So the console never lets a configuration reach Squid without proving it loads
first, and never applies one without keeping the previous one to hand.

## Staged changes

As you add rules, edit lists or change settings, a banner appears at the top of
the console:

> **3 pending changes** — Review · Apply changes

**Review** lists exactly what will change, in the same plain language the rules
themselves use. Nothing is running yet; the proxy is still enforcing the
configuration it had before you started.

This means you can make a set of related changes — a new list, two rules that
use it, and a settings tweak — and apply them together, rather than putting the
proxy through four separate deployments.

## What Apply does

Applying runs ten steps, streamed to the screen as they happen, so a slow apply
is visibly different from a stuck one.

| Step | What it does |
|---|---|
| **Render** | Generates `squid.conf` and the list files from your policy |
| **Stage** | Writes them to a staging directory Squid is not reading |
| **Validate** | Runs Squid's own parser over the staged file |
| **Classify** | Decides whether a reload will do, or a restart is needed |
| **Snapshot** | Copies the running configuration aside |
| **Install** | Moves the staged files into place atomically |
| **Activate** | Reloads or restarts Squid |
| **Health gate** | Four checks that the proxy actually works |
| **Commit** | Records the new version |
| **Roll back** | Only if the health gate fails |

### Validate

The staged configuration is checked with `squid -k parse` — Squid's own parser,
the same binary that will run it. Not a lookalike, not a regular expression: if
Squid will not load it, this step says so and nothing further happens.

A failure here is completely safe. The proxy is still running the previous
configuration and has not been touched.

### Reload or restart

Most changes apply without dropping a single connection. Some cannot: changing
the **listening port** or the **cache directory** requires a full restart, which
interrupts every active connection.

The console decides which is needed by comparing the live configuration with the
candidate, and it errs toward caution — wrongly reloading when a restart was
needed would leave the proxy running a configuration you cannot see. When a
restart is required, the outcome panel says so explicitly, so an interruption is
never a surprise.

### The health gate

This is the step that makes rollback meaningful. After activating, four checks
run in order:

1. **The service is active** — systemd reports it running.
2. **The port is listening** — something is accepting connections on it.
3. **Squid is answering** — its Cache Manager responds, proving it is servicing
   requests rather than merely holding a socket open.
4. **A real request goes through** — an actual HTTP request is proxied end to
   end.

The fourth check matters most. A proxy that has started but refuses every
request would pass the first three and fail its users. The console runs a
throwaway origin server on the appliance itself for this, so the check needs no
internet access — a gate that probed an external URL would roll back a perfectly
good configuration every time somebody's NAT gateway hiccupped.

!!! note "Authentication and the health gate"
    The end-to-end request is deliberately unauthenticated. When
    [directory authentication](directory-authentication.md) is enabled, Squid
    answers `407 Proxy Authentication Required` — and that counts as a pass. It
    proves Squid is running, parsing its configuration and applying policy,
    which is exactly what the gate is asking.

### Rollback

If the health gate fails, the snapshot taken earlier is restored and Squid is
reloaded again. You end up back where you started, and the outcome panel tells
you which check failed and why.

There is one outcome worse than a failed apply, and the console distinguishes
it: if the rollback *itself* fails, the message says so plainly rather than
claiming a clean recovery. That situation needs a person, and pretending
otherwise would be the worst thing the software could do.

## Reading the outcome

| Outcome | Meaning |
|---|---|
| <span class="pill-allow">Applied</span> | The proxy is running your new configuration |
| <span class="pill-neutral">Rolled back</span> | Something failed; the proxy is running the previous configuration |
| <span class="pill-block">Not applied</span> | Stopped before anything changed — usually validation |

An applied change also says whether connections were interrupted:

> Version 7 · applied without interrupting connections.

## Configuration history

Every apply is recorded — successful, rolled back or failed — under
**Configuration History**:

- **What changed**, as a policy-level difference. Not a diff of the generated
  file: nobody wrote that file, so a line-by-line comparison of it would
  describe the generator rather than your decision. The console shows added,
  removed, changed and reordered *rules and lists*.
- **Who applied it** and when.
- **The result**, including why a rollback happened.

Any version that actually reached the proxy can be **restored**. Restoring
stages the earlier policy — it does not apply it. It then goes through exactly
the same pipeline as any other change, because a restore is a deployment and
deserves the same gate.

## When Apply is unavailable

If the privileged helper is not answering, the console reports that Apply is
unavailable and explains why, rather than offering a control that cannot work.
You can still edit policy; it simply stays staged until deployment is possible
again.

## Your own directives

The generator owns `/etc/squid/squid.conf` and rewrites it on every apply. If
you need Squid directives the console does not manage, put them in:

```text
/etc/squid/local.conf
```

That file is included by the generated configuration, is never generated, and is
never overwritten or restored — so a rollback of the console's configuration
cannot undo your edits.

!!! warning "local.conf is validated with everything else"
    A mistake in `local.conf` fails the validation step of your *next* apply,
    because Squid parses the whole configuration including the include. That is
    correct behaviour, but it means the failure appears attached to an unrelated
    change. If validation fails on a change that looks innocuous, check
    `local.conf`.

!!! warning "A workaround here outlives the problem"
    Because this file is never overwritten, anything added to work around a
    problem stays after the problem is fixed, and keeps taking effect. It is
    included *above* the generated access rules, so an `http_access` line here
    can override policy set in the console without appearing anywhere in it —
    including one that would otherwise require users to sign in.

    If a rule in the console does not behave the way it reads, check this file
    first, and remove anything that is no longer earning its place.

## Related

- [Access rules](access-rules.md) — what you are usually applying
- [Health](health.md) — the checks in more detail
- [Backup & restore](backup-restore.md) — carrying configuration between appliances
- [The generated squid.conf](../reference/generated-config.md) — what the renderer produces
