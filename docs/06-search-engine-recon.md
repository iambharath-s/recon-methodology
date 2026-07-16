# Search Engine Reconnaissance

Search engines have already crawled the entire internet — including the mistakes. Configuration files left in web roots, directory listings, admin portals, database dumps, and hardcoded credentials all get indexed if they are publicly accessible. Your job is to query that index intelligently.

---

## Google Dorking

### How It Works

Google's crawlers have indexed billions of pages. When you write a dork, you are filtering that index — not scanning the target. The target never sees your queries. This makes dorking one of the most powerful passive techniques available.

The risk is on Google's side: they rate-limit aggressive querying and may show a CAPTCHA. The target has no visibility.

### Operator Reference

```
site:target.com                         → restrict all results to this domain
site:*.target.com                       → only subdomains
site:*.target.com -site:www.target.com  → subdomains, exclude www

filetype:pdf                            → PDF files only
filetype:env                            → .env files
filetype:sql                            → SQL dumps

intitle:"index of"                      → page title contains "index of"
allintitle:admin login panel            → all three words in title

inurl:/admin                            → URL contains /admin
inurl:.php?id=                          → URL pattern (SQLi hunting)
allinurl:config backup                  → all words in URL

intext:"DB_PASSWORD"                    → text in page body
allintext:username password login       → all words in body

cache:target.com                        → Google's cached copy
link:target.com                         → pages linking to target
related:target.com                      → similar sites
before:2022-01-01                       → indexed before this date
after:2023-06-01                        → indexed after this date
```

### High-Value Dork Combinations

**Infrastructure discovery:**

```
# All subdomains Google has indexed
site:*.target.com

# Login portals across all subdomains
inurl:login site:*.target.com
inurl:"/wp-admin" site:target.com
intitle:"admin panel" site:target.com
inurl:"/phpmyadmin" site:target.com
inurl:"/cpanel" site:target.com
intitle:"Dashboard [Jenkins]" site:target.com

# Open directory listings
intitle:"index of" site:target.com
intitle:"index of" "parent directory" site:target.com
intitle:"index of" /backup site:target.com
intitle:"index of" /config site:target.com
intitle:"index of" ".git" site:target.com
```

**Credential and secret exposure:**

```
# Environment files with credentials
filetype:env site:target.com
filetype:env "DB_PASSWORD" site:target.com
filetype:env "AWS_SECRET" site:target.com

# Config files
filetype:conf site:target.com
filetype:config site:target.com
filetype:ini site:target.com
filetype:yaml intext:password site:target.com

# Private keys
intext:"-----BEGIN RSA PRIVATE KEY-----" site:target.com
intext:"-----BEGIN OPENSSH PRIVATE KEY-----" site:target.com
filetype:pem site:target.com

# API keys and tokens
intext:"api_key" site:target.com
intext:"access_token" site:target.com
intext:"AKIA" site:target.com            ← AWS access key prefix
intext:"sk_live_" site:target.com        ← Stripe live secret key

# Database files
filetype:sql site:target.com
filetype:db site:target.com
filetype:sqlite site:target.com

# Credential files
filetype:txt intext:password site:target.com
filetype:csv intext:password site:target.com
```

**Technology identification:**

```
# WordPress
inurl:"/wp-content" site:target.com
inurl:"/wp-includes" site:target.com

# Error messages revealing stack info
intext:"Warning: mysql_connect()" site:target.com
intext:"Fatal error:" site:target.com
intext:"Traceback (most recent call" site:target.com
intext:"stack trace at" site:target.com
intext:"DisallowedHost at" site:target.com       ← Django debug mode

# Version disclosure
intitle:"Apache2 Ubuntu Default Page" site:target.com
intitle:"Welcome to nginx" site:target.com

# Elasticsearch / Kibana
intitle:"Kibana" site:target.com
inurl:":9200/_cat" site:target.com

# Laravel debug
intext:"APP_DEBUG=true" site:target.com
intitle:"Whoops! There was an error." site:target.com
```

**Document intelligence:**

```
# Company documents — metadata reveals internal info
filetype:pdf site:target.com
filetype:docx site:target.com
filetype:xlsx site:target.com
filetype:pptx site:target.com

# VPN configs
filetype:pcf site:target.com
filetype:ovpn site:target.com

# Backup files
filetype:bak site:target.com
inurl:backup site:target.com
inurl:".backup" site:target.com
```

### Workflow for a Dorking Session

Start broad, narrow down based on what you find. Do not run every dork — pick ones relevant to what you know about the target.

```
1. site:*.target.com                        → how many pages indexed, what subdomains
2. filetype:env OR filetype:conf site:target.com  → immediate high-value check
3. intitle:"index of" site:target.com       → open directories
4. inurl:login OR inurl:admin site:target.com → portals
5. intext:"api_key" OR intext:"secret" site:target.com → credential exposure
6. filetype:pdf OR filetype:docx site:target.com → documents for metadata
```

---

## Google Hacking Database (GHDB)

`exploit-db.com/google-hacking-database` — a library of 6000+ pre-built dorks organized by category. Before crafting a custom dork, search GHDB first.

**Categories and what to look for:**

| Category | Best used for |
|---|---|
| Footholds | Finding file managers, shell upload forms |
| Files containing usernames | Config files, htpasswd files |
| Sensitive directories | Open directory listings of specific types |
| Web server detection | Identifying specific server versions |
| Vulnerable files | Files directly associated with CVEs |
| Vulnerable servers | Servers with known-vulnerable software |
| Error messages | DB connection errors, stack traces |
| Files containing passwords | Text files, config files with creds |
| Network/vulnerability data | Firewall and router configs |
| Various online devices | IoT, cameras, printers, industrial systems |
| Login portals | Admin panels, CMS logins |

**Workflow:** Go to GHDB, search for the technology you identified (e.g. "Jenkins"), find the relevant dork, add `site:target.com` to it.

---

## Shodan

Shodan scans every IP address on the internet continuously, stores what it finds — open ports, service banners, TLS certificates, HTTP headers — and makes it searchable. Unlike Google which indexes web content, Shodan indexes raw internet-facing services.

### Web Interface Searches

```
# By organization name
org:"Target Corp"

# By ASN
asn:AS12345

# By IP range
net:185.45.12.0/24

# Combine
org:"Target Corp" port:22
org:"Target Corp" port:3389      ← RDP
org:"Target Corp" port:27017     ← MongoDB
org:"Target Corp" port:9200      ← Elasticsearch
org:"Target Corp" port:6379      ← Redis
org:"Target Corp" port:5900      ← VNC
org:"Target Corp" port:21        ← FTP

# By certificate CN (finds hidden subdomains)
ssl.cert.subject.CN:"*.target.com"
ssl.cert.subject.O:"Target Corp"

# By product
org:"Target Corp" product:nginx
org:"Target Corp" product:"Apache httpd"
org:"Target Corp" product:MySQL

# Find specific vulnerability indicators
org:"Target Corp" http.title:"Apache2 Ubuntu Default Page"
org:"Target Corp" http.title:"phpMyAdmin"
org:"Target Corp" http.title:"Dashboard [Jenkins]"
```

### Shodan CLI

```bash
# Setup
pip install shodan
shodan init YOUR_API_KEY

# Search — returns structured results
shodan search "org:Target Corp"

# Get IPs only
shodan search --fields ip_str "org:Target Corp"

# Get IP + port + banner
shodan search --fields ip_str,port,data "org:Target Corp"

# Specific host info
shodan host 185.45.12.10

# Count results without downloading
shodan count "org:Target Corp"

# Download results for offline analysis
shodan download results.json.gz "org:Target Corp"
shodan parse --fields ip_str,port results.json.gz

# Find hosts in a network range
shodan search "net:185.45.12.0/24"

# Get all open ports for an org
shodan search --fields ip_str,port "org:Target Corp" | sort -t',' -k2 -n
```

### What Shodan Reveals That Port Scanning Misses

- **Historical data** — Shodan shows what a host was running months ago, not just now
- **TLS certificate details** — subject, issuer, SANs, expiry — all searchable
- **Banner content** — full HTTP response headers including server version
- **Geographic and ISP context** — where physically, who provides connectivity
- **Vulnerabilities** — some Shodan plans show CVEs for detected software versions

---

## Censys

Censys continuously scans all IPv4 space and indexes TLS certificates, HTTP/S responses, and service banners. Stronger than Shodan for certificate-based asset discovery.

### Search Syntax

```
# By organization
autonomous_system.organization: "Target Corp"

# By IP range
ip: 185.45.12.0/24

# Hosts with specific port open
services.port: 8443

# Combine
services.port: 8443 and autonomous_system.organization: "Target Corp"

# By certificate
parsed.names: target.com
parsed.names: *.target.com

# Find assets by cert organization
parsed.subject.organization: "Target Corp"

# Hosts running specific software
services.software.product: "Apache httpd" and autonomous_system.organization: "Target Corp"
```

**Censys advantage over Shodan for recon:**
Censys is better for certificate-based discovery. Search `parsed.names: target.com` and you find every host that has ever had a certificate mentioning `target.com` — including internal hostnames that leaked into SANs.

---

## ZoomEye

Chinese internet-wide scanner — indexes different infrastructure than Shodan and Censys. Useful for finding assets not visible in the other two.

```
# Basic search
app:"Apache" hostname:target.com

# Port + hostname
port:8080 hostname:target.com

# Country filter
country:IN org:"Target Corp"
```

---

## Combining Search Engines

Each search engine has different coverage. Run all three for maximum asset discovery.

```bash
# Collect all IPs found across sources
shodan search --fields ip_str "org:Target Corp" > ips-shodan.txt
# (Censys and ZoomEye via web interface — export IPs)

# Merge and deduplicate
cat ips-shodan.txt ips-censys.txt | sort -u > ips-all.txt

# Cross-reference with your DNS findings
# IPs in Shodan but not in DNS → servers with no hostname (hidden assets)
comm -23 <(sort ips-all.txt) <(sort dns-ips.txt)
```

---

## Bing

Bing indexes differently from Google and occasionally finds content Google misses. Worth running for completeness.

```
site:target.com
site:target.com filetype:pdf
site:target.com inurl:admin
```

Also available as a source in theHarvester: `theHarvester -d target.com -b bing`

---

## Common Mistakes

**Running dorks in the wrong order.** Start with `site:*.target.com` to understand how much Google has indexed. If it returns 50,000 results, you know the site is heavily indexed and dorking will be productive. If it returns 3 results, broad dorking will not help — move to other techniques.

**Not checking for time-sensitive data.** Add `after:2023-01-01` to find recently indexed content — new subdomains, recently exposed files, fresh admin portals.

**Missing non-www subdomains.** `site:target.com` also matches `www.target.com`. Use `site:*.target.com -site:www.target.com` to see only subdomains.

**Ignoring error messages.** Dorks for verbose error messages (`intext:"Warning: mysql"`, `intext:"Fatal error:"`) find application errors that reveal database type, file paths, code structure, and sometimes credentials — all indexed by Google because a crawlable page threw an error.

---

*Previous: [WHOIS and IP Intelligence](05-whois-ip-intel.md)*
*Next: [Email Enumeration](07-email-enumeration.md)*
*Cheat sheet: [Google Dorks](../cheatsheets/google-dorks-cheatsheet.md)*
