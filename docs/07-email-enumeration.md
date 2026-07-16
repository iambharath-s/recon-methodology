# Email Enumeration

Email addresses serve two purposes in a reconnaissance engagement: they are credentials for password spraying and account takeover, and they are delivery vehicles for phishing. Collecting as many as possible — and understanding the email format — is one of the highest-value early recon tasks.

---

## What You Are Looking For

**Volume** — more addresses = larger phishing target pool, better chance one credential works in a spray.

**Format** — finding `john.smith@target.com` and `sarah.jones@target.com` tells you the format is `firstname.lastname@target.com`. You can generate addresses for every employee on LinkedIn.

**Role accounts** — `admin@`, `it@`, `support@`, `security@`, `helpdesk@` are high-value social engineering targets. These are people who receive unusual requests regularly and are trained to help — which makes them easier to manipulate.

**Breach data** — an old email/password pair from a 2019 breach may still work if the employee reuses passwords.

---

## theHarvester

The standard starting point for email harvesting. Queries multiple search engines and data sources in one command.

```bash
# Single source queries
theHarvester -d target.com -l 500 -b google
theHarvester -d target.com -l 500 -b bing
theHarvester -d target.com -l 500 -b linkedin
theHarvester -d target.com -l 500 -b yahoo
theHarvester -d target.com -l 500 -b baidu

# Run all sources — outputs HTML report
theHarvester -d target.com -l 500 -b all -f harvest-report.html

# Save raw results to XML
theHarvester -d target.com -l 500 -b google -f results.xml
```

**Flag breakdown:**

| Flag | What it does |
|---|---|
| `-d` | Target domain |
| `-l` | Limit — max results to retrieve per source |
| `-b` | Source to query |
| `-f` | Output filename (generates .html and .xml) |
| `-s` | Start at result number (for pagination) |

**Source reference:**

| Source flag | What it searches | Notes |
|---|---|---|
| `google` | Google search | Best for emails in indexed pages |
| `bing` | Bing search | Finds different results than Google |
| `linkedin` | LinkedIn | Returns employee names more than emails |
| `yahoo` | Yahoo search | Supplementary |
| `baidu` | Baidu | Good for companies with China presence |
| `hunter` | Hunter.io API | Best for format confirmation — needs key |
| `intelx` | Intelligence X | Breach data — needs key |
| `shodan` | Shodan | Returns hosts, sometimes emails in banners |
| `duckduckgo` | DuckDuckGo | Privacy-focused, different index |
| `all` | Everything | Runs all available sources |

---

## Email Format Discovery

Once you have a few confirmed email addresses, derive the format and generate a full list.

**Common formats:**

| Format | Example | theHarvester result that reveals it |
|---|---|---|
| `firstname.lastname` | `john.smith@target.com` | Most common corporate format |
| `firstnamelastname` | `johnsmith@target.com` | |
| `f.lastname` | `j.smith@target.com` | |
| `firstname` | `john@target.com` | Small companies |
| `flastname` | `jsmith@target.com` | |
| `firstname_lastname` | `john_smith@target.com` | |
| `lastname.firstname` | `smith.john@target.com` | Less common |

**Confirming format with Hunter.io:**

```bash
# Hunter.io API — finds verified email addresses and confirms format
curl "https://api.hunter.io/v2/domain-search?domain=target.com&api_key=YOUR_KEY"

# Returns:
# "pattern": "firstname.lastname"       ← confirmed format
# "emails": [...]                       ← list of found addresses
```

**Generating addresses from LinkedIn employee names:**

```bash
# CrossLinked — scrapes LinkedIn, generates email combinations
pip3 install crosslinked

crosslinked -f '{first}.{last}@target.com' "Target Corp"
crosslinked -f '{f}{last}@target.com' "Target Corp"   # alternate format
```

**Manual approach — build a list from known names:**

```bash
# names.txt contains: John Smith, Sarah Jones, Mike Brown
while IFS=" " read -r first last; do
    echo "${first,,}.${last,,}@target.com"
done < names.txt > emails-generated.txt
```

---

## Email Validation — Verifying Addresses Exist

After generating a list, validate which addresses are real before using them.

### Hunter.io Verification

```bash
# Verify a single address
curl "https://api.hunter.io/v2/email-verifier?email=john.smith@target.com&api_key=YOUR_KEY"

# Returns: deliverable / undeliverable / risky / unknown
```

### SMTP Verification (Active — Leaves Logs)

Direct SMTP connection to the mail server to test if an address exists. This is active — it hits the target's mail server.

```bash
# Find the mail server
dig target.com MX +short
# → 10 mail.target.com

# Connect and test
nc mail.target.com 25
EHLO test.com
MAIL FROM: test@test.com
RCPT TO: john.smith@target.com
# 250 OK          → address accepted (likely exists)
# 550 No such user → does not exist
QUIT
```

**Caveats:**
- Modern mail systems (O365, Google Workspace) return 250 for all addresses to prevent enumeration
- Self-hosted Exchange and older systems still respond differently per address
- This technique hits the mail server and will appear in logs
- Some systems block or rate-limit connections that test many addresses

### smtp-user-enum Tool

```bash
# Install
sudo apt install smtp-user-enum

# VRFY method
smtp-user-enum -M VRFY -U emails.txt -t mail.target.com

# RCPT method (more reliable when VRFY is disabled)
smtp-user-enum -M RCPT -U emails.txt -t mail.target.com -f test@test.com

# EXPN method
smtp-user-enum -M EXPN -U emails.txt -t mail.target.com
```

**Three SMTP methods:**

| Method | Command | Supported by |
|---|---|---|
| VRFY | Directly ask "does this user exist?" | Old sendmail, some qmail |
| RCPT TO | Send to address and check response | Most servers |
| EXPN | Expand a mailing list | Rare, usually disabled |

---

## Email Header Analysis

When a target sends you an email — as part of a social engineering pretext, or if you are assessing a mail infrastructure — the headers reveal internal infrastructure.

**How to read a full email header:**

```
Received: from mail.target.com (mail.target.com [185.45.12.11])
          by mx.gmail.com with ESMTPS
          id xyz for <you@gmail.com>

Received: from TARGETCORP-EXCH01.target.local (10.0.0.20)
          by mail.target.com (185.45.12.11)

From: John Smith <john.smith@target.com>
To: you@gmail.com
Date: Mon, 15 Jan 2024 09:23:01 +0530
Message-ID: <abc123@TARGETCORP-EXCH01.target.local>
X-Mailer: Microsoft Outlook 16.0
X-Originating-IP: 10.0.0.20
```

**What each field reveals:**

| Header field | Reveals |
|---|---|
| `Received: from ... [IP]` | Mail server IP (`185.45.12.11`) — the actual server |
| `Received: from HOSTNAME (internal-IP)` | Internal hostname (`TARGETCORP-EXCH01`) and IP (`10.0.0.20`) |
| `Message-ID: <...@HOSTNAME>` | Internal hostname used as mail server identifier |
| `X-Originating-IP` | Sender's internal IP — confirms internal network range |
| `X-Mailer` | Email client and version |
| `MIME-Version + Content-Type` | Sometimes reveals email platform |

**From the example above:**
- Mail server public IP: `185.45.12.11`
- Exchange server internal hostname: `TARGETCORP-EXCH01.target.local`
- Internal domain: `target.local` (different from `target.com` — AD internal domain)
- Sender's internal IP: `10.0.0.20`
- Email client: Outlook 2016

All from one email header. No active scanning.

---

## Email Tracking (Attacker-Side Technique)

When attackers send phishing or pretexting emails, they embed tracking pixels to gather intelligence on the recipient when they open the email.

**How it works:**
1. Attacker embeds `<img src="https://attacker-server.com/track/uuid.gif">` in HTML email
2. Recipient opens email → their email client fetches the image
3. Attacker's server logs: recipient's IP, browser/client user agent, timestamp, OS

**What the log entry captures:**

```
IP: 10.0.0.45                        ← recipient's internal IP (if no proxy)
IP: 203.45.67.89                     ← or their external IP (if proxied)
User-Agent: Microsoft Outlook/16.0   ← email client and version
Timestamp: 2024-01-15 09:31:22 UTC   ← when they opened it
X-Forwarded-For: 10.0.0.45          ← internal IP if behind corporate proxy
```

**Legitimate security use:** Red team engagements use this to assess employee awareness, measure open rates, and map internal IP ranges when corporate mail clients don't proxy image requests.

**Services used legitimately for this:** CanaryTokens (canarytokens.org), Mailtrack, custom Burp Collaborator payloads.

---

## Breach Data — Finding Existing Credentials

Before building a phishing campaign, check if credentials already exist from previous breaches.

**haveibeenpwned — check if a domain appears in breaches:**

```
https://haveibeenpwned.com/DomainSearch
```

Requires domain ownership verification. Shows which breaches contained addresses from that domain.

**Dehashed (paid) — search actual credentials:**

```bash
# Dehashed API — finds email:password pairs from breaches
curl "https://api.dehashed.com/search?query=email:@target.com" \
  -H "Authorization: Basic BASE64_ENCODED_CREDS"
```

**Intelligence X — broader breach search:**

```
https://intelx.io/
# Search for target.com — returns breach data, pastes, leaks
```

**What to do with breach data:**

1. Extract email:password pairs for accounts ending in `@target.com`
2. Try pairs against O365 (`login.microsoftonline.com`) or Google Workspace login — people reuse passwords
3. Even if passwords are changed, the email list is confirmed-valid addresses
4. Password patterns reveal the company's password requirements and user habits

---

## Email Security Assessment

While harvesting emails, note the organization's email security posture — this determines whether phishing is viable.

```bash
# Check SPF
dig target.com TXT +short | grep spf

# Check DMARC
dig _dmarc.target.com TXT +short

# Check DKIM (need the selector — often "default", "google", "mail")
dig default._domainkey.target.com TXT +short
dig google._domainkey.target.com TXT +short
```

**Quick assessment:**

| Finding | Phishing viability |
|---|---|
| No SPF record | Spoof any `@target.com` address freely |
| SPF `~all` + DMARC `p=none` | Spoof successfully — email delivered |
| SPF `-all` + DMARC `p=quarantine` | Spoofed email goes to spam |
| SPF `-all` + DMARC `p=reject` | Spoofing blocked — use lookalike domains instead |

If spoofing is blocked, move to lookalike domain registration — `targetc0rp.com`, `target-corp.com`, `targetsupport.com` — and build phishing infrastructure from there.

---

## Full Email Recon Workflow

```bash
TARGET="target.com"

echo "[1] Harvest emails from search engines"
theHarvester -d $TARGET -l 500 -b google,bing,linkedin -f harvest.html

echo "[2] Check email security posture"
dig $TARGET TXT +short | grep -E "spf|v=spf"
dig _dmarc.$TARGET TXT +short

echo "[3] Identify email provider from MX"
dig $TARGET MX +short

echo "[4] Derive email format"
# Review harvest.html output for patterns

echo "[5] Get employee names from LinkedIn for format + generation"
# Manual step — or use CrossLinked
# crosslinked -f '{first}.{last}@$TARGET' "Target Corp"

echo "[6] Validate generated emails"
# smtp-user-enum -M RCPT -U generated-emails.txt -t $(dig $TARGET MX +short | awk '{print $2}' | head -1)

echo "[7] Check for breach exposure"
# Check intelx.io and dehashed for @target.com credentials
```

---

*Previous: [Search Engine Recon](06-search-engine-recon.md)*
*Next: [Subdomain Enumeration](08-subdomain-enumeration.md)*
