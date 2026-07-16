# OSINT and Social Media Intelligence

Open-source intelligence from social platforms, people search services, and public databases is some of the most actionable recon data available. It reveals the human layer — who has access to what, what technologies they use, and how to convincingly impersonate or target them.

---

## LinkedIn — The Most Valuable Passive Source

LinkedIn is an accidental threat intelligence platform. Employees publicly document their roles, tools, certifications, and project history. Security teams rarely audit what their staff post there.

### What to Extract

**From employee profiles:**

| Profile section | What it reveals | How to use it |
|---|---|---|
| Job title | Role and likely system access level | Target sysadmins and network engineers first |
| Skills and endorsements | Confirmed technologies they work with | Validates tech stack from other recon |
| Certifications | Security posture (CISSP, CISM = mature sec team) | Gauge how sophisticated defenses are |
| Work history | Previous employers and roles | Supply chain — maybe their old company is a vendor |
| Connections | Org chart clues | Who reports to whom |
| Activity / posts | Current projects, tools being deployed | What new tech is being implemented right now |

**From job postings:**

Job postings are the most honest documentation a company produces about its infrastructure. They are written for technical candidates and therefore contain specific, accurate details.

```
Senior Network Engineer — TargetCorp
Requirements:
• 5+ years with Palo Alto firewalls (PCNSE preferred)
• Experience with Cisco Catalyst 9000 series
• Proficiency in VMware NSX-T
• AWS VPC and Transit Gateway experience
• ServiceNow ITSM administration
• Familiarity with Splunk SIEM
```

From one job posting you now know: firewall vendor and model series, switch platform, network virtualization stack, cloud provider and specific services, ITSM platform, and SIEM solution. This is the complete security architecture without scanning a single port.

Search: `site:linkedin.com/jobs "TargetCorp"`, `site:indeed.com "TargetCorp" network engineer`

### LinkedIn Recon Tools

```bash
# theHarvester — LinkedIn source
theHarvester -d target.com -b linkedin -l 500

# CrossLinked — scrapes employee names, generates email combinations
pip3 install crosslinked
crosslinked -f '{first}.{last}@target.com' "Target Corp"
crosslinked -f '{f}{last}@target.com' "Target Corp"    # alternate format

# Output: list of email addresses based on every employee name found
```

---

## Twitter / X and Other Social Platforms

**What to look for:**

- Employees posting about outages → reveals infrastructure components ("our AWS us-east-1 instance is down")
- Developers sharing code snippets → sometimes with API keys or config details
- IT staff complaining about tools → reveals what they are switching from and to
- Security team announcing new controls → tells you what just got hardened

**Search operators:**

```
from:@employee_handle "vpn"
from:@employee_handle "server" OR "database"
site:twitter.com "TargetCorp" "outage"
site:twitter.com "TargetCorp" "firewall" OR "palo alto"
```

---

## People Search Services

Useful for building profiles on specific high-value targets — executives, IT administrators, security staff.

| Service | URL | What it finds |
|---|---|---|
| Spokeo | spokeo.com | Phone, address, relatives, social profiles |
| Pipl | pipl.com | Deep people search — professional and personal |
| BeenVerified | beenverified.com | Background records, address history |
| Intelius | intelius.com | Public records, criminal, address |
| Whitepages | whitepages.com | Phone and address |
| FastPeopleSearch | fastpeoplesearch.com | Free, address and relatives |

**How this feeds attacks:**
Knowing an IT admin's home address, phone number, and personal email enables highly convincing pretexting — calling the helpdesk impersonating them with accurate personal details that would normally verify identity.

---

## Sherlock — Username Search Across Platforms

When you identify an employee's username from one platform, Sherlock checks 300+ other platforms for the same username.

```bash
# Install
pip3 install sherlock-project

# Basic search
sherlock johndoe

# Multiple usernames
sherlock johndoe j.doe john.doe.target

# Output to file
sherlock johndoe --output results.txt

# CSV output
sherlock johndoe --csv

# Timeout per site
sherlock johndoe --timeout 5
```

**Why this matters:**
A developer uses `jtarget_dev` as their username on GitHub, Stack Overflow, and Reddit. Their GitHub has repositories that include internal tools or old API keys. Their Stack Overflow questions reveal which version of a specific database they are running ("I'm having this error on PostgreSQL 13.4..."). Their Reddit history has complained about their company's specific software.

---

## GitHub and Code Repository Intelligence

Public code repositories are one of the highest-value OSINT sources. Developers accidentally commit credentials, internal hostnames, API keys, and architecture diagrams.

### Searching GitHub

```bash
# Via web interface — advanced search
site:github.com "target.com" password
site:github.com "target.com" api_key
site:github.com "@target.com" secret

# GitHub Search API
curl "https://api.github.com/search/code?q=target.com+password" \
  -H "Authorization: token YOUR_GITHUB_TOKEN"
```

**GitHub search queries:**

```
# Search for company domain in code
"target.com" language:python
"target.com" extension:env
"target.com" extension:yml password
"@target.com" extension:json api_key

# Search for internal hostnames (found from DNS recon)
"dc1.target.local"
"erp.target.com"
"jenkins.target.com"

# Search for specific tech (found from fingerprinting)
"target.com" "palo alto"
"target.com" aws_access_key_id
```

### trufflehog — Secret Scanning in Repos

```bash
# Install
pip3 install trufflehog

# Scan a GitHub org
trufflehog github --org=targetcorp

# Scan a specific repo
trufflehog github --repo=https://github.com/targetcorp/internal-tool

# Scan all branches
trufflehog github --repo=https://github.com/targetcorp/internal-tool --branch=all

# Scan local directory
trufflehog filesystem /path/to/cloned/repo
```

### gitleaks — Another Secret Scanner

```bash
# Scan a repo
gitleaks detect --source https://github.com/targetcorp/internal-tool

# Scan with report
gitleaks detect --source . --report-format json --report-path report.json
```

**What these tools find:** AWS keys, GitHub tokens, Slack webhooks, database connection strings, private SSH keys, API credentials for internal services, and hardcoded passwords.

---

## SpiderFoot — Automated OSINT Framework

SpiderFoot runs 200+ OSINT modules automatically, correlates findings, and builds relationship maps between discovered entities.

```bash
# Install
pip3 install spiderfoot

# Start web interface
spiderfoot -l 127.0.0.1:5001
# Open http://127.0.0.1:5001 in browser
# Create new scan → enter target domain → select modules → run

# CLI mode
spiderfoot -s target.com -t INTERNET_NAME -o csv -f output.csv

# Specific module types
# INTERNET_NAME → domain and subdomain recon
# IP_ADDRESS → IP intelligence
# EMAILADDR → email-based OSINT
# PHONE_NUMBER → phone intelligence
```

**What SpiderFoot correlates automatically:**
Domain → IPs → ASN → netblocks → hostnames → emails → breach records → social media profiles → company info → related domains. Each discovery feeds the next module automatically.

---

## Maltego — Visual Link Analysis

Maltego builds visual graphs of relationships between entities. The free Community Edition is sufficient for most recon work.

**Core concepts:**
- **Entities** — domains, IPs, people, email addresses, companies, social profiles
- **Transforms** — data queries that run against an entity and return new entities
- **Graph** — the visual map of all discovered relationships

**Basic workflow:**

1. Create new graph
2. Add entity: `target.com` (Domain entity)
3. Right-click → Run Transform → `To DNS Name [Found in DNS]`
4. Result: subdomains appear as connected nodes
5. Select IP node → Run Transform → `To Whois [Owner Name]`
6. Result: organization name attached to IP
7. Select organization → Run Transform → `To Email Address`
8. Result: email addresses connected to the org

**Useful transforms:**

| Starting entity | Transform | Returns |
|---|---|---|
| Domain | To DNS Name | Subdomains |
| Domain | To IP Address | IP addresses |
| IP Address | To Netblock | IP range ownership |
| IP Address | To Domain | Reverse DNS hostnames |
| Email Address | To Person | Person entity |
| Person | To Social Profiles | LinkedIn, Twitter, etc. |
| Domain | To Shodan Results | Open ports and services |
| Email | Have I Been Pwned | Breach records |

---

## Recon-ng — Modular OSINT Framework

Recon-ng works like Metasploit — install modules from a marketplace, run them against targets, store results in a database.

```bash
# Start
recon-ng

# Create a workspace for the engagement
[recon-ng] > workspaces create targetcorp

# Add the target domain
[recon-ng][targetcorp] > db insert domains
domain: target.com

# Browse available modules
[recon-ng][targetcorp] > marketplace search

# Search for specific functionality
[recon-ng][targetcorp] > marketplace search domain

# Install a module
[recon-ng][targetcorp] > marketplace install recon/domains-hosts/hackertarget

# Load and run
[recon-ng][targetcorp] > modules load recon/domains-hosts/hackertarget
[recon-ng][targetcorp][hackertarget] > run

# View results
[recon-ng][targetcorp] > show hosts
[recon-ng][targetcorp] > show contacts

# Generate report
[recon-ng][targetcorp] > modules load reporting/html
[recon-ng][targetcorp][html] > run
```

**Useful module categories:**

| Category | What it does |
|---|---|
| `recon/domains-hosts` | Find hosts for a domain |
| `recon/domains-contacts` | Find email addresses for a domain |
| `recon/hosts-hosts` | Resolve hostnames, reverse lookup |
| `recon/contacts-credentials` | Find breach data for emails |
| `recon/companies-contacts` | Find contacts for a company |
| `reporting/html` | Generate HTML report of all findings |

---

## Dark Web and Paste Sites

### Breach Data and Leaked Credentials

```bash
# haveibeenpwned — check domain exposure
# https://haveibeenpwned.com/DomainSearch (requires domain ownership verification)

# Intelligence X — search breach data, paste sites, dark web
# https://intelx.io/ → search for target.com

# Dehashed — find email:password pairs (paid)
curl "https://api.dehashed.com/search?query=email:@target.com" \
  -H "Authorization: Basic BASE64_ENCODED_KEY"
```

### Paste Site Monitoring

Pastebin, GitHub Gist, and similar sites are used for sharing data dumps from breaches. Sensitive internal data sometimes appears there.

```bash
# pwnbin — search paste sites
pip3 install pwnbin
pwnbin target.com

# pastego — Pastebin scraper
go install github.com/edoz90/pastego@latest
pastego -s "target.com"

# Manual search
site:pastebin.com "target.com"
site:pastebin.com "@target.com"
site:pastebin.com "target.com" password
```

---

## Competitive Intelligence Sources

| Source | URL | Best for |
|---|---|---|
| SEC EDGAR | edgar.sec.gov | Annual reports, financial filings, risk factors, infrastructure mentions |
| Crunchbase | crunchbase.com | Funding, team, acquisitions, investors |
| D&B Hoovers | hoovers.com | Company size, revenue, subsidiaries |
| Glassdoor | glassdoor.com | Internal tools mentioned in reviews, IT complaints |
| BuiltWith | builtwith.com | Web technology stack history |
| Job boards | indeed.com, dice.com | Tech stack from postings |
| Patent databases | patents.google.com | R&D direction, technology investments |
| News archives | Factiva, LexisNexis | Press coverage, incidents, executive changes |

**SEC EDGAR** is particularly valuable for large companies. 10-K annual filings contain:
- Risk factors mentioning specific technologies and their security implications
- Descriptions of IT infrastructure in business continuity sections
- Disclosure of past incidents and their causes
- Details of major vendors and third-party dependencies

---

## OSINT Workflow

```bash
TARGET="target.com"
COMPANY="Target Corp"

echo "[1] LinkedIn intelligence"
# Manual — search LinkedIn for:
# - Job postings (tech stack)
# - Employees in IT/Security (names for email generation)
# - Company page (headcount, recent announcements)

echo "[2] Email harvest from search engines"
theHarvester -d $TARGET -l 500 -b google,bing,linkedin -f harvest.html

echo "[3] Username search on found employees"
# sherlock [employee_usernames]

echo "[4] GitHub secret search"
# Search: site:github.com "$TARGET" password
# Search: site:github.com "$TARGET" api_key
# trufflehog github --org=targetcorp

echo "[5] Automated OSINT framework"
# spiderfoot -s $TARGET -t INTERNET_NAME -o csv -f spiderfoot.csv

echo "[6] Breach data check"
# Check intelx.io for @$TARGET credentials
# Check dehashed.com for $TARGET email:password pairs

echo "[7] Paste site search"
# site:pastebin.com "$TARGET"
# pwnbin $TARGET

echo "[8] Competitive intelligence"
# edgar.sec.gov — search company name
# crunchbase.com — company profile
# glassdoor.com — employee reviews for tool mentions
```

---

## OPSEC for Social Media OSINT

- **LinkedIn scraping** — aggressive automated scraping violates ToS and may result in account suspension. Manual browsing is safer for targeted profiles.
- **Creating fake profiles** — never create fake LinkedIn or social media profiles for recon. This crosses both ethical and legal lines in most jurisdictions and in virtually all engagement scopes.
- **Viewing LinkedIn profiles** — your LinkedIn account shows in the profile owner's "who viewed my profile" if your privacy settings are not set to anonymous. Set to anonymous mode before profiling targets: Settings → Privacy → Profile viewing options → Anonymous.
- **GitHub searches** — completely anonymous, no account required for public repo searches.
- **IntelX and Dehashed** — these services log your API key against your searches. Use a dedicated key, not a personal one.

---

*Previous: [Website Enumeration](10-website-enumeration.md)*
*Back to: [README](../README.md) · [Tool Reference](tool-reference.md)*
