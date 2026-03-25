# VICIdial Security Hardening: CVEs, Firewalls & Access Control

*You deployed VICIdial. It's dialing. Agents are happy. Then your provider sends a nastygram about toll fraud running $14,000 in international calls over a weekend. Or worse — you wake up Monday morning to a defaced admin panel and a MySQL dump floating around a Telegram channel. You search the VICIdial forums for "security hardening" and find... a handful of threads from 2014 telling you to change the default AMI password.*

*This is the security guide that should have existed from day one.*

We've audited, hardened, and incident-responded on more than 100 VICIdial deployments. We've read every CVE filing, tracked every exploit disclosed on the forums and in security advisories, and seen firsthand what happens when a contact center treats security as an afterthought. The damage is always the same: toll fraud, data exfiltration, regulatory fines, and a week of downtime nobody budgeted for.

This isn't a theoretical checklist. This is what actually gets exploited, how it gets exploited, and the specific configurations that stop it. Every firewall rule, every config snippet, every permission change — pulled from production hardening playbooks we use on our own infrastructure.

---

## The CVE History You Need to Know

VICIdial has a documented history of critical vulnerabilities. Some have been patched upstream. Some require manual intervention. All of them are still being actively scanned for by automated exploit kits. If you're running an unpatched VICIdial installation exposed to the internet, the question isn't whether you'll be compromised — it's whether you already have been.

### CVE-2024-8503 — SQL Injection in Authentication (Critical)

This one made headlines. Discovered by Kali Linux's Offensive Security team in 2024, this SQL injection vulnerability exists in the `/vicidial/user_stats.php` endpoint. An unauthenticated attacker can inject arbitrary SQL through the `user` parameter, extract the entire `vicidial_users` table — including plaintext or weakly hashed passwords — and gain full administrative access.

The attack is trivial. A single crafted HTTP request dumps credentials. Automated tools for this CVE were published within days of disclosure.

**How to patch:** Update to VICIdial SVN revision 3848 or later. If you can't update immediately, block direct internet access to `/vicidial/user_stats.php` and every other PHP file that accepts user input. This is a band-aid — update your code.

### CVE-2024-8504 — Authenticated Remote Code Execution (Critical)

Paired with CVE-2024-8503, this vulnerability allows an authenticated admin user to inject arbitrary OS commands through the admin interface. Combined, these two CVEs form a full attack chain: unauthenticated SQL injection to steal admin credentials, then authenticated RCE to own the server.

**How to patch:** Same fix — SVN revision 3848+. But this CVE is a reminder that VICIdial's admin panel should **never** be exposed to the public internet. Even after patching, defense-in-depth means the admin interface lives behind a VPN or IP whitelist.

### CVE-2021-28854 — Unauthenticated Command Injection

An older but still-relevant vulnerability allowing unauthenticated command injection through crafted input parameters. Affects VICIdial installations that haven't been updated since 2021. If you're running a "set it and forget it" VICIdial box from a few years ago, this is almost certainly exploitable on your system.

### CVE-2013-4467 / CVE-2013-4468 — The Classics

These SQL injection and command injection vulnerabilities are over a decade old. They've been patched for years. And we still find them in production. Why? Because someone installed VICIdial from a 2012 ISO, never updated it, and it's been dialing ever since. We've personally encountered these in active exploitation during audits as recently as 2025.

### The Pattern Is Clear

Every major VICIdial CVE follows the same template: **unauthenticated access to a web-facing PHP script that accepts unsanitized input.** The attack surface is the web layer. If you do nothing else from this guide, do these two things:

1. **Keep your VICIdial code updated.** Run `svn update` on your installation directory and review changelogs for security fixes.
2. **Never expose the VICIdial web interface directly to the internet without access controls.**

```bash
# Check your current SVN revision
cd /usr/share/astguiclient
svn info | grep "Revision"

# Update to latest
svn update
```

---

## Firewall Configuration: The Foundation of Everything

A properly configured firewall is the single highest-impact security measure you can implement. It costs nothing, takes 30 minutes, and eliminates entire categories of attack. Yet the majority of VICIdial installations we audit have either no firewall at all, or a firewall so permissive it might as well not exist.

### The Default-Deny Principle

Start from zero. Block everything. Open only what's needed. This is the opposite of VICIdial's default install, which assumes a trusted network and opens ports liberally.

Here's the complete firewalld configuration for a single-server VICIdial installation:

```bash
# Reset to default deny
firewall-cmd --set-default-zone=drop

# Create a dedicated zone for VICIdial
firewall-cmd --permanent --new-zone=vicidial
firewall-cmd --permanent --zone=vicidial --set-target=DROP

# SSH — restrict to your management IPs only
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="YOUR.OFFICE.IP/32" port port="22" protocol="tcp" accept'

# HTTP/HTTPS — agent access
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="YOUR.AGENT.SUBNET/24" port port="443" protocol="tcp" accept'
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="YOUR.AGENT.SUBNET/24" port port="80" protocol="tcp" accept'

# SIP from carriers ONLY (not the whole internet)
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="CARRIER.SIP.IP/32" port port="5060" protocol="udp" accept'

# RTP media ports — carrier IPs only
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="CARRIER.SIP.IP/32" port port="10000-20000" protocol="udp" accept'

# WebRTC/ViciPhone (if used)
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="YOUR.AGENT.SUBNET/24" port port="8089" protocol="tcp" accept'

# Apply the zone to your public interface
firewall-cmd --permanent --zone=vicidial --change-interface=eth0
firewall-cmd --reload
```

For **iptables** (CentOS 6 or custom setups), the equivalent:

```bash
# Flush existing rules
iptables -F
iptables -X

# Default deny
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Loopback
iptables -A INPUT -i lo -j ACCEPT

# SSH from management IPs only
iptables -A INPUT -p tcp --dport 22 -s YOUR.OFFICE.IP/32 -j ACCEPT

# HTTPS for agents
iptables -A INPUT -p tcp --dport 443 -s YOUR.AGENT.SUBNET/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -s YOUR.AGENT.SUBNET/24 -j ACCEPT

# SIP from carriers
iptables -A INPUT -p udp --dport 5060 -s CARRIER.SIP.IP/32 -j ACCEPT

# RTP from carriers
iptables -A INPUT -p udp --dport 10000:20000 -s CARRIER.SIP.IP/32 -j ACCEPT

# AMI — localhost only (NEVER expose externally)
iptables -A INPUT -p tcp --dport 5038 -s 127.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 5038 -j DROP

# Save
iptables-save > /etc/sysconfig/iptables
```

### Cluster Firewall Rules

In a [multi-server cluster](/blog/vicidial-cluster-guide/), you need inter-server communication. The correct approach: **dual NICs.** Public NIC faces the internet with strict rules. Private NIC connects to an isolated VLAN with no firewall between cluster nodes.

If dual NICs aren't possible (cloud deployments, budget constraints), here are the cluster-specific ports that must be open between nodes:

| Port | Protocol | Between | Purpose |
|------|----------|---------|---------|
| 3306 | TCP | All nodes -> DB server | MySQL |
| 4569 | UDP | Dialer <-> Dialer | IAX2 inter-server |
| 5060 | UDP | Carriers -> Dialers | [SIP](/glossary/sip/) signaling |
| 5038 | TCP | Localhost only | AMI (never cross-server) |
| 10000-20000 | UDP | All nodes (bidirectional) | RTP audio |
| 80/443 | TCP | Agents -> Web servers | HTTP/HTTPS |
| 8089 | TCP | Agents -> Dialers | WebSocket ([WebRTC](/glossary/webrtc/)) |
| 21 | TCP | Dialers -> Archive | FTP recordings |

**The critical rule people forget:** Port 5038 (AMI) must be **localhost only.** The Asterisk Manager Interface grants full control of Asterisk — originating calls, hanging up channels, reading configuration. If AMI is reachable from the network, an attacker with the credentials (which are often defaults) can make international calls, wipe voicemail, or crash the PBX. We've seen this exploited more than any other single misconfiguration.

> **Your Firewall Rules Shouldn't Be Guesswork.**
> ViciStack clusters ship with dual-NIC isolation, carrier-specific SIP rules, and zero internet-exposed management ports. [Get Your Free Security Assessment -->](/free-audit/)

---

## Asterisk SIP Security

Asterisk is the telephony engine underneath VICIdial, and it's the most directly exploitable component in your stack. A misconfigured Asterisk server exposed to the internet will be found and exploited within hours — not days, hours. SIP scanners run 24/7 across the entire IPv4 space, probing port 5060 for open registrations.

### Lock Down SIP Registration

The default `sip.conf` that ships with most VICIdial installations is far too permissive. Here are the changes that matter:

```ini
[general]
; Only allow connections from known IPs
allowguest=no
alwaysauthreject=yes

; Bind to specific interface, not 0.0.0.0
bindaddr=YOUR.SERVER.IP

; Disable DNS SRV lookups (prevents DNS rebinding attacks)
srvlookup=no

; NAT settings — see /glossary/nat-traversal/
externip=YOUR.PUBLIC.IP
localnet=192.168.1.0/255.255.255.0
directmedia=no
```

**`allowguest=no`** prevents Asterisk from accepting calls from unregistered endpoints. Without this, anyone who can reach port 5060 can attempt to place calls through your system.

**`alwaysauthreject=yes`** is subtle but important. By default, Asterisk returns different error codes for "user exists but wrong password" vs. "user doesn't exist." Attackers use this to enumerate valid extensions. `alwaysauthreject` returns the same rejection regardless, eliminating the information leak.

### PJSIP: The Modern Alternative

If you're running VICIdial on Asterisk 16+ with [PJSIP](/glossary/pjsip/), the configuration is different but the principles are identical:

```ini
; /etc/asterisk/pjsip.conf

[transport-udp]
type=transport
protocol=udp
bind=YOUR.SERVER.IP:5060
external_media_address=YOUR.PUBLIC.IP
external_signaling_address=YOUR.PUBLIC.IP
local_net=192.168.1.0/24

[global]
type=global
max_initial_qualify_time=4
unidentified_request_period=2
unidentified_request_count=3
```

The `unidentified_request_period` and `unidentified_request_count` settings in PJSIP automatically throttle unauthenticated requests — a built-in defense against SIP scanners.

### Fail2ban for SIP: Non-Negotiable

Fail2ban monitors your Asterisk logs for authentication failures and automatically bans offending IPs via firewall rules. Without it, a brute-force attack against your SIP endpoints will eventually succeed — it's just a matter of time and password complexity.

```ini
# /etc/fail2ban/jail.local

[asterisk]
enabled  = true
filter   = asterisk
action   = iptables-allports[name=ASTERISK, protocol=all]
logpath  = /var/log/asterisk/messages
maxretry = 3
findtime = 600
bantime  = 86400
ignoreip = 127.0.0.1/8 YOUR.OFFICE.IP/32
```

The matching filter in `/etc/fail2ban/filter.d/asterisk.conf`:

```ini
[INCLUDES]
before = common.conf

[Definition]
failregex = SECURITY.* SecurityEvent="FailedACL".*RemoteAddress=".+?/.+?/.*?/<HOST>/.*?"
            SECURITY.* SecurityEvent="InvalidAccountID".*RemoteAddress=".+?/.+?/.*?/<HOST>/.*?"
            SECURITY.* SecurityEvent="ChallengeResponseFailed".*RemoteAddress=".+?/.+?/.*?/<HOST>/.*?"
            SECURITY.* SecurityEvent="InvalidPassword".*RemoteAddress=".+?/.+?/.*?/<HOST>/.*?"
            NOTICE.* .*: Registration from '.*' failed for '<HOST>(:\d+)?' - Wrong password
            NOTICE.* .*: Registration from '.*' failed for '<HOST>(:\d+)?' - No matching peer found
            NOTICE.* .*: Registration from '.*' failed for '<HOST>(:\d+)?' - Username/auth name mismatch
            NOTICE.* <HOST> failed to authenticate as '.*'
            NOTICE.* .*: No registration for peer '.*' \(from <HOST>\)
            NOTICE.* .*: Host <HOST> failed MD5 authentication for '.*' (.*)
            VERBOSE.* logger.c: -- .*IP/<HOST>-.* Playing 'vm-incorrect-mailbox' .*

ignoreregex =
```

**Set `bantime` to at least 86400 (24 hours).** The default of 600 seconds is a joke — automated scanners simply wait 10 minutes and resume. We run 7-day bans on production systems. For repeat offenders, consider `bantime.increment = true` with `bantime.factor = 24` for exponential ban durations.

After configuring fail2ban, verify it's working:

```bash
# Check fail2ban status
fail2ban-client status asterisk

# View currently banned IPs
fail2ban-client status asterisk | grep "Banned IP"

# Test the filter against your actual log
fail2ban-regex /var/log/asterisk/messages /etc/fail2ban/filter.d/asterisk.conf
```

If you're seeing [SIP registration failures](/errors/sip-registration-failed/) from legitimate endpoints after enabling fail2ban, add those IPs to the `ignoreip` list. Don't disable fail2ban — whitelist the known-good sources.

### SIP ACLs — The Second Layer

Fail2ban is reactive. ACLs are proactive. Asterisk supports IP-based Access Control Lists that reject connections before authentication is even attempted:

```ini
; /etc/asterisk/acl.conf

[carrier-acl]
deny=0.0.0.0/0.0.0.0
permit=CARRIER.PRIMARY.IP/255.255.255.255
permit=CARRIER.SECONDARY.IP/255.255.255.255

[internal-acl]
deny=0.0.0.0/0.0.0.0
permit=192.168.1.0/255.255.255.0
permit=10.0.0.0/255.0.0.0
```

Then apply the ACL to your SIP trunk definitions:

```ini
[your-carrier]
type=peer
host=CARRIER.SIP.IP
acl=carrier-acl
; ... rest of trunk config
```

This means even if fail2ban hasn't caught an attacker yet, and even if your firewall has a gap, Asterisk itself will reject the connection at the application layer.

### Strong SIP Credentials

This shouldn't need to be said in 2026, but we still find VICIdial installations with SIP passwords like `1234` or `test123`. Every SIP endpoint — every phone, every trunk, every agent extension — needs a unique, randomly generated password of at least 20 characters.

```bash
# Generate a strong SIP password
openssl rand -base64 24
```

SIP authentication uses digest auth (MD5-based), which is not the strongest protocol. Long random passwords compensate for the protocol's limitations. Never reuse SIP passwords across endpoints.

---

## MySQL Access Control

VICIdial's MySQL database contains everything: agent credentials, call recordings metadata, lead data (names, phone numbers, addresses), campaign configurations, and API keys. A compromised database is a total compromise.

### Restrict MySQL Binding

By default, MySQL listens on all interfaces (`bind-address = 0.0.0.0`). On a single-server installation, there's zero reason for MySQL to be reachable from the network.

```ini
# /etc/my.cnf
[mysqld]
bind-address = 127.0.0.1
```

In a [cluster environment](/blog/vicidial-cluster-guide/), MySQL must be reachable from other servers — but only from other cluster nodes:

```ini
# Bind to the private interface only
bind-address = 192.168.1.10
```

Combined with firewall rules restricting port 3306 to cluster node IPs, this ensures the database is never accessible from the public internet.

### Eliminate Default and Overprivileged Users

Fresh VICIdial installations often create MySQL users with overly broad privileges. Audit yours:

```sql
-- List all users and their hosts
SELECT user, host FROM mysql.user;

-- Check for wildcard hosts (dangerous)
SELECT user, host FROM mysql.user WHERE host = '%';

-- Check for users with GRANT privileges
SELECT user, host FROM mysql.user WHERE Grant_priv = 'Y';
```

**What you should find:** A `cron` user for VICIdial's background processes, a `custom` user for the web interface, and a `root` user accessible only from localhost. What you often find instead: users with `%` host wildcards (accessible from anywhere) and `GRANT ALL` privileges.

Fix it:

```sql
-- Remove wildcard access
DROP USER 'cron'@'%';
CREATE USER 'cron'@'192.168.1.%' IDENTIFIED BY 'STRONG_RANDOM_PASSWORD';
GRANT SELECT, INSERT, UPDATE, DELETE ON asterisk.* TO 'cron'@'192.168.1.%';

-- Separate read-only user for reporting
CREATE USER 'reports'@'192.168.1.%' IDENTIFIED BY 'ANOTHER_STRONG_PASSWORD';
GRANT SELECT ON asterisk.* TO 'reports'@'192.168.1.%';

-- Lock down root
DROP USER 'root'@'%';
-- Ensure root only works from localhost
ALTER USER 'root'@'localhost' IDENTIFIED BY 'VERY_STRONG_PASSWORD';

FLUSH PRIVILEGES;
```

### VICIdial Database Credentials

VICIdial stores its MySQL credentials in two places:

1. **`/etc/astguiclient.conf`** — Used by all Perl background processes
2. **PHP configuration files in the web directory** — Used by the admin and agent interfaces

Both files must have restrictive permissions:

```bash
# astguiclient.conf — readable only by root and asterisk
chmod 640 /etc/astguiclient.conf
chown root:asterisk /etc/astguiclient.conf

# Web config files
chmod 640 /var/www/html/vicidial/dbconnect_mysqli.php
chown root:apache /var/www/html/vicidial/dbconnect_mysqli.php
```

If an attacker can read `dbconnect_mysqli.php` through a web server misconfiguration (directory traversal, exposed `.php` source via misconfigured mod_php), they have your database credentials. File permissions are the last line of defense.

---

## HTTPS Enforcement

Running VICIdial over plain HTTP in 2026 is negligent. Agent credentials, lead data, and session tokens traverse the network in cleartext. On a shared network or any network segment an attacker can sniff, every agent login is compromised.

### SSL/TLS Certificate Setup

Use Let's Encrypt for free, automated certificates. If your VICIdial server has a public FQDN:

```bash
# Install certbot
yum install -y certbot python3-certbot-apache

# Obtain certificate
certbot --apache -d dialer.yourdomain.com

# Verify auto-renewal
certbot renew --dry-run
```

For internal/private deployments without public DNS, use a private CA or self-signed certificates — but understand that [WebRTC](/glossary/webrtc/) (ViciPhone) will not work with self-signed certificates in modern browsers.

### Apache HTTPS Configuration

Force all HTTP traffic to HTTPS and configure modern TLS settings:

```apache
# /etc/httpd/conf.d/ssl.conf

<VirtualHost *:80>
    ServerName dialer.yourdomain.com
    Redirect permanent / https://dialer.yourdomain.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName dialer.yourdomain.com
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/dialer.yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/dialer.yourdomain.com/privkey.pem

    # Modern TLS configuration
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    SSLHonorCipherOrder on

    # Security headers
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    # Disable server signature
    ServerSignature Off
    ServerTokens Prod
</VirtualHost>
```

**Disable TLSv1.0 and TLSv1.1.** They're deprecated, they're insecure, and no modern browser needs them. If you have agents on Internet Explorer 10, you have bigger problems than TLS versions.

---

## Admin Panel Security

The VICIdial admin panel (`/vicidial/admin.php`) is the control plane for your entire operation. It manages campaigns, users, trunks, DID routes, system settings, and API access. A compromised admin panel means total control over your contact center.

### IP-Restrict the Admin Interface

The admin panel should never be accessible from the open internet. Period. Use Apache configuration to restrict access:

```apache
# /etc/httpd/conf.d/vicidial-admin.conf

<Directory "/var/www/html/vicidial">
    # Allow only from specific IPs/subnets
    <RequireAny>
        Require ip 192.168.1.0/24
        Require ip YOUR.OFFICE.IP/32
        Require ip YOUR.VPN.SUBNET/24
    </RequireAny>
</Directory>

# Completely block sensitive scripts from external access
<LocationMatch "/(agc|vicidial)/(admin|manager_send|AST_|user_stats|dbconnect)">
    <RequireAny>
        Require ip 192.168.1.0/24
        Require ip YOUR.OFFICE.IP/32
    </RequireAny>
</LocationMatch>
```

If your managers need remote access, the answer is a VPN — not opening the admin panel to the internet. WireGuard takes 10 minutes to set up and eliminates the entire class of web-application attacks against the admin interface.

### VICIdial User Permission Model

VICIdial has a granular permission system that almost nobody configures properly. The defaults give most users far more access than they need.

In the admin panel under **Admin > User Groups**, configure access levels:

- **Level 1 (Agent):** Can only log in to the agent screen. No admin access. This is what 90% of your users should be.
- **Level 7 (Manager):** Can view reports, manage lists and campaigns assigned to their user group. Cannot modify system settings, trunks, or server configurations.
- **Level 8 (Senior Manager):** Full campaign management, user management within their group. Cannot modify global system settings.
- **Level 9 (Admin):** Full access. Limit this to 2-3 people maximum.

**Critical settings in User Groups:**

```
allowed_campaigns:  Restrict to specific campaigns (don't use "ALL")
admin_viewable_groups:  Restrict to relevant groups only
admin_viewable_call_times:  Limit to business-relevant call times
qc_enabled:  Disable unless quality control is actively used
```

Change the default admin credentials immediately after installation. The default username `6666` with password `1234` is the first thing any attacker tries. Create a new admin user with a strong password, then deactivate (not delete — VICIdial doesn't like deleted users) the default accounts.

### API Authentication

VICIdial's non-agent API (`/vicidial/non_agent_api.php`) is a powerful interface that can add leads, start campaigns, pull reports, and manage users programmatically. It authenticates via username, password, and an IP whitelist.

If you're seeing [API authentication failures](/errors/api-authentication-failed/), verify the `api_allowed_functions` and `api_ip` fields for the user in question. But from a security perspective, the critical settings are:

```
# In System Settings > API Configuration
api_only_user:  Create dedicated API users that CANNOT log in to the agent or admin interface
api_ip:  Restrict to specific IPs. NEVER use 0.0.0.0/0
```

Create separate API users for each integration. If your CRM, your reporting tool, and your IVR all use the same API credentials, compromising one compromises all three. Separate users with separate IP restrictions and separate function permissions.

> **Security Isn't a Feature — It's a Foundation.**
> Every ViciStack deployment ships with IP-restricted admin access, disabled default accounts, and per-integration API users. [Get Your Free Security Audit -->](/free-audit/)

---

## Web Server Hardening

Apache is the web server that serves VICIdial's agent and admin interfaces. Out of the box, it exposes more information and accepts more requests than it should.

### Disable Directory Listing and Server Info

```apache
# /etc/httpd/conf/httpd.conf

# Remove Indexes from Options (prevents directory browsing)
<Directory "/var/www/html">
    Options -Indexes -FollowSymLinks
    AllowOverride None
</Directory>

# Disable mod_info and mod_status from external access
<Location "/server-info">
    Require ip 127.0.0.1
</Location>

<Location "/server-status">
    Require ip 127.0.0.1
</Location>

# Hide Apache version
ServerTokens Prod
ServerSignature Off

# Disable TRACE method (prevents cross-site tracing)
TraceEnable Off
```

### PHP Hardening

VICIdial's web interface runs on PHP. The default PHP configuration is designed for developer convenience, not production security.

```ini
; /etc/php.ini

; Don't expose PHP version in headers
expose_php = Off

; Disable dangerous functions VICIdial doesn't need
disable_functions = exec,passthru,shell_exec,system,proc_open,popen

; Limit file uploads
file_uploads = On
upload_max_filesize = 50M
max_file_uploads = 10

; Session security
session.cookie_httponly = 1
session.cookie_secure = 1
session.use_strict_mode = 1

; Error handling — NEVER show errors in production
display_errors = Off
log_errors = On
error_log = /var/log/php_errors.log
```

**Important caveat:** The `disable_functions` directive above may break some VICIdial admin functions that use `exec()` for system operations (like recording playback, server status checks, etc.). Test thoroughly in a staging environment. If specific admin functions break, selectively re-enable `exec` only, and compensate with stricter filesystem permissions and Apache access controls.

### Rate Limiting with mod_evasive

Brute-force attacks against the VICIdial login page are common. `mod_evasive` provides basic rate limiting:

```bash
yum install -y mod_evasive
```

```apache
# /etc/httpd/conf.d/mod_evasive.conf
<IfModule mod_evasive24.c>
    DOSHashTableSize    3097
    DOSPageCount        5
    DOSSiteCount        50
    DOSPageInterval     1
    DOSSiteInterval     1
    DOSBlockingPeriod   300
    DOSEmailNotify      admin@yourdomain.com
    DOSLogDir           "/var/log/mod_evasive"
</IfModule>
```

This blocks any IP that hits the same page more than 5 times per second or generates more than 50 requests per second site-wide. Legitimate agent usage never triggers these thresholds.

---

## Recording Access Control

Call recordings are regulated data in most jurisdictions. PCI-DSS, HIPAA, GDPR, and state-level privacy laws all have specific requirements for how recordings are stored, accessed, and retained. A VICIdial installation with world-readable recordings on an unprotected HTTP path is a compliance violation waiting to become a lawsuit.

### Restrict Recording Directory Access

By default, VICIdial stores recordings in `/var/spool/asterisk/monitorDONE/` on telephony servers and serves them via HTTP for playback in the admin interface. The archive server typically serves recordings from a web-accessible directory.

```apache
# Block direct HTTP access to recording directories
<Directory "/var/www/html/RECORDINGS">
    # Only allow access from the admin interface (referrer check + IP restriction)
    <RequireAll>
        Require ip 192.168.1.0/24
        Require ip YOUR.OFFICE.IP/32
    </RequireAll>
</Directory>
```

For enhanced security, VICIdial supports authenticated recording playback. In **System Settings > Recordings**, configure:

- `sounds_web_server`: Point to a dedicated recording server
- `sounds_web_directory`: Use a non-obvious path (not `/RECORDINGS/`)
- Authentication: VICIdial can pass session tokens to validate recording access

### Filesystem Permissions

```bash
# Recording directories — readable only by apache and asterisk
chown -R asterisk:apache /var/spool/asterisk/monitorDONE/
chmod -R 750 /var/spool/asterisk/monitorDONE/

# Archive server recordings
chown -R apache:apache /var/www/html/RECORDINGS/
chmod -R 750 /var/www/html/RECORDINGS/
```

### Recording Encryption at Rest

For PCI-DSS or HIPAA compliance, recordings should be encrypted at rest. The simplest approach is filesystem-level encryption using LUKS:

```bash
# Create encrypted partition for recordings
cryptsetup luksFormat /dev/sdb1
cryptsetup luksOpen /dev/sdb1 recordings
mkfs.ext4 /dev/mapper/recordings
mount /dev/mapper/recordings /var/spool/asterisk/monitorDONE/
```

This way, if the physical drive is stolen or the server is decommissioned without proper disk wiping, the recordings are unreadable.

---

## AMI (Asterisk Manager Interface) Security

The AMI is Asterisk's remote control interface. Through AMI, you can originate calls, hang up channels, reload configuration, execute commands, and read real-time events. VICIdial's Perl scripts communicate with Asterisk exclusively through AMI on port 5038.

### Never Expose AMI to the Network

This cannot be stated forcefully enough. AMI on the network is equivalent to SSH with a known password. The default VICIdial AMI credentials (`cron`/`1234`) are documented in every install guide on the internet.

```ini
; /etc/asterisk/manager.conf

[general]
enabled = yes
port = 5038
; CRITICAL: bind to localhost ONLY
bindaddr = 127.0.0.1

; Change from defaults immediately
[cron]
secret = USE_A_GENERATED_32_CHARACTER_PASSWORD_HERE
; Restrict permissions to only what VICIdial needs
read = system,call,log,verbose,agent,user,config,dtmf,reporting,cdr,dialplan
write = system,call,agent,user,command,reporting,originate,message
deny = 0.0.0.0/0.0.0.0
permit = 127.0.0.1/255.255.255.0
```

**After changing the AMI secret**, you must update it in `/etc/astguiclient.conf` (the `VARserver_password` field). If these don't match, every VICIdial Perl script will fail to connect to Asterisk, and your system will stop dialing immediately. Test during a maintenance window.

```bash
# After changing AMI credentials, verify connectivity
asterisk -rx "manager show connected"
```

### AMI in a Cluster

In a cluster, VICIdial's scripts on each telephony server communicate with their **local** Asterisk only via AMI. There is no legitimate reason for AMI traffic to traverse the network between servers. If you see port 5038 open in your cluster's firewall rules, that's a misconfiguration. Remove it.

The one exception: if you've built custom integrations that use AMI remotely (wallboard applications, custom dialers), those integrations should be refactored to use VICIdial's API instead, which has proper authentication and doesn't expose the full Asterisk control plane.

---

## SSH Hardening

SSH is your server management access. Compromise SSH and everything else is moot. These changes take five minutes and dramatically reduce your attack surface.

### Disable Password Authentication

```bash
# /etc/ssh/sshd_config

# Key-based auth only
PasswordAuthentication no
PubkeyAuthentication yes

# Disable root login
PermitRootLogin no

# Use SSH protocol 2 only
Protocol 2

# Restrict to specific users
AllowUsers youradmin

# Rate limit connections
MaxAuthTries 3
LoginGraceTime 30

# Disable unused features
X11Forwarding no
AllowTcpForwarding no
PermitTunnel no
```

```bash
# Reload SSH (don't restart — you'll lose your current session if something's wrong)
systemctl reload sshd
```

**Before disabling password authentication**, verify you can log in with your SSH key. Lock yourself out and you're calling the data center for KVM access.

### Fail2ban for SSH

```ini
# /etc/fail2ban/jail.local

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/secure
maxretry = 3
findtime = 600
bantime = 86400
```

### Change the Default SSH Port

Security through obscurity alone is worthless. But changing SSH from port 22 to a non-standard port eliminates 99% of automated scanning noise, making your logs readable and your fail2ban bans meaningful.

```bash
# /etc/ssh/sshd_config
Port 2222

# Update firewall rules BEFORE restarting SSH
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="YOUR.OFFICE.IP/32" port port="2222" protocol="tcp" accept'
firewall-cmd --reload
systemctl reload sshd
```

---

## Operating System Hardening

### Keep the OS Updated

```bash
# Enable automatic security updates (RHEL/CentOS/AlmaLinux)
yum install -y dnf-automatic
systemctl enable --now dnf-automatic-install.timer

# Verify timer is active
systemctl status dnf-automatic-install.timer
```

For VICIdial-specific packages (Asterisk, DAHDI, Perl modules), update manually during maintenance windows. Automatic Asterisk updates on a production dialer are a recipe for unscheduled downtime.

### Disable Unnecessary Services

A VICIdial server should run VICIdial. Not Postfix (unless you need email functionality). Not Cups. Not Bluetooth services. Every unnecessary service is a potential attack vector.

```bash
# Common unnecessary services on VICIdial servers
systemctl disable --now cups
systemctl disable --now bluetooth
systemctl disable --now avahi-daemon
systemctl disable --now rpcbind

# List all active services and review
systemctl list-unit-files --type=service --state=enabled
```

### SELinux

The VICIdial community's standard advice is "disable SELinux." We disagree — but we understand why they say it. SELinux in enforcing mode breaks VICIdial's web interface, Perl scripts, and Asterisk interactions unless you write custom policies.

The pragmatic middle ground:

```bash
# Set to permissive (logs violations but doesn't block)
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

Permissive mode gives you an audit log of what SELinux would block, without breaking anything. On high-security deployments, use the audit log to create targeted policies:

```bash
# Generate policy from audit log
audit2allow -a -M vicidial_custom
semodule -i vicidial_custom.pp
```

---

## Network-Level Protections

### SIP-Specific Firewall Rules

Beyond basic port filtering, implement SIP-aware protections:

```bash
# Rate-limit SIP REGISTER attempts (prevents registration floods)
iptables -A INPUT -p udp --dport 5060 -m string --string "REGISTER" --algo bm -m recent --name sip_register --set
iptables -A INPUT -p udp --dport 5060 -m string --string "REGISTER" --algo bm -m recent --name sip_register --update --seconds 60 --hitcount 20 -j DROP

# Rate-limit SIP INVITE (prevents call flooding)
iptables -A INPUT -p udp --dport 5060 -m string --string "INVITE" --algo bm -m recent --name sip_invite --set
iptables -A INPUT -p udp --dport 5060 -m string --string "INVITE" --algo bm -m recent --name sip_invite --update --seconds 10 --hitcount 30 -j DROP
```

### VPN for Remote Management

If your team manages VICIdial servers remotely, all management traffic should traverse a VPN. WireGuard is the simplest option:

```bash
# Install WireGuard
yum install -y wireguard-tools

# Generate server keys
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
chmod 600 /etc/wireguard/server_private.key
```

```ini
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.200.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.200.0.2/32
```

```bash
systemctl enable --now wg-quick@wg0
```

Then restrict SSH and admin panel access to the WireGuard subnet (`10.200.0.0/24`). Remote management without VPN simply does not exist.

---

## Cloud Deployment Security Considerations

If you're running VICIdial in [AWS, GCP, or DigitalOcean](/blog/vicidial-cloud-deployment/), the security model is different from bare metal. Cloud providers add their own layer — security groups, VPCs, IAM — but they also add new risks.

### Security Groups Are Not Firewalls

Cloud security groups are stateful packet filters, not application-aware firewalls. They don't inspect SIP traffic, they don't understand [NAT traversal](/glossary/nat-traversal/), and they don't rate-limit. You still need host-based firewalls (iptables/firewalld) and application-level protections (fail2ban, SIP ACLs) on every instance.

### VPC Architecture

```
Internet
    |
    v
[ NAT Gateway / Load Balancer ]
    |
    v
[ Public Subnet: Web Servers Only ]
    |
    v
[ Private Subnet: DB, Dialers, Archive ]
```

Database and telephony servers belong in a private subnet with no public IP. Web servers in the public subnet proxy all traffic. This mirrors the dual-NIC approach for bare metal deployments.

### IAM and Instance Roles

Never store AWS/GCP credentials on VICIdial servers. Use instance roles (IAM roles in AWS, service accounts in GCP) for any cloud service interaction (S3 for recording storage, SNS for notifications, etc.). Credentials on disk are credentials waiting to be exfiltrated.

---

## The 22-Point VICIdial Security Hardening Checklist

Here's everything above distilled into an actionable checklist. Print it. Tape it to the wall. Run through it on every deployment.

**CVE and Patch Management:**
1. SVN revision is 3848+ (patches CVE-2024-8503, CVE-2024-8504)
2. Automatic OS security updates enabled
3. Asterisk version is current LTS (18.x or 20.x)

**Firewall:**
4. Default-deny policy on all interfaces
5. SIP (5060) restricted to carrier IPs only
6. RTP (10000-20000) restricted to carrier IPs only
7. AMI (5038) bound to 127.0.0.1 only
8. MySQL (3306) restricted to cluster nodes or localhost
9. SSH restricted to management IPs only

**Asterisk/SIP:**
10. `allowguest=no` and `alwaysauthreject=yes` in sip.conf
11. Fail2ban running with 24-hour bans for SIP brute force
12. SIP ACLs configured for all trunks
13. All SIP passwords are 20+ character random strings

**MySQL:**
14. No MySQL users with `host='%'`
15. Root accessible from localhost only
16. Separate users for cron, web, and reporting with minimal privileges
17. Database config files (`dbconnect_mysqli.php`, `astguiclient.conf`) have 640 permissions

**Web/HTTPS:**
18. HTTPS enforced, TLSv1.0/1.1 disabled
19. Admin panel IP-restricted (VPN or whitelist)
20. Default admin accounts (6666) deactivated; strong passwords on all admin users
21. Directory listing disabled; `ServerTokens Prod`

**Access Control:**
22. SSH key-only auth; root login disabled; non-standard port

---

## Ongoing Security Maintenance

Hardening isn't a one-time project. It's an ongoing discipline.

**Weekly:**
- Review fail2ban logs for patterns (targeted attacks vs. background noise)
- Check `svn log` for VICIdial security patches
- Verify firewall rules haven't been modified

**Monthly:**
- Rotate SIP trunk passwords with your carrier
- Review VICIdial user accounts — deactivate departed staff immediately
- Test recording access controls (try accessing recordings without authentication)
- Review MySQL user privileges

**Quarterly:**
- Full VICIdial SVN update and regression test
- SSL certificate expiry check (automate this with certbot, but verify)
- Review and update firewall IP whitelists
- Penetration test the web interface from an external IP

**Annually:**
- Full security audit of all VICIdial servers
- Review compliance requirements (PCI-DSS, HIPAA, TCPA) against current configuration
- Evaluate whether current Asterisk version still receives security patches
- Update this checklist

---

## Frequently Asked Questions

### Has VICIdial been hacked before?

Yes. Multiple times, across multiple organizations. The CVE-2024-8503/8504 pair was particularly damaging because it allowed unauthenticated remote code execution through a two-step attack chain. Prior to that, CVE-2013-4467 and CVE-2013-4468 were exploited in the wild for years before many installations were patched. Toll fraud — where attackers use a compromised Asterisk server to route international calls — is the most common post-exploitation activity, followed by lead data exfiltration. The common thread in every incident we've investigated: internet-exposed VICIdial web interfaces with no IP restriction and outdated code.

### Is VICIdial PCI-DSS compliant?

Not out of the box. PCI-DSS compliance for a contact center handling payment card data requires encrypted recordings, restricted access to cardholder data, network segmentation, logging and monitoring, and regular vulnerability assessments. VICIdial can be configured to meet these requirements — encrypted storage via LUKS, recording access controls, network segmentation via VLANs, and the audit logging built into the admin interface. But it takes deliberate configuration. If you're processing payments over the phone, engage a PCI QSA (Qualified Security Assessor) who understands VoIP environments. Do not assume your VICIdial installation is compliant because someone told you "we set it up right."

### Should I run VICIdial behind a VPN?

For the admin panel and SSH access, absolutely. For agent access, it depends on your deployment model. If your agents are in a physical call center on a controlled network, IP whitelisting is sufficient. If agents work remotely, a VPN adds meaningful security but also adds latency and a point of failure — which matters for real-time voice. The compromise for remote agents: require VPN for admin functions, use HTTPS with strong authentication for the agent interface, and implement per-agent IP logging to detect compromised credentials.

### How do I know if my VICIdial server has been compromised?

Check for these indicators: unexpected cron jobs (`crontab -l` for all users), unknown processes listening on network ports (`ss -tlnp`), modified PHP files in the web directory (compare against SVN with `svn status`), new MySQL users you didn't create, and unexplained outbound SIP traffic (especially to international destinations). The most reliable indicator is your carrier bill — a sudden spike in international call volume over a weekend is almost always toll fraud from a compromised Asterisk instance. Set up carrier-side spending alerts as a safety net.

### Can I use VICIdial with WebRTC securely?

Yes, but WebRTC actually improves your security posture in some ways. [WebRTC](/glossary/webrtc/) requires HTTPS (enforcing encrypted transport), uses DTLS-SRTP for media encryption (voice traffic is encrypted end-to-end), and doesn't require opening SIP ports to agent endpoints. The tradeoff is that each telephony server needs a valid SSL certificate and a publicly accessible WebSocket port (8089). Configure that port with a firewall rule restricting source IPs to your agent subnet, and you've got encrypted voice transport with a smaller attack surface than traditional [SIP](/glossary/sip/) phones. See our [Asterisk configuration guide](/blog/vicidial-asterisk-configuration/) for detailed [PJSIP](/glossary/pjsip/) and WebRTC setup.

### What's the biggest security risk most VICIdial admins overlook?

Stale user accounts. When agents leave — and in contact centers, turnover is brutal — their VICIdial accounts stay active. Those credentials are known to the former employee and potentially to anyone they shared a workstation with. We audit environments with 500 active VICIdial user accounts and 40 actual agents. Each of those 460 dormant accounts is a potential entry point. Run a monthly audit: compare active VICIdial users against your HR headcount, and deactivate every account that doesn't have a warm body behind it. VICIdial doesn't support automatic account expiration, so this has to be a manual process or a cron job you write yourself.

### How often should I update VICIdial for security patches?

Monitor the VICIdial SVN changelog weekly. Security patches are released as they're developed — there's no fixed schedule. Critical CVE fixes (like the 2024 SQL injection patch) are committed to SVN and announced on the VICIdial community forums. Subscribe to the VICIdial community mailing list and check `svn log` for commits mentioning "security," "SQL injection," "XSS," or "CVE." For Asterisk, subscribe to the Asterisk security advisory mailing list at asterisk.org — critical Asterisk vulnerabilities affect VICIdial directly since Asterisk handles all call processing. Apply Asterisk security patches within 72 hours of release; apply VICIdial patches within a week, after testing in a non-production environment.

### Is fail2ban enough to protect Asterisk?

Fail2ban is necessary but not sufficient. It's a reactive measure — it bans IPs after they've already attempted attacks. A sophisticated attacker using a botnet rotates source IPs faster than fail2ban can ban them. The correct layered approach: firewall rules restricting [SIP](/glossary/sip/) port access to carrier IPs only (prevents 99% of attacks from ever reaching Asterisk), SIP ACLs as a second layer within Asterisk itself, strong SIP authentication as a third layer, and fail2ban as a fourth layer catching anything that gets through. If you're relying on fail2ban alone with SIP open to the internet, you will get compromised. It's just a matter of volume and time.

---

## Where ViciStack Fits In

Everything in this guide is exactly what we implement on every deployment. The firewall rules, the fail2ban configuration, the MySQL access controls, the SSL enforcement, the disabled default accounts — it's all baked into our standard build.

But security isn't a set-and-forget exercise. Vulnerabilities get disclosed. Carrier IP ranges change. Agents join and leave. Asterisk releases patches. Someone adds a "temporary" firewall exception that becomes permanent. Maintaining a hardened VICIdial deployment is an ongoing operational commitment.

**ViciStack handles the security lifecycle so you can focus on dialing.** Continuous patching, proactive monitoring, carrier-side fraud alerts, and incident response when something goes sideways. Your agents dial. We make sure nobody else does.

[Get a free security assessment of your current deployment -->](/free-audit/)

---

*This guide is maintained by the ViciStack team and updated as new vulnerabilities, patches, and best practices emerge. Last updated: March 2026.*

*Found a VICIdial security issue this guide doesn't cover? Reach out directly — responsible disclosure keeps the whole community safer.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-security-hardening).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/)
- [ViciStack Pricing](https://vicistack.com/pricing/)
- [All ViciStack Guides](https://vicistack.com/blog/)
