# Race Conditions — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional (Repeater Parallel Send, Turbo Intruder)  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Race conditions occur when websites process concurrent requests without adequate concurrency controls (locks, mutexes, transactions), leading to Time-of-Check to Time-of-Use (TOCTOU) flaws.

```
Identify Collision Window (Sub-second processing gap)
  ├── Limit Overruns: Applying coupon, redeeming gift card, voting, rating
  ├── Multi-Endpoint: Modifying cart/address while payment is in-flight
  └── State Desync: Password reset token generation, user registration
          │
Prepare Synchronized Parallel Requests
  ├── Method A: Burp Suite Repeater (Send group in parallel - HTTP/2 single-packet)
  └── Method B: Turbo Intruder (gate mechanism)
          │
Execute Attack & Analyze Responses
  ├── Check for multiple successes (200 OK on multiple coupon redemptions)
  └── Detect state desynchronization
```

---

## 2. Burp Repeater Parallel Engine (Single-Packet Attack)

Modern HTTP/2 multiplexes multiple requests onto a single TCP connection. Burp Repeater's parallel engine can synchronize requests down to sub-millisecond precision by sending them within the **exact same TCP packet**.

### Step-by-Step Setup:
1. Identify the target request (e.g. `POST /cart/coupon` with `csrf=...&coupon=PROMO20`).
2. Send the request to **Repeater**.
3. Duplicate the tab 20–30 times (`Ctrl+R` repeatedly).
4. In Repeater, click the **+** (plus icon) next to the tabs → **Create tab group**.
5. Select all cloned tabs and create the group.
6. Click the dropdown arrow next to the **Send** button and select:
   **Send group in parallel (single-packet attack)**.
7. Click **Send group**.
8. Inspect the response table: If multiple requests return `200 OK` (coupon applied), the limit overrun succeeded.

---

## 3. High-Yield CTF Scenarios

### 1. Limit Overruns (Coupon / Gift Card Reuse)
- **Vulnerability:** Application checks:
  1. `if (!coupon_used) { apply_discount(); set_coupon_used(true); }`
- **Exploitation:** Send 25 parallel coupon application requests using the single-packet technique. All 25 requests pass the `if (!coupon_used)` check simultaneously before the database updates `coupon_used = true`.

---

### 2. Multi-Endpoint Race Conditions
Occurs when distinct endpoints interact with shared state (e.g. cart items and order payment).

- **Workflow:**
  - Endpoint 1: `POST /cart/checkout` (Calculates cost for cheap item, initiates order)
  - Endpoint 2: `POST /cart` (Adds expensive target item)
- **Single-Packet Attack:**
  - Tab 1: `POST /cart/checkout`
  - Tab 2: `POST /cart` (`productId=1` [Expensive Item])
  - Put both in a Repeater group, send in parallel.
  - If Tab 2 executes between Tab 1's price calculation and final fulfillment, the expensive item is included in the order for the cheap item's price.

---

### 3. Single-Endpoint Race Conditions (Token Collision)
- **Vulnerability:** Password reset endpoint generates and stores tokens in temporary shared variables before writing to user records:
  ```http
  POST /forgot-password HTTP/2
  username=carlos
  ```
- **Exploitation:**
  - Tab 1: `POST /forgot-password` with `username=carlos`
  - Tab 2: `POST /forgot-password` with `username=wiener`
  - Send in parallel.
  - The server generates Carlos's token, but the concurrent execution for Wiener overwrites the active variable or binds the token to both accounts.
  - Check Wiener's inbox: the received token works to reset Carlos's password.

---

### 4. Partial Construction Race Conditions
During user registration, the application inserts the user record with default administrative permissions, and only revokes them in a subsequent step:
- **Exploitation:** Send user registration request concurrently with an immediate API request using the newly registered credentials. If executed within the sub-millisecond gap, administrative API actions succeed.

---

## 4. Turbo Intruder Script (Race Engine)

When you need high concurrency or automated iteration:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2  # Enables HTTP/2 single-packet attack
                           )

    # Queue 30 requests behind a gate
    for i in range(30):
        engine.queue(target.req, gate='race1')

    # Release all requests simultaneously
    engine.openGate('race1')

def handleResponse(req, interesting):
    table.add(req)
```

---

## 5. CTF Quick Reference

| Flaw | Target Endpoints | Parallel Setup | Indicator |
|---|---|---|---|
| **Limit Overrun** | `POST /cart/coupon` | 20x cloned tabs | Multiple `200 OK` applying discount |
| **Multi-Endpoint** | `/checkout` + `/cart` | Mixed tabs in 1 group | Target item acquired for base price |
| **Reset Collision**| `/forgot-password` | Victim tab + Attacker tab | Attacker receives valid token for victim |
| **Bypass Rate Limit**| `/login` | 20x brute-force attempts | Passwords tested without triggering lockout |

---

## 6. Remediation

- **Database-Level Atomic Transactions & Locks:** Use row-level locking (`SELECT ... FOR UPDATE` in SQL) or atomic update operations (`findAndModify` in MongoDB).
- **Idempotency Keys & Mutual Exclusion:** Enforce distributed locks (e.g. Redis `SETNX`) around critical transaction blocks.
- **Synchronize Shared State:** Eliminate shared global/session variables during multi-step asynchronous operations.
