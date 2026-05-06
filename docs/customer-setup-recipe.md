# Customer Setup Recipe

Generic step-by-step guide to set up a new customer website and domain. Replace `<customer.domain>` with the actual domain (e.g. `hallo-car.ch`).

---

## Prerequisites

- [ ] panaxis-ltd.ch reseller login at [myhosttech.eu](https://www.myhosttech.eu/login)
- [ ] Reseller Plesk login at [cp.hosttech.eu](https://cp.hosttech.eu/)
- [ ] Customer contact details (name, email, company, address)
- [ ] Sufficient reseller quota (domains + subscriptions)

---

## Step 1 — Check Domain Availability

**Where:** myhosttech Kundencenter → Domaincenter

1. Search for `<customer.domain>`
2. Verify it is available (or ready for transfer)
3. Note the TLD registry:
   - `.ch` / `.li` → SWITCH
   - `.com` / `.net` / `.org` → ICANN registrars

---

## Step 2 — Register the Domain

**Where:** myhosttech Kundencenter → Domaincenter → Domain registrieren

1. Select `<customer.domain>`
2. Fill in registrant data:

   | Field | Value |
   |-------|-------|
   | Owner (Holder) | Customer's legal name / company |
   | Admin-C | panaxis-ltd.ch (your reseller contact) |
   | Tech-C | panaxis-ltd.ch |
   | Email | Customer's email |
   | Address | Customer's address |

3. Confirm registration
4. DNS is auto-assigned to hosttech nameservers:
   - `ns1.hosttech.eu`
   - `ns2.hosttech.eu`
   - `ns3.hosttech.eu`

**For domain transfers:** Obtain the auth/transfer code from the current registrar first, then initiate transfer in Domaincenter.

---

## Step 3 — Create Customer in Plesk

**Where:** [Plesk](https://cp.hosttech.eu/) → Customers → Add Customer

| Field | Value |
|-------|-------|
| Contact name | Customer's name |
| Email | Customer's email |
| Company | Customer's company (optional) |
| Login | e.g. `hallo-car` (customer chooses) |
| Password | Generate a strong password |

---

## Step 4 — Create Hosting Subscription

**Where:** Plesk → Subscriptions → Add Subscription

| Field | Value |
|-------|-------|
| Domain | `<customer.domain>` |
| Customer | Select from Step 3 |
| Service plan | Choose from your reseller plans |
| System user | e.g. `hallocar` |
| Password | Generate a strong password |

Plesk will create:
- Web space directory (`httpdocs/`)
- FTP account
- Default DNS zone
- System user for SSH/FTP

---

## Step 5 — Configure DNS Records

**Where:** Plesk → `<customer.domain>` → DNS Settings
**Or via API:** `https://api.ns1.hosttech.eu/api/documentation/`

### Required records

| Type | Host | Value | TTL |
|------|------|-------|-----|
| A | `<customer.domain>` | `<server-ip>` | 3600 |
| A | `www.<customer.domain>` | `<server-ip>` | 3600 |
| MX | `<customer.domain>` | `mail.<customer.domain>` (priority 10) | 3600 |
| TXT | `<customer.domain>` | `v=spf1 a mx ~all` | 3600 |

### Optional records

| Type | Host | Value | Purpose |
|------|------|-------|---------|
| CNAME | `webmail` | `<plesk-server>` | Webmail access |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; rua=mailto:admin@<customer.domain>` | DMARC policy |
| AAAA | `<customer.domain>` | `<ipv6-address>` | IPv6 support |

> **Find your server IP:** Plesk → Subscriptions → `<customer.domain>` → the IP is shown in the subscription details.

---

## Step 6 — Install SSL Certificate (Let's Encrypt)

**Where:** Plesk → `<customer.domain>` → SSL/TLS Certificates

1. Click **Install** → select **Let's Encrypt**
2. Email: admin email for expiry notices
3. Check **Include www.<customer.domain>**
4. Check **Assign the certificate to the mail domain**
5. Confirm

Auto-renewal is enabled by default. Certificate renews every 90 days.

---

## Step 7 — Set Up Email Accounts

**Where:** Plesk → `<customer.domain>` → Mail

### Create mailboxes

| Address | Purpose |
|---------|---------|
| `info@<customer.domain>` | General contact |
| `admin@<customer.domain>` | Administrative |
| Custom per customer request | — |

### Configure email security

1. **SPF** — already set in Step 5
2. **DKIM** — Plesk → Mail Settings → enable DKIM signing
3. **DMARC** — add TXT record from Step 5 (optional records)

### Webmail access

- URL: `https://webmail.<customer.domain>` or `https://<plesk-server>:8443/`
- Clients: Roundcube or Horde (configurable in Plesk)

---

## Step 8 — Install Website / CMS

### Option A: WordPress (recommended)

**Where:** Plesk → `<customer.domain>` → WordPress

1. Click **Install**
2. Configure:

   | Setting | Value |
   |---------|-------|
   | Site title | Customer's business name |
   | Admin user | Customer chooses |
   | Admin email | Customer's email |
   | Language | `de_CH` (or as needed) |
   | Path | `/` (root) or `/blog` |

3. After install, enable in WordPress Toolkit:
   - Auto-updates (minor + security)
   - Search engine indexing (when ready)

### Option B: Static site / custom CMS

1. Upload via FTP to `httpdocs/`
   - Host: `ftp.<customer.domain>` or server IP
   - User: system user from Step 4
   - Port: 21 (FTP) or 22 (SFTP)
2. Or use Plesk File Manager

### Option C: hosttech Website Creator

If the customer needs a drag-and-drop builder, provision via the Website Creator API or myhosttech.

---

## Step 9 — Verify Everything Works

### Checklist

- [ ] `https://<customer.domain>` loads correctly
- [ ] `https://www.<customer.domain>` redirects or loads correctly
- [ ] SSL certificate is valid (padlock icon, no warnings)
- [ ] Email send/receive works (`info@<customer.domain>`)
- [ ] Webmail login works
- [ ] FTP/SFTP login works
- [ ] DNS propagation complete: `dig <customer.domain> A`
- [ ] SPF/DKIM/DMARC pass: send test email to [mail-tester.com](https://www.mail-tester.com)

### DNS propagation check

```bash
dig <customer.domain> A +short
dig <customer.domain> MX +short
dig <customer.domain> TXT +short
nslookup <customer.domain> ns1.hosttech.eu
```

Allow up to 24-48h for full global propagation (usually under 1h with hosttech).

---

## Step 10 — Hand Over to Customer

Send the customer their credentials:

### Credentials document

```
=== Hosting Credentials for <customer.domain> ===

Plesk Control Panel
  URL:      https://cp.hosttech.eu/
  Login:    <plesk-login>
  Password: <plesk-password>

FTP / SFTP
  Host:     <customer.domain>
  User:     <system-user>
  Password: <ftp-password>
  Port:     21 (FTP) / 22 (SFTP)

Email: info@<customer.domain>
  IMAP:     mail.<customer.domain> (port 993, SSL)
  SMTP:     mail.<customer.domain> (port 465, SSL)
  Webmail:  https://webmail.<customer.domain>

WordPress Admin (if installed)
  URL:      https://<customer.domain>/wp-admin
  User:     <wp-admin-user>
  Password: <wp-admin-password>
```

> **Security:** Send passwords via a separate secure channel (e.g. encrypted message, phone call, or a one-time link like [onetimesecret.com](https://onetimesecret.com)).

---

## Quick Reference — Hosttech Portals

| Portal | URL | Purpose |
|--------|-----|---------|
| myhosttech | [myhosttech.eu](https://www.myhosttech.eu/login) | Domain registration, billing, reseller settings |
| Plesk | [cp.hosttech.eu](https://cp.hosttech.eu/) | Hosting management, subscriptions, email |
| DNS API | [api.ns1.hosttech.eu](https://api.ns1.hosttech.eu/api/documentation/) | Programmatic DNS management |
| Support | [support.hosttech.ch](https://support.hosttech.ch/) | Knowledge base and tickets |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Domain not resolving | DNS not propagated or wrong NS | Verify NS records point to `ns1/ns2/ns3.hosttech.eu` |
| SSL certificate error | DNS not yet propagated when Let's Encrypt tried | Wait for DNS, retry in Plesk SSL settings |
| Email bouncing | Missing or wrong MX/SPF records | Check `dig <domain> MX` and SPF TXT record |
| FTP timeout | Wrong port or firewall | Use port 21 (FTP) or 22 (SFTP), check Plesk firewall |
| "403 Forbidden" on website | Empty `httpdocs/` or wrong permissions | Upload `index.html`/`index.php`, check file permissions (644/755) |
| WordPress white screen | PHP memory limit or plugin conflict | Plesk → PHP Settings → increase `memory_limit` |
