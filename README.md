# Homelab Internal PKI: Private CA + ACME on Proxmox

Build a private two-tier certificate authority in a Proxmox LXC container, issue certificates automatically with ACME, and put self-hosted apps behind a single HTTPS reverse proxy, so internal services get trusted HTTPS and browsers stop showing "Not secure".

**Stack:** Proxmox VE 9 · Debian 13 LXC containers · [Smallstep `step-ca`](https://smallstep.com/docs/step-ca/) · [Caddy](https://caddyserver.com/) · Ubiquiti UniFi (DNS/DHCP)

> **Disclaimer:** This is a homelab guide, not a hardened enterprise PKI. Verify commands against the current upstream documentation (linked in [References](#references)) before running them. Tool versions and menu locations change.

---

## Contents

1. [Architecture](#architecture)
2. [Placeholders used in this guide](#placeholders-used-in-this-guide)
3. [Part 1 – CA container](#part-1--ca-container)
4. [Part 2 – Install step-ca](#part-2--install-step-ca)
5. [Part 3 – Initialize a two-tier CA and take the root key offline](#part-3--initialize-a-two-tier-ca-and-take-the-root-key-offline)
6. [Part 4 – Run step-ca as a service](#part-4--run-step-ca-as-a-service)
7. [Part 5 – Enable ACME](#part-5--enable-acme)
8. [Part 6 – Proxmox web UI certificate](#part-6--proxmox-web-ui-certificate)
9. [Part 7 – Trust the root CA on client devices](#part-7--trust-the-root-ca-on-client-devices)
10. [Part 8 – Central reverse proxy (Caddy)](#part-8--central-reverse-proxy-caddy)
11. [Part 9 – Move apps behind the proxy](#part-9--move-apps-behind-the-proxy)
12. [Part 10 – Apps that already serve HTTPS (Nessus example)](#part-10--apps-that-already-serve-https-nessus-example)
13. [Adding a new app (checklist)](#adding-a-new-app-checklist)
14. [Troubleshooting and lessons learned](#troubleshooting-and-lessons-learned)
15. [Maintenance](#maintenance)
16. [Security notes and trade-offs](#security-notes-and-trade-offs)
17. [References](#references)

---

## Architecture

```
                    ┌──────────────────────────────┐
                    │ CA container (step-ca)        │
                    │ ca.home.arpa                  │
                    │ intermediate key (online)     │
                    │ root key: OFFLINE (NAS + pw   │
                    │ manager, stored separately)   │
                    └──────────────▲───────────────┘
                                   │ ACME (http-01)
         ┌─────────────────────────┼──────────────────────────┐
         │                         │                          │
┌────────┴────────┐      ┌─────────┴─────────┐                │
│ Proxmox host    │      │ Proxy container   │                │
│ pve.home.arpa   │      │ (Caddy)           │                │
│ built-in ACME   │      │ proxy.home.arpa   │                │
└─────────────────┘      └─────────┬─────────┘                │
                                   │ reverse proxy            │
            ┌──────────────────────┼──────────────────────┐   │
            ▼                      ▼                      ▼   │
     app1 (HTTP)            app2 (HTTP)         app3 (HTTPS, own cert
                                                 from the CA, auto-renewed)
```

- **Two-tier PKI:** the root CA signs one intermediate CA. The intermediate signs everything day to day. The root private key lives offline.
- **ACME:** Proxmox and Caddy request and renew certificates automatically, the same way they would with Let's Encrypt.
- **One proxy for all apps:** every app's DNS name points at the proxy. The proxy holds the certificates and forwards traffic to each app's own IP and port.
- **Clients** (browsers, phones) only need to trust **one** root certificate.

---

## Placeholders used in this guide

Replace these with your own values.

| Placeholder | Example | Meaning |
|---|---|---|
| `home.arpa` | `home.arpa` | Internal DNS domain. `home.arpa` is reserved for home networks (RFC 8375). |
| `PVE_IP` | `192.168.10.10` | Proxmox host |
| `CA_IP` | `192.168.10.11` | CA container |
| `PROXY_IP` | `192.168.10.12` | Reverse proxy container |
| `APP1_IP` | `192.168.10.21` | Example app (Joplin Server) |
| `APP2_IP` | `192.168.10.22` | Example app (Immich) |
| `APP3_IP` | `192.168.10.23` | Example app that serves its own HTTPS (Nessus) |
| `Homelab` | `Homelab` | PKI name (appears as "Homelab Root CA" / "Homelab Intermediate CA") |
| `<ROOT_FINGERPRINT>` | 64 hex chars | SHA-256 fingerprint of your root certificate, printed by `step ca init` |

> **Domain choice:** using a top-level domain that isn't reserved for private use risks a future clash with a real public TLD. `home.arpa` (RFC 8375) is reserved for exactly this purpose.

---

## Part 1 – CA container

### 1.1 Create the LXC

In the Proxmox UI, download the Debian 13 template (**local → CT Templates → Templates**, entry starting with `debian-13-standard`), then **Create CT**:

| Setting | Value |
|---|---|
| Hostname | `ca` |
| Unprivileged container | ✅ (default) |
| Disk / CPU / RAM | ~8 GB / 1 core / 512 MB (estimate; step-ca is lightweight) |
| Network | Static IP `CA_IP`, gateway = your router |
| DNS | Use host settings (the host must use your UniFi DNS) |
| Options → Start at boot | Yes |

**Why unprivileged:** root inside the container maps to an unprivileged user on the host, which limits the damage if the container is compromised. step-ca needs nothing that a privileged container provides.

### 1.2 Fixed IP and DNS (UniFi)

- If you use a DHCP reservation instead of a static IP, reserve the address in UniFi.
- Add a DNS record `ca.home.arpa` → `CA_IP` (see [UniFi DNS notes](#unifi-dns-notes)).

### 1.3 Verify

```bash
getent hosts pve.home.arpa
getent hosts ca.home.arpa
apt update
```

### 1.4 Backups and snapshots

Before creating keys, check **Datacenter → Backup** and the container's **Snapshots**. A backup or snapshot taken while the root key is still on the container would capture it. Pause any job covering this container until Part 3 is finished.

---

## Part 2 – Install step-ca

From Smallstep's install guide (Debian/Ubuntu), run as root in the CA container:

```bash
apt-get update && apt-get install -y --no-install-recommends curl gpg ca-certificates
install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://packages.smallstep.com/keys/apt/repo-signing-key.gpg -o /etc/apt/keyrings/smallstep.asc
```

```bash
cat << EOF > /etc/apt/sources.list.d/smallstep.sources
Types: deb
URIs: https://packages.smallstep.com/stable/debian
Suites: debs
Components: main
Signed-By: /etc/apt/keyrings/smallstep.asc
EOF
```

```bash
apt-get update && apt-get -y install step-cli step-ca
step version
step-ca version
```

Tested with `step` CLI 0.31.0 and `step-ca` 0.30.2.

---

## Part 3 – Initialize a two-tier CA and take the root key offline

This guide creates both keys in the container and then moves the root key off. That's simpler than generating the root on an air-gapped machine. Two trade-offs: the root key briefly exists on a networked host, and deleted files may remain recoverable on SSD or copy-on-write storage. The root key is encrypted with a strong, unique password, which carries most of the protection.

### 3.1 Service user and step path

```bash
useradd --user-group --system --create-home --home /etc/step-ca --shell /bin/false step
export STEPPATH=/etc/step-ca
echo 'export STEPPATH=/etc/step-ca' >> /root/.bashrc
```

Setting `STEPPATH` **before** `step ca init` creates the files directly in `/etc/step-ca`. That avoids moving them later and editing paths.

### 3.2 Initialize

Generate a long random password in a password manager first; it becomes the **root key password**.

```bash
step ca init --deployment-type=standalone --name="Homelab" \
  --dns=ca.home.arpa --dns=CA_IP --address=:443 --provisioner=admin
```

Save the **root fingerprint** that it prints. It isn't secret, but you'll need it later.

Check the result. These commands print only paths and public data:

```bash
ls -l /etc/step-ca/certs /etc/step-ca/secrets
grep -E '"(root|crt|key|dataSource)"' /etc/step-ca/config/ca.json
```

The config should point to the **intermediate** key, not the root key.

### 3.3 Give the intermediate key its own password

By default, `step ca init` encrypts both keys with the same password. Smallstep recommends changing the intermediate key's password right away.

```bash
step crypto change-pass /etc/step-ca/secrets/intermediate_ca_key
# current password = root password; new password = new "intermediate" password; answer y to overwrite
```

Store the intermediate password in a file for the service. This way the password doesn't appear on screen or in shell history:

```bash
install -m 600 /dev/null /etc/step-ca/password.txt
read -rs P && printf '%s' "$P" > /etc/step-ca/password.txt; unset P
# paste the intermediate password, press Enter
```

Test that the CA starts (Ctrl+C to stop):

```bash
step-ca /etc/step-ca/config/ca.json --password-file /etc/step-ca/password.txt
```

### 3.4 Copy the root key offline and verify the copy

1. `cat /etc/step-ca/secrets/root_ca_key` → copy it into a file named `root_ca_key` on offline storage. **Don't paste key material into chats, tickets or logs.**
2. `cat /etc/step-ca/certs/root_ca.crt` → save it as `root_ca.crt` next to the key (it's public).
3. Keep the **root password in a different place** from the key file, for example a password manager vs. a NAS share.

Verify the offline copy before deleting anything. Paste it back into the container as `/root/check_key` (`cat > /root/check_key`, paste, Enter, Ctrl+D), then run the following. These commands only print results, never key material:

```bash
wc -l /root/check_key
diff <(tr -d '\r' < /root/check_key) /etc/step-ca/secrets/root_ca_key > /dev/null && echo "MATCH" || echo "DIFFERENT"
openssl pkey -in /root/check_key -noout && echo "DECRYPT OK"   # enter the ROOT password
```

> ⚠️ Don't run a plain `diff` without `> /dev/null`. If the files differ, `diff` prints the key.

### 3.5 Remove the root key from the container

```bash
shred -u /etc/step-ca/secrets/root_ca_key /root/check_key
ls -l /etc/step-ca/secrets                       # only intermediate_ca_key should remain
step-ca /etc/step-ca/config/ca.json --password-file /etc/step-ca/password.txt   # still starts; Ctrl+C
```

`shred` is a best effort only; SSDs and copy-on-write storage may keep old data. The root password is the real protection.

---

## Part 4 – Run step-ca as a service

From Smallstep's production guide:

```bash
chown -R step:step /etc/step-ca
cat /etc/step-ca/config/defaults.json      # paths should point into /etc/step-ca
```

Create `/etc/systemd/system/step-ca.service` with the unit file from Smallstep's [production considerations page](https://smallstep.com/docs/step-ca/certificate-authority-server-production/#running-step-ca-as-a-daemon) (also on [GitHub](https://github.com/smallstep/certificates/blob/master/systemd/step-ca.service)). Key lines:

```ini
[Service]
User=step
Group=step
Environment=STEPPATH=/etc/step-ca
WorkingDirectory=/etc/step-ca
ExecStart=/usr/bin/step-ca config/ca.json --password-file password.txt
AmbientCapabilities=CAP_NET_BIND_SERVICE
ReadWriteDirectories=/etc/step-ca/db
```

The full upstream unit, including its sandboxing options, ran fine in an unprivileged Debian 13 LXC.

```bash
systemctl daemon-reload
systemctl enable --now step-ca
step ca health          # → ok
```

---

## Part 5 – Enable ACME

step-ca issues **24-hour** certificates by default. Proxmox's built-in ACME client renews once a day, whenever a certificate expires within the next 30 days. So 24-hour certificates would be renewed daily, with a risk of gaps. Use **90 days** (like Let's Encrypt) for the ACME provisioner instead:

```bash
step ca provisioner add acme --type ACME --x509-default-dur 2160h --x509-max-dur 2160h
chown step:step /etc/step-ca/config/ca.json
systemctl restart step-ca
sleep 5
step ca provisioner list | grep -E '"(type|name)"'
```

Smallstep recommends one month or less for host certificates. That doesn't fit Proxmox's fixed 30-day renewal window, so 90 days is the practical compromise here.

Test end to end (the `step` tool briefly listens on port 80 for the http-01 challenge):

```bash
step ca certificate --provisioner acme ca.home.arpa /tmp/test.crt /tmp/test.key
step certificate inspect /tmp/test.crt --short    # validity ≈ 90 days
rm /tmp/test.crt /tmp/test.key
```

The ACME directory URL is `https://ca.home.arpa/acme/acme/directory`. The second `acme` is the provisioner name.

---

## Part 6 – Proxmox web UI certificate

Run in the **Proxmox host** shell.

### 6.1 Trust the root CA on the host

Download without verification, then check the fingerprint yourself:

```bash
curl -k https://ca.home.arpa/roots.pem -o /usr/local/share/ca-certificates/homelab-root-ca.crt
openssl x509 -in /usr/local/share/ca-certificates/homelab-root-ca.crt -noout -fingerprint -sha256 \
  | cut -d= -f2 | tr -d : | tr 'A-F' 'a-f'          # must equal <ROOT_FINGERPRINT>
update-ca-certificates
curl https://ca.home.arpa/health                    # → {"status":"ok"} with no -k
```

### 6.2 Register an ACME account and order

```bash
pvenode acme account register homelab you@example.com --directory https://ca.home.arpa/acme/acme/directory
pvenode config set --acme account=homelab,domains=pve.home.arpa
pvenode acme cert order
systemctl restart pveproxy
pvenode cert info        # pveproxy-ssl.pem should show the Homelab intermediate as its issuer
```

- step-ca publishes no Terms of Service; Proxmox handles that and proceeds.
- The email address is required by the command but not used by step-ca.
- Renewal is automatic through Proxmox's daily update job.

---

## Part 7 – Trust the root CA on client devices

Distribute **`root_ca.crt`** (public) to each device.

### Windows

1. Double-click `root_ca.crt` → **Details** → compare the **Thumbprint** (SHA-1) with:
   `openssl x509 -in /etc/step-ca/certs/root_ca.crt -noout -fingerprint -sha1` (run on the CA container).
2. **General → Install Certificate → Local Machine → Place all certificates in the following store → Trusted Root Certification Authorities**.
3. If you had previously clicked through warnings for a site, turn warnings back on and **fully** restart the browser. Edge may keep running in the background. A private window is a quick way to test without the old decision.

Edge, Chrome and Firefox on Windows showed a padlock after this.

### Android

1. Copy `root_ca.crt` to the phone. A screen lock is required.
2. **Settings → Security & privacy → More security settings → Encryption & credentials → Install a certificate → CA certificate.** The path varies by manufacturer; search Settings for "CA certificate".
3. Confirm under **Trusted credentials → User**.

Notes:
- Android apps don't trust user-installed CAs unless the app opts in. Test each app. The Immich Android app worked against this setup.
- If the browser shows `ERR_CONNECTION_ABORTED` (not a certificate error), check **firewall rules** between the phone's network and the proxy. That was the cause here.
- If a site "can't be reached" by name, check Android **Private DNS**. Your phone must use your local DNS to resolve internal names.

### Linux (Debian/Ubuntu family) – not yet tested in this build

System trust:

```bash
sudo curl -k https://ca.home.arpa/roots.pem -o /usr/local/share/ca-certificates/homelab-root-ca.crt
openssl x509 -in /usr/local/share/ca-certificates/homelab-root-ca.crt -noout -fingerprint -sha256 | cut -d= -f2 | tr -d : | tr 'A-F' 'a-f'
sudo update-ca-certificates
```

Linux browsers often keep their own certificate store:
- **Firefox:** Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import → "Trust this CA to identify websites".
- **Chrome/Brave** (only if they still warn): `certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "Homelab Root CA" -i /usr/local/share/ca-certificates/homelab-root-ca.crt` (package `libnss3-tools`; run as your user, not root). Snap/Flatpak browsers may keep their stores in different locations.

### iOS – not yet tested in this build

Commonly documented flow: open/download `root_ca.crt` → **Settings → Profile Downloaded → Install**, then **Settings → General → About → Certificate Trust Settings** → enable full trust for the root. Verify against Apple's current documentation.

---

## Part 8 – Central reverse proxy (Caddy)

### 8.1 Proxy container

Create a Debian 13 LXC as in Part 1: hostname `proxy`, static IP `PROXY_IP`, DNS record `proxy.home.arpa` → `PROXY_IP`.

### 8.2 Install Caddy

From Caddy's install docs (stable channel). `gpg` is added because the Debian template may not include it:

```bash
apt install --yes debian-keyring debian-archive-keyring apt-transport-https curl gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
chmod o+r /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install -y caddy
caddy version
```

> Run these one line at a time if pasting is unreliable. In our build, a pasted block silently failed at the repository step, and `caddy` turned out not to be installed.

Tested with Caddy v2.11.7.

### 8.3 Trust the root CA on the proxy

Same as [6.1](#61-trust-the-root-ca-on-the-host): download `roots.pem` to `/usr/local/share/ca-certificates/homelab-root-ca.crt`, check the fingerprint, then run `update-ca-certificates`.

### 8.4 Point Caddy at step-ca

`/etc/caddy/Caddyfile`. **Indent with spaces.** Pressing Tab in a console paste triggers bash auto-complete and corrupts the file.

```caddyfile
{
    acme_ca https://ca.home.arpa/acme/acme/directory
    acme_ca_root /usr/local/share/ca-certificates/homelab-root-ca.crt
}

proxy.home.arpa {
    respond "Caddy proxy OK"
}
```

```bash
cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.orig   # before the first edit
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
journalctl -u caddy --no-pager -n 30 | grep -iE 'obtain|error|certificate'
curl https://proxy.home.arpa                         # → Caddy proxy OK
```

On first use, Caddy logs `no account for configured email is known to us` at **info** level. That's expected: it then creates an ACME account. Caddy renews certificates after about two-thirds of their lifetime.

---

## Part 9 – Move apps behind the proxy

### UniFi DNS notes

- UniFi can create a DNS name directly on a client ("Local DNS Record", next to Fixed IP). Names created that way always point to the **client's own** IP.
- For an app behind the proxy:
  1. On the app's client entry, **turn off Local DNS Record** but **keep the Fixed IP**.
  2. Create a separate **Host (A)** record in the DNS policy: `app.home.arpa` → `PROXY_IP`.
- Ubiquiti's help center lists the DNS records location as **Settings → Policy Table → Create New Policy → DNS** (Network 9.4) or **Settings → Policy Engine → DNS** (Network 9.3). Newer versions may differ.
- Before reloading Caddy, confirm from **both** the CA and proxy containers that `getent hosts app.home.arpa` returns `PROXY_IP`. The CA's http-01 check must reach the proxy.

### Example: Joplin Server (helper-script install)

Joplin Server must know its public URL (`APP_BASE_URL`). In helper-script installs, it's in `/opt/joplin-server/.env`, which the systemd unit loads with `EnvironmentFile=`.

Proxy block:

```caddyfile
joplin.home.arpa {
    reverse_proxy APP1_IP:22300
}
```

On the Joplin container:

```bash
cp /opt/joplin-server/.env /opt/joplin-server/.env.bak
sed -i 's#^APP_BASE_URL=.*#APP_BASE_URL=https://joplin.home.arpa#' /opt/joplin-server/.env
grep -E '^APP_BASE_URL' /opt/joplin-server/.env
systemctl restart joplin-server
```

Confirm the running process picked up the new value, printing only that one variable:

```bash
PID=$(ss -tlnp | grep ':22300' | grep -o 'pid=[0-9]*' | head -1 | cut -d= -f2)
tr '\0' '\n' < /proc/$PID/environ | grep '^APP_BASE_URL'
```

Then change the sync URL in each Joplin client to `https://joplin.home.arpa`.

### Example: Immich (helper-script install)

```caddyfile
immich.home.arpa {
    reverse_proxy APP2_IP:2283
}
```

Optionally set **Administration → Settings → Server Settings → External Domain** to `https://immich.home.arpa`, so share links use the new address.

Check what an app listens on with `ss -tlnp` inside its container. For Immich, `2283` is the web/API port; the machine-learning service on `3003` isn't proxied.

### Reloading safely

Rewrite or append to the Caddyfile, then always:

```bash
caddy validate --config /etc/caddy/Caddyfile && systemctl reload caddy
```

If a reload fails, Caddy keeps running with the previous working config.

---

## Part 10 – Apps that already serve HTTPS (Nessus example)

Nessus serves HTTPS on 8834 with a self-signed certificate whose name is **only in the CN field** (no Subject Alternative Name). Caddy is written in Go, and Go has ignored CN for hostname verification since Go 1.15. So Caddy can't properly verify that certificate. Rather than using `tls_insecure_skip_verify` (Caddy's docs warn against it), give Nessus a certificate from the CA and have the proxy verify it.

ACME won't work for this name: once `nessus.home.arpa` points to the proxy, the http-01 check reaches the proxy, not Nessus. Use a dedicated **JWK provisioner** with 1-year certificates and automated renewal instead.

### 10.1 Dedicated provisioner (CA container)

```bash
step ca provisioner add nessus --type JWK --create --x509-default-dur 8760h --x509-max-dur 8760h
chown step:step /etc/step-ca/config/ca.json
systemctl restart step-ca
sleep 5
step ca provisioner list | grep -E '"(type|name)"'
```

This provisioner gets its own password. Don't reuse the `admin` provisioner: `step ca init` protected it with the same password as the CA keys, which is the root password.

> If you list provisioners immediately after a restart, you may get `connection refused`; the CA takes a moment to start.

### 10.2 `step` CLI on the Nessus container

Install `step-cli` only (same repository as in Part 2), then trust the CA, pinned to its fingerprint:

```bash
step ca bootstrap --ca-url https://ca.home.arpa --fingerprint <ROOT_FINGERPRINT> --install
step ca health
curl https://ca.home.arpa/health
```

### 10.3 Issue the certificate (key never leaves the Nessus container)

```bash
install -d -m 700 /etc/nessus-cert
step ca certificate nessus.home.arpa /etc/nessus-cert/nessus.crt /etc/nessus-cert/nessus.key \
  --provisioner nessus --kty RSA --size 2048
step certificate inspect /etc/nessus-cert/nessus.crt | grep -A1 'Subject Alternative Name'
```

RSA was chosen for compatibility; whether Nessus accepts ECDSA keys was not tested.

### 10.4 Import into Nessus

Back up first:

```bash
cp -a /opt/nessus/com/nessus/CA /root/nessus-CA-com.bak
cp -a /opt/nessus/var/nessus/CA /root/nessus-CA-var.bak
```

`nessuscli import-certs` rejected `--cacert` set to **either** the root alone **or** the intermediate alone. It accepted a **bundle of intermediate + root**:

```bash
curl -sS https://ca.home.arpa/intermediates.pem -o /etc/nessus-cert/intermediate.pem
cat /etc/nessus-cert/intermediate.pem $(step path)/certs/root_ca.crt > /etc/nessus-cert/ca-bundle.pem
/opt/nessus/sbin/nessuscli import-certs --serverkey=/etc/nessus-cert/nessus.key \
  --servercert=/etc/nessus-cert/nessus.crt --cacert=/etc/nessus-cert/ca-bundle.pem
systemctl restart nessusd
```

Verify from the proxy that the certificate checks out against the CA under the right name:

```bash
curl --resolve nessus.home.arpa:8834:APP3_IP -sSI https://nessus.home.arpa:8834 | head -1
```

`405 Method not Allowed` is fine. Nessus rejects `HEAD`, but the TLS verification succeeded.

### 10.5 Automated renewal (no stored password)

The import turned out to be a plain copy. These files were byte-identical after import, and the key wasn't encrypted:
- `servercert.pem` = `nessus.crt`
- `serverkey.pem` = `nessus.key`
- `cacert.pem` = `ca-bundle.pem`

So renewal can copy the files and restart Nessus, without `nessuscli` and without its password. Check this on your install with `cmp` before relying on it.

`/usr/local/bin/nessus-cert-deploy.sh` (mode `700`):

```sh
#!/bin/sh
set -e
cp /etc/nessus-cert/nessus.crt /opt/nessus/com/nessus/CA/servercert.pem
cp /etc/nessus-cert/nessus.key /opt/nessus/var/nessus/CA/serverkey.pem
systemctl restart nessusd
```

`/etc/systemd/system/nessus-cert-renew.service`:

```ini
[Unit]
Description=Renew Nessus certificate from internal CA
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=STEPPATH=/root/.step
ExecStart=/usr/bin/step ca renew --daemon --exec /usr/local/bin/nessus-cert-deploy.sh /etc/nessus-cert/nessus.crt /etc/nessus-cert/nessus.key
Restart=on-failure
RestartSec=60

[Install]
WantedBy=multi-user.target
```

```bash
step ca renew --force /etc/nessus-cert/nessus.crt /etc/nessus-cert/nessus.key   # one-off test
/usr/local/bin/nessus-cert-deploy.sh
systemctl daemon-reload
systemctl enable --now nessus-cert-renew
journalctl -u nessus-cert-renew --no-pager -n 5      # "first renewal in ~5568h"
```

- `step ca renew` authenticates with the current certificate (mTLS), so no provisioner password is stored.
- It scheduled renewal at about two-thirds of the 1-year lifetime.
- Each renewal restarts Nessus, which interrupts any scan running at that moment.

### 10.6 Proxy block with full upstream verification

```caddyfile
nessus.home.arpa {
    reverse_proxy https://APP3_IP:8834 {
        transport http {
            tls_trust_pool file /usr/local/share/ca-certificates/homelab-root-ca.crt
            tls_server_name nessus.home.arpa
        }
    }
}
```

Users now open `https://nessus.home.arpa` (no `:8834`).

**Bonus:** paste `root_ca.crt` into Nessus **Settings → Custom CA**. Scans of your own servers then stop reporting plugin #51192 ("SSL Certificate Cannot Be Trusted"). This setting affects scanning only, not the Nessus web certificate.

---

## Adding a new app (checklist)

1. **Container:** fixed IP in UniFi, **no** Local DNS Record.
2. **DNS:** Host (A) record `newapp.home.arpa` → `PROXY_IP`.
3. **Check:** `getent hosts newapp.home.arpa` on the CA **and** proxy containers → `PROXY_IP`.
4. **Append** on the proxy. Note `>>` (append). A single `>` overwrites the whole file.
   ```bash
   cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak
   cat << 'EOF' >> /etc/caddy/Caddyfile

   newapp.home.arpa {
       reverse_proxy APP_IP:PORT
   }
   EOF
   ```
5. **Validate and reload:** `caddy validate --config /etc/caddy/Caddyfile && systemctl reload caddy`
6. **Confirm:** `journalctl -u caddy --no-pager -n 20 | grep -i newapp` → "certificate obtained successfully".
7. **App settings:** if the app has a "base URL", "external URL" or "domain" setting, set it to `https://newapp.home.arpa`.
8. **Rollback:** `cp /etc/caddy/Caddyfile.bak /etc/caddy/Caddyfile && systemctl reload caddy`

Apps that serve their own HTTPS need a block like [10.6](#106-proxy-block-with-full-upstream-verification), not the simple one.

---

## Troubleshooting and lessons learned

| Symptom | Cause / fix |
|---|---|
| `step ca provisioner list` → `connection refused` right after a restart | The CA wasn't listening yet. Wait a few seconds. |
| `caddy: command not found` after a pasted install block | A step in the block failed silently. Rerun one line at a time. |
| Certificate shows as valid, but the page still says "Not secure" | The browser remembered an earlier "skip warnings" decision. Turn warnings back on, fully restart the browser, or test in a private window. |
| UniFi refuses to give an app the proxy's IP | You're editing the app's **Fixed IP**. Change the **DNS record** instead. |
| Phone: `ERR_CONNECTION_ABORTED` | Firewall between the phone's network and the proxy. Allow `PROXY_IP:443` (and `:80` for the redirect). |
| Phone app "Server is not reachable" | Test the same URL in the phone's browser first: DNS error → Private DNS / resolver; certificate warning → CA not installed; padlock → app-specific trust. |
| `nessuscli import-certs`: "could not be validated with the new CA certificate" | Pass a bundle of **intermediate + root** as `--cacert`. |
| Caddy can't verify an HTTPS backend | The backend certificate may lack a SAN (CN-only). Issue it a proper certificate from the CA. |

**Avoid leaking secrets while debugging:**
- `diff` prints differing lines, so a failed comparison against a key file prints the key. Add `> /dev/null`.
- Broad `grep -r` across `/opt` can match dump files full of environment variables, passwords included.
- Prefer commands that print only counts, `MATCH`/`OK`, or single named settings.

---

## Maintenance

- **Back up the CA container.** The intermediate key (encrypted) and the CA database live only there. The root key and its password are needed to rebuild the CA if it's lost.
- **Expiry dates:** `step ca init` defaults to a **10-year** root and intermediate. Replacing the intermediate requires the offline root key; see Smallstep's "Rotating an intermediate CA".
- **Automatic renewals:**
  - Proxmox (daily job; renews when ≤ 30 days remain)
  - Caddy (after about ⅔ of the lifetime)
  - Nessus (`nessus-cert-renew` service, about ⅔ of 1 year)
- **Quick health checks:**
  ```bash
  step ca health                                   # CA container
  systemctl status caddy --no-pager | head -3      # proxy
  pvenode cert info                                # Proxmox host
  systemctl status nessus-cert-renew --no-pager    # Nessus container
  ```

---

## Security notes and trade-offs

- **ACME trusts DNS and the network.** Anyone who can make a name resolve to a machine they control, and answer on port 80, can obtain a certificate for that name. Protect UniFi DNS and admin access. step-ca supports [policies](https://smallstep.com/docs/step-ca/policies/) to restrict which names ACME may issue for.
- **The root CA is trusted everywhere it's installed.** Anyone with the root key and password could impersonate any website to those devices. Keep both offline and stored separately.
- **Apps still listen on their own IPs.** After the switch, apps remain reachable over plain HTTP at `APP_IP:PORT`. Consider firewalling them so that only `PROXY_IP` can connect.
- **Mobile "allow self-signed certificates" toggles** make an app accept *any* certificate. Prefer installing the root CA and leaving such toggles off.
- **Creating the root in the container (Part 3)** is a deliberate homelab compromise. For stronger isolation, generate the root on an air-gapped machine or hardware security module, and import only the intermediate.

---

## References

- Smallstep – [Install step-ca](https://smallstep.com/docs/step-ca/installation/)
- Smallstep – [Production considerations](https://smallstep.com/docs/step-ca/certificate-authority-server-production/)
- Smallstep – [ACME basics](https://smallstep.com/docs/step-ca/acme-basics/)
- Smallstep – [Configure ACME clients (incl. Caddy)](https://smallstep.com/docs/tutorials/acme-protocol-acme-clients/)
- Smallstep – [`step ca init`](https://smallstep.com/docs/step-cli/reference/ca/init/) · [`step ca provisioner add`](https://smallstep.com/docs/step-cli/reference/ca/provisioner/add/)
- Proxmox – [Certificate Management](https://pve.proxmox.com/wiki/Certificate_Management) · [`pvenode(1)`](https://pve.proxmox.com/pve-docs/pvenode.1.html)
- Caddy – [Install](https://caddyserver.com/docs/install) · [`reverse_proxy` directive](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- Tenable – [Upload a custom server certificate and CA certificate](https://docs.tenable.com/nessus/Content/UploadACustomServerAndCACertificate.htm)
- Ubiquiti – [UniFi DNS records and local hostnames](https://help.ui.com/hc/en-us/articles/15179064940439)
- IETF – [RFC 8375: Special-Use Domain `home.arpa.`](https://www.rfc-editor.org/rfc/rfc8375)
