# Customer Setup Recipe

Generic step-by-step guide to set up a new customer website and domain. Replace `<customer.domain>` with the actual domain (e.g. `hallo-car.ch`).

---

## Prerequisites

- [ ] Reseller login at [domain.hosttech.eu](https://domain.hosttech.eu/) (expert mode)
- [ ] Reseller Plesk login (server-specific, e.g. `355.hostserv.eu:8443`)
- [ ] Customer contact details (name, email, company, address)
- [ ] Sufficient reseller quota (domains + subscriptions)
- [ ] `HOSTTECH_API_KEY` environment variable set (for API-managed zones like panaxis-ltd.ch)

---

## Step 1 — Check Domain Availability

**Where:** domain.hosttech.eu → Domains → Domains

1. Search for `<customer.domain>`
2. Verify it is available (or ready for transfer)
3. Note the TLD registry:
   - `.ch` / `.li` → SWITCH
   - `.com` / `.net` / `.org` → ICANN registrars

---

## Step 2 — Register the Domain

**Where:** domain.hosttech.eu → Domains → Domains → Register

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
4. Default nameservers will be assigned (typically `ns1.hosttech.eu` / `ns2.hosttech.eu`)

**For domain transfers:** Obtain the auth/transfer code from the current registrar first, then initiate transfer in domain.hosttech.eu → Domains → Pending Transfers.

---

## Step 3 — Create DNS Zone

**Where:** domain.hosttech.eu → Domains → `<customer.domain>` → DNS Zone tab

> **Important:** Domains registered through the reseller panel use a separate DNS system from the end-user panel. The DNS API (`api.ns1.hosttech.eu`) only manages zones created in the end-user panel (e.g. panaxis-ltd.ch). Reseller domains must be managed through the reseller panel or zone file import.

1. Click **DNS Zone** tab on the domain configuration page
2. Click **Create DNS Zone**
3. Import the zone template or add records manually (see Step 5)

### Zone template

Use `templates/zone-template.txt` for quick setup:

1. Copy the template and replace `DOMAIN` with `<customer.domain>` and `SERVERIP` with the Plesk server IP
2. In the DNS Zone page, click **Import Zone File** and paste/upload the modified template

This creates all required records in one step (A, CNAME www/mail/webmail, MX, SPF).

---

## Step 4 — Change Nameservers to rrpproxy

**Where:** domain.hosttech.eu → Domains → `<customer.domain>` → Configure → Nameserver tab

> **Why:** The reseller DNS zones are mastered on rrpproxy nameservers (`ns1.rrpproxy.net`), not on the default hosttech nameservers. The default `ns1/ns2.hosttech.eu` nameservers will return REFUSED for reseller-created zones. You **must** change the nameservers for the domain to resolve.

Change nameservers to:

| # | Nameserver |
|---|------------|
| 1 | `NS1.RRPPROXY.NET` |
| 2 | `NS2.RRPPROXY.NET` |
| 3 | `NS3.RRPPROXY.NET` |

Click **Speichern** (Save). The `.ch` registry update typically propagates within 30-60 minutes.

### Verify propagation

```bash
# Check if NS delegation has updated at the registry
dig <customer.domain> NS +trace | tail -10

# Once propagated, verify records resolve
dig <customer.domain> A +short
dig www.<customer.domain> A +short
dig <customer.domain> MX +short
dig <customer.domain> TXT +short
```

---

## Step 5 — Configure DNS Records

**Where:** domain.hosttech.eu → Domains → `<customer.domain>` → DNS Zone → Resource Records

> **If you imported the zone template in Step 3, skip this step.** All records are already created.

### Required records

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `@` | `<server-ip>` | 3600 |
| CNAME | `www` | `<customer.domain>` | 3600 |
| CNAME | `mail` | `<customer.domain>` | 3600 |
| CNAME | `webmail` | `<customer.domain>` | 3600 |
| MX | `@` | `<customer.domain>` (priority 10) | 3600 |
| TXT | `@` | `v=spf1 a mx -all` | 3600 |

### Optional records

| Type | Name | Value | Purpose |
|------|------|-------|---------|
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; rua=mailto:postmaster@<customer.domain>` | DMARC policy |
| TXT | `default._domainkey` | (from Plesk DKIM setup) | DKIM signing |
| AAAA | `@` | `<ipv6-address>` | IPv6 support |

> **Use CNAMEs, not A records** for subdomains (`www`, `mail`, `webmail`). If the server IP changes, you only update the root A record.

> **Find your server IP:** Plesk → Websites & Domains → `<customer.domain>` → Connection Info → IP address.

---

## Step 6 — Create Hosting Subscription in Plesk

**Where:** Plesk (server-specific URL, e.g. `355.hostserv.eu:8443`)

### Create customer (if new)

Plesk → Customers → Add Customer

| Field | Value |
|-------|-------|
| Contact name | Customer's name |
| Email | Customer's email |
| Company | Customer's company (optional) |
| Login | e.g. `hallo-car` (customer chooses) |
| Password | Generate a strong password |

### Create subscription

Plesk → Subscriptions → Add Subscription

| Field | Value |
|-------|-------|
| Domain | `<customer.domain>` |
| Customer | Select customer |
| Service plan | Choose from your reseller plans |
| System user | e.g. `hallocar` |
| Password | Generate a strong password |

Plesk will create:
- Web space directory (`httpdocs/`)
- FTP account
- System user for SSH/FTP

---

## Step 7 — Install SSL Certificate (Let's Encrypt)

**Where:** Plesk → Websites & Domains → `<customer.domain>` → SSL/TLS Certificates

> **Prerequisite:** DNS must be fully propagated first (Step 4). Let's Encrypt validates via HTTP, so the domain must resolve to the server.

1. Click **Install** → select **Let's Encrypt**
2. Email: admin email for expiry notices
3. Check **Include www.`<customer.domain>`**
4. Check **Assign the certificate to the mail domain**
5. Confirm

After installation:
- [ ] Enable **Redirect from http to https**
- [ ] Enable **HSTS**
- [ ] **Reissue** the certificate to include `webmail.<customer.domain>` if needed

Auto-renewal is enabled by default. Certificate renews every 90 days.

---

## Step 8 — Set Up Email Accounts

**Where:** Plesk → Websites & Domains → `<customer.domain>` → Mail

### Create mailboxes

| Address | Purpose |
|---------|---------|
| `postmaster@<customer.domain>` | RFC-required, abuse/admin contact |
| `info@<customer.domain>` | General contact |
| Custom per customer request | — |

### Configure email security

1. **SPF** — already set in Step 5
2. **DKIM** — Plesk → Mail → Mail Settings → enable DKIM signing
   - Since DNS is external, copy the DKIM TXT record value from Plesk
   - Add it as a TXT record (`default._domainkey`) in the reseller DNS panel
3. **DMARC** — add TXT record from Step 5 (optional records)

### Webmail access

- URL: `https://webmail.<customer.domain>` (requires CNAME + SSL from Steps 5/7)
- Or: `https://<plesk-server>:8443/` → Webmail
- Client: Roundcube (default in Plesk)

---

## Step 9 — Install Website / CMS

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
   - Host: `<customer.domain>` or server IP
   - User: system user from Step 6
   - Port: 21 (FTP) or 22 (SFTP)
2. Or use Plesk File Manager

### Option C: hosttech Website Creator

If the customer needs a drag-and-drop builder, provision via the Website Creator API or myhosttech.

---

## Step 10 — Verify Everything Works

### Checklist

- [ ] `https://<customer.domain>` loads correctly
- [ ] `https://www.<customer.domain>` redirects or loads correctly
- [ ] SSL certificate is valid (padlock icon, no warnings)
- [ ] HTTP → HTTPS redirect enabled
- [ ] HSTS enabled
- [ ] Email send/receive works (`info@<customer.domain>`)
- [ ] Webmail login works
- [ ] FTP/SFTP login works
- [ ] DNS propagation complete (all records resolve)
- [ ] SPF/DKIM/DMARC pass: send test email to [mail-tester.com](https://www.mail-tester.com)

### DNS verification

```bash
dig <customer.domain> A +short          # Should return server IP
dig www.<customer.domain> A +short      # Should return server IP (via CNAME)
dig <customer.domain> MX +short         # Should return: 10 <customer.domain>.
dig <customer.domain> TXT +short        # Should include SPF record
dig <customer.domain> NS +short         # Should show ns{1,2,3}.rrpproxy.net.

# Check authoritative nameservers directly
dig @ns1.rrpproxy.net <customer.domain> A +short
```

---

## Step 11 — Hand Over to Customer

Send the customer their credentials:

### Credentials document

```
=== Hosting Credentials for <customer.domain> ===

Plesk Control Panel
  URL:      https://<plesk-server>:8443/
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
| Reseller panel | [domain.hosttech.eu](https://domain.hosttech.eu/) | Domain registration, DNS zones (expert mode) |
| myhosttech | [myhosttech.eu](https://www.myhosttech.eu/login) | End-user DNS (panaxis-ltd.ch), billing |
| Plesk | Server-specific (e.g. `355.hostserv.eu:8443`) | Hosting, subscriptions, email, SSL |
| DNS API | [api.ns1.hosttech.eu](https://api.ns1.hosttech.eu/api/documentation/) | API for end-user zones only (not reseller zones) |
| Support | [support.hosttech.ch](https://support.hosttech.ch/) | Knowledge base and tickets |

---

## Architecture Notes

### Two DNS systems

Hosttech has two separate DNS management systems:

| System | Panel | Nameservers | API |
|--------|-------|-------------|-----|
| End-user DNS | myhosttech.eu → DNS Editor | `ns1/ns2/ns3.hosttech.eu` (= `hostserv.eu`) | `api.ns1.hosttech.eu` with `HOSTTECH_API_KEY` |
| Reseller DNS | domain.hosttech.eu → DNS Zone | `ns1/ns2/ns3.rrpproxy.net` | Not available |

Domains registered through the reseller panel use the **reseller DNS** system. Their zones must be created in the reseller panel, and the domain's nameservers must be changed to `rrpproxy.net` — the default `hosttech.eu` nameservers will return REFUSED for these zones.

### SOA record

Reseller zones are mastered on `ns1.rrpproxy.net` with SOA email `tech.rrpproxy.net`. This is set automatically and should not be changed.

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Domain not resolving (REFUSED) | Reseller zone with default hosttech NS | Change nameservers to `ns1/ns2/ns3.rrpproxy.net` (Step 4) |
| Domain not resolving (NXDOMAIN) | DNS zone not created | Create zone in reseller panel (Step 3) |
| DNS zone not in API | Zone created via reseller panel | Reseller zones are not accessible via `api.ns1.hosttech.eu` — manage in reseller panel |
| NS change not propagating | `.ch` registry batch update | Wait 30-60 min, verify with `dig +trace` |
| SSL certificate error | DNS not yet propagated when Let's Encrypt tried | Wait for DNS, retry in Plesk SSL settings |
| SSL cert name mismatch | Old/wrong cert still active | Reissue Let's Encrypt in Plesk |
| Email bouncing | MX points to wrong server | Verify MX points to `<customer.domain>` (Plesk mail) not `mail1.hosttech.eu` |
| Email marked as spam | Missing DKIM/DMARC | Enable DKIM in Plesk, add DKIM TXT record to DNS, add DMARC record |
| FTP timeout | Wrong port or firewall | Use port 21 (FTP) or 22 (SFTP), check Plesk firewall |
| "403 Forbidden" on website | Empty `httpdocs/` or wrong permissions | Upload `index.html`/`index.php`, check file permissions (644/755) |
| WordPress white screen | PHP memory limit or plugin conflict | Plesk → PHP Settings → increase `memory_limit` |
| Webmail not accessible | Missing `webmail` CNAME | Add CNAME record `webmail` → `<customer.domain>` |
