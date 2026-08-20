# Backup & Restore

A backup captures the configuration somebody decided on — the access rules,
lists and proxy settings — so it can be restored onto this appliance or a
different one.

## What a backup contains

| Included | Not included |
|---|---|
| Access rules | Administrator passwords |
| Domain and network lists | Session keys |
| Proxy settings | TLS certificates |
| Retention settings | Traffic history |
| Where and when it was taken | The generated `squid.conf` |

Two deliberate omissions are worth explaining.

**No credentials.** Nothing in a backup is a secret. A backup that leaks costs
you your policy, not your appliance — which makes it safe to store in ordinary
version control or send to support.

**No `squid.conf`.** The generated configuration is a build artefact; nobody
wrote it. Backing it up would preserve the generator's output rather than your
decisions, and it would be tied to the machine that produced it.

## Why it restores onto a different appliance

This is the case backups are actually taken for: you rebuild from a newer image
and want your policy back.

A filesystem-level backup carries paths, user IDs, cache directories and a
`squid.conf` belonging to the machine it came from. This carries none of that —
just the policy — so restoring onto a fresh appliance works.

## Taking one

**Backup & Restore → Download backup.**

Optionally set a **passphrase**. Without one you get a readable JSON document you
can diff and keep in version control. With one, the file is encrypted with
AES-256-GCM using a key derived from your passphrase with Argon2id.

!!! warning "There is no passphrase recovery"
    The passphrase is not stored anywhere. If you lose it, the backup cannot be
    opened by anyone, including us.

The download is named for the date it was taken, and the fact that a backup was
made — and whether it was protected — is recorded in the
[audit trail](administration.md#audit-trail).

## Restoring

**Backup & Restore → Restore from a backup.**

1. Choose the file, and enter its passphrase if it has one.
2. Press **Check the file**. Nothing changes yet.
3. Review what the file contains — when it was taken, by whom, from which
   appliance, and how many rules and lists.
4. Press **Restore this backup**.

The check step is inert by design. A restore replaces your entire policy, and
discovering you uploaded last year's backup should come from a summary, not from
your proxy.

### What restoring does

It **stages** the policy. Nothing reaches Squid until you press **Apply
changes**, and then it goes through the same validate, snapshot and health-gate
pipeline as any other change — because a restore is a deployment.

[:material-arrow-right: How applying works](applying-changes.md)

### One thing that is deliberately not restored

The **listening port**.

Your appliance's port is set by its image and its security group. Silently moving
it because a backup from another machine said so would take the proxy off the
network its clients use, and the rollback could not detect that — from the
appliance's point of view, listening on the new port would be a complete success.

Everything else in the settings is restored.

## Errors you might see

| Message | Means |
|---|---|
| *That backup is protected. Enter its passphrase* | Encrypted file, no passphrase given — a prompt, not a failure |
| *That passphrase did not open the backup* | Wrong passphrase, or the file was altered |
| *That file is not a CloudInfra Proxy Manager backup* | Wrong file |
| *Written by a newer version* | Taken from an appliance running a newer release |

Encrypted backups are authenticated, so a file that has been modified will not
open even with the correct passphrase. A wrong passphrase and a tampered file
give the same message deliberately — distinguishing them would be a tampering
oracle.

## A sensible routine

- Take one **before** any significant policy change. It is faster than
  reconstructing what you had.
- Take one **after** you are happy with a configuration, so you have a known-good
  state.
- Keep them somewhere that survives the appliance. A backup on the instance is
  no help when the instance is the problem.
- Use a passphrase if they leave your control. The file contains your full access
  policy, which is not a secret but is not public either.

!!! tip "Version control works well"
    An unencrypted backup is indented JSON. Committing it gives you history,
    diffs and review on your proxy policy for free — and it contains no
    credentials, so there is nothing dangerous about doing so.

## Related

- [Applying changes](applying-changes.md) — how a restored policy is deployed
- [Configuration history](applying-changes.md#configuration-history) — rolling back one change rather than restoring everything
- [Security model](../about/security-model.md) — what is protected and how
