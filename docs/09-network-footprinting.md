# Network Footprinting

Network footprinting maps the infrastructure between you and the target — routers, firewalls, CDNs, load balancers, and ISP peering points. It tells you what security controls exist, where they sit, and what the actual network layout looks like before you start attacking services.

---

## Traceroute — How It Works

Traceroute exploits the TTL (Time To Live) field in IP packets. Every packet has a TTL counter — each router it passes through decrements it by 1. When TTL hits 0, the router discards the packet and sends an ICMP "Time Exceeded" message back to the sender, including the router's own IP address.

Traceroute sends packets with incrementing TTL values:
- TTL=1 → first router responds → you learn hop 1's IP
- TTL=2 → second router responds → you learn hop 2's IP
- TTL=N → target responds → path complete

```
Your machine (TTL=1)  →  Router 1 drops it, responds  →  Hop 1 IP known
Your machine (TTL=2)  →  Router 1 passes it, Router 2 drops it  →  Hop 2 IP known
...continues until destination responds
```

---

## Traceroute Variants

Three protocols, each with different behavior against firewalls.

| Variant | Default on | Protocol | Port | Firewall evasion |
|---|---|---|---|---|
| ICMP | Windows (`tracert`) | ICMP Echo Request | — | Poor — ICMP often blocked |
| UDP | Linux (`traceroute`) | UDP | 33434+ | Medium — UDP sometimes filtered |
| TCP | Manual (`tcptraceroute`) | TCP SYN | 80 or 443 | Best — web ports rarely blocked |

```bash
# Linux — UDP default
traceroute target.com

# Windows — ICMP default
tracert target.com

# TCP on port 443 — best firewall bypass
sudo tcptraceroute target.com 443
sudo tcptraceroute target.com 80

# Nmap traceroute — TCP SYN based
nmap --traceroute -sn target.com

# hping3 — custom TTL, full control
sudo hping3 -S -p 443 --traceroute target.com   # TCP SYN traceroute
sudo hping3 --icmp --traceroute target.com       # ICMP traceroute
```

---

## Reading Traceroute Output

```
traceroute to target.com (185.45.12.10)

 1  192.168.1.1      2.1 ms    ← your local router
 2  10.0.0.1         8.4 ms    ← ISP gateway (RFC1918 — your ISP's internal)
 3  203.0.113.1     14.2 ms    ← ISP backbone
 4  *  *  *                    ← firewall drops TTL-exceeded ICMP
 5  185.45.0.1      44.1 ms    ← ISP peering / upstream transit
 6  185.45.12.1     47.3 ms    ← target's edge router
 7  185.45.12.10    49.1 ms    ← destination — web server
```

**Pattern analysis:**

| What you see | What it means | Implication |
|---|---|---|
| `* * *` | Router drops ICMP TTL-exceeded — does not respond | Security device or filter present at this hop |
| RTT spikes (e.g. 10ms → 150ms) | Geographic jump — different country/continent | Traffic crosses an ocean or long-haul link |
| Hop goes RFC1918 → public | You crossed into target's internal network boundary | Internal router identified |
| CDN hostname in hop | Target uses a CDN — Cloudflare, Fastly, Akamai | Real origin IP is hidden behind CDN |
| Very few hops (3-5) | Target is close to internet exchange or uses CDN PoP | CDN node nearby — real server elsewhere |
| Same RTT for multiple hops | All within same data center or network segment | Servers colocated |
| Hop 6 = edge router, Hop 7 = server (1ms gap) | Web server directly behind edge router, no internal firewall | Misconfig — no defense-in-depth |

---

## When ICMP Is Blocked — Switching to TCP

A common scenario: standard traceroute shows `* * *` after a few hops, making the network appear opaque.

```bash
# Problem — ICMP blocked
traceroute target.com
 3  203.0.113.1    14ms
 4  *  *  *               ← blocked from here
 5  *  *  *
 6  *  *  *

# Solution — TCP on port 443 (HTTPS must pass through firewall)
sudo tcptraceroute target.com 443
 3  203.0.113.1    14ms
 4  185.45.0.1     44ms   ← visible now
 5  185.45.12.1    47ms   ← edge router
 6  185.45.12.10   49ms   ← web server
```

Port 443 traffic must reach the web server — the firewall has to let it through. TCP traceroute rides that permitted path and exposes every hop along the way.

---

## Multi-Path Analysis

Running traceroutes to multiple IPs in the same target range reveals the network topology.

```bash
# Traceroute to three different hosts in target's range
traceroute 185.45.12.10    # www
traceroute 185.45.12.11    # mail
traceroute 185.45.12.80    # admin
```

If all three share the same last few hops:
```
 5  185.45.0.1     44ms   ← same for all three
 6  185.45.12.1    47ms   ← same for all three (gateway)
 7  185.45.12.X    49ms   ← each goes to different final host
```

Hop 6 (`185.45.12.1`) is the internal router — single point through which all target traffic flows. If you can reach that IP, you can potentially reach everything behind it.

---

## BGP and ASN Intelligence

BGP (Border Gateway Protocol) routes traffic between autonomous systems on the internet. Each organization with its own IP space has an ASN (Autonomous System Number). Querying BGP data reveals all IP ranges an organization controls.

```bash
# Find ASN from an IP
curl -s ipinfo.io/185.45.12.10 | jq '{org, asn}'
# → "org": "AS12345 TargetCorp Inc."

# All prefixes announced by that ASN
curl -s "https://api.bgpview.io/asn/12345/prefixes" \
  | jq '.data.ipv4_prefixes[].prefix'

# Via BGP toolkit
whois -h whois.radb.net "!gas12345"  # all routes for ASN

# BGP looking glass — real-time routing table
# bgp.he.net — search by IP or ASN
# bgpview.io  — clean web interface
```

**Why this matters:** A company may have multiple IP blocks from different RIRs, registered at different times. WHOIS on one IP gives you one block. BGP gives you the complete picture — every prefix the organization announces to the internet.

---

## Network Topology Mapping

Combine traceroute data with DNS and WHOIS to build a network diagram.

```bash
# Step 1 — get all live hosts
dnsrecon -r 185.45.12.0-185.45.12.255 | grep PTR > hosts.txt

# Step 2 — traceroute to each, note last 3 hops
for ip in $(awk '{print $3}' hosts.txt); do
    echo "=== $ip ==="
    traceroute -m 15 $ip | tail -5
done

# Step 3 — identify common hops (routers, firewalls)
# The hop that appears in ALL traceroutes is the default gateway
# Hops that appear in SOME traceroutes are segment routers

# Step 4 — infer topology
# Same last-hop IP for www and mail → on same network segment
# Different last-hop for admin panel → different segment, possibly VLAN
```

---

## CDN and Load Balancer Detection

When `dig target.com A` returns multiple IPs or a CDN IP, you are not talking to the origin server.

```bash
# Check if multiple IPs are returned (load balancing)
dig target.com A
# If 3+ IPs returned → load balanced

# Check if IP belongs to a CDN
curl -s ipinfo.io/185.45.12.10 | grep org
# → "org": "AS13335 Cloudflare, Inc."  ← CDN, not origin

# Check for CDN headers
curl -I https://target.com | grep -iE "CF-Ray|X-Cache|X-Varnish|X-Amz|Fastly|Via"
# CF-Ray present → Cloudflare
# X-Amz-Cf-Id → AWS CloudFront
# X-Varnish → Varnish cache

# Try to find origin IP behind CDN
# 1. Historical DNS — SecurityTrails, ViewDNS history
#    If they added Cloudflare recently, old A records show real IP
# 2. Check MX record — often points to real IP, not CDN
# 3. Check subdomains — direct.target.com, origin.target.com sometimes skip CDN
# 4. TLS certificate SAN — may include internal hostnames
# 5. Shodan — searches historical data, may have pre-CDN IP
```

---

## Tools Summary

### traceroute / tracert
Built-in. UDP on Linux, ICMP on Windows. First thing to run.

```bash
traceroute target.com           # Linux
tracert target.com              # Windows
traceroute -I target.com        # Force ICMP on Linux
traceroute -T -p 443 target.com # TCP port 443 on Linux
```

### tcptraceroute
TCP-based traceroute. Install separately. Best for firewall bypass.

```bash
sudo apt install tcptraceroute
sudo tcptraceroute target.com 443
sudo tcptraceroute target.com 80
```

### hping3
Full-control packet crafting. Supports ICMP, TCP, UDP traceroute with custom flags.

```bash
sudo apt install hping3

# TCP SYN traceroute to port 443
sudo hping3 -S -p 443 --traceroute -V target.com

# ICMP traceroute
sudo hping3 --icmp --traceroute -V target.com

# Test if a specific port is reachable
sudo hping3 -S -p 443 -c 3 target.com
```

### Nmap (network mapping)
Beyond port scanning — list scan for reverse DNS, traceroute, and topology inference.

```bash
# List scan — pure DNS, no port scanning, completely passive
nmap -sL 185.45.12.0/24

# Traceroute without port scan
nmap --traceroute -sn target.com

# Find live hosts in range (ping sweep)
nmap -sn 185.45.12.0/24
```

### PingPlotter
GUI tool for continuous traceroute analysis. Shows packet loss and latency over time. Useful for identifying intermittent filtering. Available for Windows and macOS.

---

## Findings and What They Mean for Attacks

| Network finding | Attack implication |
|---|---|
| No hops between edge router and server | No internal firewall — network layer attacks possible |
| CDN in front of target | WAF likely active — look for IPv6 bypass, direct IP access |
| `* * *` at specific hop | Firewall present — switch to TCP traceroute for visibility |
| Multiple IPs for same domain | Load balanced — attacks must reach all nodes |
| RFC1918 IP appears in traceroute | You can see internal addressing scheme |
| Same last-hop for all servers | Single gateway — if compromised, reaches all internal hosts |
| Very low hop count | Server is close to internet — possibly colocated or cloud-hosted |

---

## OPSEC Notes

Traceroute is active — every router in the path sees your TTL-expired packets and sends responses back. The target's firewall logs will show ICMP, UDP, or TCP connections from your IP.

Mitigations:
- Use TCP traceroute on 443 — blends with normal HTTPS traffic
- Route through VPN or proxy — hides your real IP
- Limit to one traceroute per target IP — repeated traceroutes look suspicious

---

*Previous: [Subdomain Enumeration](08-subdomain-enumeration.md)*
*Next: [Website Enumeration](10-website-enumeration.md)*
