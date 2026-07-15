# DNS Records — Recon Reference

Every record type, what it reveals, and how to query it.

---

## Record Type Reference

| Record | Full Name | Query | What an Attacker Extracts |
|---|---|---|---|
| `A` | Address | `dig target.com A` | IPv4 — primary server IPs, scan targets |
| `AAAA` | IPv6 Address | `dig target.com AAAA` | IPv6 — often bypasses WAF/CDN |
| `MX` | Mail Exchange | `dig target.com MX` | Mail provider, phishing/spoofing vector |
| `NS` | Name Server | `dig target.com NS` | Authoritative servers → zone transfer targets |
| `CNAME` | Canonical Name | `dig sub.target.com CNAME` | Third-party services, subdomain takeover candidates |
| `SOA` | Start of Authority | `dig target.com SOA` | Admin email, zone serial, DNS server info |
| `TXT` | Text | `dig target.com TXT` | SPF/DMARC gaps, service verification tokens |
| `SRV` | Service | `dig _kerberos._tcp.target.com SRV` | Internal services — AD, LDAP, VoIP |
| `PTR` | Pointer | `dig -x 185.45.12.10` | Hostname from IP — labels a netblock |
| `HINFO` | Host Info | `dig target.com HINFO` | CPU type, OS (rare but valuable if set) |
| `RP` | Responsible Person | `dig target.com RP` | Admin contact details |
| `CAA` | CA Authorization | `dig target.com CAA` | Which CAs can issue certs — reveals cert provider |

---

## MX Record Analysis

```bash
dig target.com MX +short
```

| MX value | Email provider | Attack implication |
|---|---|---|
| `*.protection.outlook.com` | Microsoft 365 | Password spray at login.microsoftonline.com |
| `*.google.com` | Google Workspace | Credential attack at accounts.google.com |
| `*.mimecast.com` | Mimecast | Email security gateway — bypass or target |
| `*.proofpoint.com` | Proofpoint | Email gateway — bypass or target |
| `mail.target.com` | Self-hosted | Attack the mail server directly |
| `*.sendgrid.net` | SendGrid | Bulk email platform — not primary mail |

**Priority numbers:** Lower = higher priority. `10 mail.target.com`, `20 backup-mail.target.com` — backup mail servers are almost always less hardened than primary.

---

## TXT Record Analysis

```bash
dig target.com TXT +short
```

### SPF Record Breakdown

| SPF value | Meaning | Attack implication |
|---|---|---|
| `v=spf1 ... ~all` | Softfail — unlisted servers suspicious but accepted | Spoofed emails land in inbox |
| `v=spf1 ... -all` | Hardfail — unlisted servers rejected | Spoofing blocked |
| `v=spf1 +all` | Pass all — any server can send | Full spoofing capability |
| No SPF record | No policy | No enforcement, spoofing trivial |
| `include:sendgrid.net` | Uses SendGrid | Third-party email platform |
| `include:amazonses.com` | Uses AWS SES | AWS email infrastructure |
| `ip4:185.45.12.11` | Direct IP in SPF | Mail server IP directly revealed |

### DMARC Record Breakdown

```
v=DMARC1; p=none; rua=mailto:dmarc@target.com
```

| DMARC policy | Enforcement | Spoofing impact |
|---|---|---|
| `p=none` | Monitor only | Zero enforcement — spoofed email delivered |
| `p=quarantine` | Route to spam | Spoofed email goes to spam |
| `p=reject` | Hard reject | Spoofed email blocked at server |

**Most common finding:** `p=none` with SPF `~all` = no enforcement whatsoever. Email spoofing will succeed against any recipient.

### Service Verification Tokens

| TXT value | What it confirms |
|---|---|
| `MS=ms12345678` | Microsoft 365 / Azure AD |
| `google-site-verification=...` | Google Workspace / Search Console |
| `facebook-domain-verification=...` | Facebook Business account |
| `docusign=...` | DocuSign in use |
| `atlassian-domain-verification=...` | Atlassian (Jira/Confluence) |
| `stripe-verification=...` | Stripe payment processing |
| `_github-challenge-...` | GitHub organization account |

Each token reveals a SaaS platform in use — each is a potential account to target.

---

## SRV Record Analysis

SRV records directly name internal services. Often overlooked but high value.

```bash
# Query patterns
dig _ldap._tcp.target.com SRV          # LDAP / Active Directory
dig _kerberos._tcp.target.com SRV      # Kerberos / Domain Controller
dig _kpasswd._tcp.target.com SRV       # Kerberos password change
dig _gc._tcp.target.com SRV            # Global Catalog (AD)
dig _sip._tcp.target.com SRV           # SIP / VoIP
dig _xmpp-server._tcp.target.com SRV  # XMPP chat
dig _autodiscover._tcp.target.com SRV  # Outlook autodiscover (Exchange)
dig _dmarc.target.com TXT              # DMARC policy
```

| SRV record | Service | Attack value |
|---|---|---|
| `_ldap._tcp` → port 389 | Active Directory LDAP | Confirms Windows AD, LDAP enumeration target |
| `_kerberos._tcp` → port 88 | Domain Controller | DC hostname and IP directly revealed |
| `_gc._tcp` → port 3268 | AD Global Catalog | Forest-level DC identified |
| `_sip._tcp` → port 5060 | VoIP | Eavesdropping, toll fraud |
| `_autodiscover._tcp` → port 443 | Exchange/O365 | Confirms mail infrastructure |

---

## CNAME Subdomain Takeover Candidates

When a CNAME points to an external service that the company no longer uses, an attacker can register that service and take over the subdomain.

```bash
dnsx -l subs.txt -cname -resp -silent
```

| CNAME destination | Service | Takeover possible? |
|---|---|---|
| `*.github.io` | GitHub Pages | Yes — claim the repo/username |
| `*.s3.amazonaws.com` | AWS S3 | Yes — claim the bucket name |
| `*.azurewebsites.net` | Azure App Service | Yes — claim the app name |
| `*.github.com` | GitHub | Yes — via custom domain in repo settings |
| `*.herokuapp.com` | Heroku | Yes — claim the app name |
| `*.netlify.app` | Netlify | Yes — claim the site name |
| `*.ghost.io` | Ghost CMS | Yes — create the account |
| `*.greenhouse.io` | Greenhouse (ATS) | Yes — contact Greenhouse to claim |
| `*.zendesk.com` | Zendesk | Yes — claim the subdomain |
| `*.myshopify.com` | Shopify | Yes — create the store |

**To confirm takeover opportunity:** Try to access the CNAME destination — if it returns a 404 or "not found" page from the third-party service (not from the company's own 404 page), the service is unclaimed.

---

## Zone Transfer Quick Reference

```bash
# Manual — against specific NS
dig axfr target.com @ns1.target.com
dig axfr target.com @ns2.target.com

# Automated — tries all NS
dnsrecon -d target.com -t axfr

# With fierce — tries AXFR then falls back to brute-force
fierce --domain target.com

# Via nslookup (Windows)
nslookup
> server ns1.target.com
> set type=any
> ls -d target.com
```

**Refused response:**
```
; Transfer failed.
; AXFR query to ns1.target.com failed: REFUSED
```
→ Server is correctly locked down. Try the next NS server — secondary servers are often misconfigured independently.

**Successful response** gives you every record in the zone including internal IPs, hidden subdomains, and infrastructure details that were never meant to be public.

---

*Back to [Cheat Sheets](recon-cheatsheet.md) | Full DNS guide: [docs/04-dns-enumeration.md](../docs/04-dns-enumeration.md)*
