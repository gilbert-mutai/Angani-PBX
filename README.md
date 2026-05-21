# 📘 WHITE-LABEL PBX (FusionPBX + FreeSWITCH)

**Full Implementation Guide — From Zero to SaaS Launch**

---

## 🧭 1. Architecture Overview

Your system is organised into six layers — from client devices at the top down to your SaaS business layer at the bottom.

```mermaid
flowchart TD
    subgraph Clients["👥 Client Layer"]
        A[🌐 Web Browser]
        B[💻 Softphone App]
        C[📱 Mobile SIP App]
    end

    subgraph SIPLayer["🔀 SIP / WebRTC Layer"]
        D[SIP Proxy / WebRTC Gateway]
    end

    subgraph Carriers["📡 Telecom Carriers"]
        N[🇰🇪 Local Trunks\nTelkom · Safaricom]
        O[🌍 International Providers\nTwilio · DIDWW]
    end

    subgraph Voice["📞 Voice Engine"]
        E[FreeSWITCH\nMedia · RTP · NAT · Codecs]
    end

    subgraph Control["🧠 PBX Control Layer"]
        F[FusionPBX\nMulti-tenant UI · API · Dial Plans]
    end

    subgraph Data["🗄️ Data Layer"]
        G[(PostgreSQL)]
        H[📁 Call Recordings]
        I[📋 CDR Logs]
    end

    subgraph Business["💼 Business Layer"]
        J[🖥️ Customer Portal]
        K[🔧 Admin Portal]
        L[💳 Billing\nStripe · Flutterwave]
        M[🤖 Automation\nCRM · AI · WhatsApp]
    end

    A & B & C -->|SIP / WebRTC| D
    D -->|SIP| E
    N & O <-->|SIP Trunks| E
    E <-->|ESL / API| F
    F --> G & H & I
    F --> J & K
    J & K --> L
    L --> M
```

---

## 🧰 2. Requirements

### 🖥️ Infrastructure

**Minimum Production Setup (small SaaS start)**

| Role | Count |
|------|-------|
| Application Server (FusionPBX) | 1× |
| Media Server (FreeSWITCH) | 1× |
| Database Server (PostgreSQL) | 1× (can be combined initially) |

**Recommended Cloud Specs**

- CPU: 4–8 cores per server
- RAM: 8–16 GB minimum
- SSD: 100 GB+
- OS: Debian 11/12 or Ubuntu 22.04 LTS

### 🌐 Networking Requirements

- Static public IP (**VERY important**)
- Open ports:
  - SIP: `5060` / `5061`
  - RTP media: `16384–32768` (UDP)
  - Web: `80` / `443`
- Firewall: UFW or iptables
- NAT configuration (critical for audio quality)

### 🔧 Software Stack

**Core**

- FreeSWITCH
- FusionPBX
- PostgreSQL
- Nginx / Apache (FusionPBX web server)

**Supporting tools**

- Fail2Ban (security)
- Certbot (SSL)
- Redis (optional caching layer)
- Kamailio/OpenSIPS (optional scaling later)

**📊 Business Layer (you will add)**

- Billing system (custom or Stripe integration)
- CRM / provisioning tool
- Customer portal (your white-label layer)
- Monitoring (Grafana + Prometheus — optional)

---

## ⚙️ 3. Installation Phase (Step-by-Step)

### 🧱 STEP 1: Prepare Server

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl wget nano unzip -y
```

Set hostname:

```bash
hostnamectl set-hostname pbx.yourdomain.com
```

### 🔐 STEP 2: Firewall Setup

```bash
ufw allow 22
ufw allow 80
ufw allow 443
ufw allow 5060:5061/udp
ufw allow 16384:32768/udp
ufw enable
```

### 📦 STEP 3: Install FreeSWITCH

Install dependencies:

```bash
apt install -y gnupg2 wget lsb-release
```

Add FreeSWITCH official repo and install:

```bash
wget -O - https://files.freeswitch.org/repo/deb/debian-release/fsstretch-archive-keyring.asc | apt-key add -
echo "deb [signed-by=/usr/share/keyrings/freeswitch-archive-keyring.gpg] https://files.freeswitch.org/repo/deb/debian-release/ $(lsb_release -sc) main" \
  > /etc/apt/sources.list.d/freeswitch.list
apt update
apt install -y freeswitch-meta-all
```

> 👉 For production SaaS, building from source is preferred for full codec and module control.

### 🧠 STEP 4: Install FusionPBX

```bash
cd /usr/src
git clone https://github.com/fusionpbx/fusionpbx-install.sh.git
cd fusionpbx-install.sh/debian
./install.sh
```

This installs:

- FreeSWITCH
- FusionPBX
- PostgreSQL
- Web UI
- Default configuration

### 🌐 STEP 5: Configure Domain

Access:

```
https://your-server-ip
```

Then:

1. Create admin account
2. Set domain (your SaaS domain)
3. Enable SSL (Let's Encrypt)

---

## 🏢 4. Multi-Tenant (White Label Core)

> This is the **MOST IMPORTANT PART**.

FusionPBX supports **Domains = Tenants**.

For each customer, create a new domain:

```
client1.yourpbx.com
client2.yourpbx.com
```

Or use a separate SIP realm per tenant:

```
client1-sip.yourdomain.com
```

### 👤 Tenant Isolation Model

Each tenant gets:

- Extensions
- SIP users
- IVR
- Call queues
- Voicemail
- CDR logs

> 👉 Fully isolated inside PostgreSQL schema logic.

### 🗄️ PostgreSQL Multi-Tenant Schema Design

FusionPBX uses a **shared schema** with `domain_uuid` as the tenant key on every row. Understanding this is critical for custom billing queries and reporting.

**Key tables and their tenant key:**

```sql
-- Every major table carries domain_uuid
SELECT table_name
FROM information_schema.columns
WHERE column_name = 'domain_uuid'
  AND table_schema = 'public';
```

**Useful queries for tenant management:**

```sql
-- List all tenants (domains)
SELECT domain_uuid, domain_name, domain_enabled
FROM v_domains
ORDER BY domain_name;

-- Count extensions per tenant
SELECT d.domain_name, COUNT(e.extension_uuid) AS extensions
FROM v_domains d
LEFT JOIN v_extensions e USING (domain_uuid)
GROUP BY d.domain_name
ORDER BY extensions DESC;

-- Pull CDR for a specific tenant (last 30 days)
SELECT start_epoch, caller_id_number, destination_number,
       duration, billsec, hangup_cause
FROM v_xml_cdr
WHERE domain_uuid = '<tenant-uuid>'
  AND start_epoch > EXTRACT(EPOCH FROM NOW() - INTERVAL '30 days')
ORDER BY start_epoch DESC;
```

**Creating a new tenant via SQL (programmatic provisioning):**

```sql
INSERT INTO v_domains (domain_uuid, domain_name, domain_enabled, domain_description)
VALUES (gen_random_uuid(), 'client1.yourpbx.com', 'true', 'Client One');
```

### 🔐 Role-Based Access Control (RBAC)

FusionPBX ships with built-in groups. Map these to your SaaS roles:

| FusionPBX Group | Your SaaS Role | Permissions |
|-----------------|----------------|-------------|
| superadmin | Platform Admin | Full system access |
| admin | Tenant Admin | Manage own domain only |
| user | End User | Own extension + voicemail |
| agent | Call Centre Agent | Queues + own CDR |

**Assign a user to a group:**

```sql
INSERT INTO v_group_users (group_user_uuid, domain_uuid, group_name, user_uuid)
VALUES (gen_random_uuid(), '<domain_uuid>', 'admin', '<user_uuid>');
```

**⚠️ Production pitfall:** Never give tenant admins access to `Advanced → Defaults` — they can overwrite global FreeSWITCH config.

---

## 🎨 5. White-Label Branding

### File Locations

```
/var/www/fusionpbx/                  ← PHP source
/var/www/fusionpbx/themes/default/   ← CSS, images, layout
/var/www/fusionpbx/app/              ← Per-app templates
```

**Key changes:**

- Replace FusionPBX logo
- Change "FusionPBX" text in templates
- Apply custom CSS theme

### Nginx Reverse Proxy Branding Layer (Recommended)

Rather than editing FusionPBX source directly (hard to maintain across upgrades), run a branded Nginx reverse proxy in front of FusionPBX:

```nginx
# /etc/nginx/sites-available/pbx-whitelabel

server {
    listen 443 ssl http2;
    server_name portal.yourpbx.com;

    ssl_certificate     /etc/letsencrypt/live/portal.yourpbx.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/portal.yourpbx.com/privkey.pem;

    # Inject custom branding CSS on every HTML response
    sub_filter '</head>'
        '<link rel="stylesheet" href="/custom/brand.css">
         <script src="/custom/brand.js"></script></head>';
    sub_filter_once on;
    sub_filter_types text/html;

    # Serve your own branding assets
    location /custom/ {
        alias /var/www/whitelabel/assets/;
    }

    # Override the FusionPBX logo
    location = /themes/default/images/logo.png {
        alias /var/www/whitelabel/assets/logo.png;
    }

    # Proxy everything else to FusionPBX
    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto https;
        proxy_read_timeout 90;
    }
}
```

**`/var/www/whitelabel/assets/brand.css` example:**

```css
/* Hide "FusionPBX" text in footer and header */
.footer-copyright, #footer { display: none !important; }

/* Replace page title */
.navbar-brand::after { content: "MyPBX Portal"; }
.navbar-brand img { content: url("/custom/logo.png"); }

/* Brand colours */
:root {
  --primary: #0066cc;
  --secondary: #004499;
}
```

> **⚠️ Production pitfall:** `sub_filter` requires `ngx_http_sub_module`. Confirm with `nginx -V 2>&1 | grep sub`.

---

## 📞 6. SIP & Call Setup

### Configure Trunks

Connect:

- Local carriers in Kenya (Telkom, Safaricom SIP where available)
- International SIP providers (Twilio, DIDWW, etc.)

In FusionPBX: **Accounts → Gateways → Add SIP Trunk**

**Example gateway config (Twilio):**

```xml
<!-- /etc/freeswitch/sip_profiles/external/twilio.xml -->
<gateway name="twilio">
  <param name="username"     value="ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"/>
  <param name="password"     value="your_auth_token"/>
  <param name="proxy"        value="your-account.pstn.twilio.com"/>
  <param name="register"     value="false"/>
  <param name="caller-id-in-from" value="true"/>
  <param name="contact-params" value="transport=tls"/>
</gateway>
```

### RTP Optimization (**VERY IMPORTANT**)

Edit `/etc/freeswitch/autoload_configs/switch.conf.xml`:

```xml
<param name="rtp-start-port" value="16384"/>
<param name="rtp-end-port"   value="32768"/>

<!-- NAT traversal — set to your server's PUBLIC IP -->
<param name="ext-rtp-ip"  value="autonat:YOUR_PUBLIC_IP"/>
<param name="ext-sip-ip"  value="YOUR_PUBLIC_IP"/>
```

Edit `/etc/freeswitch/sip_profiles/internal.xml`:

```xml
<param name="ext-rtp-ip"  value="$${external_rtp_ip}"/>
<param name="ext-sip-ip"  value="$${external_sip_ip}"/>
<param name="local-network-acl" value="localnet.auto"/>
<param name="aggressive-nat-detection" value="true"/>
```

> **⚠️ Production pitfall:** Misconfigured NAT is the #1 cause of one-way audio. Always test with `sngrep` to trace SIP and verify RTP flows.

---

## 📱 7. Softphone Strategy (Customer Experience)

**Option A: Use existing SIP apps (fastest)**

- Zoiper
- Linphone
- Grandstream Wave

> ✔ No development needed

**Option B: Branded softphone (recommended SaaS path)**

- Build React Native or Flutter app
- Use SIP SDK: PJSIP or Linphone SDK

**Option C: WebRTC browser phone**

- Built into FusionPBX (limited)
- Or custom WebRTC SIP client using [JsSIP](https://jssip.net/) + FreeSWITCH `mod_verto`

**Recommended provisioning flow for softphones:**

```
Customer signs up → Portal generates SIP credentials → 
QR code / email sent → Customer scans in softphone app → Connected
```

---

## 💳 8. Billing System Design

FusionPBX does **NOT** provide full billing. You must add your own.

### PostgreSQL Billing Schema

Add these tables to your own database (separate from FusionPBX's DB):

```sql
-- Tenants (mirrors v_domains)
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_uuid     UUID UNIQUE NOT NULL,   -- FK to FusionPBX v_domains
    company_name    TEXT NOT NULL,
    email           TEXT NOT NULL,
    plan            TEXT NOT NULL DEFAULT 'starter',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Subscription plans
CREATE TABLE plans (
    plan_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,          -- e.g. 'starter', 'business', 'enterprise'
    monthly_fee     NUMERIC(10,2) NOT NULL,
    included_minutes INT NOT NULL DEFAULT 0,
    max_extensions  INT NOT NULL DEFAULT 5,
    price_per_min   NUMERIC(8,4) NOT NULL DEFAULT 0.0200
);

-- Extensions being billed
CREATE TABLE billed_extensions (
    ext_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(tenant_id),
    extension_uuid  UUID NOT NULL,          -- FK to FusionPBX v_extensions
    extension_num   TEXT NOT NULL,
    monthly_fee     NUMERIC(10,2) NOT NULL DEFAULT 5.00,
    active          BOOLEAN DEFAULT TRUE
);

-- CDR-based call billing (populated from FreeSWITCH CDR)
CREATE TABLE call_records (
    cdr_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(tenant_id),
    call_uuid       UUID NOT NULL,
    caller          TEXT,
    destination     TEXT,
    direction       TEXT,                   -- 'inbound' | 'outbound'
    start_time      TIMESTAMPTZ,
    duration_sec    INT,
    billable_sec    INT,
    cost            NUMERIC(10,4),
    carrier         TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Monthly invoices
CREATE TABLE invoices (
    invoice_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(tenant_id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    subtotal        NUMERIC(10,2),
    tax             NUMERIC(10,2) DEFAULT 0,
    total           NUMERIC(10,2),
    status          TEXT DEFAULT 'draft',   -- draft | sent | paid | overdue
    stripe_invoice_id TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### CDR Processing Flow

```
FreeSWITCH call ends
        ↓
mod_cdr_pg_csv writes to v_xml_cdr
        ↓
Your billing worker polls v_xml_cdr every 60s
        ↓
Lookup tenant by domain_uuid
        ↓
Calculate cost: (billsec / 60) × rate_per_minute
        ↓
INSERT into call_records
        ↓
Aggregate monthly → generate invoice
```

**Python billing worker (minimal example):**

```python
import psycopg2, time

FUSIONPBX_DSN = "host=localhost dbname=fusionpbx user=fusionpbx password=secret"
BILLING_DSN   = "host=localhost dbname=billing   user=billing   password=secret"

RATE_PER_MIN = 0.02  # $0.02/min default

def process_new_cdrs():
    fpbx = psycopg2.connect(FUSIONPBX_DSN)
    bill = psycopg2.connect(BILLING_DSN)

    with fpbx.cursor() as cur:
        cur.execute("""
            SELECT xml_cdr_uuid, domain_uuid, caller_id_number,
                   destination_number, start_epoch, billsec
            FROM v_xml_cdr
            WHERE start_epoch > EXTRACT(EPOCH FROM NOW() - INTERVAL '2 minutes')
              AND billsec > 0
        """)
        rows = cur.fetchall()

    with bill.cursor() as cur:
        for row in rows:
            cdr_id, domain_uuid, caller, dest, start_ep, billsec = row
            cost = round((billsec / 60) * RATE_PER_MIN, 4)
            cur.execute("""
                INSERT INTO call_records
                    (call_uuid, tenant_id, caller, destination,
                     start_time, billable_sec, cost)
                SELECT %s, tenant_id, %s, %s,
                       to_timestamp(%s), %s, %s
                FROM tenants WHERE domain_uuid = %s
                ON CONFLICT DO NOTHING
            """, (cdr_id, caller, dest, start_ep, billsec, cost, domain_uuid))
        bill.commit()

while True:
    process_new_cdrs()
    time.sleep(60)
```

### Stripe Integration

```python
import stripe
stripe.api_key = "sk_live_..."

def charge_tenant(tenant, invoice):
    # Create or retrieve Stripe customer
    if not tenant.stripe_customer_id:
        customer = stripe.Customer.create(
            email=tenant.email,
            name=tenant.company_name,
            metadata={"tenant_id": str(tenant.tenant_id)}
        )
        tenant.stripe_customer_id = customer.id

    # Create invoice
    inv = stripe.Invoice.create(
        customer=tenant.stripe_customer_id,
        auto_advance=True,
        collection_method="charge_automatically",
    )
    stripe.InvoiceItem.create(
        customer=tenant.stripe_customer_id,
        invoice=inv.id,
        amount=int(invoice.total * 100),  # cents
        currency="usd",
        description=f"PBX Service {invoice.period_start} – {invoice.period_end}"
    )
    stripe.Invoice.finalize_invoice(inv.id)
    return inv.id
```

**Flutterwave (for Africa — KES / NGN):**

```python
import requests

def charge_flutterwave(tenant, invoice_total_kes):
    resp = requests.post("https://api.flutterwave.com/v3/charges?type=mobile_money_kenya", 
        headers={"Authorization": "Bearer FLWSECK_TEST-..."},
        json={
            "phone_number": tenant.phone,
            "amount": invoice_total_kes,
            "currency": "KES",
            "email": tenant.email,
            "tx_ref": str(invoice.invoice_id),
            "network": "SAFARICOM"
        }
    )
    return resp.json()
```

---

## 🖥️ 9. Customer Portal Architecture

### Recommended Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Frontend | Next.js (React) | SSR + API routes, fast |
| Backend API | FastAPI (Python) or Node.js | Easy PostgreSQL integration |
| Auth | JWT + refresh tokens | Stateless, works with SIP |
| Database | PostgreSQL (billing DB above) | Unified with billing |
| Hosting | VPS behind Nginx | Same server or separate |

### Core API Endpoints

```
POST   /auth/login                  → JWT token
GET    /dashboard                   → usage summary, active calls
GET    /extensions                  → list extensions
POST   /extensions                  → provision new extension
DELETE /extensions/:id              → remove extension
GET    /cdr?from=&to=               → call history
GET    /invoices                    → invoice list
GET    /invoices/:id/pdf            → download invoice
POST   /billing/payment-method      → attach Stripe card
GET    /sip-credentials/:ext_id     → return SIP username/password
POST   /support/ticket              → create support ticket
```

### Customer Portal Flow

```mermaid
flowchart LR
    A[Customer signs up] --> B[Portal creates FusionPBX domain]
    B --> C[Assigns plan & extensions]
    C --> D[Generates SIP credentials]
    D --> E[Sends welcome email with QR code]
    E --> F[Customer logs into portal]
    F --> G[Views CDR, invoices, usage]
    G --> H[Pays via Stripe / Flutterwave]
```

### Auto-Provisioning a New Tenant (Python + requests)

```python
import requests, uuid

FUSIONPBX_API = "https://pbx.yourpbx.com"
API_KEY = "your-fusionpbx-api-key"

def provision_tenant(company_name: str, domain: str, admin_email: str):
    # 1. Create domain in FusionPBX
    requests.post(f"{FUSIONPBX_API}/app/api/index.php", json={
        "method": "domain_add",
        "domain_name": domain,
        "domain_enabled": "true"
    }, headers={"X-API-Key": API_KEY})

    # 2. Create admin user
    requests.post(f"{FUSIONPBX_API}/app/api/index.php", json={
        "method": "user_add",
        "domain_name": domain,
        "username": admin_email,
        "password": generate_password(),
        "user_enabled": "true",
        "groups": ["admin"]
    }, headers={"X-API-Key": API_KEY})

    # 3. Record in your billing DB
    db.execute("""
        INSERT INTO tenants (domain_uuid, company_name, email)
        VALUES ((SELECT domain_uuid FROM v_domains WHERE domain_name=%s), %s, %s)
    """, (domain, company_name, admin_email))
```

---

## 📊 10. Call Data & Analytics

Enable:

- Call Detail Records (CDR)
- Call recording storage
- Real-time monitoring

You will use:

- FusionPBX CDR tables
- FreeSWITCH event socket (advanced analytics)

### Enable CDR in FreeSWITCH

```xml
<!-- /etc/freeswitch/autoload_configs/cdr_pg_csv.conf.xml -->
<configuration name="cdr_pg_csv.conf" description="CDR to PostgreSQL">
  <settings>
    <param name="db-info" value="host=localhost dbname=fusionpbx user=fusionpbx password=secret connect_timeout=10"/>
    <param name="log-b-leg" value="true"/>
  </settings>
</configuration>
```

### Real-Time Call Monitoring via ESL

```python
from ESL import ESLconnection

con = ESLconnection("127.0.0.1", "8021", "ClueCon")
con.events("plain", "CHANNEL_CREATE CHANNEL_DESTROY CHANNEL_ANSWER")

while True:
    e = con.recvEvent()
    if e:
        event_name  = e.getHeader("Event-Name")
        call_uuid   = e.getHeader("Unique-ID")
        caller      = e.getHeader("Caller-Caller-ID-Number")
        destination = e.getHeader("Caller-Destination-Number")
        domain      = e.getHeader("variable_domain_name")
        print(f"{event_name} | {domain} | {caller} → {destination}")
        # Push to Redis pub/sub for live dashboard
```

---

## 🔐 11. Security Hardening

### Fail2Ban — SIP Protection

```bash
apt install fail2ban
```

Create `/etc/fail2ban/jail.d/freeswitch.conf`:

```ini
[freeswitch]
enabled  = true
port     = 5060,5061
protocol = udp
filter   = freeswitch
logpath  = /var/log/freeswitch/freeswitch.log
maxretry = 5
findtime = 60
bantime  = 3600

[freeswitch-tcp]
enabled  = true
port     = 5060,5061
protocol = tcp
filter   = freeswitch
logpath  = /var/log/freeswitch/freeswitch.log
maxretry = 5
bantime  = 3600
```

Create `/etc/fail2ban/filter.d/freeswitch.conf`:

```ini
[Definition]
failregex = \[WARNING\] sofia_reg.c:\d+ SIP auth failure \(REGISTER\) on sofia profile \'[^']+\' for \[.*\] from ip <HOST>
            \[WARNING\] sofia_reg.c:\d+ SIP auth failure \(INVITE\) on sofia profile \'[^']+\' for \[.*\] from ip <HOST>
ignoreregex =
```

```bash
systemctl restart fail2ban
fail2ban-client status freeswitch   # verify
```

### Anti-Fraud (Toll Fraud Prevention)

Toll fraud is the #1 risk in VoIP SaaS. Implement all of these:

**1. Disable anonymous SIP calls:**

```xml
<!-- /etc/freeswitch/sip_profiles/internal.xml -->
<param name="auth-calls" value="true"/>
<param name="auth-all-packets" value="false"/>
```

**2. International call restrictions by default:**

```xml
<!-- dialplan: block international unless tenant has it enabled -->
<condition field="destination_number" expression="^(00|011|\+)" break="on-true">
  <action application="check_acl" data="${domain_name} international_allowed"/>
  <anti-action application="hangup" data="CALL_REJECTED"/>
</condition>
```

**3. Per-tenant concurrent call limits:**

```xml
<action application="limit" data="db ${domain_name} max_calls 10 !NORMAL_CLEARING"/>
```

**4. Per-extension daily spend cap (check in billing worker):**

```sql
-- Alert if a single extension spends >$20 in one day
SELECT extension_num, SUM(cost) AS daily_spend
FROM call_records
WHERE DATE(start_time) = CURRENT_DATE
GROUP BY extension_num
HAVING SUM(cost) > 20;
```

**5. IP whitelisting for the FusionPBX admin panel:**

```nginx
location /core/ {
    allow 196.216.0.0/16;   # your office IP range
    deny all;
}
```

### Backup & Disaster Recovery

```bash
# Daily PostgreSQL backup
pg_dump fusionpbx | gzip > /backups/fusionpbx_$(date +%F).sql.gz

# Sync to S3-compatible storage (Backblaze B2 / AWS S3)
aws s3 sync /backups/ s3://your-pbx-backups/ --delete

# Back up FreeSWITCH config
tar czf /backups/freeswitch_conf_$(date +%F).tar.gz /etc/freeswitch/
```

**Recovery time target:** With daily backups + config snapshots, you should be able to restore a new server in under 2 hours.

---

## 📈 12. Scaling Strategy (**VERY IMPORTANT** for SaaS)

| Phase | Setup |
|-------|-------|
| Phase 1 | Single FusionPBX + FreeSWITCH server |
| Phase 2 | Separate DB server, PBX controller, and media servers |
| Phase 3 (carrier-grade) | Kamailio SIP proxy, multiple FreeSWITCH nodes, Kubernetes or VM cluster |

### Phase 3 — Kamailio SIP Proxy Setup

Kamailio sits in front of multiple FreeSWITCH nodes and load-balances SIP:

```
SIP Clients
    ↓
Kamailio (dispatcher module)
    ↓            ↓
FreeSWITCH-1  FreeSWITCH-2
```

**`/etc/kamailio/kamailio.cfg` dispatcher block:**

```
loadmodule "dispatcher.so"

modparam("dispatcher", "db_url", "postgres://kamailio:pass@localhost/kamailio")
modparam("dispatcher", "ds_ping_interval", 10)
modparam("dispatcher", "ds_probing_mode", 1)

route[DISPATCH] {
    if (!ds_select_dst("1", "4")) {   # group 1, round-robin
        send_reply("503", "Service Unavailable");
        exit;
    }
    t_relay();
}
```

**Insert FreeSWITCH nodes into dispatcher table:**

```sql
INSERT INTO dispatcher (setid, destination, attrs)
VALUES (1, 'sip:192.168.1.10:5060', 'weight=50'),
       (1, 'sip:192.168.1.11:5060', 'weight=50');
```

---

## 🧩 13. Your White-Label SaaS Layer (What YOU Build)

**Customer Portal**

- Sign up / login
- Buy extensions
- Manage users
- View usage
- Pay bills

**Admin Portal**

- Provision tenants
- Assign SIP trunks
- Monitor usage
- Suspend accounts

**Automation**

- WhatsApp integration
- AI call routing
- Ticket creation from calls/messages

---

## 🤖 14. Automation & AI Integration

### WhatsApp + ERPNext Ticketing

When a call ends, automatically open a support ticket in ERPNext:

```python
from ESL import ESLconnection
import requests

ERPNEXT_URL = "https://erp.yourcompany.com"
ERPNEXT_KEY = "token api_key:api_secret"

con = ESLconnection("127.0.0.1", "8021", "ClueCon")
con.events("plain", "CHANNEL_HANGUP_COMPLETE")

while True:
    e = con.recvEvent()
    if e and e.getHeader("Event-Name") == "CHANNEL_HANGUP_COMPLETE":
        caller      = e.getHeader("Caller-Caller-ID-Number")
        destination = e.getHeader("Caller-Destination-Number")
        duration    = e.getHeader("variable_duration")
        domain      = e.getHeader("variable_domain_name")

        if int(duration or 0) < 10:   # missed / very short call
            requests.post(f"{ERPNEXT_URL}/api/resource/Issue", 
                headers={"Authorization": ERPNEXT_KEY},
                json={
                    "subject": f"Missed call from {caller}",
                    "description": f"Tenant: {domain}\nCaller: {caller}\nDialled: {destination}",
                    "issue_type": "Missed Call",
                    "status": "Open"
                }
            )
```

### WhatsApp Notification on Missed Call (Twilio)

```python
from twilio.rest import Client

twilio = Client("ACxxxxx", "auth_token")

def notify_whatsapp(to_number: str, caller: str):
    twilio.messages.create(
        from_="whatsapp:+14155238886",
        to=f"whatsapp:{to_number}",
        body=f"📞 You missed a call from {caller}. Reply to call back or open a ticket."
    )
```

### AI Call Routing

Use Claude / OpenAI to classify the caller's intent from a short IVR speech input and route accordingly:

```python
import anthropic, subprocess

client = anthropic.Anthropic()

def classify_intent(transcript: str) -> str:
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=64,
        messages=[{
            "role": "user",
            "content": (
                f"Classify this caller intent into one word: "
                f"sales | support | billing | other\n\nTranscript: {transcript}"
            )
        }]
    )
    return msg.content[0].text.strip().lower()

# In your FreeSWITCH dialplan, call this via mod_lua or mod_python
# and transfer to the correct queue based on the returned intent
def route_call(call_uuid: str, transcript: str):
    intent = classify_intent(transcript)
    queue_map = {
        "sales":   "sales_queue@yourpbx.com",
        "support": "support_queue@yourpbx.com",
        "billing": "billing_queue@yourpbx.com",
        "other":   "general_queue@yourpbx.com",
    }
    destination = queue_map.get(intent, queue_map["other"])
    subprocess.run(["fs_cli", "-x",
        f"uuid_transfer {call_uuid} {destination} XML default"])
```

---

## 🚀 15. Deployment Checklist

Before launch:

- [x] SIP trunk tested
- [x] Outbound calls working
- [x] Inbound routing working
- [x] NAT audio OK
- [x] SSL active
- [x] Tenant isolation verified
- [x] Billing tracking working
- [x] Fail2Ban active
- [x] Backups configured
- [x] Toll fraud limits set
- [x] Admin panel IP-restricted
- [x] CDR worker running
- [x] Customer portal deployed
- [x] Stripe / Flutterwave connected
- [x] Missed-call WhatsApp alerts tested

---

## 🧠 16. Common Mistakes to Avoid

- ❌ Editing FreeSWITCH core without backup
- ❌ Ignoring NAT/RTP configuration (causes no-audio issues)
- ❌ Not planning billing early
- ❌ Running single server for too long
- ❌ Exposing FusionPBX admin publicly without IP restriction
- ❌ Skipping toll fraud limits — one compromised account can cost thousands overnight
- ❌ Storing SIP passwords in plaintext in your portal DB (always hash or use a secrets manager)
- ❌ Forgetting to set concurrent call limits per tenant
- ❌ Not testing NAT audio from an external network before going live

---

## 🎯 Final Summary

| Component | Role |
|-----------|------|
| FusionPBX | PBX core + UI |
| FreeSWITCH | Carrier-grade voice engine |
| PostgreSQL | Tenant data + CDR + billing |
| Nginx (proxy) | White-label branding layer |
| Billing worker | CDR → invoice automation |
| Customer portal | Next.js self-service dashboard |
| Kamailio | SIP load balancer (Phase 3) |
| Fail2Ban + ACLs | Security & fraud prevention |
| Your Layer | Branding + billing + customer portal |
| End Result | SaaS PBX provider (3CX alternative) |
