# Metasploitable_2_Vuln# 🛡️ Metasploitable 2 Vulnerability Research

> A comprehensive documentation of vulnerabilities discovered in the **Metasploitable 2** intentionally vulnerable virtual machine, for educational and ethical penetration testing purposes.

---

## ⚠️ Disclaimer

> **This repository is strictly for educational purposes.**
> All findings documented here were performed in a controlled, isolated lab environment on the Metasploitable 2 VM — a machine deliberately designed to be vulnerable.
> **Do NOT use any of these techniques on systems you do not own or do not have explicit written permission to test. Unauthorized access to computer systems is illegal.**

---

## 📖 About This Repository

This repository documents vulnerabilities, exploitation techniques, and findings from a structured penetration test of the **Metasploitable 2** virtual machine — a deliberately insecure Linux-based VM created by Rapid7 for security training.

Each vulnerability entry includes:
- CVE ID (if applicable)
- Affected service and port
- Description of the vulnerability
- Steps to reproduce
- Tools used
- Proof of concept / screenshots
- Recommended remediation

---

## 🖥️ Target Information

| Property        | Details                        |
|----------------|-------------------------------|
| OS             | Ubuntu 8.04 (Hardy Heron)     |
| VM             | Metasploitable 2              |
| Architecture   | 32-bit x86                    |
| Default IP     | 192.168.x.x (DHCP / Host-Only)|
| Purpose        | Intentionally Vulnerable VM   |
| Creator        | Rapid7                        |

---

## 🗂️ Vulnerability Index

| # | Service | Port | Vulnerability | Severity | CVE | Status
|---|---------|------|--------------|----------|-----|-------
| 1 | FTP | 21 | vsFTPd 2.3.4 Backdoor | 🔴 Critical | CVE-2011-2523 | ✅
| 2 | SSH | 22 | Weak Credentials / Brute Force | 🟠 High | — |
| 3 | Telnet | 23 | Cleartext Authentication | 🟠 High | — |
| 4 | SMTP | 25 | Open Mail Relay | 🟡 Medium | — |
| 5 | HTTP | 80 | DVWA / Multiple Web Vulns | 🔴 Critical | — |
| 6 | RPC | 111 | RPC Enumeration | 🟡 Medium | — |
| 7 | NetBIOS | 139/445 | Samba Usermap Script | 🔴 Critical | CVE-2007-2447 |✅
| 8 | Java RMI | 1099 | Java RMI Server RCE | 🔴 Critical | — |
| 9 | NFS | 2049 | World-Readable NFS Shares | 🟠 High | — |
| 10 | IRC | 6667 | UnrealIRCd Backdoor | 🔴 Critical | CVE-2010-2075 |✅
| 11 | Postgres | 5432 | Default Credentials | 🟠 High | — |
| 12 | MySQL | 3306 | No Root Password | 🔴 Critical | — |✅
| 13 | Tomcat | 8180 | Default Credentials / WAR Upload | 🔴 Critical | — |✅
| 14 | Distcc | 3632 | Remote Code Execution | 🔴 Critical | CVE-2004-2687 |✅
| 15 | PHP | 80 | PHP CGI Argument Injection | 🔴 Critical | CVE-2012-1823 |✅

> ✏️ *This table will be updated as new vulnerabilities are documented.*

---

## 📁 Repository Structure

```
Metasploitable_2_Vuln/
│
├── README.md                     # This file
│
├── recon/                        # Reconnaissance & enumeration
│   ├── nmap_scan.md
│   └── service_enumeration.md
│
├── vulnerabilities/              # Individual vulnerability write-ups
│   ├── vsftpd_backdoor/
│   ├── samba_usermap/
│   ├── unrealircd_backdoor/
│   ├── distcc_rce/
│   ├── java_rmi_rce/
│   ├── mysql_no_root_pass/
│   ├── tomcat_manager/
│   └── ...
│
├── web/                          # Web application vulnerabilities
│   ├── dvwa/
│   ├── mutillidae/
│   └── phpMyAdmin/
│
├── post-exploitation/            # Post-exploitation notes
    └── privilege_escalation.md
              
```

---

## 🔬 Lab Setup

### Requirements
- [VirtualBox](https://www.virtualbox.org/) or VMware
- [Metasploitable 2 ISO](https://sourceforge.net/projects/metasploitable/)
- [Kali Linux](https://www.kali.org/) (attacker machine)
- NAT network adapter (isolated environment)

### Network Configuration
```
Attacker (Kali)  →  [NAT Network]  →  Target (Metasploitable 2)
192.168.133.x                               192.168.133.x
```

> ⚡ **Always run in an isolated network. Never expose Metasploitable 2 to the internet.**

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning & service enumeration |
| `Metasploit Framework` | Exploitation |
| `Hydra` / `Medusa` | Brute-force attacks |
| `Nikto` | Web vulnerability scanning |
| `enum4linux` | SMB/Samba enumeration |
| `sqlmap` | SQL injection automation |
| `Burp Suite` | Web application testing |
| `netcat` | Reverse shells & listeners |
| `searchsploit` | Exploit research |

---

## 📚 References

- [Metasploitable 2 Official Documentation](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [Exploit-DB](https://www.exploit-db.com/)
- [NVD - National Vulnerability Database](https://nvd.nist.gov/)
- [CVE Details](https://www.cvedetails.com/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

---

## 👤 Author

**Your Name**
- GitHub: [@Gupteswar-Achary](https://github.com/Gupteswar-Achary)
- LinkedIn: [Gupteswar Achary](https://www.linkedin.com/in/gupteswar-achary-141151341/)

---


<div align="center">
  <sub>Built for learning. Hack responsibly. 🔐</sub>
</div>
