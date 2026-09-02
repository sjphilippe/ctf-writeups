## Jangow: 1.0.1 — VulnHub

**Date: 8/31** <br>
**Difficulty:** Easy Initial access level: None — given only an IP address on a login screen, no credentials or shell access <br>
**Objective:** Find the flag(s) on the box. The author's own hint for this machine: "the secret to this box is enumeration." <br> 

## Enumeration

Ran nmap against the target IP to identify open ports and running services:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
80/tcp open  http    Apache httpd 2.4.18
|_http-server-header: Apache/2.4.18 (Ubuntu)
```

## Initial Foothold
Started with port 80 since it was the more approachable service — navigated to the IP in a browser and found a standard website. Browsing didn't turn up much until a tab labeled "Buscar" (Spanish for "to search/look") stood out. Checking the URL showed it was a PHP page: `busque.php?buscar=`, which suggested the parameter might be passed straight into a command rather than sanitized.

I tested the `buscar` parameter with  `echo` to confirm the input was being processed by the page. Once that confirmed the parameter was live, tested a Linux command (`ls -lha /`) and got a filesystem output back — confirming this was an actual OS command injection.  I continued using bash commands to explore filesystem, but was limited to `www-data`'s permissions. 

I tried `sudo -l` to increase my access as www-data, but it didn't work. 

## Privilege Escalation

Eventually I found login credentials in `config.php` on the web server (**flag 1**). Used those creds to log into the machine directly, though the shell was limited/buggy (backslash, quotes, and other special characters didn't work correctly). Used the same credentials over FTP instead, which gave clean access to the full filesystem structure without the shell quirks.

Still only had standard user-level access at this point. I checked the kernel version and searched for known privilege escalation exploits matching that kernel. I decided to use Dirty COW (CVE-2016-5195) since I had limited read/write access and I wanted to dig deeper into the filesystem. Dirty COW exploits a timing gap in how linux handles copy-on-write memory which allows a low privileged user to write files they should only be to read (like those under root).

STEPS:
- Found Dirty COW (CVE-2016-5195) exploit source (`.c` file) on GitHub
- Transferred it to the target via FTP (tested root first — no write access there — then found `/home/jangow01` was writable)
- Confirmed on the terminal side that the file landed
- Compiled it with gcc: `gcc exploit.c -o moo` (turning readable source into a runnable binary)
- Made it executable (`chmod +x`)
- Ran 'moo'
- Landed root, navigated to `/root/proof.txt` and got the flag!

## Flags
- Flag 1 (creds): found in `config.php` on the web server
- Flag 2 (root): `/root/proof.txt`

## Defensive Takeaway
1. The initial foothold came from a command injection vulnerability in `busque.php` — user input was passed directly into a shell command with no sanitization. This should have been fixed at the code level when the server was first created. There is also the issue of privlege handling.  The `www-data` service account way too much filesystem access. Additional security would come from a Web Application Firewall (WAF) to catch additional attacks on the network and proper privilege's hardening.

2. The credentials found in `config.php` point to a secrets management gap. Credentials like this belong in a dedicated secrets manager or at least belongs outside the web root. 

3. The final escalation used Dirty COW (CVE-2016-5195), a known and patched Linux kernel vulnerability — meaning this machine was running outdated, unpatched software. This is a common vulnerability management gap.  Regular scanning against known CVEs and  patch management would have caught this before it was exploitable. Notably, the enumeration step of this walkthrough (identifying exact service versions via nmap, then researching CVEs against them) is a manual version of exactly what automated vulnerability scanning tools are built to do continuously.

## Tools Used
- `nmap` — port/service scanning
- Bash (via web command injection, then direct shell) — filesystem enumeration and command execution
- FTP client — file transfer and clean filesystem access
-  Dirty COW exploit (CVE-2016-5195) — privilege escalation
- `gcc` — compiling the Dirty COW exploit source

## Images

<p align="center">
  <img width="704" height="595" alt="buscar" src="https://github.com/user-attachments/assets/cf3eccc6-11d1-44c3-9dfe-87b185453cc4" /><br>
  <sub><b>Command injection confirmed via busque.php</b></sub>
</p>

<p align="center">
  <img width="786" height="637" alt="flag" src="https://github.com/user-attachments/assets/1746c7bc-1423-4808-9c46-e0fee7cfc850" />><br>
  <sub><b>Root access confirmed, flag captured</b></sub>
</p>

