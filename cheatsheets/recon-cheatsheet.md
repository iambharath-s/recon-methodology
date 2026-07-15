# Recon Cheat Sheet

Quick reference for active assessments. Commands only — for explanations see the [full docs](../docs/).

---

## Phase 1 — First 5 Minutes

```bash
# Resolve target IP
dig target.com A +short

# Get the netblock
whois $(dig target.com A +short | head -1) | grep -E "NetRange|CIDR|OrgName"

# Pull all DNS records
dig target.com ANY +noall +answer

# Attempt zone transfer
dnsrecon -d target.com -t axfr

# Check for subdomains via crt.sh (fully passive)
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sort -u
```

---

## DNS Commands

```bash
# Record types
dig target.com A +short
dig target.com AAAA +short
dig target.com MX +short
dig target.com NS +short
dig target.com TXT +short
dig target.com SOA +short
dig -x 185.45.12.10 +short                     # PTR reverse lookup

# Zone transfer
dig axfr target.com @ns1.target.com
dnsrecon -d target.com -t axfr                 # tries all NS servers

# Standard full enum
dnsrecon -d target.com -t std

# Subdomain brute-force
dnsrecon -d target.com -t brt -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --threads 10
puredns bruteforce /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt target.com

# Reverse lookup a range
dnsrecon -r 185.45.12.0-185.45.12.255
prips 185.45.12.0/24 | dnsx -ptr -resp -silent
nmap -sL 185.45.12.0/24                        # list scan — DNS only, no port scan

# Wildcard check
dig randomstringthatdoesnotexist123.target.com A

# Fierce — auto zone transfer + brute-force + near-IP
fierce --domain target.com
fierce --domain target.com --traverse 10
```

---

## Subdomain Enumeration

```bash
# Passive
subfinder -d target.com -silent -o subs.txt
findomain -t target.com -u subs-fd.txt
assetfinder --subs-only target.com >> subs.txt
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' >> subs.txt

# Merge and deduplicate
sort -u subs.txt subs-fd.txt > subs-final.txt

# Resolve list to live hosts
dnsx -l subs-final.txt -a -resp -silent -o live.txt

# Active brute-force
ffuf -u https://FUZZ.target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fc 404
gobuster dns -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50

# Find CNAME chains (subdomain takeover candidates)
dnsx -l subs-final.txt -cname -resp -silent
```

---

## WHOIS and IP Intel

```bash
# Domain WHOIS
whois target.com
whois target.com | grep -E "Registrar|Name Server|Admin|Expir|Creat"

# IP WHOIS
whois 185.45.12.10
whois 185.45.12.10 | grep -E "NetRange|CIDR|OrgName|OrgId"

# All blocks for an org (ARIN)
whois -h whois.arin.net "o ORGID-ARIN"

# RIR web portals
# ARIN:    search.arin.net
# RIPE:    apps.db.ripe.net
# APNIC:   wq.apnic.net
# AFRINIC: afrinic.net/whois
# LACNIC:  query.milacnic.lacnic.net
```

---

## Email Harvesting

```bash
theHarvester -d target.com -l 500 -b google
theHarvester -d target.com -l 500 -b bing
theHarvester -d target.com -l 500 -b linkedin
theHarvester -d target.com -l 500 -b all -f report.html
```

---

## HTTP Probing and Fingerprinting

```bash
# Probe live hosts
httpx -l subs.txt -title -sc -tech-detect -server -ip -silent

# Filter to 200-OK only
httpx -l subs.txt -sc -mc 200 -silent -o live-200.txt

# Full data + JSON
httpx -l subs.txt -title -sc -tech-detect -server -ip -cname -json -o http-results.json

# Screenshots
httpx -l subs.txt -screenshot -o screenshots/

# Single target fingerprint
whatweb -a 3 https://target.com

# HTTP headers only
curl -IL https://target.com
```

---

## Web Content Discovery

```bash
# Directory fuzzing
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -fc 404 -t 40

# Recursive
feroxbuster -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  --depth 3 -t 40

# With extension brute-force
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,html,txt,js,env,bak -t 40

# URL history
gau target.com | tee gau-urls.txt
waybackurls target.com | tee wayback-urls.txt
cat gau-urls.txt wayback-urls.txt | sort -u | grep -E "admin|api|config|backup|key"
```

---

## Google Dorks

```bash
# Subdomains Google has indexed
site:*.target.com -site:www.target.com

# Exposed files
filetype:env OR filetype:config OR filetype:conf site:target.com
filetype:sql OR filetype:db site:target.com
filetype:pdf site:target.com

# Login portals
inurl:"/admin" OR inurl:"/login" OR inurl:"/wp-admin" site:target.com

# Open directories
intitle:"index of" site:target.com

# Exposed credentials
filetype:txt intext:password site:target.com
intext:"api_key" OR intext:"secret_key" site:target.com

# phpMyAdmin
intitle:"phpMyAdmin" inurl:"phpmyadmin" site:target.com

# Private keys
intext:"-----BEGIN RSA PRIVATE KEY-----" site:target.com
```

---

## Network Mapping

```bash
# Traceroute
traceroute target.com          # Linux/macOS — UDP
tracert target.com             # Windows — ICMP
sudo tcptraceroute target.com 443   # TCP — bypasses ICMP blocking

# Nmap — specific to recon phase
nmap -sL 185.45.12.0/24       # list scan — pure DNS, no packets to hosts
nmap --traceroute -sn target.com
nmap -sV -O target.com        # service + OS detection
```

---

## Metadata

```bash
# Harvest public documents + extract metadata
metagoofil -d target.com -t pdf,doc,docx,xls,xlsx,ppt,pptx -o docs/ -f report.html

# Usernames, paths, software versions from metadata
# Check docs/ output for author fields, creator fields, internal paths
```

---

## Full Pipeline (copy-paste)

```bash
TARGET="target.com"

# DNS
dnsrecon -d $TARGET -t std -o dns-std.json
dnsrecon -d $TARGET -t axfr
dig $TARGET TXT +short

# Passive subdomains
subfinder -d $TARGET -silent -o subs.txt
curl -s "https://crt.sh/?q=%.$TARGET&output=json" | jq -r '.[].name_value' | sort -u >> subs.txt
sort -u subs.txt -o subs.txt

# Resolve and probe
dnsx -l subs.txt -a -cname -resp -silent | tee resolved.txt
awk '{print $1}' resolved.txt | httpx -title -sc -tech-detect -silent | tee http-probe.txt

# Email harvest
theHarvester -d $TARGET -l 500 -b google,bing -f harvest.html

# Network
whois $(dig $TARGET A +short | head -1) | grep -E "NetRange|CIDR|OrgName"
nmap -sL $(whois $(dig $TARGET A +short | head -1) | grep CIDR | awk '{print $2}' | head -1)
```

---

*Full documentation: [docs/](../docs/) | DNS records reference: [dns-records-cheatsheet.md](dns-records-cheatsheet.md)*
