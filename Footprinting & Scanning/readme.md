# Assessment Methodologies: Footprinting & Scanning CTF 1

## Overview

This lab focused entirely on the reconnaissance and footprinting phase — no exploitation required. The objective was to extract information passively and actively from a single target by reading service banners, inspecting web server metadata, enumerating anonymous FTP access, and probing a MySQL instance using discovered credentials. Every flag was reachable through careful observation rather than exploitation.

**Target:** `192.248.144.3`

**Flags to capture:**

- **Flag 1** — Something unusual hidden in the server's identity announcement
- **Flag 2** — Information revealed in the web server's gatekeeper instructions
- **Flag 3** — A file accessible via anonymous FTP
- **Flag 4** — A well-named database discoverable through MySQL enumeration

---

## Enumeration

### Phase 1 — Full Port Scan

A fast SYN scan across all 65535 ports was run first to identify every open service:

```bash
nmap -sS -p- -T4 192.248.144.3
```

This revealed the full set of open ports before committing to a slower, deeper scan.

### Phase 2 — Service & Version Detection

With the open ports identified, a targeted aggressive scan was run to pull service versions, OS fingerprints, and default NSE script output:

```bash
nmap -sS -p21,22,25,80,143,993,3306,33060 -A -T4 192.248.144.3
```

**Services identified:**

| Port | Service | Notes |
|---|---|---|
| 21 | FTP | Anonymous access enabled |
| 22 | SSH | Standard OpenSSH |
| 25 | SMTP | Mail server |
| 80 | HTTP | Web server with banner |
| 143 | IMAP | Mail access |
| 993 | IMAPS | Encrypted IMAP |
| 3306 | MySQL | Database server |
| 33060 | MySQLx | MySQL extended protocol |

The breadth of services — FTP, HTTP, SMTP, IMAP, and MySQL all on one host — made this a rich target for passive information gathering, with multiple potential channels to extract flags without any active exploitation.

---

## Flag 1 — Server Identity Banner

The hint referred to the server "proudly announcing its identity in every response." The natural place to look was the HTTP response headers returned by the web server on port 80, which commonly include a `Server:` header disclosing the web software version.

Examining the HTTP headers closely from the Nmap scan output revealed an unusual entry embedded in the server's standard identity disclosure:

```text
Flag 1 embedded in the HTTP Server header
```
![alt text](image.png)
> **Takeaway:** HTTP `Server:` headers are often overlooked but are a prime source of version disclosure and occasionally deliberately-placed lab flags. Real-world servers should suppress or sanitize this header.

---

## Flag 2 — Web Server Gatekeeper Instructions

The hint about "gatekeeper instructions" and reading "between the lines" pointed directly to `robots.txt` — the file web servers use to instruct crawlers which paths should not be indexed. These directives are meant to discourage crawlers, but since they are publicly readable, they frequently disclose the existence of sensitive or hidden paths:

```bash
curl http://192.248.144.3/robots.txt
```

The `robots.txt` file contained a `Disallow:` entry referencing a path that held the second flag — a classic example of the file intended to hide something instead advertising exactly where it is.

![alt text](image-1.png)

> **Takeaway:** `robots.txt` is one of the first files to check on any web target. Disallowed paths are often the most interesting ones. Never assume that listing a path in `robots.txt` keeps it hidden — it does the opposite.

---

## Flag 3 — Anonymous FTP Access

The Nmap scan confirmed that port 21 was running an FTP service with anonymous access enabled. This means any client can authenticate using the username `anonymous` with any string as a password (commonly an email address or just `anonymous` again):

```bash
curl ftp://anonymous:anonymous@192.248.144.3/flag.txt -o flag.txt
cat flag.txt
```

The flag was sitting as a plain file in the FTP root, readable without any credentials.

During this step, a second file was also noticed in the directory listing:

```text
flag.txt
creds.txt
```

`creds.txt` was immediately downloaded as well — any file named after credentials in an accessible share warrants immediate attention:

```bash
curl ftp://anonymous:anonymous@192.248.144.3/creds.txt -o creds.txt
cat creds.txt
```

Contents:

```text
db_admin : password@123
```
![alt text](image-2.png)
> **Takeaway:** Anonymous FTP is a significant misconfiguration in any production environment. File names like `creds.txt` left in accessible directories represent a compounded failure — not only is anonymous access permitted, but plaintext credentials for another service were left inside. Every file in an anonymous share should be downloaded and inspected.

---

## Flag 4 — MySQL Database Enumeration

With the credentials from `creds.txt`, the MySQL instance on port 3306 became accessible. Before connecting directly, Nmap's MySQL script collection was run to gather as much passive information as possible:

```bash
nmap -p 3306 -T4 -v --script=mysql-* 192.248.144.3
```

This enumerated accessible databases, users, and any information-schema details exposed without authentication.

A direct authenticated connection was then established:

```bash
mysql -h 192.248.144.3 -u db_admin -p'password@123'
```

To quickly surface all database names — including any with unusual or flag-like names — a full database dump was performed:

```bash
mysqldump -h 192.248.144.3 -u db_admin -p'password@123' --all-databases > dump.sql
```

Inspecting the dump revealed a database whose name was the flag itself:

```sql
CREATE DATABASE IF NOT EXISTS `FLAG4_e5a80e79ad8b4a6785ad5c218113cfea`
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci
  DEFAULT ENCRYPTION='N';
```

```text
Flag 4: FLAG4_e5a80e79ad8b4a6785ad5c218113cfea
```
![alt text](image-3.png)
> **Takeaway:** Database names, table names, and column names are all worth enumerating — they can reveal the application's structure, environment details, or in this case, deliberately hidden flags. `mysqldump --all-databases` is one of the fastest ways to surface everything a user account can access in a single pass.

---

## Flags Captured

| Flag | Location | Value |
|---|---|---|
| Flag 1 | HTTP `Server:` response header | *(captured from banner — fill in from screenshot)* |
| Flag 2 | `robots.txt` disallowed path | *(captured from robots.txt — fill in from screenshot)* |
| Flag 3 | `/flag.txt` via anonymous FTP | *(captured from flag.txt — fill in from screenshot)* |
| Flag 4 | MySQL database name | `FLAG4_e5a80e79ad8b4a6785ad5c218113cfea` |


---

## Key Takeaways

- Running a full port scan (`-p-`) before a targeted version scan is a habit worth keeping — services on non-standard or high ports (33060 here) are easily missed if you only scan the top 1000.
- HTTP response headers and `robots.txt` are passive information sources that require no authentication and yield immediate results — they should always be checked before anything more active.
- Anonymous FTP is a legacy misconfiguration that still appears regularly. Any file in a world-readable FTP share should be treated as potentially sensitive and downloaded immediately.
- Credential files discovered on one service (`creds.txt` on FTP) should be tested immediately against every other running service — the `db_admin` credential would not have been tried against MySQL without first finding it on FTP.
- `mysqldump --all-databases` is a fast, comprehensive way to enumerate everything a database user can access, including database names that might themselves carry information.

## Skills Practiced

- Full TCP Port Scanning (Nmap SYN scan)
- Service Version & OS Fingerprinting (`-A` flag)
- HTTP Banner Analysis & Header Inspection
- `robots.txt` Enumeration
- Anonymous FTP Enumeration & File Retrieval (`curl`)
- Credential Discovery from Exposed Files
- MySQL Enumeration via Nmap Scripts (`mysql-*`)
- Authenticated MySQL Access & Database Enumeration
- Full Database Dump (`mysqldump`)
- Passive Information Gathering Methodology