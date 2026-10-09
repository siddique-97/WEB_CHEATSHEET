# Essential Skills & Burp Suite Techniques — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Match and Replace, Macros, Collaborator  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Essential skills encompass core Burp Suite mechanics, intercept manipulation, automated session handling, and out-of-band detection needed to solve complex challenges.

```
Reconnaissance & Scope Definition
          │
Target Interception & Tampering
  ├── Request Manipulation (Repeater / Intercept)
  └── Response Manipulation (Do Intercept → Response to this request)
          │
Automated Testing & Bypasses
  ├── Intruder Attacks (Sniper, Battering Ram, Pitchfork, Cluster Bomb)
  ├── Match & Replace (Global header injection, IP spoofing)
  └── Session Handling Rules & Macros (Dynamic token/cookie refreshes)
          │
Out-of-Band (OAST) Verification
  └── Burp Collaborator (DNS, HTTP, SMTP payload verification)
```

---

## 2. Response Manipulation (Client-Side Bypasses)

When server logic relies on client-side JavaScript interpreting an API response:

1. In **Proxy → Intercept**, click **Intercept is on**.
2. Submit the target action (e.g., incorrect 2FA code).
3. Right-click the intercepted request → **Do intercept → Response to this request**.
4. Click **Forward**.
5. When the server response appears in Intercept:
   - Modify status code: `HTTP/2 401 Unauthorized` → `HTTP/2 200 OK`
   - Modify JSON body:
     ```json
     {"result": "failure"}  -->  {"result": "success"}
     ```
   - Strip redirect headers: remove `Location: /login`.
6. Click **Forward**. The browser receives the modified response and logs into the target session.

---

## 3. Match and Replace Rules

Configure global substitutions in **Proxy → Proxy settings → Match and Replace**:

### Common CTF Use Cases:
1. **Bypass IP Blacklists globally:**
   - Type: `Request header`
   - Match: *(empty)*
   - Replace: `X-Forwarded-For: 127.0.0.1`
2. **Emulate Mobile / Internal User-Agent:**
   - Type: `Request header`
   - Match: `User-Agent: .*`
   - Replace: `User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS...)`
3. **Disable Client-Side Validation in HTML:**
   - Type: `Response body`
   - Match: `required|pattern="[^"]*"`
   - Replace: *(empty)*

---

## 4. Session Handling Rules & Burp Macros

Used during brute-force or blind attacks where requests require a dynamic CSRF token or session refresh.

### Step-by-Step Macro Configuration:
1. Go to **Settings → Sessions → Macros → Add**.
2. Select the request that generates a fresh CSRF token (e.g. `GET /login`).
3. Click **Configure item** → select the `csrf` parameter and define the custom parameter extraction regex:
   `name="csrf" value="([^"]+)"`
4. Go to **Session handling rules → Add**.
5. Under **Rule Actions**, select **Run a macro**. Select your newly created macro.
6. Under **Update parameter**, select the parameter to update: `csrf`.
7. Under **Scope**, select **Include all URLs** or select **Intruder / Repeater**.
8. Burp will now fetch a fresh CSRF token before every Intruder payload is fired.

---

## 5. Burp Collaborator Workflow (OAST)

Used for detecting completely blind SSRF, XXE, SQLi, and OS command execution.

1. Open **Burp Collaborator Client** (from top menu or Project settings).
2. Click **Copy to clipboard** to obtain an ephemeral payload subdomain:
   `xyz123abc.oastify.com`
3. Inject the domain into target vectors:
   ```text
   http://xyz123abc.oastify.com
   '; nslookup xyz123abc.oastify.com;
   <!ENTITY % x SYSTEM "http://xyz123abc.oastify.com">
   ```
4. Click **Poll now** in Collaborator:
   - **DNS Interaction:** Confirms backend resolved the hostname.
   - **HTTP Interaction:** Confirms backend established a TCP/HTTP connection.

---

## 6. Burp Intruder Attack Modes Quick Reference

| Attack Type | Payload Positions | Payload Lists | Typical Use Case |
|---|---|---|---|
| **Sniper** | Multiple markers | 1 list | Testing single parameters sequentially (Fuzzing, XSS) |
| **Battering Ram** | Multiple markers | 1 list | Injects the same payload simultaneously into all markers |
| **Pitchfork** | N markers | N lists | Tests synchronized pairs (Username list + Password list line-by-line) |
| **Cluster Bomb** | N markers | N lists | Tests every permutation (All Users × All Passwords, Char extraction) |

---

## 7. Remediation & Best Practices

- **Never Trust Client-Side State:** Perform all critical authorization and validation checks strictly on the backend.
- **Enforce Anti-Automation:** Use CAPTCHA, exponential backoff, and IP rate limiting on sensitive authentication interfaces.
- **Harden Server Headers:** Filter untrusted client-controlled headers like `X-Forwarded-For` unless coming from verified upstream proxies.
