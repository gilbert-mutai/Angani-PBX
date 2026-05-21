📘 WHITE-LABEL PBX (FusionPBX + FreeSWITCH)
Full Implementation Guide (From Zero → SaaS Launch)
🧭 1. Architecture Overview

Your system will have 5 main layers:

Clients (Web / Softphone / Mobile SIP apps)
        ↓
SIP / WebRTC Layer
        ↓
FreeSWITCH (Media + Call Engine)
        ↓
FusionPBX (Control + Multi-tenant UI)
        ↓
PostgreSQL + Storage + Logs
        ↓
Billing + CRM + Your Custom Portal
🧰 2. Requirements
🖥️ Infrastructure
Minimum Production Setup (small SaaS start)
1 × Application Server (FusionPBX)
1 × Media Server (FreeSWITCH)
1 × Database Server (PostgreSQL) (can be combined initially)
Recommended Cloud Specs
CPU: 4–8 cores per server
RAM: 8–16GB minimum
SSD: 100GB+
OS: Debian 11/12 or Ubuntu 22.04 LTS
🌐 Networking Requirements
Static public IP (VERY important)
Open ports:
SIP: 5060 / 5061
RTP media: 16384–32768 (UDP)
Web: 80 / 443
Firewall: UFW or iptables
NAT configuration (critical for audio quality)
🔧 Software Stack
Core
FreeSWITCH
FusionPBX
PostgreSQL
Nginx / Apache (FusionPBX web server)
Supporting tools
Fail2Ban (security)
Certbot (SSL)
Redis (optional caching layer)
Kamailio/OpenSIPS (optional scaling later)
📊 Business Layer (you will add)
Billing system (custom or Stripe integration)
CRM / provisioning tool
Customer portal (your white-label layer)
Monitoring (Grafana + Prometheus optional)
⚙️ 3. Installation Phase (Step-by-Step)
🧱 STEP 1: Prepare Server
sudo apt update && sudo apt upgrade -y
sudo apt install git curl wget nano unzip -y

Set hostname:

hostnamectl set-hostname pbx.yourdomain.com
🔐 STEP 2: Firewall Setup
ufw allow 22
ufw allow 80
ufw allow 443
ufw allow 5060:5061/udp
ufw allow 16384:32768/udp
ufw enable
📦 STEP 3: Install FreeSWITCH

Install dependencies:

apt install -y gnupg2 wget lsb-release

Install FreeSWITCH (official packages or build from source depending on scale).

👉 For production SaaS, building from source is preferred later.

🧠 STEP 4: Install FusionPBX

Typical install flow:

cd /usr/src
git clone https://github.com/fusionpbx/fusionpbx-install.sh.git
cd fusionpbx-install.sh/debian
./install.sh

This installs:

FreeSWITCH
FusionPBX
PostgreSQL
Web UI
Default configuration
🌐 STEP 5: Configure Domain

Access:

https://your-server-ip

Then:

Create admin account
Set domain (your SaaS domain)
Enable SSL (Let’s Encrypt)
🏢 4. Multi-Tenant (White Label Core)

This is the MOST IMPORTANT PART.

FusionPBX supports Domains = Tenants

For each customer:

Create a new DOMAIN:

client1.yourpbx.com
client2.yourpbx.com

OR:

Separate SIP realm per tenant:

client1-sip.yourdomain.com
👤 Tenant Isolation Model

Each tenant gets:

Extensions
SIP users
IVR
Call queues
Voicemail
CDR logs

👉 Fully isolated inside PostgreSQL schema logic.

🎨 5. White-Label Branding
Change FusionPBX branding:

Modify:

Logo
Login page CSS
Footer branding
System name

Paths:

/var/www/fusionpbx

Key changes:

Replace FusionPBX logo
Change “FusionPBX” text in templates
Custom CSS theme
Recommended approach:

Instead of deep editing:

👉 Use a reverse proxy (Nginx) to:

inject branding layer
override UI assets
control login screen appearance
📞 6. SIP & Call Setup
Configure Trunks

You will connect:

Local carriers in Kenya (Telkom, Safaricom SIP where available)
International SIP providers (Twilio, DIDWW, etc.)

In FusionPBX:

Accounts → Gateways → Add SIP Trunk
RTP Optimization (VERY IMPORTANT)

Edit:

/etc/freeswitch/autoload_configs/switch.conf.xml

Set:

RTP port range
NAT settings
📱 7. Softphone Strategy (Customer Experience)

You have 3 options:

Option A: Use existing SIP apps (fastest)
Zoiper
Linphone
Grandstream Wave

✔ No development needed

Option B: Branded softphone (recommended SaaS path)
Build React Native or Flutter app
Use SIP SDK:
PJSIP
Linphone SDK
Option C: WebRTC browser phone
Built into FusionPBX (limited)
Or custom WebRTC SIP client
💳 8. Billing System (Critical for SaaS)

FusionPBX does NOT provide full billing.

You must add:

Options:
🔹 Simple start
Manual billing per extension
🔹 SaaS model (recommended)

Build:

Customer portal
Subscription system
Usage tracking (CDR-based billing)
Tools:
Stripe (global)
Flutterwave (Africa-friendly)
Custom PostgreSQL billing tables
📊 9. Call Data & Analytics

Enable:

Call Detail Records (CDR)
Call recording storage
Real-time monitoring

You will use:

FusionPBX CDR tables
FreeSWITCH event socket (advanced analytics)
🔐 10. Security Hardening

Install:

apt install fail2ban

Configure:

SIP brute-force protection
IP whitelisting for admin panel
SSL everywhere
Disable anonymous SIP
📈 11. Scaling Strategy (VERY IMPORTANT for SaaS)

When you grow:

Phase 1
Single FusionPBX + FreeSWITCH server
Phase 2
Separate:
DB server
PBX controller
Media servers
Phase 3 (carrier-grade)
Kamailio (SIP proxy load balancing)
Multiple FreeSWITCH nodes
Kubernetes or VM cluster
🧩 12. Your White-Label SaaS Layer (What YOU build)

This is what makes you 3CX competitor:

Customer Portal
Sign up / login
Buy extensions
Manage users
View usage
Pay bills
Admin Portal
Provision tenants
Assign SIP trunks
Monitor usage
Suspend accounts
Automation (your strength)
WhatsApp integration (ERPNext idea fits here)
AI call routing
Ticket creation from calls/messages
🚀 13. Deployment Checklist

Before launch:

✔ SIP trunk tested
✔ Outbound calls working
✔ Inbound routing working
✔ NAT audio OK
✔ SSL active
✔ Tenant isolation verified
✔ Billing tracking working
✔ Fail2Ban active
✔ Backups configured

🧠 14. Common Mistakes to Avoid

❌ Editing FreeSWITCH core without backup
❌ Ignoring NAT/RTP configuration (causes no audio issues)
❌ Not planning billing early
❌ Running single server for too long
❌ Exposing FusionPBX admin publicly without IP restriction

🎯 FINAL SUMMARY

If you choose this path:

✔ FusionPBX gives you PBX core + UI
✔ FreeSWITCH gives you carrier-grade voice engine
✔ You add branding + billing + customer portal
✔ You become a SaaS PBX provider (like 3CX alternative)