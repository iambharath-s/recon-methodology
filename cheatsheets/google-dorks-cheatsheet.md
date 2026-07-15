# Google Dorks — Cheat Sheet

Operators, combinations, and ready-to-use queries. Replace `target.com` with your target domain.

---

## Core Operators

| Operator | Syntax | Function |
|---|---|---|
| `site:` | `site:target.com` | Restrict to domain |
| `filetype:` | `filetype:pdf` | File extension filter |
| `intitle:` | `intitle:"login"` | Match in page title |
| `allintitle:` | `allintitle:admin panel` | All words in title |
| `inurl:` | `inurl:/admin` | Match in URL |
| `allinurl:` | `allinurl:config backup` | All words in URL |
| `intext:` | `intext:"api_key"` | Match in page body |
| `cache:` | `cache:target.com` | Google's cached snapshot |
| `link:` | `link:target.com` | Pages linking to target |
| `related:` | `related:target.com` | Similar sites |
| `before:` | `before:2022-01-01` | Content before date |
| `after:` | `after:2023-01-01` | Content after date |
| `-` | `-inurl:www` | Exclude results |
| `OR` | `filetype:env OR filetype:conf` | Either term |
| `"..."` | `"index of"` | Exact phrase match |
| `*` | `inurl:admin*` | Wildcard |

---

## Subdomain and Asset Discovery

```
site:*.target.com
site:*.target.com -site:www.target.com
site:*.*.target.com
```

---

## Exposed Files by Type

```
# Config and environment files
filetype:env site:target.com
filetype:conf site:target.com
filetype:config site:target.com
filetype:ini site:target.com
filetype:cfg site:target.com

# Database files
filetype:sql site:target.com
filetype:db site:target.com
filetype:sqlite site:target.com
filetype:mdb site:target.com

# Backup files
filetype:bak site:target.com
filetype:backup site:target.com
inurl:backup site:target.com

# Log files
filetype:log site:target.com
inurl:access.log site:target.com
inurl:error.log site:target.com

# Office documents (metadata gold)
filetype:pdf site:target.com
filetype:docx site:target.com
filetype:xlsx site:target.com
filetype:pptx site:target.com

# VPN config files
filetype:pcf site:target.com
filetype:ovpn site:target.com

# Private keys
intext:"-----BEGIN RSA PRIVATE KEY-----" site:target.com
intext:"-----BEGIN OPENSSH PRIVATE KEY-----" site:target.com
filetype:pem site:target.com
filetype:key site:target.com
```

---

## Credentials and Sensitive Data

```
# Passwords in text files
filetype:txt intext:password site:target.com
filetype:txt intext:passwd site:target.com

# API keys
intext:"api_key" site:target.com
intext:"apikey" site:target.com
intext:"api_secret" site:target.com
intext:"secret_key" site:target.com
intext:"access_token" site:target.com

# AWS credentials
intext:"AKIA" site:target.com
intext:"aws_access_key_id" site:target.com

# Database connection strings
intext:"DB_PASSWORD" site:target.com
intext:"database_password" site:target.com
intext:"mysql_password" site:target.com
intext:"MONGODB_URI" site:target.com

# Generic credential patterns
inurl:credentials site:target.com
filetype:xml intext:password site:target.com
```

---

## Login and Admin Portals

```
inurl:"/admin" site:target.com
inurl:"/login" site:target.com
inurl:"/wp-admin" site:target.com
inurl:"/administrator" site:target.com
inurl:"/phpmyadmin" site:target.com
inurl:"/cpanel" site:target.com
inurl:"/webmail" site:target.com
inurl:"/outlook" site:target.com
inurl:"/portal" site:target.com
intitle:"admin panel" site:target.com
intitle:"login" inurl:admin site:target.com
```

---

## Open Directory Listings

```
intitle:"index of" site:target.com
intitle:"index of" "parent directory" site:target.com
intitle:"index of" /backup site:target.com
intitle:"index of" /admin site:target.com
intitle:"index of" /config site:target.com
intitle:"index of" /uploads site:target.com
intitle:"index of" /logs site:target.com
intitle:"index of" ".git" site:target.com
```

---

## Technology-Specific Dorks

```
# WordPress
inurl:"/wp-content" site:target.com
inurl:"/wp-includes" site:target.com
intitle:"WordPress" site:target.com

# phpMyAdmin
intitle:"phpMyAdmin" inurl:"phpmyadmin" site:target.com

# Joomla
inurl:"/components/com_" site:target.com
intitle:"Joomla! Administration" site:target.com

# Django
intitle:"Django" intext:"Debug = True" site:target.com
intext:"DisallowedHost at" site:target.com

# Apache / Nginx misconfiguration
intitle:"Apache2 Ubuntu Default Page" site:target.com
intitle:"Welcome to nginx" site:target.com
intitle:"Test Page for Apache" site:target.com

# Jenkins
intitle:"Dashboard [Jenkins]" site:target.com
inurl:"/job/" inurl:"/build" site:target.com

# GitLab / GitHub
inurl:"/-/pipelines" site:target.com
inurl:"/raw/" site:target.com

# Kibana / Elasticsearch
intitle:"Kibana" site:target.com
inurl:":9200/_cat" site:target.com

# Laravel debug
intext:"APP_DEBUG=true" site:target.com
intitle:"Whoops! There was an error." site:target.com
```

---

## Error Messages (Verbose Errors = Info Disclosure)

```
intext:"SQL syntax" site:target.com
intext:"mysql_fetch" site:target.com
intext:"Warning: mysql" site:target.com
intext:"Fatal error:" site:target.com
intext:"Parse error:" site:target.com
intext:"stack trace" site:target.com
intext:"Traceback (most recent call" site:target.com
intext:"undefined index" site:target.com
intext:"pg_query()" site:target.com
```

---

## Network and Infrastructure

```
# Network devices
intitle:"RouterOS" site:target.com
intitle:"Cisco" "last login" site:target.com

# Webcams and surveillance
inurl:"view/index.shtml" site:target.com
intitle:"IP Camera" site:target.com

# VoIP
intitle:"Asterisk Management" site:target.com

# Remote access
intitle:"VNC Desktop" inurl:":5901" site:target.com
inurl:":8080/RDWeb" site:target.com
```

---

## Shodan Dorks (shodan.io)

```
# By organization
org:"Target Corp"

# By ASN
asn:AS12345

# By IP range
net:185.45.12.0/24

# Find specific services
org:"Target Corp" port:21        # FTP
org:"Target Corp" port:22        # SSH
org:"Target Corp" port:3389      # RDP
org:"Target Corp" port:27017     # MongoDB (often unauthenticated)
org:"Target Corp" port:9200      # Elasticsearch
org:"Target Corp" port:6379      # Redis
org:"Target Corp" port:5900      # VNC
org:"Target Corp" product:nginx
org:"Target Corp" product:Apache

# Find by certificate
ssl.cert.subject.CN:"*.target.com"
ssl.cert.subject.O:"Target Corp"
```

---

## GHDB Categories

Pre-built dorks organized by category at `exploit-db.com/google-hacking-database`:

| Category | What it finds |
|---|---|
| Footholds | Entry points — shell upload forms, file managers |
| Files containing usernames | Text files, config files with username fields |
| Sensitive directories | Open directories, backup locations |
| Web server detection | Server type and version identification |
| Vulnerable files | Files associated with known vulnerabilities |
| Vulnerable servers | Servers running exploitable software |
| Error messages | Verbose errors revealing code, paths, DB info |
| Files containing passwords | Direct credential exposure |
| Sensitive online shopping info | E-commerce credential exposure |
| Network or vulnerability data | Firewall configs, IDS data |
| Various online devices | Cameras, printers, routers, SCADA |

---

*Back to [Cheat Sheets](recon-cheatsheet.md)*
