# JSON Web Tokens (JWT) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, JWT Editor BApp, hashcat  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

A JWT consists of three Base64URL-encoded parts separated by periods (`.`):
`Header.Payload.Signature`

1. **Decode & Inspect Token:** In Burp Proxy / Inspector, identify the token format, claims (`"sub"`, `"admin"`, `"role"`), and algorithm (`"alg": "RS256"`, `"HS256"`).
2. **Test Signature Verification Flaws:**
   - Modify payload (`"sub": "administrator"`). Does the server check the signature at all?
3. **Test Algorithm `none`:**
   - Change `"alg": "none"` and delete the signature part.
4. **Brute-Force Symmetric Secrets (HS256):**
   - Run `hashcat` against weak HMAC secret keys.
5. **Test Header Injection (Asymmetric Keys):**
   - Self-signed key embedded in header (`"jwk"` injection).
   - Remote key set URL pointing to your server (`"jku"` injection).
   - Path traversal in Key ID (`"kid": "/dev/null"`).
6. **Test Algorithm Confusion (RS256 → HS256):**
   - Re-sign using the server's public key as an HMAC symmetric secret.

---

## 2. High-Yield JWT Attacks

### 1. Unverified Signature
Many applications decode the token payload but skip calling the verification function:
1. Decode the payload in Burp Inspector.
2. Change `"sub": "wiener"` to `"sub": "administrator"`.
3. Leave the signature unaltered and send the request.

---

### 2. The `none` Algorithm Bypass
Some libraries accept tokens marked with `"alg": "none"`:
1. Modify header:
   ```json
   {"alg": "none", "typ": "JWT"}
   ```
2. Modify payload:
   ```json
   {"sub": "administrator"}
   ```
3. Remove the signature trailing characters, but **keep the trailing dot**:
   `eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbmlzdHJhdG9yIn0.`
*(Try case variations if rejected: `"None"`, `"NONE"`, `"nOnE"`).*

---

### 3. Weak HMAC Secret Brute-Forcing (HS256)
If the application signs tokens using HS256 and a weak secret passphrase:

```bash
# Save raw JWT to jwt.txt
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
```
Once cracked (e.g., secret is `secret123`), use **Burp JWT Editor** or `jwt.io` to sign a tampered payload with that key.

---

### 4. Embedded JWK Header Injection (`"jwk"`)
The server validates the signature using an embedded public key provided in the token's own header:

1. In Burp **JWT Editor Keys** tab, click **New RSA Key** → Generate (2048 bits).
2. In Repeater, go to the **JSON Web Token** tab.
3. Modify payload to `"sub": "administrator"`.
4. Click **Attack → Embedded JWK** and select your generated key.
5. Send request.

---

### 5. Remote JWKS Injection (`"jku"`)
The server fetches the verification key from the URL specified in the `"jku"` header:

1. Generate a new RSA Key in Burp JWT Editor.
2. Right-click the key → **Copy Public Key as JWK**.
3. Create `/jwks.json` on your **Exploit Server** containing:
   ```json
   {
       "keys": [
           { ...PASTED_JWK_HERE... }
       ]
   }
   ```
4. In Repeater, update the JWT Header:
   ```json
   {
       "alg": "RS256",
       "typ": "JWT",
       "jku": "https://EXPLOIT-SERVER-SUBDOMAIN/jwks.json"
   }
   ```
5. Sign the token with your private key and send.

---

### 6. Key ID (`kid`) Path Traversal
The `"kid"` header parameter specifies which key file on the server's filesystem should be used. If vulnerable to path traversal, point `"kid"` to an empty or predictable system file:

```json
{
    "alg": "HS256",
    "typ": "JWT",
    "kid": "../../../../../../../dev/null"
}
```
- `/dev/null` is an empty file on Linux (0 bytes).
- In Burp JWT Editor, create a new symmetric key with Base64 value `AA==` (empty/null byte) or empty string `""`.
- Sign the modified payload (`"sub": "administrator"`) using this empty key.

---

### 7. Algorithm Confusion (RS256 → HS256)
The server uses asymmetric RSA (RS256), but the library verifies HS256 if supplied. In HS256 mode, the server uses its public verification certificate as an HMAC symmetric secret:

1. Obtain the server's public key (from `/.well-known/jwks.json` or certificate on the site).
2. Save the public key in standard PEM format.
3. In Burp JWT Editor, create a new Symmetric key, set `k` to the Base64-encoded string of the raw PEM certificate.
4. Modify token header to `"alg": "HS256"`.
5. Modify payload to `"sub": "administrator"`.
6. Sign with the newly created symmetric key.

---

## 3. CTF Quick Reference

| Attack Pattern | Header Vector | Exploit Mechanism |
|---|---|---|
| **No Verification** | Original header | Tamper payload without changing signature |
| **Algorithm None** | `"alg": "none"` | Strip signature, keep trailing dot |
| **Weak Secret** | `"alg": "HS256"` | Crack with `hashcat -m 16500` |
| **JWK Injection** | `"jwk": {...}` | Embed attacker public key in header |
| **JKU Injection** | `"jku": "https://..."`| Host malicious JWKS on exploit server |
| **KID Traversal** | `"kid": "/dev/null"` | Sign with empty HMAC key against `/dev/null` |
| **Algorithm Confusion**| `"alg": "HS256"` | Sign HS256 with server's RSA public key |

---

## 4. Remediation

- **Enforce Algorithm Allowlist:** Hardcode the verification algorithm in code (e.g. `algorithms=['RS256']`). Never allow the token's `"alg"` header to define verification logic.
- **Reject `none` Algorithm:** Explicitly disallow unsigned tokens.
- **Do Not Trust User-Controlled Headers:** Ignore untrusted `"jwk"`, `"jku"`, and `"kid"` parameters supplied directly by clients.
- **Strong HMAC Secrets:** Use cryptographically secure keys of at least 256 bits for symmetric algorithms.
