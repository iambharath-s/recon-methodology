# DNS Enumeration

DNS is the single most information-dense target in any reconnaissance engagement. A misconfigured DNS server can hand you a complete infrastructure map before you send a single packet to an application.

---

## How DNS Works (The Bits That Matter for Recon)

DNS is a distributed database. Every domain's records live on an **authoritative name server** controlled by the organization. To reach it, queries flow through a chain: your resolver → root servers → TLD servers → authoritative server.

For recon purposes, the authoritative server is the target. That is where zone transfers happen and where brute-force subdomain queries land.

```
Your tool
    │
    ▼  "What is mail.target.com?"
Recursive Resolver (8.8.8.8)
    │
    ▼  "Who handles .com?"
Root Server
    │  → TLD Server (for .com)
    ▼
.com TLD Server
    │  → ns1.target.com (authoritative)
    ▼
Authoritative Server (ns1.target.com)
    │  → "185.45.12.11"
    ▼
Your tool ← answer
```

**Key detail:** DNS queries are plaintext UDP on port 53 by default. Unencrypted. Anyone on the network path can read them. Zone transfers use TCP port 53.

---

## DNS Record Types — Recon Value

| Record | Queried with | What it reveals |
|---|---|---|
| `A` | `dig target.com A` | IPv4 — primary web/server IPs |
| `AAAA` | `dig target.com AAAA` | IPv6 — often bypasses WAF/CDN |
| `MX` | `dig target.com MX` | Mail servers — email provider, phishing vector |
| `NS` | `dig target.com NS` | Authoritative DNS servers — zone transfer targets |
| `CNAME` | `dig sub.target.com CNAME` | Third-party services — subdomain takeover candidates |
| `SOA` | `dig target.com SOA` | Admin email, serial number, zone info |
| `TXT` | `dig target.com TXT` | SPF/DMARC gaps, service verification tokens |
| `SRV` | `dig _kerberos._tcp.target.com SRV` | Internal services — LDAP, Kerberos, SIP, XMPP |
| `PTR` | `dig -x 185.45.12.10` | Reverse — IP to hostname, labels netblocks |
| `HINFO` | `dig target.com HINFO` | CPU + OS — if set (rare but valuable) |

### Reading TXT Records for Weaknesses

```bash
dig target.com TXT +short
```

| TXT value | What it means |
|---|---|
| `v=spf1 ... ~all` | Softfail SPF — spoofing succeeds and lands in inbox |
| `v=spf1 ... -all` | Hardfail SPF — spoofed emails rejected |
| `v=spf1 +all` | No SPF enforcement — spoof anything |
| `v=DMARC1; p=none` | DMARC monitor only — zero enforcement |
| `v=DMARC1; p=reject` | DMARC enforced — spoofing blocked |
| `MS=ms12345` | Microsoft 365 verification — company uses O365 |
| `google-site-verification=` | Google Workspace confirmed |
| `include:sendgrid.net` | Uses SendGrid for bulk email |
| `include:amazonses.com` | Uses AWS SES |

---

## Zone Transfer — The Full Breakdown

### What It Is

Zone transfer (AXFR) is the mechanism DNS servers use to replicate records from a primary to a secondary server. When misconfigured to allow any requester, a single query returns every DNS record in the zone.

**AXFR** = full transfer (all records)
**IXFR** = incremental transfer (only changes since last sync)

### Step-by-Step Attack

```bash
# Step 1 — identify name servers
dig target.com NS +short
# ns1.target.com.
# ns2.target.com.

# Step 2 — resolve NS IPs (some servers only accept queries to their IP directly)
dig ns1.target.com A +short
# 185.45.12.5

# Step 3 — attempt AXFR against each NS
dig axfr target.com @ns1.target.com
dig axfr target.com @ns2.target.com
dig axfr target.com @185.45.12.5

# Step 4 — automate across all NS with dnsrecon
dnsrecon -d target.com -t axfr
```

### Successful Transfer — What You Get

```
target.com.          SOA  ns1.target.com. admin.target.com. 2024010101
target.com.          NS   ns1.target.com.
target.com.          NS   ns2.target.com.
target.com.          A    185.45.12.10
mail.target.com.     A    185.45.12.11
vpn.target.com.      A    185.45.12.15        ← VPN gateway
dev.target.com.      A    185.45.12.45        ← dev server, likely unpatched
staging.target.com.  A    185.45.12.46
admin.target.com.    A    185.45.12.80        ← admin panel
jenkins.target.com.  A    185.45.12.90        ← CI/CD pipeline
db.target.com.       A    10.0.0.50           ← internal IP leaked
dc1.target.com.      A    10.0.0.5            ← domain controller
backup.target.com.   A    185.45.12.20
```

The leaked internal IPs (`10.0.0.x`) are especially valuable — they reveal the internal network scheme, which becomes actionable the moment you get any foothold inside the network.

### Why Secondary NS Servers Are Worth Trying Separately

Primary and secondary name servers are often configured by different people at different times. The primary has AXFR locked down; the secondary was set up by a contractor three years ago and nobody touched it since. Always try every NS server individually.

---

## Tools

### dig

The baseline. Every DNS tool is essentially a wrapper around what `dig` does manually.

```bash
# All record types
dig target.com ANY +noall +answer

# Specific record types
dig target.com A +short
dig target.com MX +short
dig target.com NS +short
dig target.com TXT +short
dig target.com SOA +short

# Reverse lookup
dig -x 185.45.12.10 +short

# Zone transfer
dig axfr target.com @ns1.target.com

# Query a specific resolver
dig target.com A @8.8.8.8

# Trace full resolution path (shows every server in chain)
dig target.com +trace

# TCP instead of UDP
dig target.com +tcp
```

**When to use dig over other tools:** When you want to verify exactly what a specific DNS server returns, or when debugging results from other tools. dig is the ground truth — it makes the raw DNS query, shows you the exact response, and nothing more.

---

### dnsrecon

The workhorse for structured DNS recon. Handles multiple scan types, outputs to JSON/CSV, and tries every NS server automatically.

```bash
# Full standard enumeration — the first command you run
dnsrecon -d target.com -t std

# Zone transfer attempt only
dnsrecon -d target.com -t axfr

# Brute-force subdomains with a wordlist
dnsrecon -d target.com -t brt -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --threads 10

# Reverse lookup an entire range
dnsrecon -r 185.45.12.0-185.45.12.255

# Google-based passive subdomain discovery
dnsrecon -d target.com -t goo

# Save output to JSON
dnsrecon -d target.com -t std -o results.json

# Verbose — see failed attempts too
dnsrecon -d target.com -t std -v
```

**Scan type reference:**

| `-t` value | What it runs |
|---|---|
| `std` | All record types + zone transfer attempt |
| `axfr` | Zone transfer only, all NS servers |
| `brt` | Subdomain brute-force (requires `-D wordlist`) |
| `goo` | Google subdomain discovery |
| `bing` | Bing subdomain discovery |
| `snoop` | DNS cache snooping |
| `tld` | TLD expansion (target.net, .org, .io, etc.) |
| `zonewalk` | DNSSEC zone walk via NSEC records |

---

### fierce

DNS recon tool that automatically chains zone transfer attempts with brute-force fallback. Unique feature: `--traverse` for near-IP discovery.

```bash
# Basic scan — zone transfer attempt, then brute-force
fierce --domain target.com

# Test specific subdomains only
fierce --domain target.com --subdomains admin vpn dev staging jenkins backup api

# Near-IP sweep — finds hosts adjacent to discovered IPs via PTR
fierce --domain target.com --subdomains mail --traverse 10

# Use custom DNS servers
fierce --domain target.com --dns-servers 8.8.8.8 1.1.1.1

# Add delay between queries (stealth)
fierce --domain target.com --delay 2

# HTTP connection attempt on found hosts
fierce --domain target.com --subdomains admin jenkins --connect
```

**The `--traverse` flag explained:**
When fierce finds `mail.target.com → 185.45.12.11`, `--traverse 10` runs PTR lookups on `185.45.12.1` through `185.45.12.21`. Companies cluster servers in sequential IPs. Near-IP sweeps find servers that have no subdomain in DNS but do have a PTR record — meaning they exist but were never publicly listed.

---

### dnsx

ProjectDiscovery's DNS toolkit. Best used as a pipeline stage — resolves large lists fast and filters clean results.

```bash
# Resolve a subdomain list and keep only live ones
dnsx -l subs.txt -a -resp -silent

# Enumerate multiple record types at once
dnsx -l subs.txt -a -cname -mx -txt -resp -silent

# Reverse PTR lookup on a range
prips 185.45.12.0/24 | dnsx -ptr -resp -silent

# Filter only domains that resolve (remove NXDOMAIN)
dnsx -l subs.txt -a -resp-only -silent > live-ips.txt

# JSON output for downstream processing
dnsx -l subs.txt -a -cname -txt -json -o dns-results.json

# Show CDN and ASN info
dnsx -l subs.txt -a -resp -cdn -asn -silent

# Wildcard detection (important before brute-force)
dnsx -d target.com -wc
```

**Pipeline use — the typical flow:**

```bash
subfinder -d target.com -silent | dnsx -a -resp -silent | tee live-resolved.txt
```

Subfinder outputs raw subdomain names. dnsx resolves each one — domains that don't exist return NXDOMAIN and are silently dropped. What lands in `live-resolved.txt` is a clean list of confirmed live hosts with IPs.

---

## Wildcard DNS — The Problem and How to Handle It

Some domains configure wildcard DNS: `*.target.com → 185.45.12.10`. This means *every* subdomain query returns an IP, even for subdomains that don't actually exist. Brute-force tools that don't handle wildcards will return thousands of false positives.

```bash
# Detect wildcard DNS
dig randomstringthatshouldnotexist123.target.com A

# If it returns an IP → wildcard is configured
# If it returns NXDOMAIN → no wildcard, brute-force results are clean
```

Tools that handle wildcards correctly: **puredns** (best), **dnsx** with `-wc` flag, **amass** (built-in detection).

```bash
# puredns handles wildcards automatically
puredns bruteforce /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt target.com
```

---

## Common Mistakes

**Running only against the first NS server.** Secondary servers are often misconfigured differently. Always try all of them, preferably with dnsrecon `-t axfr` which does this automatically.

**Querying `ANY` and trusting the result.** Many modern resolvers return minimal data for `ANY` queries as an anti-amplification measure. Query each record type individually for complete results.

**Ignoring AAAA records.** IPv6 addresses are frequently on different infrastructure with different security controls. An IPv6 address might bypass a WAF that only inspects IPv4 traffic.

**Forgetting SRV records.** SRV records directly name internal services. `_kerberos._tcp.target.com` gives you the domain controller hostname. `_ldap._tcp.target.com` gives you the LDAP server. These are not queried by most tools by default.

**Not doing reverse DNS on the full netblock.** Forward DNS (name → IP) only finds subdomains that someone configured a DNS record for. Reverse DNS (IP → name) finds everything with a PTR record, including servers that have no public subdomain — test environments, forgotten servers, decommissioned systems that still answer.

---

## Decision Logic

```
Start DNS recon
│
├─ Run dnsrecon -t std → get all record types
│
├─ Attempt zone transfer (dnsrecon -t axfr)
│   ├─ Success → you have the full zone, map everything, move to active scanning
│   └─ Failure → continue below
│
├─ Check TXT records for SPF/DMARC gaps → note for phishing potential
│
├─ Check MX records → identify mail provider
│   ├─ *.protection.outlook.com → Microsoft 365 → password spray opportunity
│   └─ *.google.com → Google Workspace
│
├─ Check SRV records → find internal services
│   └─ _kerberos, _ldap → Windows AD environment
│
├─ Get WHOIS netblock → run reverse DNS on full range
│   └─ dnsrecon -r [netrange]
│
└─ Brute-force subdomains
    └─ puredns or dnsrecon -t brt with seclists wordlist
```

---

*Next: [Subdomain Enumeration](08-subdomain-enumeration.md) — passive discovery at scale*
*Previous: [Active Recon](03-active-recon.md)*
