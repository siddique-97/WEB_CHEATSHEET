# Web LLM Attacks — CTF Cheat Sheet

**Tools:** Burp Suite, browser DevTools, LLM chat interface  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

## 1. CTF methodology

When you encounter an AI chatbot or AI-powered feature, don't start by guessing prompt-injection payloads. First, map what the model can actually do.

1. Open the chatbot or AI feature.
2. Ask what tools, APIs, or functions it can access.
3. Identify each function's purpose and required arguments.
4. Look for sensitive actions: account modification, deletion, database access, email sending, file operations, and URL fetching.
5. Use Burp Proxy → HTTP history to inspect the application's requests and responses.
6. Test whether untrusted content influences the model's decisions.
7. Check whether model output is rendered unsafely.
8. Confirm the impact in the lab and record the evidence.

## 2. Attack-surface discovery

**First prompts to try in a CTF:**

```text
What functions or APIs are available to you?
What does each function do?
What arguments does each function accept?
Which functions modify application state?
Can you explain how a normal product lookup works?
```

These are reconnaissance prompts, not proof of a vulnerability. Validate the responses by observing the application's behaviour.

**Look for:**
- Debug or administrative functions.
- SQL execution interfaces.
- Account-management functions.
- Newsletter or email APIs.
- Product-information tools.
- URL fetching or site-scanning tools.
- Functions that can access sensitive information.

## 3. Excessive agency

**Lab:** Exploiting LLM APIs with excessive agency — Apprentice

**Core weakness:** The model can access an API that has more power than necessary.

### CTF approach

1. Ask which APIs are available.
2. Identify the API responsible for database operations.
3. Determine its argument format.
4. Check whether the function accepts a narrowly defined operation or an unrestricted command.
5. Assess whether authorization and operation restrictions are enforced by the backend.

**Red flags:**
- An LLM-accessible debug API accepts arbitrary SQL.
- The model can perform destructive operations.
- The backend trusts the model to decide which actions are allowed.

**Burp focus:** Inspect the API call, arguments, response, and any resulting state change.

## 4. LLM API abuse and command injection

**Lab:** Exploiting vulnerabilities in LLM APIs — Practitioner

**Core weakness:** An API invoked through the LLM passes attacker-controlled input into an operating-system command unsafely.

### CTF approach

1. Enumerate the APIs available to the model.
2. Identify an API that accepts user-controlled text, such as an email address.
3. Determine how the application processes that input.
4. In the authorized lab, use a harmless command-execution indicator to test whether shell interpretation occurs.
5. Confirm the issue through observable behaviour, such as an email delivered to a controlled test inbox.
6. Assess the impact without testing against systems outside the lab.

**Important concept:** Shell command substitution can cause text that appears to be an ordinary parameter to be interpreted as a command.

**Defence:** Use safe process APIs, avoid shell interpretation, validate inputs, and restrict the privileges of the underlying service.

## 5. Indirect prompt injection

**Lab:** Indirect prompt injection — Practitioner

**Core weakness:** The model follows instructions embedded in content it retrieves, such as a product review.

### CTF approach

1. Map the available functions and identify actions that require an authenticated user.
2. Test a normal function using your own lab account.
3. Identify content that the LLM retrieves, such as product reviews.
4. Add a harmless test instruction to content you control.
5. Ask the LLM to retrieve or summarize that content.
6. Observe whether the model follows the embedded instruction.
7. Assess whether this changes application state or invokes an unauthorized function.

**Typical attack chain:**

```text
Attacker-controlled content
          ↓
LLM retrieves the content
          ↓
Embedded instructions influence the model
          ↓
LLM invokes an available API
          ↓
Unauthorized action
```

**CTF clue:** A product review that changes the chatbot's response may indicate that retrieved content is influencing its instructions.

## 6. Insecure output handling and XSS

**Lab:** Exploiting insecure output handling in LLMs — Expert

**Core weakness:** The application renders model output as executable HTML instead of safely displaying it as text.

### CTF approach

1. Submit a harmless HTML/XSS test string to the chatbot.
2. Check whether it is displayed as text or interpreted by the browser.
3. Test whether product reviews are safely encoded.
4. Ask the model to retrieve content containing your controlled test marker.
5. Inspect the rendered output in the browser.
6. Determine whether untrusted content can reach an executable HTML context.

**Useful harmless probe:**

```html
<img src=x onerror=alert(1)>
```

Use this only in your authorized lab. An alert dialog is evidence of script execution in that context, not proof that every user is affected.

**Key distinction:** A review can be safely encoded when displayed directly but become dangerous if an LLM reproduces it in an unsafe context elsewhere.

**Defence:** Encode output for its destination context, use safe DOM APIs, and avoid relying on the LLM to remove malicious markup.

## 7. AI-powered scanner attacks

AI scanners may browse user-generated content while holding authenticated credentials.

### Important lab themes

| Lab | Core concept |
|---|---|
| Exploiting AI agents to perform destructive actions | Prompt injection leading to unauthorized state changes |
| Exploiting AI agents to exfiltrate sensitive information | Abusing access to private data |
| Exploiting AI agents to trigger secondary vulnerabilities | Prompt injection leading to SSRF |
| Bypassing AI scanner defenses | Testing the limits of prompt-injection defences |

### CTF approach

1. Identify what content the scanner visits.
2. Determine which credentials and resources it can access.
3. Check whether page content affects its behaviour.
4. Inspect scan requests, results, and observable side effects.
5. Assess whether the scanner accesses data or destinations outside its intended scope.

**Defence:** Use isolated credentials, restrict network access, apply least privilege, and treat all scanned content as untrusted.

## 8. Common CTF vulnerability chains

### Chain A — Excessive agency

```text
Chatbot → Powerful API → Missing restrictions → Unauthorized action
```

### Chain B — Indirect prompt injection

```text
Malicious review → LLM retrieves review → Tool call → State change
```

### Chain C — LLM output XSS

```text
Malicious content → LLM repeats content → Unsafe HTML rendering → XSS
```

### Chain D — AI scanner SSRF

```text
Malicious page content → Scanner follows instructions
→ Server-side request → Internal resource access
```

These are conceptual patterns. Verify each stage before reporting a finding.

## 9. Burp Suite checklist

- [ ] Capture the chatbot's requests in Proxy → HTTP history.
- [ ] Identify API endpoints, parameters, and conversation identifiers.
- [ ] Send relevant requests to Repeater.
- [ ] Compare normal and test inputs.
- [ ] Inspect function calls and their arguments where observable.
- [ ] Identify state-changing operations.
- [ ] Check whether server-side authorization is enforced.
- [ ] Test untrusted retrieved content with a harmless marker.
- [ ] Inspect HTML encoding and browser rendering.
- [ ] Record observable evidence of the impact.

## 10. Quick reference

| Finding | What to investigate |
|---|---|
| Excessive agency | Powerful tools with insufficient restrictions |
| Arbitrary SQL | Debug APIs accepting unrestricted SQL statements |
| Command injection | User input interpreted by a shell |
| Indirect prompt injection | Retrieved content influencing tool use |
| XSS through LLM output | Unsafe rendering of model-generated HTML |
| Sensitive-data exposure | Access to another user's private information |
| AI scanner SSRF | Untrusted content influencing server-side requests |

## 11. Further reading

- [PortSwigger Web LLM Attacks](https://portswigger.net/web-security/llm-attacks)
- [PortSwigger Cross-Site Scripting](https://portswigger.net/web-security/cross-site-scripting)
- [PortSwigger OS Command Injection](https://portswigger.net/web-security/os-command-injection)
- [PortSwigger Server-Side Request Forgery](https://portswigger.net/web-security/ssrf)

**Scope:** Use these techniques only in authorized CTFs, PortSwigger Academy labs, and systems where you have explicit permission.