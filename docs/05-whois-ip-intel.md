# WHOIS and IP Intelligence

WHOIS is one of the first techniques you run on any target. The data is public, the query is passive, and it directly reveals the IP ranges you need for every subsequent network-level technique.

---

## Domain WHOIS

```bash
whois target.com
```

### Key Fields and What to Do With Them

| Field | Example value | What to do with it |
|---|---|---|
| Registrar | GoDaddy Inc. | Potential registrar-level attack vector |
| Registrant email | admin@target.com | Spear phishing target, confirms email format |
| Admin email | it-admin@target.com | IT contact — DNS change social engineering |
| Name servers | ns1.target.com, ns2.target.com | Zone transfer targets |
| Created date | 2008-03-14 | Domain age — older = more legacy systems |
| Expiry date | 2025-03-14 | Domain hijack if they miss renewal |
| Updated date | 2024-01-10 | Recent DNS/registrar activity |

**Extract just the important fields:**

```bash
whois target.com | grep -iE "registrar|name server|admin email|tech email|expir|creat|registrant"
```

### WHOIS Models (CEH exam topic)

| Model | How it works | Used by |
|---|---|---|
| **Thick** | Complete registrant data in one central database | .com, .net (Verisign stores everything) |
| **Thin** | Central database stores only the registrar — you must query the registrar separately | .org and some ccTLDs |
| **Decentralized** | Multiple independent registrars each store complete data | Newer gTLDs |

**Protocol:** WHOIS runs on TCP port 43.

---

## IP WHOIS and Netblock Discovery

### Finding the Netblock

```bash
# Step 1 — get the IP
dig target.com A +short
# → 185.45.12.10

# Step 2 — query WHOIS for that IP
whois 185.45.12.10

# Step 3 — extract the range
whois 185.45.12.10 | grep -E "NetRange|CIDR|OrgName|OrgId"
```

**Example output:**

```
NetRange:     185.45.12.0 - 185.45.12.255
CIDR:         185.45.12.0/24
OrgName:      TargetCorp Inc.
OrgId:        TC-1234
NetType:      Direct Allocation
RegDate:      2018-06-14
```

The netblock is `185.45.12.0/24` — 256 IPs all belonging to TargetCorp.

### Finding All Blocks Owned by the Same Org

Large companies often have multiple IP allocations. Once you have the OrgId:

```bash
# ARIN (North America)
whois -h whois.arin.net "o TC-1234"

# This returns every netblock registered to that organization
# Look for multiple NetRange entries
```

### CIDR Size Reference

| CIDR | IPs in range | Typical org size |
|---|---|---|
| `/32` | 1 | Single host |
| `/29` | 8 | Very small allocation |
| `/28` | 16 | Small |
| `/27` | 32 | Small |
| `/26` | 64 | Small-medium |
| `/25` | 128 | Medium |
| `/24` | 256 | Common for mid-size companies |
| `/23` | 512 | Medium-large |
| `/22` | 1,024 | Large |
| `/21` | 2,048 | Large |
| `/20` | 4,096 | Enterprise |
| `/16` | 65,536 | ISP-scale |

---

## Regional Internet Registries

Each RIR manages IP allocations for its region. When you run `whois` on an IP, the query automatically routes to the correct RIR.

| RIR | Region | Web portal |
|---|---|---|
| ARIN | North America, parts of Caribbean and sub-Saharan Africa | arin.net |
| RIPE NCC | Europe, Middle East, Central Asia | ripe.net |
| APNIC | Asia-Pacific | apnic.net |
| AFRINIC | Africa | afrinic.net |
| LACNIC | Latin America, Caribbean | lacnic.net |

**Direct queries:**

```bash
whois -h whois.arin.net 185.45.12.10     # force ARIN query
whois -h whois.ripe.net 193.0.0.1        # RIPE query
whois -h whois.apnic.net 202.12.29.0     # APNIC query
```

---

## IP Geolocation

Geolocation maps an IP to a physical location and ISP — without sending a single packet to the target.

```bash
# curl-based geolocation
curl ipinfo.io/185.45.12.10
curl ip-api.com/json/185.45.12.10

# CLI tool
pip3 install ipinfo
ipinfo 185.45.12.10
```

**What geolocation reveals:**

| Data | Recon value |
|---|---|
| City / region | Physical office location — dumpster diving, physical access |
| ISP | On-premises hosting vs. cloud vs. colocation |
| Organization | Confirm the IP belongs to the right target |
| Timezone | Know what hours are "off-hours" for the target's team |
| ASN | Find all IPs in the same autonomous system |

**ASN-based discovery** — once you have the ASN, you can find all IP ranges announced by that autonomous system:

```bash
# Find ASN from IP
curl ipinfo.io/185.45.12.10 | grep "org"
# → "AS12345 TargetCorp Inc."

# Find all prefixes for that ASN
curl https://api.bgpview.io/asn/12345/prefixes | jq '.data.ipv4_prefixes[].prefix'
```

This finds IP ranges that may not appear in WHOIS directly — cloud-hosted assets, CDN ranges, and secondary allocations through ISPs.

---

## Putting It Together — Full WHOIS and IP Recon

```bash
TARGET="target.com"

# 1. Domain WHOIS
echo "=== DOMAIN WHOIS ===" && whois $TARGET | grep -iE "registrar|name server|admin|expir|creat"

# 2. Resolve IP
IP=$(dig $TARGET A +short | head -1)
echo "IP: $IP"

# 3. IP WHOIS → netblock
echo "=== NETBLOCK ===" && whois $IP | grep -E "NetRange|CIDR|OrgName|OrgId"

# 4. ASN → all prefixes
ASN=$(curl -s ipinfo.io/$IP | jq -r '.org' | awk '{print $1}' | tr -d 'AS')
echo "ASN: AS$ASN"
curl -s "https://api.bgpview.io/asn/$ASN/prefixes" | jq '.data.ipv4_prefixes[].prefix'

# 5. Reverse DNS on the netblock
CIDR=$(whois $IP | grep CIDR | awk '{print $2}' | head -1)
nmap -sL $CIDR | grep "report for"
```

---

*Next: [Search Engine Recon and Google Dorking](06-search-engine-recon.md)*
*Related: [DNS Enumeration](04-dns-enumeration.md) · [Network Footprinting](09-network-footprinting.md)*
