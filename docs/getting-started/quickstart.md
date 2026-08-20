# Your First 10 Minutes

The proxy is already running by the time you read this. This page gets you
signed in, proves traffic is flowing, and leaves you with one access rule of
your own.

## 1. Find your password

The initial password is random, unique to this instance, and written to the
instance console output — so you can retrieve it by proving you control the
instance, without SSH.

=== "AWS"

    ```bash
    aws ec2 get-console-output --instance-id i-0123456789abcdef0 \
        --output text | grep -A6 "CloudInfra Proxy Manager is ready"
    ```

    Or in the EC2 console: select the instance, then
    **Actions → Monitor and troubleshoot → Get system log**.

=== "Azure"

    In the portal, select the VM, then **Help → Boot diagnostics → Serial log**.

    ```bash
    az vm boot-diagnostics get-boot-log --name proxy-01 --resource-group my-rg
    ```

=== "Google Cloud"

    ```bash
    gcloud compute instances get-serial-port-output proxy-01 --zone us-east1-b
    ```

You are looking for:

```text
================================================================
 CloudInfra Proxy Manager is ready

   Console:  https://10.20.1.4:8443
   Username: admin
   Password: <generated for this instance>
```

!!! info "If you have SSH access"
    The same details are in `/var/lib/cloudinfra/initial-credentials.txt`,
    readable by root only. That file deletes itself once you have changed the
    password.

## 2. Sign in

Open `https://<instance-ip>:8443`.

Your browser will warn about the certificate. That is expected — it was
generated on the instance at first boot, because there is no way to ship a
trusted certificate in an image without also shipping its private key to every
customer.

!!! tip "Confirm the warning is about your appliance"
    Once signed in, go to **Administration → Security** and compare the SHA-256
    fingerprint shown there with the one your browser reported. Matching
    fingerprints mean the warning was about a self-signed certificate, not about
    something between you and the appliance.

    You can replace the certificate with your own at any time — see
    [Administration](../guide/administration.md).

**You will be asked to change the password immediately.** That is deliberate:
the generated password appeared in your instance console output, which is a
bootstrap token rather than a credential. Choose something at least 12
characters.

## 3. Confirm the proxy is healthy

You land on the **Dashboard**. The appliance header across the top shows the
hostname, address, Squid version, uptime and a health pill.

Go to **Health**. You should see a score of 100 and fourteen checks, all green.

If the score is lower, the page tells you exactly what took the points off and
what to do about it — every deduction is a named check with a published weight.
A brand-new appliance with no traffic yet is normal and still scores 100.

[:material-arrow-right: What each health check means](../guide/health.md)

## 4. Send a request through it

From a machine that can reach the proxy:

```bash
curl -x http://<instance-ip>:3128 -I https://example.com
```

You should get `HTTP/1.1 200 OK` (or `200 Connection established` for the
tunnel). Now go to **Live Traffic** in the console — your request is there,
within a second or two.

??? failure "It returned 403 Forbidden"
    The proxy accepts traffic from the client network it detected at first boot.
    If your test machine is outside that range, it is denied by policy — which
    is the proxy working correctly.

    Check **Proxy Settings** for the detected network, and see
    [Pointing clients at the proxy](clients.md) for how to widen it.

??? failure "It timed out"
    That is a network path problem rather than a proxy problem: the request
    never arrived. Check the security group allows 3128 from your test machine,
    and that it has a route to the instance.

## 5. Write one rule

Go to **Access Rules → Add rule**. The wizard asks four questions and then shows
you the sentence it will enforce:

> Deny clients in 10.20.0.0/16 to reach facebook.com at all times.

Save it. Notice what happens: the rule is **staged**, not applied. A banner
appears saying you have a pending change.

This is the core safety property of the product. Nothing you do in the console
reaches Squid until you press **Apply changes**, and when you do, the change is
validated, snapshotted, applied and health-checked — and rolled back
automatically if the proxy does not come back healthy.

Press **Apply changes** and watch it run through the steps.

Now test it:

```bash
curl -x http://<instance-ip>:3128 -I https://facebook.com
```

You get `403 Forbidden`. Go to **Live Traffic**, click the blocked request, and
the detail panel names the rule that blocked it — not just that something did.

[:material-arrow-right: How applying changes works](../guide/applying-changes.md)

## What next

<div class="grid cards" markdown>

-   :material-lan-connect: __Get your clients using it__

    ---

    Browser settings, system proxy, PAC files and the WPAD approach.

    [:material-arrow-right: Pointing clients at the proxy](clients.md)

-   :material-filter: __Block categories, not just domains__

    ---

    Build reusable allowlists and blocklists rather than one rule per site.

    [:material-arrow-right: URL filtering](../guide/url-filtering.md)

-   :material-account-key: __Identify people, not addresses__

    ---

    Authenticate against Active Directory, Entra ID or OpenLDAP.

    [:material-arrow-right: Directory authentication](../guide/directory-authentication.md)

-   :material-content-save: __Protect your configuration__

    ---

    Take a backup you can restore onto a rebuilt appliance.

    [:material-arrow-right: Backup & restore](../guide/backup-restore.md)

</div>
