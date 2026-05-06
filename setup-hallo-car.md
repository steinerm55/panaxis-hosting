# Setup Instructions: hallo-car.ch at hosttech.ch (Reseller: panaxis-ltd.ch)

## Part 1 — Manual Setup (Step-by-Step)

### Step 1: Register the Domain hallo-car.ch

**Via myhosttech Kundencenter** (https://www.myhosttech.eu/login):

1. Log in with your panaxis-ltd.ch reseller credentials
2. Go to **Domaincenter** → **Domain registrieren**
3. Search for `hallo-car.ch`
4. If available, register it (SWITCH registry for `.ch` domains)
5. Enter registrant contact data for the customer (owner = customer, admin-c = panaxis-ltd.ch)
6. Confirm and complete the order

**DNS will be assigned to hosttech nameservers automatically:**
- `ns1.hosttech.eu`
- `ns2.hosttech.eu`
- `ns3.hosttech.eu`

### Step 2: Create a Hosting Subscription in Plesk

**Via Plesk** (https://cp.hosttech.eu/):

1. Log in with your reseller Plesk credentials
2. **Customers** → **Add Customer**
   - Contact name, email, company for hallo-car.ch owner
   - Set login credentials for the customer's Plesk access
3. **Subscriptions** → **Add Subscription**
   - Domain: `hallo-car.ch`
   - Assign to the customer created above
   - Select a service plan (from your reseller quota)
   - System username + password for FTP/SSH
4. Plesk creates the web space, FTP account, and default DNS zone

### Step 3: Configure DNS Records

**In Plesk** (under the subscription's DNS settings) or via the **DNS API**:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `hallo-car.ch` | `<server IP>` | 3600 |
| A | `www.hallo-car.ch` | `<server IP>` | 3600 |
| MX | `hallo-car.ch` | `mail.hallo-car.ch` (prio 10) | 3600 |
| CNAME | `webmail` | `<plesk-server>` | 3600 |
| TXT | `hallo-car.ch` | SPF record | 3600 |

The server IP comes from your Plesk subscription details.

### Step 4: Install SSL Certificate

In Plesk under the subscription:

1. **Websites & Domains** → `hallo-car.ch` → **SSL/TLS Certificates**
2. Click **Install** → **Let's Encrypt**
3. Enter email, check "Include www.hallo-car.ch"
4. Confirm — auto-renewal is enabled by default

### Step 5: Set Up Email (if needed)

1. In Plesk → **Mail** → **Create Email Address**
2. Example: `info@hallo-car.ch`, `kontakt@hallo-car.ch`
3. Configure SPF/DKIM/DMARC records for deliverability

### Step 6: Install CMS / Upload Website

**Option A — WordPress via Plesk:**
1. **Applications** → **Install WordPress**
2. Configure site title, admin user, language (German)

**Option B — Manual upload:**
1. FTP to `ftp.hallo-car.ch` with the credentials from Step 2
2. Upload files to `httpdocs/`

### Step 7: Hand Over Credentials

Provide the customer with:
- Plesk login URL + credentials (restricted to their subscription)
- FTP credentials
- Email account credentials
- Webmail URL

---

## Part 2 — Automation Possibilities

### Level 1: WHMCS (Turnkey, Included Free)

The most complete solution — hosttech includes WHMCS with reseller plans.

**What it automates:**
- Customer signs up on your website → WHMCS provisions Plesk subscription automatically
- Domain registration triggered on order
- Invoicing, payment processing (PayPal, Stripe, bank transfer)
- Renewal reminders, suspension on non-payment
- Customer self-service portal (password resets, upgrades)

**Setup:** Download the 3 free WHMCS domain modules from myhosttech → configure Plesk server connection in WHMCS → build product packages.

**Best for:** Running a real hosting business with multiple customers and billing.

### Level 2: API-Based Scripting (Custom Automation)

hosttech provides three APIs you can script against:

| API | URL | Auth | Use Case |
|-----|-----|------|----------|
| **DNS REST API** | `https://api.ns1.hosttech.eu/api/` | Bearer token | Zone + record management |
| **Domain Reselling API** | via myhosttech Servercenter | Multiple protocols (HTTP, SOAP, XML/RPC) | Domain registration/transfer |
| **Plesk REST/XML API** | `https://cp.hosttech.eu:8443/` | API key or credentials | Subscription provisioning |

**Example: Automate DNS setup with a script**

```bash
#!/bin/bash
# Create DNS zone and records for a new customer domain
API_KEY="your-hosttech-dns-api-key"
BASE="https://api.ns1.hosttech.eu/api/user/v1"
DOMAIN="hallo-car.ch"
IP="185.x.x.x"  # Your Plesk server IP

# Create zone
curl -X POST "$BASE/zones" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"$DOMAIN\", \"type\": \"NATIVE\"}"

# Get zone ID from response, then add records
ZONE_ID=12345  # from response

# A record
curl -X POST "$BASE/zones/$ZONE_ID/records" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"type\": \"A\", \"name\": \"\", \"ipv4\": \"$IP\", \"ttl\": 3600}"

# www CNAME
curl -X POST "$BASE/zones/$ZONE_ID/records" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"type\": \"CNAME\", \"name\": \"www\", \"cname\": \"$DOMAIN\", \"ttl\": 3600}"

# MX record
curl -X POST "$BASE/zones/$ZONE_ID/records" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"type\": \"MX\", \"name\": \"\", \"ownmx\": \"mail.$DOMAIN\", \"pref\": 10, \"ttl\": 3600}"
```

### Level 3: Infrastructure as Code

For repeatable, version-controlled setups:

| Tool | What it does | Existing integration |
|------|-------------|---------------------|
| **Ansible** | DNS record management, server config | `felixfontein/ansible-hosttech_dns` collection |
| **Terraform** | DNS + SSL cert automation | ACME provider with hosttech DNS challenge |

**Example: Ansible playbook for new customer DNS**

```yaml
- name: Setup hallo-car.ch DNS
  hosts: localhost
  tasks:
    - name: Create A record
      community.dns.hosttech_dns_record:
        zone_name: hallo-car.ch
        type: A
        record: ""
        value: "185.x.x.x"
        ttl: 3600
        api_token: "{{ hosttech_api_key }}"
```

### Level 4: Full Pipeline (End-to-End)

Combine the pieces into a single provisioning script:

```
New customer order
  → 1. Domain API: Register hallo-car.ch
  → 2. Plesk API: Create customer + subscription
  → 3. DNS API: Configure zone + records
  → 4. Plesk API: Install Let's Encrypt SSL
  → 5. Plesk API: Install WordPress (optional)
  → 6. Email: Send credentials to customer
```

This can be a Python/Node.js script, a WHMCS hook, or an Ansible playbook.

---

## Recommendation

| Scenario | Approach |
|----------|----------|
| **1-5 customers, occasional** | Manual via Plesk + myhosttech (Part 1) |
| **5-20 customers, growing** | WHMCS (free with your plan) + DNS API scripts |
| **20+ customers, professional** | WHMCS fully configured + custom API integrations |
| **DevOps-oriented** | Ansible/Terraform + DNS API + Plesk API |

For **hallo-car.ch right now**, the manual process (Part 1) is the practical path. If you plan to onboard more customers regularly, setting up WHMCS is the highest-value next step since it's already included in your reseller plan.

---

## Key URLs

| Resource | URL |
|----------|-----|
| DNS API docs | https://api.ns1.hosttech.eu/api/documentation/ |
| Domain API docs | Available in myhosttech → Servercenter → Solutions |
| Plesk panel | https://cp.hosttech.eu/ |
| myhosttech login | https://www.myhosttech.eu/login |
| WHMCS modules | Downloadable from myhosttech Kundencenter |
| Ansible hosttech_dns | https://github.com/felixfontein/ansible-hosttech_dns |
| Terraform ACME provider | https://registry.terraform.io/providers/vancluever/acme/latest/docs/guides/dns-providers-hosttech |
