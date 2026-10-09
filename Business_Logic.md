# Business Logic Vulnerabilities — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, Repeater, Intruder  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

Business logic flaws occur when an application's design or implementation allows an attacker to manipulate legitimate functionality to achieve unintended business outcomes.

1. **Map the Business Workflow:** Identify multi-step workflows (e.g., cart checkout, account registration, coupon application, password reset, role assignment).
2. **Challenge Assumptions:**
   - Can quantities be zero or negative (`-1`)?
   - Can prices or currencies be tampered with directly?
   - Can a multi-step sequence be executed out of order?
   - Can coupons be applied repeatedly or alternated?
   - What happens on integer overflow (values exceeding 32-bit limits)?
   - Can inputs be truncated (e.g. extremely long email addresses)?
3. **Execute & Observe:** Modify parameters in Burp Repeater, observing whether backend logic strictly validates values and workflow states.

---

## 2. Common Attack Patterns & Exploitation

### 1. Excessive Trust in Client-Side Controls (Price Tampering)
The application assumes prices sent in HTTP POST requests are immutable:
```http
POST /cart HTTP/2
Content-Type: application/x-www-form-urlencoded

productId=1&price=1&quantity=1
```
- **Exploit:** Change `price=133700` to `price=1` or `price=0` before submitting.

---

### 2. Negative Quantities (Cart Balancing)
The application validates that individual items have a quantity, but fails to check for negative numbers or enforce a positive total order value:
```http
# Add expensive target product (Total: $1000)
POST /cart HTTP/2
productId=1&quantity=1

# Add negative quantity of cheap item (Price: $10, Quantity: -99, Total discount: -$990)
POST /cart HTTP/2
productId=2&quantity=-99
```
- **Result:** Net cart total is reduced to a nominal positive amount ($10.00), allowing checkout within the user's available store balance.

---

### 3. Workflow Bypass (Skipping Mandatory Steps)
A checkout workflow consists of:
- Step 1: `/cart`
- Step 2: `/cart/checkout` (Calculates cost, reserves funds)
- Step 3: `/cart/order-confirmation` (Dispatches goods)
- **Exploit:** Add items to cart in Step 1, then issue a direct `POST` to Step 3 (`/cart/order-confirmation`), bypassing payment calculation completely.

---

### 4. Flawed Coupon Enforcement (Alternating Coupons)
The application restricts using the same coupon code twice consecutively, but resets its check when a different coupon is applied:
1. Apply Coupon A (`NEWUSER`) → Discount applied.
2. Apply Coupon B (`DISCOUNT20`) → Discount applied.
3. Apply Coupon A (`NEWUSER`) → Accepted!
- Use **Burp Intruder** to alternate coupon requests until the cart total drops to $0.

---

### 5. Integer Truncation & Overflows
The application calculates cart totals using 32-bit signed integers (Maximum: `2,147,483,647`):
- **Exploit:** Add massive quantities of an item (e.g., quantity `214748` of a $10,000 item).
- The total price exceeds the integer boundary, overflowing into negative numbers, or wrapping around to a small positive number.

---

### 6. Email Truncation (Admin Impersonation)
Database truncates email strings at a fixed length (e.g., 255 characters):
- **Target:** `administrator@target.net`
- **Exploitation:** Register with an email crafted so that the domain falls past character 255:
  ```text
  administrator@target.net[230 spaces]@attacker.com
  ```
- Verification email arrives at `@attacker.com`.
- When stored in the database, the string past character 255 is truncated, leaving `administrator@target.net`.

---

### 7. Dual-Use Endpoint Flaws
An account update endpoint allows updating personal details and password simultaneously:
```http
POST /my-account/change-password HTTP/2
Content-Type: application/x-www-form-urlencoded

username=carlos&current-password=oldpass&new-password-1=newpass&new-password-2=newpass
```
- **Exploit:** Delete the `current-password` parameter completely. The backend code branches into a path intended for administrative password resets, changing the password without verification.

---

## 3. CTF Quick Reference

| Logic Flaw | Test Action | Expected Impact |
|---|---|---|
| **Price Tampering** | Modify `price` parameter in POST | Buy item for $0.01 / $0 |
| **Negative Quantity** | Submit `quantity=-10` | Offset expensive items |
| **Step Skipping** | Direct call to `/order-confirmation` | Order items without paying |
| **Coupon Alternation**| Submit Coupon A, B, A, B | Stack infinite discounts |
| **Integer Overflow** | Submit massive integer quantities | Wrap cost to negative/low sum |
| **Field Truncation** | Input string > 255 characters | Truncate account name/email |
| **Missing Parameter**| Omit `current-password` | Bypass authentication checks |

---

## 4. Remediation

- **Never Trust Client-Supplied Business Values:** Always retrieve product prices, discount rules, and tax rates directly from trusted server-side databases.
- **Enforce State Machines:** Track workflow progression in server-side session state. Reject requests to Step N unless Step N-1 has been strictly completed and validated.
- **Strict Boundary & Type Validation:** Enforce positive bounds on numerical quantities (`quantity > 0`). Use arbitrary-precision decimal libraries for financial transactions.
