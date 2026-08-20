# Contact & Support

Need help with your CloudInfra Proxy Manager appliance, or have a pre-sales
question? Our team is here to assist.

## Contact us

Reach our team through our contact page — it goes straight to support:

[Contact Cloud Infrastructure Services :material-open-in-new:](https://cloudinfrastructureservices.co.uk/contact-us/){ .md-button .md-button--primary target="_blank" rel="noopener" }

Prefer email? Use the contact form on that page and your message reaches our
support team directly — we keep our address off these pages to cut down on spam.

## Before you get in touch

To help us resolve your query quickly, please include as much of the following
as you can. All of it is on **Administration → Application** in the console:

- The **Proxy Manager version** and the **Squid version**.
- The **cloud and region**, and the instance size.
- What you were doing, and what happened instead.
- The **health score** and any failing checks, from the Health page.
- For a failed deployment: which **step** failed, and the message it gave —
  Configuration History records both.
- For a traffic problem: the entry from **Live Traffic**, including the rule it
  names.

For anything involving the proxy misbehaving, the relevant lines from
**Logs → Proxy error log**, filtered to warnings and errors, are usually the
single most useful thing you can send.

!!! warning "Review before sharing"
    Proxy logs contain the hostnames your users visited, and configuration
    exports contain your full access policy. Please review anything you send and
    redact what you consider sensitive.

    You do not need to send us a backup file to get help, and we will not ask
    for credentials.

## Try these first

Many questions are answered directly in the documentation:

- [Your First 10 Minutes](getting-started/quickstart.md) — sign in, verify, and
  prove traffic is flowing.
- [Pointing clients at the proxy](getting-started/clients.md) — including the
  troubleshooting table for 403, 407 and timeouts.
- [Applying changes](guide/applying-changes.md) — why a change was rolled back.
- [Health](guide/health.md) — what each check means and what to do about it.
- [Frequently Asked Questions](about/faq.md) — the common ones, including HTTPS
  visibility and cache hit rates.

## Reporting a security issue

If you believe you have found a security vulnerability, please use the contact
page above and say so in the subject line. Please do not open a public issue.

We will acknowledge it, keep you informed while we investigate, and credit you
when it is fixed unless you would rather we did not.

## Feature requests

Feature requests go to the same place and are genuinely read — most of the
[roadmap](about/roadmap.md) came from customers. Saying which of your problems a
feature would solve is more useful than describing the feature, because there is
often a way to solve it today.

---

*CloudInfra Proxy Manager is a product of InfraSOS FZCO, provided by
[Cloud Infrastructure Services](https://cloudinfrastructureservices.co.uk/).*
