# Subdomain Enumeration

The main domain is hardened. The subdomains are where you find the real attack surface — dev servers, forgotten test environments, admin panels, CI/CD pipelines, and decommissioned applications that still answer requests.

---

## Why Subdomains Matter

| Subdomain pattern | What is typically there | Why it is valuable |
|---|---|---|
| `dev.target.com` | Development environment | Debug mode enabled, verbose errors, no WAF |
| `staging.target.com` | Pre-production copy | Production data copy, less audited |
| `old.target.com` | Legacy system | Years of unpatched CVEs |
| `admin.target.com` | Admin panel | One credential away from full access |
| `vpn.target.com` | VPN gateway | Credential stuffing target |
| `jenkins.target.com` | CI/CD pipeline | Code execution on build infrastructure |
| `api.target.com` | API gateway | Business logic, rate limit issues, key exposure |
| `backup.target.com` | Backup system | Literal data backups |
| `mail.target.com` | Mail server | Email infrastructure, webmail |
| `gitlab.target.com` | Internal GitLab | Source code, secrets in repos |

---

## Method 1 — Passive Discovery (No Contact With Target)

### Certificate Transparency

Every public TLS certificate is logged. Search the logs to find every subdomain that has ever had a certificate — including internal ones.

```bash
# crt.sh — most complete
curl -s "https://crt.sh/?q=%.target.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u

# With date filtering — only recent certs
curl -s "https://crt.sh/?q=%.target.com&output=json" \
  | jq -r '.[] | select(.not_before > "2023-01-01") | .name_value' \
  | sort -u
```

### Subfinder

Queries 50+ passive data sources simultaneously. The standard starting point.

```bash
# Basic — free sources only
subfinder -d target.com -silent -o subs-subfinder.txt

# With API keys configured (~/.config/subfinder/provider-config.yaml)
subfinder -d target.com -all -silent -o subs-subfinder.txt

# Multiple domains
subfinder -dL domains.txt -silent -o subs-all.txt

# Verbose — shows which source found each subdomain
subfinder -d target.com -v 2>&1 | grep "\[" | head -50
```

**Setting up API keys (increases results significantly):**
Edit `~/.config/subfinder/provider-config.yaml` — add keys for Shodan, Censys, SecurityTrails, VirusTotal, GitHub. Free tiers are sufficient for most.

### Other Passive Tools

```bash
# findomain — cert transparency focused
findomain -t target.com -u subs-findomain.txt

# assetfinder — lightweight, fast
assetfinder --subs-only target.com > subs-assetfinder.txt

# amass passive mode
amass enum -passive -d target.com -o subs-amass.txt
```

### Combining Passive Sources

```bash
# Run all, merge, deduplicate
subfinder -d target.com -silent > subs-raw.txt
findomain -t target.com -q >> subs-raw.txt
assetfinder --subs-only target.com >> subs-raw.txt
curl -s "https://crt.sh/?q=%.target.com&output=json" \
  | jq -r '.[].name_value' | sed 's/\*\.//' >> subs-raw.txt

# Clean and deduplicate
sort -u subs-raw.txt | grep "\.target\.com$" > subs-passive.txt
wc -l subs-passive.txt
```

---

## Method 2 — Active Brute-Force

When passive discovery is done, brute-force fills the gaps. Active — DNS queries hit target's name servers.

### Check for Wildcard DNS First

```bash
dig randomstringxyz999abc.target.com A +short
```

- Returns nothing (NXDOMAIN) → no wildcard, brute-force results are valid
- Returns an IP → wildcard DNS configured, use puredns which handles this

### puredns (Recommended — Handles Wildcards)

```bash
# Install massdns first (puredns dependency)
sudo apt install massdns

# Brute-force
puredns bruteforce \
  /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  target.com \
  -r /usr/share/seclists/Miscellaneous/dns-resolvers.txt

# Validate an existing list
puredns resolve subs-passive.txt
```

### dnsrecon Brute-Force

```bash
dnsrecon -d target.com -t brt \
  -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --threads 10
```

### ffuf DNS Brute-Force

```bash
ffuf -u https://FUZZ.target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fc 404 \
  -t 50 \
  -o ffuf-subs.json
```

### Wordlist Recommendations

| Wordlist | Path | Size | Use case |
|---|---|---|---|
| `subdomains-top1million-5000.txt` | SecLists/Discovery/DNS/ | 5K | Quick pass |
| `subdomains-top1million-20000.txt` | SecLists/Discovery/DNS/ | 20K | Standard |
| `deepmagic.com-prefixes-top500.txt` | SecLists/Discovery/DNS/ | 500 | Common prefixes |
| `dns-Jhaddix.txt` | SecLists/Discovery/DNS/ | 1.1M | Bug bounty thorough |
| `best-dns-wordlist.txt` | Assetnote | 9.5M | Maximum coverage |

---

## Method 3 — Zone Transfer

If the target's DNS server allows AXFR queries from any IP, you get every subdomain at once.

```bash
# Get name servers
dig target.com NS +short

# Attempt zone transfer
dig axfr target.com @ns1.target.com
dig axfr target.com @ns2.target.com

# Automate
dnsrecon -d target.com -t axfr

# fierce also tries zone transfer first
fierce --domain target.com
```

If zone transfer succeeds, you are done with subdomain discovery. The zone file contains everything — forward to DNS analysis and HTTP probing.

---

## Method 4 — Reverse DNS on Netblock

Finds hosts that have a PTR record but no public subdomain — forgotten servers.

```bash
# Get the netblock
whois $(dig target.com A +short | head -1) | grep CIDR

# Reverse lookup the entire range
dnsrecon -r 185.45.12.0-185.45.12.255
prips 185.45.12.0/24 | dnsx -ptr -resp -silent
nmap -sL 185.45.12.0/24 | grep "report for"
```

---

## Resolving and Validating the Final List

After running all discovery methods, you have a raw list that includes duplicates and non-existent domains. Validate it:

```bash
# Merge everything
cat subs-passive.txt subs-brute.txt | sort -u > subs-all.txt

# Resolve — keep only domains that return an IP
dnsx -l subs-all.txt -a -resp -silent -o subs-resolved.txt

# Show counts
wc -l subs-all.txt          # total discovered
wc -l subs-resolved.txt     # live and resolving
```

---

## Subdomain Takeover Detection

A CNAME pointing to an unclaimed external service is a takeover opportunity — you register the service and take control of the subdomain.

```bash
# Find all CNAMEs
dnsx -l subs-resolved.txt -cname -resp -silent -o cnames.txt

# Check if the CNAME destination is claimed
# Look for patterns like:
# careers.target.com → [targetcorp.greenhouse.io]
# blog.target.com → [targetcorp.ghost.io]

# Verify manually — visit the CNAME destination
# If you see "not found" from the third-party service (not target's 404), it is unclaimed
```

**Automating with nuclei:**

```bash
nuclei -l subs-resolved.txt -t nuclei-templates/takeovers/
```

---

## Full Pipeline

```bash
TARGET="target.com"

echo "[1] Passive discovery"
subfinder -d $TARGET -silent > subs-raw.txt
findomain -t $TARGET -q >> subs-raw.txt
assetfinder --subs-only $TARGET >> subs-raw.txt
curl -s "https://crt.sh/?q=%.$TARGET&output=json" | jq -r '.[].name_value' | sed 's/\*\.//' >> subs-raw.txt

echo "[2] Clean"
sort -u subs-raw.txt | grep "\.$TARGET$" > subs-passive.txt
echo "$(wc -l < subs-passive.txt) unique passive subdomains"

echo "[3] Wildcard check"
dig random999xyz.$TARGET A +short && echo "WILDCARD DETECTED — use puredns" || echo "No wildcard"

echo "[4] Brute-force"
puredns bruteforce /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt $TARGET >> subs-brute.txt

echo "[5] Zone transfer"
dnsrecon -d $TARGET -t axfr

echo "[6] Merge and resolve"
cat subs-passive.txt subs-brute.txt | sort -u > subs-all.txt
dnsx -l subs-all.txt -a -resp -silent -o subs-live.txt
echo "$(wc -l < subs-live.txt) live subdomains"

echo "[7] HTTP probe"
awk '{print $1}' subs-live.txt | httpx -title -sc -tech-detect -silent -o http-results.txt

echo "[8] Check for takeover candidates"
dnsx -l subs-live.txt -cname -resp -silent | tee cnames.txt
```

---

*Previous: [Email Enumeration](07-email-enumeration.md)*
*Next: [Network Footprinting](09-network-footprinting.md)*
*Related: [DNS Enumeration](04-dns-enumeration.md)*
