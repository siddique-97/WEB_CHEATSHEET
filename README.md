# 🎯 Burp Suite & Web Security CTF Cheat Sheets

[![PortSwigger](https://img.shields.io/badge/PortSwigger-Web%20Security%20Academy-orange?style=flat-square)](https://portswigger.net/web-security)
[![Cheat Sheets](https://img.shields.io/badge/Cheat%20Sheets-31%20Topics-blue?style=flat-square)](#-all-cheat-sheets)
[![Focus](https://img.shields.io/badge/Focus-CTF%20%26%20Pentesting-red?style=flat-square)](#)

A concise, field-tested reference library for CTF competitions and web penetration testing based on the **PortSwigger Web Security Academy**. Includes rapid testing workflows, copy-paste payloads, Burp Suite automation tips, and filter bypasses.

---

## 📚 All Cheat Sheets

| Category | Topic | File |
|---|---|---|
| **Injection** | SQL Injection | [SQL_Injection.md](./SQL_Injection.md) |
| **Injection** | NoSQL Injection | [NoSQL_Injection.md](./NoSQL_Injection.md) |
| **Injection** | OS Command Injection | [OS_Command_Injection.md](./OS_Command_Injection.md) |
| **Injection** | Server-Side Template Injection (SSTI) | [SSTI.md](./SSTI.md) |
| **Injection** | XML External Entity (XXE) Injection | [XXE_Injection.md](./XXE_Injection.md) |
| **Client-Side** | Cross-Site Scripting (XSS) | [Cross_Site_Scripting.md](./Cross_Site_Scripting.md) |
| **Client-Side** | Cross-Site Request Forgery (CSRF) | [CSRF.md](./CSRF.md) |
| **Client-Side** | Cross-Origin Resource Sharing (CORS) | [CORS.md](./CORS.md) |
| **Client-Side** | Clickjacking (UI Redressing) | [Clickjacking.md](./Clickjacking.md) |
| **Client-Side** | DOM-Based Vulnerabilities | [DOM_Based_Vulnerabilities.md](./DOM_Based_Vulnerabilities.md) |
| **Client-Side** | Prototype Pollution | [Prototype_Pollution.md](./Prototype_Pollution.md) |
| **Auth & Access** | Authentication Vulnerabilities | [Authentication.md](./Authentication.md) |
| **Auth & Access** | Access Control & IDOR | [Access_Control.md](./Access_Control.md) |
| **Auth & Access** | JSON Web Tokens (JWT) | [JWT.md](./JWT.md) |
| **Auth & Access** | OAuth 2.0 & OpenID Connect | [OAuth_Authentication.md](./OAuth_Authentication.md) |
| **Server-Side** | Server-Side Request Forgery (SSRF) | [SSRF.md](./SSRF.md) |
| **Server-Side** | Path Traversal | [Path_Traversal.md](./Path_Traversal.md) |
| **Server-Side** | File Upload Vulnerabilities | [File_Upload.md](./File_Upload.md) |
| **Server-Side** | Insecure Deserialization | [Insecure_Deserialization.md](./Insecure_Deserialization.md) |
| **Advanced Web** | HTTP Request Smuggling | [HTTP_Request_Smuggling.md](./HTTP_Request_Smuggling.md) |
| **Advanced Web** | Web Cache Poisoning | [Web_Cache_Poisoning.md](./Web_Cache_Poisoning.md) |
| **Advanced Web** | Web Cache Deception | [Web-Cache-Deception.md](./Web-Cache-Deception.md) |
| **Advanced Web** | HTTP Host Header Attacks | [Host_Header_Attacks.md](./Host_Header_Attacks.md) |
| **Advanced Web** | Race Conditions | [Race_Conditions.md](./Race_Conditions.md) |
| **Advanced Web** | WebSockets | [WebSockets.md](./WebSockets.md) |
| **APIs** | GraphQL API Vulnerabilities | [GraphQL.md](./GraphQL.md) |
| **APIs** | API Testing | [API_BURP.md](./API_BURP.md) |
| **Modern / Logic** | Web LLM Attacks | [Web_LLM_Attacks.md](./Web_LLM_Attacks.md) |
| **Modern / Logic** | Business Logic Vulnerabilities | [Business_Logic.md](./Business_Logic.md) |
| **Recon & Core** | Information Disclosure | [Information_Disclosure.md](./Information_Disclosure.md) |
| **Recon & Core** | Essential Skills & Burp Setup | [Essential_Skills.md](./Essential_Skills.md) |

---

## ⚡ Quick Burp Suite Pro-Tips

- **URL-Encode on the fly:** Highlight string in Repeater/Intruder and press `Ctrl + U`.
- **Parallel Race Conditions:** Put requests in a Repeater group → select **Send group in parallel (single-packet attack)**.
- **Match & Replace:** Use `Proxy → Proxy Settings → Match and Replace` to automatically inject `X-Forwarded-For: 127.0.0.1` or bypass client validations.
- **OAST Probing:** Use `Burp Collaborator Client` to verify blind SSRF, XXE, and Command Injection via DNS.

---

## ⚠️ Disclaimer

These cheat sheets are intended **strictly for educational purposes, authorized CTF competitions, and legal penetration testing**. Never test systems without explicit prior permission.
