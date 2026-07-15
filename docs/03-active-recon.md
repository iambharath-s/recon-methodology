# Active Reconnaissance

Active recon means sending packets directly to target systems. Every technique here creates log entries somewhere — on the target's firewall, DNS server, web server, or IDS. Do passive recon first. Switch to active only when you have enough context to know what you are looking for.

---

## Before You Start

Ask these questions before running active recon:

- **Is this in scope?** Active scanning against out-of-scope systems is a contract violation and potentially illegal.
- **What is the detection risk tolerance?** A stealth engagement requires slow, rate-limited scans. A time-boxed assessment can go loud.
- **Do you have a starting IP or a domain?** The starting point determines which technique comes first.

---

## DNS Active Techniques

### Zone Transfer

The highest-value active DNS technique. If it works, you get the entire infrastructure map in one query.

```bash
# Get name servers first
dig target.com NS +short

# Attempt zone transfer against each
dig axfr target.com @ns1.target.com
dig axfr target.com @ns2.target.com

# Automate — tries all NS servers
dnsrecon -d target.com -t axfr
```

### DNS Brute-Force

When zone transfer fails, systematically guess subdomain names.

```bash
# dnsrecon with SecLists wordlist
dnsrecon -d target.com -t brt \
  -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  --threads 10

# puredns — handles wildcard DNS correctly, fastest option
puredns bruteforce \
  /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  target.com

# fierce — falls back to brute-force automatically after failed zone transfer
fierce --domain target.com
```

**Wildcard check before brute-forcing:**

```bash
dig randomstringxyz123abc.target.com A
```

If this resolves → wildcard DNS is set, brute-force will return false positives. Use puredns which handles this automatically.

### Reverse DNS on Netblock

Turn an IP range into a labeled infrastructure map.

```bash
# Get netblock from WHOIS
whois $(dig target.com A +short | head -1) | grep -E "NetRange|CIDR"

# Run reverse DNS across the full range
dnsrecon -r 185.45.12.0-185.45.12.255

# Faster — dnsx with prips
prips 185.45.12.0/24 | dnsx -ptr -resp -silent

# Nmap list scan — pure DNS, no port scanning
nmap -sL 185.45.12.0/24 | grep "report for"
```

---

## Network Footprinting

### Traceroute

Maps the network path to the target. Reveals routers, firewalls, ISPs, CDNs, and sometimes internal network structure.

```bash
# Linux — UDP by default
traceroute target.com

# Windows — ICMP by default
tracert target.com

# TCP traceroute — bypasses ICMP-blocking firewalls
sudo tcptraceroute target.com 443
sudo tcptraceroute target.com 80

# Nmap traceroute (TCP SYN based)
nmap --traceroute -sn target.com
```

**Reading traceroute output:**

```
1   192.168.1.1      2ms     ← your local router
2   10.0.0.1         8ms     ← ISP edge
3   203.0.113.1     15ms     ← ISP backbone
4   *  *  *                  ← firewall — drops ICMP TTL exceeded
5   185.45.0.1      44ms     ← ISP peering point
6   185.45.12.1     47ms     ← target's edge router
7   185.45.12.10    49ms     ← destination (web server)
```

**What each pattern means:**

| Pattern | Meaning |
|---|---|
| `* * *` | Firewall or router dropping ICMP — security device present |
| RTT doubles between hops | Geographic jump — different cities or data centers |
| Hop goes from public to RFC1918 IP | Entered the target's internal network |
| Last hop is a CDN hostname | Target is behind a CDN — real origin IP hidden |
| Very few hops (3-4) | Target is at ISP edge or using a CDN PoP |

**When ICMP is blocked — switch to TCP:**

```bash
# ICMP traceroute shows nothing after hop 4
traceroute target.com
# 4  *  *  *
# 5  *  *  *

# TCP on port 443 works because firewall must allow HTTPS
sudo tcptraceroute target.com 443
# 4  185.45.0.1    44ms   ← now visible
# 5  185.45.12.1   47ms
# 6  185.45.12.10  49ms   ← target
```

---

## Web and Service Fingerprinting

### httpx — HTTP Probing at Scale

```bash
# Probe live hosts from a subdomain list
httpx -l subs.txt -title -sc -tech-detect -server -ip -silent

# Filter to only 200-OK responses
httpx -l subs.txt -sc -mc 200 -silent -o live-200.txt

# Take screenshots
httpx -l subs.txt -screenshot -o screenshots/

# Full recon pass
httpx -l subs.txt -title -sc -tech-detect -server -ip -cname -json -o http-results.json
```

**Reading httpx output:**

```
https://admin.target.com [200] [Admin Panel] [Django 4.2,Python] [gunicorn/21.2] [185.45.12.80]
```

Left to right: URL, status code, page title, detected technologies, server header, IP. Each bracketed segment corresponds to a flag passed.

### WhatWeb — Technology Fingerprinting

```bash
# Single target
whatweb https://target.com

# Aggressive — more requests, more info
whatweb -a 3 https://target.com

# Scan multiple targets from a file
whatweb -i hosts.txt --log-json=results.json

# Quiet output — only important findings
whatweb -q https://target.com
```

### Banner Grabbing with nc and curl

Sometimes you just want the raw HTTP headers.

```bash
# HTTP headers only
curl -I https://target.com

# Follow redirects and show headers
curl -IL https://target.com

# Raw connection — see the exact server banner
nc target.com 80
HEAD / HTTP/1.0
[press enter twice]

# HTTPS banner
openssl s_client -connect target.com:443 </dev/null 2>/dev/null | head -20
```

---

## Subdomain Brute-Force

When passive discovery is exhausted, brute-force fills the gaps.

```bash
# ffuf — fastest option
ffuf -u https://FUZZ.target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fc 404 -t 50

# gobuster DNS mode
gobuster dns -d target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 50

# amass enum — most thorough (slow)
amass enum -d target.com -brute -w /usr/share/seclists/Discovery/DNS/deepmagic.com-prefixes-top500.txt
```

**Wordlist recommendation — SecLists:**

```bash
# Install SecLists
sudo apt install seclists
# Location: /usr/share/seclists/Discovery/DNS/

# Good starting wordlists (in order of preference):
# subdomains-top1million-5000.txt   — fast, good coverage
# subdomains-top1million-20000.txt  — thorough
# deepmagic.com-prefixes-top500.txt — targeted at common prefixes
# dns-Jhaddix.txt                   — comprehensive, bug bounty focused
```

---

## Directory and Content Discovery

Once you have live web hosts, discover hidden paths.

```bash
# ffuf — most flexible
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -fc 404 -t 40

# feroxbuster — recursive, follows into found directories
feroxbuster -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  --depth 3 -t 40

# gobuster dir mode
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 40 -x php,html,txt,js
```

---

## Email Active Techniques

### SMTP User Enumeration

If a mail server is exposed, you can verify email address existence directly.

```bash
# Manual — VRFY command
nc mail.target.com 25
EHLO test.com
VRFY john.smith
# 250 2.1.5 john.smith@target.com   ← user exists
# 550 5.1.1 unknown user            ← does not exist

# RCPT TO method (more reliable — VRFY often disabled)
MAIL FROM: test@test.com
RCPT TO: john.smith@target.com
# 250 OK                   ← address accepted (user likely exists)
# 550 No such user         ← does not exist
```

**Note:** Many mail servers have disabled VRFY. RCPT TO is more reliable but depends on server configuration. Modern mail security (Office 365, Google Workspace) returns 250 for all addresses to prevent enumeration.

---

## Active Recon OPSEC

Active recon creates logs. These measures reduce your footprint:

**Slow down scans:** Most IDS/IPS systems detect scans by volume and rate. Adding delays makes scans look more like legitimate traffic.

```bash
# dnsrecon — add sleep between queries
dnsrecon -d target.com -t brt -D wordlist.txt --lifetime 5

# ffuf — rate limit requests
ffuf -u https://target.com/FUZZ -w wordlist.txt -rate 10

# nmap — slow scan timing
nmap -T2 target.com
```

**Rotate source IPs for large-scale scanning:** Use multiple VPS nodes or a cloud scanning service like Axiom. Fingerprinting by source IP is the most common detection method.

**Use expected protocols:** Scanning port 443 with TCP SYN from a residential IP looks like normal HTTPS traffic. Sending ICMP pings to every IP in a /24 looks like a scan.

**Avoid scanning during business hours where detection matters:** Security teams are not watching dashboards at 3am on a Sunday. This is not always a valid consideration (some engagements have time windows) but worth noting.

---

## Common Mistakes

**Starting active before finishing passive.** Active recon alerts defenders and narrows your opportunity. Exhaust passive sources first — you often find more with less risk.

**Not checking for CDN before scanning.** If the target uses Cloudflare, direct IP scans hit Cloudflare's infrastructure, not the target's. Check whether the A record points to a CDN range before assuming you are talking to the target's own server.

**Brute-forcing without wildcard detection.** If `*.target.com` resolves to something, every subdomain in your wordlist will appear to "exist." Check for wildcards first.

**Forgetting IPv6.** You scan the IPv4 address, hit a WAF, and move on. The AAAA record for the same host points directly to the origin without WAF protection. Always check for AAAA records.

---

*Previous: [Passive Reconnaissance](02-passive-recon.md)*
*Next: [DNS Enumeration](04-dns-enumeration.md)*
