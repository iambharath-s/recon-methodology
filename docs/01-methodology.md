# Reconnaissance Methodology

This document covers the logic behind the recon workflow — why techniques happen in a specific order and how findings in one area unlock techniques in another.

---

## The Core Principle

Recon is not a checklist. It is a decision tree. What you find determines what you do next. A successful zone transfer means you skip subdomain brute-force entirely. Finding the target uses Microsoft 365 changes your phishing approach. Finding an old Apache version on a staging server changes your attack priority.

The workflow in this guide reflects how experienced pentesters actually work — not the linear order topics appear in textbooks.

---

## Reconnaissance Phases

### Phase 0 — Define Scope

Before any technique, answer these questions:

- What is in scope? (domains, IP ranges, cloud accounts)
- What is out of scope? (subsidiaries, third-party infrastructure)
- What is the engagement type? (black box / grey box / white box)
- Is there a detection risk? (stealth engagement vs. loud assessment)

Scope violations are a serious problem. If `shop.target.com` resolves to `target.myshopify.com`, Shopify's infrastructure is almost certainly out of scope.

---

### Phase 1 — Passive Reconnaissance

Zero contact with target systems. The goal is to build as complete a picture as possible before touching anything.

**What you are building:**
- Domain and IP inventory
- Employee names and email addresses
- Technology stack (from job postings, error messages, metadata)
- Existing breach data (credentials, internal documents)
- Org chart (who has privileged access)

**Minimum passive recon checklist:**

- [ ] WHOIS — domain registrant, name servers, netblock
- [ ] DNS — all record types via dig, check TXT for SPF/DMARC gaps
- [ ] Certificate transparency — crt.sh for subdomain history
- [ ] Passive subdomain discovery — subfinder + findomain + assetfinder
- [ ] Google dorking — exposed files, login portals, error messages
- [ ] Shodan/Censys — exposed services on known IP ranges
- [ ] Email harvesting — theHarvester with multiple sources
- [ ] LinkedIn — employee names, roles, tech stack from job postings
- [ ] Wayback Machine — deleted content, old endpoints
- [ ] Metadata from public documents — metagoofil

**Rule:** Do not move to Phase 2 until passive recon is complete. You will almost always find more passively than you expect.

---

### Phase 2 — Active Reconnaissance

Direct interaction with target systems. Every technique here creates log entries somewhere.

**What you are building on top of Phase 1:**
- Confirmed live hosts and services (validate passive findings)
- Full DNS zone (zone transfer)
- Complete subdomain map (brute-force to fill gaps)
- HTTP fingerprints (what software version, what CMS)
- Network path (traceroute — where are the firewalls)
- Reverse DNS of the full netblock (what is on IPs with no subdomain)

**Minimum active recon checklist:**

- [ ] DNS zone transfer — against all NS servers
- [ ] DNS brute-force — if zone transfer failed
- [ ] Reverse DNS — full netblock from WHOIS
- [ ] Traceroute — map network path, identify CDN/firewall
- [ ] HTTP probing — httpx against all resolved subdomains
- [ ] Tech fingerprinting — WhatWeb / httpx -tech-detect
- [ ] Content discovery — ffuf on high-value live hosts
- [ ] SMTP user enum — if mail server is in scope

---

### Phase 3 — Analysis and Hand-off

Collate everything. The output of recon is a structured attack surface, not raw command output.

**Deliverables:**
- List of all live hosts with IPs and services
- Technology versions mapped to CVEs (before you start exploitation)
- Email addresses formatted for credential attacks
- List of high-value targets with justification
- Network diagram showing infrastructure layout
- SPF/DMARC gaps if phishing is in scope

---

## How Findings Chain Together

This is the logic that separates experienced recon from running tools blindly:

```
WHOIS → netblock → reverse DNS → hosts labeled
     ↓
NS records → zone transfer → full subdomain list
     ↓                            ↓
SOA record → admin email      dev/staging subdomains
     ↓                            ↓
social engineering target     WhatWeb → old Apache version
                                         ↓
MX record → mail provider          CVE lookup → exploit
     ↓
O365 → password spray
     ↓
TXT SPF ~all + DMARC p=none
     ↓
Phishing as any @target.com address
```

Every piece of information creates a path to the next technique.

---

## Engagement-Type Playbooks

### Black Box (No prior knowledge, stealth preferred)

1. Passive recon only for the first session
2. Build domain, subdomain, email inventory passively
3. Switch to active only for DNS (zone transfer + brute-force)
4. HTTP probe the subdomain list — this is the lowest-noise active technique
5. Do targeted content discovery only on the highest-value hosts
6. Avoid broad port scans unless the engagement explicitly allows it

### Grey Box (Some information provided, standard engagement)

1. Start with provided scope to validate and expand
2. Run full passive + active recon in parallel
3. Standard subdomain + DNS + HTTP probing pipeline
4. Content discovery on all live web hosts
5. Light port scanning (top 1000 ports) on all resolved IPs

### White Box (Full information, no detection constraint)

1. Validate all provided information
2. Full port scan all in-scope IPs
3. Comprehensive subdomain brute-force
4. Aggressive content and parameter discovery
5. SMTP user enumeration
6. Full OSINT on all employees

---

## Tool Chain Overview

```
Target Domain
    │
    ├─ WHOIS ─────────────────────────────── netblock → RIR lookup
    │
    ├─ DNS Records (dig/dnsrecon) ────────── MX → mail provider
    │   │                                    TXT → SPF/DMARC gaps
    │   └─ Zone Transfer ─────────────────── full subdomain list (if works)
    │
    ├─ Passive Subdomain (subfinder/crt.sh) ─ subdomain list (always run)
    │
    ├─ Resolve + Filter (dnsx) ──────────── live hosts only
    │
    ├─ HTTP Probe (httpx) ───────────────── titles, status, tech, IPs
    │
    ├─ Fingerprint (WhatWeb) ────────────── exact versions
    │
    ├─ Content Discovery (ffuf) ─────────── hidden paths and files
    │
    ├─ Email (theHarvester) ─────────────── addresses for phishing/spray
    │
    └─ Network (traceroute) ─────────────── path, firewalls, CDN
```

---

## Passive vs Active Decision Table

| Technique | Type | Leaves logs on target? | Use first? |
|---|---|---|---|
| Google dorking | Passive | No | Yes |
| WHOIS lookup | Passive | No | Yes |
| crt.sh query | Passive | No | Yes |
| Subfinder | Passive | No | Yes |
| Shodan/Censys | Passive | No | Yes |
| theHarvester (search engines) | Passive | No | Yes |
| Wayback Machine | Passive | No | Yes |
| DNS record queries | Active | Yes (DNS resolver logs) | Yes — low risk |
| Zone transfer attempt | Active | Yes (NS server logs) | Yes — low noise |
| Subdomain brute-force | Active | Yes (DNS resolver logs) | After passive |
| Reverse DNS lookup | Active | Yes (DNS server logs) | After WHOIS |
| HTTP probing (httpx) | Active | Yes (web server logs) | After DNS |
| Traceroute | Active | Yes (each hop logs TTL expire) | When needed |
| Port scanning | Active | Yes (firewall, IDS) | Scanning phase, not recon |
| Content discovery | Active | Yes (web server logs) | Targeted, not blanket |

---

*Next: [Passive Reconnaissance](02-passive-recon.md)*
