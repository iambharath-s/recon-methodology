# Website Enumeration and Technology Fingerprinting

Once you have a list of live web hosts from DNS and subdomain discovery, website enumeration extracts everything the web layer exposes — what software is running, which version, what paths exist, what was there before, and what the server reveals about itself without being asked.

---

## HTTP Response Analysis

The most passive form of active web recon. Send an HTTP request, read what comes back.

```bash
# Headers only — fast, low noise
curl -I https://target.com

# Follow redirects and show each response
curl -IL https://target.com

# Full response including body
curl -s https://target.com | head -100

# With custom headers (bypass some WAFs)
curl -I https://target.com -H "X-Forwarded-For: 127.0.0.1"
```

**What HTTP headers leak:**

| Header | Example value | What it reveals |
|---|---|---|
| `Server` | `Apache/2.4.49 (Ubuntu)` | Web server + exact version + OS |
| `X-Powered-By` | `PHP/7.4.3` | Backend language + version |
| `X-Generator` | `WordPress 6.2` | CMS platform + version |
| `X-Frame-Options` | missing | Potential clickjacking if absent |
| `Content-Security-Policy` | missing | XSS mitigation absent |
| `Set-Cookie` | `PHPSESSID=...` | PHP backend, session handling |
| `Set-Cookie` | `laravel_session=...` | Laravel framework |
| `Set-Cookie` | `ASP.NET_SessionId=...` | ASP.NET application |
| `X-AspNet-Version` | `4.0.30319` | .NET version |
| `Via` | `1.1 varnish` | Varnish cache in front |
| `CF-Ray` | `abc123-LHR` | Behind Cloudflare |
| `X-Amz-Cf-Id` | `...` | AWS CloudFront CDN |

---

## httpx — HTTP Probing at Scale

After resolving a subdomain list, httpx probes all of them simultaneously and returns structured data.

```bash
# Basic live probe — title and status code
httpx -l subs.txt -title -sc -silent

# Full fingerprint pass
httpx -l subs.txt -title -sc -tech-detect -server -ip -silent

# Filter to 200-OK responses only
httpx -l subs.txt -sc -mc 200 -silent -o live-200.txt

# Filter out common dead responses
httpx -l subs.txt -sc -fc 404,400,502,503 -silent

# Full data output to JSON
httpx -l subs.txt -title -sc -tech-detect -server -ip -cname -json -o http.json

# Screenshots of every live host
httpx -l subs.txt -screenshot -o screenshots/ -sc -title

# Probe for specific paths across all hosts
httpx -l subs.txt -path /admin -sc -silent
httpx -l subs.txt -path /.git/HEAD -sc -mc 200 -silent    # exposed git repos
httpx -l subs.txt -path /phpinfo.php -sc -mc 200 -silent  # phpinfo exposure

# Follow redirects
httpx -l subs.txt -follow-redirects -title -sc -silent

# Show CNAME chain (reveals CDN and third-party services)
httpx -l subs.txt -cname -silent
```

**Flag reference:**

| Flag | Output |
|---|---|
| `-title` | HTML page title |
| `-sc` | HTTP status code |
| `-cl` | Content-Length |
| `-ct` | Content-Type |
| `-server` | Server header value |
| `-tech-detect` | Technology stack via Wappalyzer |
| `-ip` | Resolved IP address |
| `-cname` | CNAME chain |
| `-favicon` | Favicon hash (matches known tech fingerprints) |
| `-tls-probe` | TLS certificate details |
| `-screenshot` | Headless Chrome screenshot |
| `-follow-redirects` | Follow 3xx chains |
| `-mc [codes]` | Match only these status codes |
| `-fc [codes]` | Filter out these status codes |
| `-json` | JSON output |
| `-silent` | No banner/progress |

**Reading httpx output:**

```
https://admin.target.com [200] [Admin Panel] [Django 4.2,Python] [gunicorn/21.2] [185.45.12.80]
│                          │     │            │                    │               │
URL                     Status  Title      Technologies         Server header    IP
```

---

## WhatWeb — Deep Technology Fingerprinting

WhatWeb identifies 1800+ web technologies. More detailed than httpx on single targets.

```bash
# Basic scan
whatweb https://target.com

# Aggressive — makes more requests, finds more
whatweb -a 3 https://target.com

# Quiet — only non-200 responses and plugins
whatweb -q https://target.com

# Batch scan from file
whatweb -i hosts.txt --log-json=results.json

# Output formats
whatweb https://target.com --log-brief=brief.txt
whatweb https://target.com --log-json=results.json
whatweb https://target.com --log-xml=results.xml
```

**Aggressive mode levels:**

| Level | Requests made | Use when |
|---|---|---|
| `-a 1` | Passive — one request | Default, stealthy |
| `-a 3` | Aggressive — many requests | When you want thorough detection |
| `-a 4` | Very aggressive | Maximum detection, noisy |

**Example output:**

```
https://target.com [200 OK]
  Apache[2.4.49],
  Bootstrap[4.6.0],
  Cookies[PHPSESSID],
  Country[INDIA][IN],
  HTML5,
  HTTPServer[Ubuntu Linux][Apache/2.4.49 (Ubuntu)],
  IP[185.45.12.10],
  JQuery[3.5.1],
  PHP[7.4.3],
  WordPress[5.9.3],
  X-Powered-By[PHP/7.4.3]
```

From this single output: Apache 2.4.49 (has CVE-2021-41773 path traversal), PHP 7.4.3 (EOL — no security patches since Nov 2022), WordPress 5.9.3 (check WPScan for plugin vulns).

---

## Wappalyzer

Browser extension that passively identifies technologies from page source and headers as you browse. No additional requests made — reads what the browser already downloaded.

**When to use:** Quick manual check on a specific page. Good for verifying what httpx and WhatWeb detected.

**Categories detected:** CMS, ecommerce platforms, JavaScript frameworks, analytics, CDN, server software, databases, operating systems, programming languages, payment processors, marketing tools.

---

## Directory and Content Discovery

After fingerprinting, discover hidden paths and files.

### ffuf

```bash
# Directory brute-force
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -fc 404 -t 40

# File brute-force with extensions
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
  -e .php,.html,.txt,.js,.bak,.old,.conf,.env \
  -fc 404 -t 40

# Vhost discovery
ffuf -u https://target.com \
  -H "Host: FUZZ.target.com" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fc 404

# Filter by response size (remove false positives)
ffuf -u https://target.com/FUZZ \
  -w wordlist.txt \
  -fc 404 -fs 1234    # filter responses of exactly 1234 bytes

# Save matched results
ffuf -u https://target.com/FUZZ \
  -w wordlist.txt -fc 404 -o ffuf-results.json
```

### feroxbuster — Recursive Discovery

```bash
# Recursive directory scan — automatically enters found directories
feroxbuster -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  --depth 3 -t 40

# With extensions
feroxbuster -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,html,txt,js,conf,bak \
  --depth 3 -t 40 --quiet
```

### gobuster

```bash
# Directory mode
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,html,txt,js -t 40 -q

# DNS subdomain mode
gobuster dns -d target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 50
```

**Wordlist recommendations for content discovery:**

| Wordlist | Path | Best for |
|---|---|---|
| `common.txt` | SecLists/Discovery/Web-Content/ | Quick check — common paths |
| `raft-medium-directories.txt` | SecLists/Discovery/Web-Content/ | Standard directory scan |
| `raft-large-files.txt` | SecLists/Discovery/Web-Content/ | File discovery |
| `directory-list-2.3-medium.txt` | SecLists/Discovery/Web-Content/ | DirBuster classic |
| `api-endpoints.txt` | SecLists/Discovery/Web-Content/ | API endpoint discovery |

---

## Web Crawling — Extracting URLs from Live Sites

### Katana

```bash
# Basic crawl
katana -u https://target.com -o urls.txt

# With JavaScript parsing (finds dynamically loaded endpoints)
katana -u https://target.com -jc -o urls.txt

# Crawl a list of hosts
katana -list live-hosts.txt -jc -o all-urls.txt -depth 3

# Scope — only crawl pages on target domain
katana -u https://target.com -jc -o urls.txt -no-scope-check

# Find specific patterns in crawled URLs
katana -u https://target.com | grep -E "\.php\?|\.asp\?|api/"
```

### Historical URLs — Wayback Machine and gau

```bash
# gau — gets URLs from Wayback, Common Crawl, VirusTotal
gau target.com | tee gau-urls.txt

# waybackurls — Wayback Machine only
waybackurls target.com | tee wayback-urls.txt

# Merge and find interesting paths
cat gau-urls.txt wayback-urls.txt | sort -u > all-historical-urls.txt

# Filter for high-value patterns
cat all-historical-urls.txt | grep -E "admin|api|config|backup|internal|key|secret|password|token|upload"

# Find parameters (potential injection points)
cat all-historical-urls.txt | grep "?" | awk -F'?' '{print $2}' | tr '&' '\n' | cut -d'=' -f1 | sort -u
```

---

## Archive.org — Deleted Content

Pages a company has deleted, changed, or removed from their site may still be accessible via the Wayback Machine.

```bash
# What is archived for a domain
curl "https://archive.org/wayback/available?url=target.com"

# Check if a specific URL was archived
curl "https://archive.org/wayback/available?url=target.com/admin/config.php"

# From the web interface — browse snapshots
# https://web.archive.org/web/*/target.com/*

# Useful for:
# - Old admin portals that still work if not properly decommissioned
# - Configuration pages removed from the live site but still accessible on the server
# - Sitemap.xml from before it was modified to hide paths
```

---

## BuiltWith and SimilarWeb

**BuiltWith** (`builtwith.com`) shows every technology a site has ever used — including historical data. Useful for understanding how the tech stack evolved and finding legacy components.

```bash
# CLI access via API
curl "https://api.builtwith.com/v21/api.json?KEY=YOURKEY&LOOKUP=target.com"
```

**SimilarWeb** shows traffic sources, referring domains, and analytics data — useful for mapping the wider business ecosystem and finding partners and vendors.

---

## Favicon Hash Matching

Favicons have unique hashes. Shodan indexes them. If you find a favicon hash from the main domain, you can find all servers (including hidden ones) running the same application.

```bash
# Get favicon hash with httpx
httpx -u https://target.com -favicon

# Output includes: [favicon hash: 12345678]

# Search Shodan for that hash
shodan search "http.favicon.hash:12345678"
```

This finds staging and development environments that use the same application but are not linked from any DNS record — they show up because they have the same favicon.

---

## Screenshot Review — GoWitness

Taking screenshots of every live host lets you review hundreds of web apps quickly without visiting each one manually.

```bash
# Install
go install github.com/sensepost/gowitness@latest

# Screenshot a single URL
gowitness scan single -u https://target.com

# Screenshot a list
gowitness scan file -f live-hosts.txt

# View results in browser UI
gowitness server

# Generate report
gowitness report generate
```

Pattern to look for in screenshots:
- Default Apache/nginx pages → server not configured
- Login panels with software names in the title → target for known CVEs
- Jenkins/Grafana/Kibana dashboards → admin interfaces
- Error pages with stack traces → debug mode enabled
- Blank pages → could be hidden applications

---

## Full Website Enumeration Workflow

```bash
TARGET_LIST="live-hosts.txt"   # output from httpx/dnsx

echo "[1] HTTP probe all hosts"
httpx -l $TARGET_LIST -title -sc -tech-detect -server -ip -silent \
  | tee http-probe.txt

echo "[2] Filter 200-OK targets for deep dive"
httpx -l $TARGET_LIST -sc -mc 200 -silent -o live-200.txt

echo "[3] Screenshots for visual review"
gowitness scan file -f live-200.txt

echo "[4] Deep fingerprint on interesting hosts"
cat http-probe.txt | grep -iE "admin|jenkins|phpmyadmin|dashboard" \
  | awk '{print $1}' > high-value.txt
whatweb -a 3 -i high-value.txt --log-json=whatweb-hv.json

echo "[5] Content discovery on high-value hosts"
while read url; do
  ffuf -u "${url}/FUZZ" \
    -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
    -fc 404 -t 40 -silent
done < high-value.txt

echo "[6] Historical URL recovery"
while read url; do
  domain=$(echo $url | awk -F/ '{print $3}')
  gau $domain >> historical-urls.txt
done < live-200.txt

cat historical-urls.txt | grep -E "admin|config|backup|api|key" | sort -u
```

---

*Previous: [Network Footprinting](09-network-footprinting.md)*
*Next: [OSINT and Social Media Intelligence](11-osint-social-media.md)*
