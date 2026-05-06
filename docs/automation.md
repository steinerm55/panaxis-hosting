# Automation Guide

Options for automating customer provisioning at hosttech.ch, from turnkey to fully custom.

---

## Available APIs

hosttech provides three APIs for resellers:

| API | Base URL | Auth | Documentation |
|-----|----------|------|---------------|
| **DNS REST API** | `https://api.ns1.hosttech.eu/api/` | Bearer token | [Swagger docs](https://api.ns1.hosttech.eu/api/documentation/) |
| **Domain Reselling API** | via myhosttech Servercenter | 5 protocols (HTTP, SOAP, XML/RPC, Mreg, MailRobot) | [PDF](https://support.hosttech.ch/wp-content/uploads/2022/01/hosttech_DOMAIN_API.pdf) |
| **Website Creator API** | `https://wsc01.hosttech.eu/api/` | API key | [Docs](https://wsc01.hosttech.eu/api/) |

Additionally, the **Plesk REST/XML API** is available on the hosting server (`https://cp.hosttech.eu:8443/`) for subscription and email provisioning.

---

## Level 1: WHMCS (Turnkey Solution)

**Included free** with hosttech reseller plans. This is the recommended starting point.

### What WHMCS automates

| Process | Manual | With WHMCS |
|---------|--------|------------|
| Customer signup | You create in Plesk | Customer self-registers on your website |
| Domain registration | You register in myhosttech | Automatic on order confirmation |
| Hosting provisioning | You create subscription in Plesk | Automatic via Plesk module |
| SSL certificate | You install in Plesk | Auto-provisioned with Let's Encrypt |
| Invoicing | You send manually | Automatic recurring invoices |
| Payment collection | Manual bank transfer | PayPal, Stripe, bank transfer |
| Renewal reminders | You remember | Automatic emails |
| Suspension | You check manually | Auto-suspend on non-payment |

### Setup steps

1. **Download modules** from myhosttech Kundencenter:
   - hosttech domain registrar module
   - hosttech domain pricing sync module
   - hosttech DNS module
2. **Connect Plesk:** WHMCS → Setup → Servers → add Plesk server
3. **Create products:** Map your reseller plans to WHMCS products
4. **Configure payment:** Add payment gateways (PayPal, Stripe, etc.)
5. **Brand your portal:** Customize templates with panaxis-ltd.ch branding

### Customer flow with WHMCS

```
Customer visits your website
  → Selects hosting plan + domain
  → Pays via PayPal/Stripe/invoice
  → WHMCS registers domain (Domain API)
  → WHMCS creates Plesk subscription (Plesk API)
  → WHMCS sends credentials email to customer
  → Done — zero manual work
```

---

## Level 2: DNS API Scripts

The DNS REST API is modern, well-documented, and easy to script against.

### Authentication

Generate an API token in myhosttech Kundencenter → DNS → API Tokens.

```bash
export HOSTTECH_API_KEY="your-token-here"
export HOSTTECH_API="https://api.ns1.hosttech.eu/api/user/v1"
```

### Script: Provision DNS for a new customer

```bash
#!/bin/bash
# Usage: ./provision-dns.sh <domain> <server-ip>
set -euo pipefail

DOMAIN="${1:?Usage: $0 <domain> <server-ip>}"
IP="${2:?Usage: $0 <domain> <server-ip>}"
API="${HOSTTECH_API:-https://api.ns1.hosttech.eu/api/user/v1}"
TOKEN="${HOSTTECH_API_KEY:?Set HOSTTECH_API_KEY}"

header=(-H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json")

echo "==> Creating DNS zone for $DOMAIN"
ZONE_RESPONSE=$(curl -s -X POST "$API/zones" "${header[@]}" \
  -d "{\"name\": \"$DOMAIN\", \"type\": \"NATIVE\"}")
ZONE_ID=$(echo "$ZONE_RESPONSE" | jq -r '.data.id')

if [ "$ZONE_ID" = "null" ]; then
  echo "ERROR: Failed to create zone. Response:"
  echo "$ZONE_RESPONSE" | jq .
  exit 1
fi
echo "    Zone ID: $ZONE_ID"

add_record() {
  local data="$1"
  local desc="$2"
  echo "==> Adding $desc"
  curl -s -X POST "$API/zones/$ZONE_ID/records" "${header[@]}" -d "$data" | jq -r '.data.type + " " + .data.name'
}

# A record (root)
add_record "{\"type\":\"A\",\"name\":\"\",\"ipv4\":\"$IP\",\"ttl\":3600}" "A record → $IP"

# A record (www)
add_record "{\"type\":\"A\",\"name\":\"www\",\"ipv4\":\"$IP\",\"ttl\":3600}" "A www → $IP"

# MX record
add_record "{\"type\":\"MX\",\"name\":\"\",\"ownmx\":\"mail.$DOMAIN\",\"pref\":10,\"ttl\":3600}" "MX → mail.$DOMAIN"

# A record (mail)
add_record "{\"type\":\"A\",\"name\":\"mail\",\"ipv4\":\"$IP\",\"ttl\":3600}" "A mail → $IP"

# SPF
add_record "{\"type\":\"TXT\",\"name\":\"\",\"text\":\"v=spf1 a mx ~all\",\"ttl\":3600}" "SPF record"

echo ""
echo "==> DNS provisioning complete for $DOMAIN"
echo "    Zone ID: $ZONE_ID"
echo "    Server:  $IP"
echo "    Verify:  dig $DOMAIN @ns1.hosttech.eu A +short"
```

### Script: List all zones

```bash
curl -s "$HOSTTECH_API/zones" \
  -H "Authorization: Bearer $HOSTTECH_API_KEY" | jq '.data[] | {id, name, type}'
```

### Script: Bulk update IP (e.g. server migration)

```bash
# Change all A records from old IP to new IP across all zones
curl -s -X POST "$HOSTTECH_API/tools/changeip" \
  -H "Authorization: Bearer $HOSTTECH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"old_ip": "185.1.2.3", "new_ip": "185.4.5.6"}'
```

---

## Level 3: Domain Registration API

The Domain Reselling API supports five protocols. HTTP is the most practical for scripting.

### Access

Credentials are in myhosttech → Servercenter → Solutions tab.

Documentation: [Domain API PDF](https://support.hosttech.ch/wp-content/uploads/2022/01/hosttech_DOMAIN_API.pdf)

### Example: Check domain availability (SOAP)

```python
from zeep import Client

wsdl = "https://ns1.hosttech.eu/wsdl"
client = Client(wsdl)

result = client.service.domainCheck(
    username="your-reseller-user",
    password="your-reseller-pass",
    domain="example",
    tld="ch"
)
print(result)  # available / registered / ...
```

### Example: Register domain (SOAP)

```python
result = client.service.domainCreate(
    username="your-reseller-user",
    password="your-reseller-pass",
    domain="hallo-car",
    tld="ch",
    ownerContact={"firstname": "Max", "lastname": "Muster", ...},
    adminContact={"firstname": "Panaxis", ...},
    techContact={"firstname": "Panaxis", ...},
    ns1="ns1.hosttech.eu",
    ns2="ns2.hosttech.eu",
    ns3="ns3.hosttech.eu"
)
```

---

## Level 4: Infrastructure as Code

### Ansible — DNS management

Using the community collection [`felixfontein/ansible-hosttech_dns`](https://github.com/felixfontein/ansible-hosttech_dns):

```bash
ansible-galaxy collection install community.dns
```

```yaml
# playbook: provision-customer.yml
# Usage: ansible-playbook provision-customer.yml -e "domain=hallo-car.ch server_ip=185.x.x.x"
---
- name: Provision DNS for new customer
  hosts: localhost
  connection: local
  vars:
    hosttech_token: "{{ lookup('env', 'HOSTTECH_API_KEY') }}"
  tasks:
    - name: A record (root)
      community.dns.hosttech_dns_record:
        zone_name: "{{ domain }}"
        type: A
        record: ""
        value: "{{ server_ip }}"
        ttl: 3600
        api_token: "{{ hosttech_token }}"

    - name: A record (www)
      community.dns.hosttech_dns_record:
        zone_name: "{{ domain }}"
        type: A
        record: "www"
        value: "{{ server_ip }}"
        ttl: 3600
        api_token: "{{ hosttech_token }}"

    - name: MX record
      community.dns.hosttech_dns_record:
        zone_name: "{{ domain }}"
        type: MX
        record: ""
        value: "10 mail.{{ domain }}"
        ttl: 3600
        api_token: "{{ hosttech_token }}"

    - name: SPF record
      community.dns.hosttech_dns_record:
        zone_name: "{{ domain }}"
        type: TXT
        record: ""
        value: "v=spf1 a mx ~all"
        ttl: 3600
        api_token: "{{ hosttech_token }}"
```

### Terraform — SSL + DNS challenges

Using the [ACME provider with hosttech DNS](https://registry.terraform.io/providers/vancluever/acme/latest/docs/guides/dns-providers-hosttech):

```hcl
provider "acme" {
  server_url = "https://acme-v02.api.letsencrypt.org/directory"
}

resource "acme_certificate" "customer" {
  account_key_pem = tls_private_key.acme.private_key_pem
  common_name     = var.domain

  dns_challenge {
    provider = "hosttech"
    config = {
      HOSTTECH_API_KEY = var.hosttech_api_key
    }
  }
}
```

---

## Level 5: Full Automation Pipeline

Combine all APIs into a single provisioning command:

```
./provision-customer.sh hallo-car.ch

  1. Domain API  → Register hallo-car.ch
  2. DNS API     → Create zone + A/MX/SPF records
  3. Plesk API   → Create customer account
  4. Plesk API   → Create hosting subscription
  5. Plesk API   → Install Let's Encrypt SSL
  6. Plesk API   → Install WordPress (optional)
  7. Email       → Send credentials to customer
```

### Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│  Your        │────▶│  Domain API       │────▶│  SWITCH      │
│  Script /    │     │  (SOAP/HTTP)      │     │  Registry    │
│  WHMCS       │     └──────────────────┘     └─────────────┘
│              │
│              │     ┌──────────────────┐     ┌─────────────┐
│              │────▶│  DNS REST API     │────▶│  hosttech    │
│              │     │  (Bearer token)   │     │  nameservers │
│              │     └──────────────────┘     └─────────────┘
│              │
│              │     ┌──────────────────┐     ┌─────────────┐
│              │────▶│  Plesk API        │────▶│  Hosting     │
│              │     │  (REST/XML)       │     │  server      │
└─────────────┘     └──────────────────┘     └─────────────┘
```

---

## Third-Party Integrations

| Tool | Language | Purpose | Link |
|------|----------|---------|------|
| libdns/hosttech | Go | DNS record management | [GitHub](https://github.com/libdns/hosttech) |
| ansible-hosttech_dns | Ansible | DNS automation | [GitHub](https://github.com/felixfontein/ansible-hosttech_dns) |
| lego ACME | Go | Let's Encrypt DNS challenges | [Docs](https://go-acme.github.io/lego/dns/hosttech/) |
| Terraform ACME | HCL | Infrastructure as Code | [Registry](https://registry.terraform.io/providers/vancluever/acme/latest/docs/guides/dns-providers-hosttech) |
| dnsAPI | Shell | DNS API shell client | [GitHub](https://github.com/mbiegert/dnsAPI) |

---

## Recommendation by Scale

| Customers | Approach | Effort |
|-----------|----------|--------|
| 1–5 | Manual (follow the [recipe](customer-setup-recipe)) | None |
| 5–20 | WHMCS + DNS API scripts | Low — WHMCS is free with your plan |
| 20–50 | WHMCS fully configured + custom API hooks | Medium |
| 50+ | Full pipeline (WHMCS + all APIs + monitoring) | High |
| DevOps shop | Ansible/Terraform + CI/CD pipeline | Medium |
