# Tool Reference

Every tool used in this methodology. Organized by category with install commands, when to use it, and the alternatives.

---

## DNS Enumeration

### dig
**What it is:** Built-in DNS query tool. The ground truth for DNS — every other tool is an abstraction on top of what dig does directly.
**When to use:** Manual record lookups, verifying results from other tools, zone transfer attempts, debugging.
**Install:** Pre-installed on Linux/macOS. Windows: install BIND tools.

| Command | Purpose |
|---|---|
| `dig target.com A +short` | IPv4 address |
| `dig target.com ANY +noall +answer` | All record types |
| `dig target.com MX +short` | Mail servers |
| `dig target.com NS +short` | Name servers |
| `dig target.com TXT +short` | TXT records (SPF, DMARC, verification) |
| `dig target.com SOA +short` | Zone authority + admin email |
| `dig -x 185.45.12.10 +short` | Reverse lookup |
| `dig axfr target.com @ns1.target.com` | Zone transfer attempt |
| `dig target.com +trace` | Trace full resolution chain |

---

### dnsrecon
**What it is:** Python-based DNS recon framework. Multiple scan types, structured output, handles zone transfers and brute-force.
**When to use:** First full DNS pass on a target. Best option when you want structured, exportable results.
**Install:** `pip3 install dnsrecon` or pre-installed on Kali.

| Command | Purpose |
|---|---|
| `dnsrecon -d target.com -t std` | Full standard enumeration |
| `dnsrecon -d target.com -t axfr` | Zone transfer, all NS servers |
| `dnsrecon -d target.com -t brt -D wordlist.txt` | Subdomain brute-force |
| `dnsrecon -r 185.45.12.0-185.45.12.255` | Reverse lookup a range |
| `dnsrecon -d target.com -t goo` | Google-based passive enum |
| `dnsrecon -d target.com -t std -o out.json` | Save output to JSON |

---

### dnsx
**What it is:** ProjectDiscovery's DNS toolkit. Fast, concurrent, pipeline-friendly. Best for resolving large subdomain lists.
**When to use:** Filtering a subdomain list down to live hosts. Bulk PTR lookups. Checking CNAME chains across hundreds of hosts.
**Install:** `go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest`

| Command | Purpose |
|---|---|
| `dnsx -l subs.txt -a -resp -silent` | Resolve list, show IPs |
| `dnsx -l subs.txt -cname -resp -silent` | CNAME lookup (subdomain takeover) |
| `dnsx -l subs.txt -txt -resp -silent` | TXT records for all hosts |
| `prips 185.45.12.0/24 \| dnsx -ptr -resp -silent` | PTR lookup on a range |
| `dnsx -l subs.txt -a -cname -json -o out.json` | JSON output |
| `dnsx -d target.com -wc` | Wildcard detection |

---

### fierce
**What it is:** DNS recon tool with automatic fallback from zone transfer to brute-force, and near-IP traversal.
**When to use:** When you want one command that tries everything automatically, or when you want near-IP discovery (`--traverse`).
**Install:** `pip3 install fierce`

| Command | Purpose |
|---|---|
| `fierce --domain target.com` | Zone transfer + brute-force |
| `fierce --domain target.com --subdomains admin vpn jenkins` | Test specific subdomains |
| `fierce --domain target.com --traverse 10` | Sweep IPs around found hosts |
| `fierce --domain target.com --connect` | HTTP probe on found hosts |
| `fierce --domain target.com --delay 2` | Rate-limited scan |

---

## Subdomain Enumeration

### subfinder
**What it is:** Fast passive subdomain discovery via 50+ data sources (Shodan, Censys, VirusTotal, crt.sh, etc.).
**When to use:** First passive subdomain pass. No packets to the target.
**Install:** `go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest`

| Command | Purpose |
|---|---|
| `subfinder -d target.com -silent -o subs.txt` | Basic passive enum |
| `subfinder -d target.com -all -o subs.txt` | All sources (needs API keys) |
| `subfinder -dL domains.txt -o subs.txt` | Multiple domains |
| `subfinder -d target.com -v` | Verbose — shows source per result |

**Alternatives:** assetfinder (lightweight), findomain (cert transparency focused), amass (deep, slow, both passive and active)

---

### amass
**What it is:** OWASP attack surface mapping tool. Most thorough subdomain tool available — combines passive APIs, active DNS, cert transparency, and zone walks.
**When to use:** When completeness matters more than speed. Runs slow but finds what others miss.
**Install:** `go install github.com/owasp-amass/amass/v4/...@master`

| Command | Purpose |
|---|---|
| `amass enum -passive -d target.com` | Passive only |
| `amass enum -d target.com` | Passive + active |
| `amass enum -brute -d target.com -w wordlist.txt` | Active brute-force |
| `amass enum -d target.com -o subs.txt` | Save output |
| `amass db -d target.com -show` | Show stored results |

---

### puredns
**What it is:** Mass DNS resolver with correct wildcard handling. Wraps massdns.
**When to use:** Brute-force subdomain validation at scale. Best for large wordlists.
**Install:** `go install github.com/d3mondev/puredns/v2@latest` (requires massdns)

```bash
puredns bruteforce /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt target.com
puredns resolve subs.txt   # validate an existing list
```

---

## OSINT and Passive Intelligence

### theHarvester
**What it is:** Classic OSINT tool for email, subdomain, and employee harvesting from public sources.
**When to use:** Early passive recon — getting emails and names before active work.
**Install:** Pre-installed on Kali. `pip3 install theHarvester`

| Command | Purpose |
|---|---|
| `theHarvester -d target.com -l 500 -b google` | Google email harvest |
| `theHarvester -d target.com -l 500 -b linkedin` | LinkedIn harvest |
| `theHarvester -d target.com -l 500 -b all -f out.html` | All sources, HTML report |

---

### SpiderFoot
**What it is:** Automated OSINT platform with 200+ modules. Correlates IPs, domains, emails, social profiles, breach records.
**When to use:** When you want a comprehensive OSINT sweep with relationship mapping and a browsable UI.
**Install:** `pip3 install spiderfoot` or Docker.

```bash
# Start the web UI
spiderfoot -l 127.0.0.1:5001

# CLI scan
spiderfoot -s target.com -t INTERNET_NAME -o csv -f output.csv
```

**Alternative:** Maltego (GUI-based, better visualization, free community edition)

---

### Recon-ng
**What it is:** Modular web recon framework styled like Metasploit. Install modules from marketplace.
**When to use:** When you want structured, database-backed recon with module control similar to Metasploit.
**Install:** Pre-installed on Kali. `pip3 install recon-ng`

```bash
recon-ng
> marketplace install all
> workspaces create target
> db insert domains  # add target.com
> modules load recon/domains-hosts/hackertarget
> run
```

---

### Sherlock
**What it is:** Username search across 300+ social platforms.
**When to use:** Building a profile on a specific employee target.
**Install:** `pip3 install sherlock-project`

```bash
sherlock johndoe
sherlock johndoe --csv --output results.csv
```

---

## HTTP Probing and Web Fingerprinting

### httpx
**What it is:** Fast HTTP toolkit by ProjectDiscovery. The central tool for turning a subdomain list into actionable web recon data.
**When to use:** After subdomain discovery — probe everything, fingerprint what's running, filter live hosts.
**Install:** `go install github.com/projectdiscovery/httpx/cmd/httpx@latest`

| Command | Purpose |
|---|---|
| `httpx -l subs.txt -title -sc -silent` | Basic live probe |
| `httpx -l subs.txt -tech-detect -server -ip -silent` | Full fingerprint |
| `httpx -l subs.txt -sc -mc 200 -silent` | Filter 200-OK only |
| `httpx -l subs.txt -json -o out.json` | JSON output |
| `httpx -l subs.txt -screenshot -o screens/` | Screenshot all hosts |

---

### WhatWeb
**What it is:** Web technology fingerprinting. Identifies 1800+ technologies from HTTP headers and page content.
**When to use:** Targeted fingerprinting of specific hosts. Good for single-target deep analysis.
**Install:** Pre-installed on Kali. `gem install whatweb`

```bash
whatweb https://target.com
whatweb -a 3 https://target.com           # aggressive mode
whatweb -i hosts.txt --log-json=out.json  # batch scan
```

**Alternative:** Wappalyzer (browser extension — passive, good for quick checks)

---

### Katana
**What it is:** Next-gen web crawler by ProjectDiscovery. Handles JS-rendered pages, extracts endpoints, forms, parameters.
**When to use:** Enumerating all endpoints on live web hosts after fingerprinting.
**Install:** `go install github.com/projectdiscovery/katana/cmd/katana@latest`

```bash
katana -u https://target.com -o endpoints.txt
katana -list live-hosts.txt -o endpoints.txt -jc    # JS crawling enabled
```

---

## URL Discovery

### waybackurls / gau
**What it is:** Fetches all URLs a domain has ever had, from Wayback Machine, Common Crawl, VirusTotal.
**When to use:** Finding old endpoints, forgotten admin panels, removed pages that still respond.
**Install:** `go install github.com/tomnomnom/waybackurls@latest` / `go install github.com/lc/gau/v2/cmd/gau@latest`

```bash
waybackurls target.com | tee wayback.txt
gau target.com | tee gau.txt

# Find interesting paths in results
cat wayback.txt gau.txt | sort -u | grep -E "admin|api|config|backup|key|secret|password"
```

---

## Network Scanning and Mapping

### Nmap
**What it is:** The standard port scanner. Service detection, OS fingerprinting, NSE scripting.
**When to use:** After host discovery — detailed service enumeration on specific targets.
**Install:** Pre-installed on Kali. `sudo apt install nmap`

| Command | Purpose |
|---|---|
| `nmap -sV target.com` | Service version detection |
| `nmap -O target.com` | OS fingerprinting |
| `nmap -sV -O -A target.com` | Full aggressive scan |
| `nmap -sL 185.45.12.0/24` | List scan — reverse DNS, no port scan |
| `nmap -p- target.com` | All 65535 ports |
| `nmap --traceroute -sn target.com` | Traceroute without port scan |

---

### Nuclei
**What it is:** YAML-template-driven vulnerability scanner with 9000+ community templates.
**When to use:** After httpx — run vulnerability templates against all live HTTP hosts.
**Install:** `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest`

```bash
nuclei -l live-hosts.txt                          # all templates
nuclei -l live-hosts.txt -t exposures/            # exposed files/panels
nuclei -l live-hosts.txt -t cves/ -severity high  # high-severity CVEs only
nuclei -l live-hosts.txt -t misconfigurations/    # misconfigurations
```

---

## WHOIS and IP Intelligence

### whois
**What it is:** Command-line WHOIS client. Queries domain and IP registration data.
**Install:** Pre-installed on Linux/macOS.

```bash
whois target.com                                          # domain info
whois 185.45.12.10                                        # IP info + netblock
whois 185.45.12.10 | grep -E "NetRange|CIDR|OrgName"     # extract key fields
whois -h whois.arin.net "o ORGNAME"                       # all blocks for an org
```

---

## Metadata Extraction

### metagoofil
**What it is:** Extracts metadata from publicly indexed documents (PDF, DOCX, XLSX, PPTX).
**When to use:** Finding internal usernames, server paths, software versions from company-published documents.
**Install:** `pip3 install metagoofil`

```bash
metagoofil -d target.com -t pdf,docx,xlsx -o docs/ -f results.html
```

### FOCA
**What it is:** Windows-based metadata extraction and analysis tool. Better UI than metagoofil.
**When to use:** When you have downloaded documents and want thorough metadata analysis with relationship mapping.
**Platform:** Windows only. Available at github.com/ElevenPaths/FOCA

---

## Tool Selection Guide

| Task | First choice | Alternative | Notes |
|---|---|---|---|
| DNS full enum | `dnsrecon -t std` | `dig ANY` | dnsrecon structures output better |
| Zone transfer | `dnsrecon -t axfr` | `dig axfr` | dnsrecon tries all NS automatically |
| Subdomain passive | `subfinder` | `amass -passive` | Run both, merge results |
| Subdomain brute-force | `puredns` | `dnsrecon -t brt` | puredns handles wildcards correctly |
| Resolve subdomain list | `dnsx` | `massdns` | dnsx is simpler, massdns is faster at scale |
| HTTP probing | `httpx` | `httprobe` | httpx has more features |
| Tech fingerprinting | `httpx -tech-detect` | `WhatWeb` | WhatWeb is more detailed on single targets |
| Email harvest | `theHarvester` | `hunter.io` | Combine both |
| OSINT framework | `SpiderFoot` | `Maltego` | SpiderFoot = CLI/self-hosted, Maltego = visual |
| Web crawling | `katana` | `hakrawler` | katana handles JS, hakrawler is simpler |
| URL history | `gau` | `waybackurls` | gau pulls from more sources |
| Vulnerability scan | `nuclei` | `nikto` | nuclei is faster and has more templates |
| Network path | `traceroute` | `tcptraceroute` | Use TCP when ICMP is blocked |
| Port scan | `nmap` | `rustscan + nmap` | rustscan for speed, nmap for detail |

---

*Back to [README](../README.md)*
