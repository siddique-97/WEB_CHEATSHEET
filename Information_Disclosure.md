# Information Disclosure — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Scanner, git-dumper  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Information disclosure occurs when an application inadvertently reveals sensitive technical or business information to users.

1. **Trigger Verbose Error Messages:**
   - Submit invalid data types (e.g., `id=abc` instead of integer `id=1`).
   - Submit excessively long strings, negative numbers, or arrays (`id[]=1`).
   - Check error pages for stack traces, framework versions, database dialects, and internal directory paths.
2. **Recon Hidden & Debug Endpoints:**
   - Probe for debug panels: `/phpinfo.php`, `/cgi-bin/phpinfo.cgi`, `/actuator`, `/actuator/env`, `/metrics`, `/_profiler`, `/elmah.axd`.
3. **Discover Backup & Temporary Files:**
   - Scan for text editor and backup extensions: `.bak`, `.old`, `~`, `.swp`, `.backup`, `.tmp`.
4. **Inspect Source Code Repositories:**
   - Check for exposed VCS directories: `/.git/`, `/.git/HEAD`, `/.svn/`.
5. **Inspect HTTP Response Headers:**
   - Look for server disclosure headers: `Server`, `X-Powered-By`, `X-AspNet-Version`, `X-Debug-Token`.

---

## 2. Common Scenarios & Exploitation

### 1. Stack Traces & Framework Version Disclosure
- **Test:** Pass an unexpected type to a parameter:
  ```http
  GET /product?productId=' HTTP/2
  ```
- **Information Leaked:**
  - Exact framework version (e.g., `Apache Struts 2 2.3.31`).
  - Search Exploit-DB / CVE databases for known RCEs matching the disclosed framework version.

---

### 2. Exposed Debug Pages & Environment Secrets
- **Endpoints to Check:**
  ```text
  /phpinfo.php
  /cgi-bin/phpinfo.cgi
  /actuator/env
  /actuator/mappings
  /config.json
  ```
- **High-Value Data:** Look for database passwords, API credentials, `SECRET_KEY`, and internal IP addresses.

---

### 3. Backup Files Disclosing Source Code
Web servers often execute `.php` or `.java` files, but serve `.bak` or `.old` files as plaintext downloads.

- **Fuzzing Patterns for Endpoints:**
  ```text
  /ProductTemplate.java.bak
  /index.php.old
  /config.php~
  /.env
  /database.yml.bak
  ```
- **Exploit:** Download the backup file, review hardcoded database credentials, cryptographic secrets, or administrative logic flaws.

---

### 4. Hidden Internal Headers Disclosed via TRACE / Debugging
Applications behind proxies sometimes leak required authentication headers in debug output:
- **Test:** Send a `TRACE` request:
  ```http
  TRACE / HTTP/1.1
  Host: TARGET.net
  ```
- Or trigger custom error pages that reflect proxy headers:
  ```http
  X-Custom-IP-Authorization: 127.0.0.1
  ```
- Add the disclosed header to bypass administrative route restrictions.

---

### 5. Exposed `.git` Repositories
- **Test:**
  ```http
  GET /.git/HEAD HTTP/2
  ```
  If response returns `ref: refs/heads/master`, the entire repository is exposed.
- **Exploitation:**
  Use `git-dumper` to download and reconstruct the repository:
  ```bash
  git-dumper https://TARGET.net/.git/ ./dumped-repo
  cd ./dumped-repo
  git log -p          # Search commit diffs for deleted passwords or flags
  git status
  ```

---

## 3. CTF Quick Reference

| Discovery Target | Probe Path / Trigger | Leaked Asset |
|---|---|---|
| **Debug Info** | `/cgi-bin/phpinfo.cgi` | Environment secrets, server paths |
| **Framework Trace** | `?id='` or `?id=text` | Framework version, CVE lookup |
| **Source Backup** | `/filename.java.bak` | Hardcoded secrets, source logic |
| **Git Exposure** | `/.git/HEAD` | Complete commit history, deleted keys |
| **Spring Actuator** | `/actuator/env` | Database connection strings, API tokens |
| **Proxy Header Leak** | Custom 500 error / TRACE | Disclosed headers (e.g. `X-Custom-IP-Auth`) |

---

## 4. Remediation

- **Disable Verbose Error Messages:** Configure production servers to display generic, friendly error pages without stack traces.
- **Remove Debug Panels:** Disable or restrict `/actuator`, `phpinfo`, and profiler routes to localhost or internal management interfaces.
- **Web Server Configuration:** Block web server access to dotfiles (`.git`, `.env`) and backup file extensions (`.bak`, `.swp`, `~`).
- **Sanitize Version Headers:** Suppress `Server` and `X-Powered-By` headers in reverse proxies and web server configs.
