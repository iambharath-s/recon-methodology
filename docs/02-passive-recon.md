# Passive Reconnaissance

Passive recon means gathering information about a target without ever interacting with their systems directly. Every source you query is a third party — Google, ARIN, LinkedIn, Shodan — and the target has no way to detect you are looking.

This is always the starting point. Active recon leaves logs. Passive leaves nothing.

---

## What You Are Looking For

By the end of passive recon you want to answer three questions:

1. **What does the organization own?** — domains, IP ranges, subsidiaries, cloud assets
2. **Who works there and what do they know?** — employees, roles, tech stack from job postings
3. **What is exposed to the internet?** — servers, services, credentials from breaches

Everything feeds into active recon and eventually into the attack phase.

---

## Search Engine Recon

### Google Dorking

Google has indexed the internet's mistakes. Config files left in web roots, directory listings, login portals, database dumps — they all show up if you know how to ask.

**Core operators:**

| Operator | Syntax | What it finds |
|---|---|---|
| `site:` | `site:target.com` | Every indexed page on the domain |
| `filetype:` | `filetype:pdf site:target.com` | Specific file extensions |
| `intitle:` | `intitle:"index of"` | Text in the page title |
| `inurl:` | `inurl:/admin/login` | Text in the URL |
| `intext:` | `intext:"api_key"` | Text in the page body |
| `cache:` | `cache:target.com` | Google's cached snapshot |
| `link:` | `link:target.com` | Pages linking to the target |

**High-value dork combinations:**

```
# Open directory listings
intitle:"index of" site:target.com

# Exposed config and environment files
filetype:env OR filetype:config OR filetype:conf site:target.com

# Database files
filetype:sql OR filetype:db site:target.com

# Exposed credentials in text files
filetype:txt intext:password site:target.com

# Login portals
inurl:"/admin" OR inurl:"/login" OR inurl:"/wp-admin" site:target.com

# Exposed API keys
intext:"api_key" OR intext:"apikey" OR intext:"secret_key" site:target.com

# VPN configuration files
filetype:pcf OR filetype:ovpn site:target.com

# Private keys
intext:"-----BEGIN RSA PRIVATE KEY-----" site:target.com

# phpMyAdmin instances
intitle:"phpMyAdmin" inurl:"phpmyadmin" site:target.com

# Find all subdomains Google has indexed
site:*.target.com -site:www.target.com
```

**GHDB — Google Hacking Database**
`exploit-db.com/google-hacking-database` is a searchable library of pre-built dorks organized by category. Before crafting your own, search here — someone has almost certainly already built the query you need.

---

### Shodan

Shodan indexes internet-connected devices by scanning every IP address and storing what it finds — open ports, service banners, TLS certificates, HTTP headers. It is a permanent passive record of what each IP was running when it was last scanned.

**Searching by organization:**

```
# Everything registered to a company name
org:"Target Corp"

# Specific ASN
asn:AS12345

# IP range
net:185.45.12.0/24

# Combine filters
org:"Target Corp" port:22

# Find exposed databases
org:"Target Corp" product:MongoDB

# Find unprotected webcams, printers, SCADA
org:"Target Corp" port:554             # RTSP cameras
org:"Target Corp" port:9100            # printers
org:"Target Corp" port:502             # Modbus / industrial
```

**Shodan CLI — faster than the web interface for recon pipelines:**

```bash
# Install
pip install shodan

# Initialize with API key
shodan init YOUR_API_KEY

# Search and output IPs only
shodan search --fields ip_str,port,org "org:Target Corp" 

# Download full results as JSON
shodan download results.json.gz "org:Target Corp"

# Host info on a specific IP
shodan host 185.45.12.10

# Count results without pulling data
shodan count "org:Target Corp"
```

---

### Censys

Censys continuously scans IPv4 space and indexes everything it finds. Better than Shodan for TLS certificate analysis and finding assets the organization may not know they have.

```
# Search by organization
autonomous_system.organization: "Target Corp"

# Find by certificate common name
parsed.names: target.com

# IPv4 hosts with a specific port open
services.port: 8443 and autonomous_system.organization: "Target Corp"
```

Censys is especially useful for finding certificates issued for internal hostnames — a certificate for `internal.target.com` tells you the hostname exists even if no DNS record points to it publicly.

---

## WHOIS and RIR Lookups

### Domain WHOIS

```bash
# Basic lookup
whois target.com

# Key fields to extract
whois target.com | grep -E "Registrar|Name Server|Admin Email|Tech Email|Expir|Creat"
```

**What you are reading:**

| Field | Recon value |
|---|---|
| Registrant email | Social engineering target, format tells you email structure |
| Name servers | Targets for zone transfer |
| Expiry date | Domain hijack opportunity if expiry is soon |
| Admin/tech email | IT contact — spear phishing target |
| Creation date | Company age — older = more legacy systems |

### IP WHOIS and Netblock Discovery

```bash
# Find the IP first
dig target.com A +short   # returns: 185.45.12.10

# Query WHOIS for that IP — returns the netblock
whois 185.45.12.10

# Extract the range
whois 185.45.12.10 | grep -E "NetRange|CIDR|OrgName|OrgId"

# Find all netblocks registered to the same org
whois 185.45.12.10 | grep OrgId          # get: TC-1234
whois -h whois.arin.net "o TC-1234"      # get all blocks for that org
```

**RIR reference — query the right registry:**

| RIR | Region | Direct query |
|---|---|---|
| ARIN | North America | `whois -h whois.arin.net` |
| RIPE | Europe, Middle East, Central Asia | `whois -h whois.ripe.net` |
| APNIC | Asia-Pacific | `whois -h whois.apnic.net` |
| AFRINIC | Africa | `whois -h whois.afrinic.net` |
| LACNIC | Latin America, Caribbean | `whois -h whois.lacnic.net` |

---

## Certificate Transparency

Every TLS certificate issued by a public CA is logged to certificate transparency logs — publicly searchable. This includes certificates for internal subdomains that were never meant to be discovered.

```bash
# crt.sh — the fastest way
curl -s "https://crt.sh/?q=%.target.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u

# Find by organization name
curl -s "https://crt.sh/?O=Target+Corp&output=json" | jq -r '.[].name_value' | sort -u
```

Certificate transparency catches subdomains that:
- Were created for internal testing and given a real certificate
- Were used briefly and then abandoned
- Were never added to public DNS but still have active certificates

---

## Passive Subdomain Discovery

### Subfinder

Queries 50+ passive sources (Shodan, Censys, VirusTotal, SecurityTrails, ThreatCrowd, etc.) simultaneously. No packets to the target.

```bash
# Basic run
subfinder -d target.com -o subs.txt

# With all sources (needs API keys for full results)
subfinder -d target.com -all -o subs.txt

# Silent output — clean list only, no banner
subfinder -d target.com -silent

# Multiple domains at once
subfinder -dL domains.txt -o subs.txt

# Show which source found each subdomain
subfinder -d target.com -v
```

### Findomain

Watches certificate transparency logs in real time. Also good for one-off passive discovery.

```bash
findomain -t target.com -u subs.txt
```

### assetfinder

Lightweight and fast — good for quick passive lookups.

```bash
assetfinder --subs-only target.com >> subs.txt
```

### Combining sources for maximum coverage

```bash
# Run all three passive tools, merge, deduplicate
subfinder -d target.com -silent > subs-raw.txt
findomain -t target.com -q >> subs-raw.txt
assetfinder --subs-only target.com >> subs-raw.txt
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//' >> subs-raw.txt

# Deduplicate and clean
sort -u subs-raw.txt > subs-final.txt
wc -l subs-final.txt   # see how many unique subdomains found
```

---

## Email Harvesting

Emails are credentials and social engineering targets. You want as many as possible.

```bash
# theHarvester — searches multiple sources
theHarvester -d target.com -l 500 -b google
theHarvester -d target.com -l 500 -b linkedin
theHarvester -d target.com -l 500 -b bing
theHarvester -d target.com -l 500 -b baidu

# Run all sources at once
theHarvester -d target.com -l 500 -b all -f results.html
```

**Sources worth knowing:**

| Source flag | What it searches |
|---|---|
| `google` | Google search results — passive |
| `bing` | Bing search results — passive |
| `linkedin` | LinkedIn (limited without API) |
| `hunter` | Hunter.io API — good email format confirmation |
| `intelx` | Intelligence X — breach data |
| `shodan` | Shodan (requires API key) |

**From emails, derive the format:**

If you find `john.smith@target.com` and `sarah.jones@target.com`, the format is `firstname.lastname@target.com`. Use this to generate a wordlist for every employee name found on LinkedIn.

---

## OSINT — Social Media and People Search

### LinkedIn

The most valuable passive source for corporate intelligence.

**What you extract from LinkedIn:**

- Employee names and roles → build org chart, find admins
- Job postings → reveal exact tech stack (firewall vendor, cloud platform, DB)
- Company headcount → scale of infrastructure
- Recent hires in IT/Security → what new tools they are implementing
- Endorsements → confirm which technologies specific employees know

**Tools for LinkedIn recon:**

```bash
# theHarvester with linkedin source
theHarvester -d target.com -b linkedin

# CrossLinked — generates email combinations from LinkedIn profiles
crosslinked -f '{first}.{last}@target.com' "Target Corp"
```

### Job Posting Intelligence

Job postings are accidental threat intelligence. A posting for "Senior Network Engineer — Palo Alto PCNSE preferred, Cisco ASA experience required, Fortinet optional" tells you exactly what sits on the network perimeter before you scan a single port.

Search: `site:linkedin.com/jobs "Target Corp"`, `site:indeed.com "Target Corp" engineer`

### People Search Services

Spokeo, Pipl, BeenVerified, Intelius — useful for building profiles on specific targets (executives, IT staff). Cross-reference names from LinkedIn against these to find personal emails, phone numbers, and addresses.

### Sherlock — Username Hunting

```bash
# Install
pip3 install sherlock-project

# Find all social profiles for a username
sherlock johndoe
```

---

## Metadata Extraction from Public Documents

Documents published on company websites — PDFs, Word files, PPTX — contain metadata that reveals internal information: author names, internal hostnames, software versions, file paths.

```bash
# Find and download all public documents
metagoofil -d target.com -t pdf,doc,docx,xls,xlsx,ppt,pptx -o docs/ -f results.html

# What to look for in results:
# - Usernames → account names for password spraying
# - Software versions → match to CVEs
# - Internal server paths (e.g. \\fileserver01\share\docs\) → internal hostnames
# - Computer names → device naming convention
```

---

## Archive.org — Deleted Content

The Wayback Machine archives web content over time. Content the organization has deleted or changed may still be accessible.

```bash
# All URLs ever indexed for a domain
waybackurls target.com | tee wayback-urls.txt

# Alternative — gau (GetAllUrls)
gau target.com | tee gau-urls.txt

# Look for high-value paths in the results
cat wayback-urls.txt | grep -E "admin|config|backup|api|internal|password|key"
```

---

## Competitive Intelligence Sources

| Source | URL | What it provides |
|---|---|---|
| SEC EDGAR | edgar.sec.gov | Financial filings — infrastructure details, litigation, strategy |
| Crunchbase | crunchbase.com | Funding rounds, team, acquisitions |
| LinkedIn | linkedin.com | Employees, org structure, tech stack from job posts |
| Glassdoor | glassdoor.com | Internal tools, IT issues, culture — accidentally revealed |
| BuiltWith | builtwith.com | Website technology stack history |
| SimilarWeb | similarweb.com | Traffic, referral sources |
| PatentsView | patentsview.org | Technology patents — reveals R&D direction |

---

## OPSEC for Passive Recon

Passive recon is inherently low-risk but not zero-risk.

- **Google dorking is completely safe** — you are querying Google, not the target
- **Shodan and Censys are completely safe** — pre-indexed data, no contact with target
- **WHOIS is safe** — public database, no target contact
- **theHarvester with Google/Bing** — safe, queries search engines
- **LinkedIn scraping** — risk of account suspension, not target detection
- **crt.sh and certificate transparency** — completely safe
- **Wayback Machine** — completely safe

The only borderline passive technique is sending queries through APIs like Hunter.io or SecurityTrails — these log your API key against your queries. Not a target detection risk, but an operational data trail.

---

*Next: [Active Reconnaissance](03-active-recon.md)*
*Related: [DNS Enumeration](04-dns-enumeration.md) · [Subdomain Enumeration](08-subdomain-enumeration.md)*
