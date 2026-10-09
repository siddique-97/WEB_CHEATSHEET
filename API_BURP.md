# Burp Suite — API Testing

**A practical cheat sheet for API endpoint discovery, request manipulation, authorization testing, and common API vulnerabilities using Burp Suite.**

## Contents

1. [Burp Suite Workflow](#1-burp-suite-workflow)
2. [API Endpoint Discovery](#2-api-endpoint-discovery)
3. [HTTP Methods](#3-http-methods)
4. [Parameter Manipulation](#4-parameter-manipulation)
5. [Server-Side Parameter Pollution (SSPP)](#5-server-side-parameter-pollution-sspp)
6. [Mass Assignment](#6-mass-assignment)
7. [BOLA / IDOR](#7-bola--idor)
8. [Authentication and JWT](#8-authentication-and-jwt)
9. [Input Validation and Content Types](#9-input-validation-and-content-types)
10. [CORS and Rate Limiting](#10-cors-and-rate-limiting)
11. [GraphQL Testing](#11-graphql-testing)
12. [Intruder Payloads](#12-intruder-payloads)
13. [Response Analysis](#13-response-analysis)
14. [Testing Checklist](#14-testing-checklist)
15. [References](#15-references)

---

## 1. Burp Suite Workflow

- **Proxy:** Capture and inspect HTTP traffic.
- **Repeater:** Modify requests and test hypotheses.
- **Intruder:** Automate controlled payload testing.
- **Decoder:** Encode and decode input.
- **Comparer:** Identify response differences.

**Workflow:** Capture → Establish baseline → Modify one variable → Compare responses → Verify impact.

## 2. API Endpoint Discovery

Common paths:

```text
/api
/api/v1
/api/v2
/api/users
/api/admin
/api/internal
/api/products
/openapi.json
/swagger.json
/api-docs
```

Review HTTP history, JavaScript files, API documentation, and error messages to discover actual routes.

## 3. HTTP Methods

| Method | Purpose |
|---|---|
| GET | Retrieve data |
| POST | Submit or create data |
| PUT | Replace a resource |
| PATCH | Update a resource |
| DELETE | Delete a resource |
| OPTIONS | Inspect supported methods |

Test relevant methods in Repeater and verify authorization. An allowed method does not automatically indicate a vulnerability.

## 4. Parameter Manipulation

Test query parameters, path identifiers, headers, cookies, and JSON bodies.

```text
?role=user&role=admin
?username=invalid
```

Example JSON variations:

```json
{"id": 1}
{"id": "invalid"}
{"id": null}
{}
```

Investigate duplicate parameters, type confusion, missing values, URL encoding, and input validation.

## 5. Server-Side Parameter Pollution (SSPP)

SSPP occurs when user-controlled input changes a backend request's URL path or query string.

Useful test values in an authorized lab:

```text
administrator%23
administrator%3F
./administrator
../administrator
```

Investigate:

- URL parsing and path normalization.
- API documentation and route templates.
- Unexpected backend parameters.
- Supported fields and API versions.

Reference: [PortSwigger SSPP](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution)

## 6. Mass Assignment

Test whether the application accepts properties that users should not control.

Example candidate fields:

```json
{
  "role": "user",
  "isAdmin": false,
  "discount": 0,
  "verified": false
}
```

Compare normal and modified requests. Confirm whether a restricted property is accepted and affects server-side behavior.

## 7. BOLA / IDOR

Test object-level authorization using accounts and resources you are permitted to access.

```http
GET /api/users/1001 HTTP/1.1
Host: target.example
```

Compare access to another authorized test account's resource. Check whether the server verifies ownership and permissions rather than trusting user-controlled IDs.

## 8. Authentication and JWT

Test:

- Missing, invalid, and expired credentials.
- Access to privileged endpoints.
- Cross-account access.
- JWT signature validation and expiration.
- Issuer, audience, and authorization claims.

Remember: decoding a JWT does not validate its signature.

Reference: [PortSwigger JWT Testing](https://portswigger.net/web-security/jwt)

## 9. Input Validation and Content Types

Test the application's handling of JSON, form data, missing fields, unexpected properties, and invalid data types.

```http
PATCH /api/products/1 HTTP/1.1
Host: target.example
Content-Type: application/json

{"price": -1}
```

Check whether server-side validation enforces the expected type, range, and permissions.

## 10. CORS and Rate Limiting

**CORS:** Inspect `Origin`, `Access-Control-Allow-Origin`, and `Access-Control-Allow-Credentials`. Confirm whether untrusted websites can access sensitive responses.

**Rate limiting:** Check for `429 Too Many Requests`, `Retry-After`, and consistent limits across relevant accounts and endpoints. Use low request rates within the approved scope.

## 11. GraphQL Testing

Common endpoint:

```text
/graphql
```

Example request:

```json
{
  "query": "{ __typename }"
}
```

Investigate schema exposure, query and mutation authorization, excessive data exposure, input validation, and query complexity limits.

## 12. Intruder Payloads

Common parameter names:

```text
id
username
email
role
userId
accountId
field
token
isAdmin
discount
price
```

Use targeted payload lists and compare status codes, response lengths, and response bodies. Validate interesting results in Repeater.

## 13. Response Analysis

| Status | What to investigate |
|---|---|
| 200 | Unexpected data or successful unauthorized action |
| 400 | Input validation |
| 401 | Authentication |
| 403 | Authorization |
| 404 | Missing or concealed resources |
| 405 | Method handling |
| 429 | Rate limiting |
| 500 | Unexpected server error |

A status code alone does not prove a vulnerability. Compare the complete response and resulting application behavior.

## 14. Testing Checklist

- [ ] Discover endpoints and documentation.
- [ ] Test relevant HTTP methods.
- [ ] Examine parameters and input validation.
- [ ] Check object-level and function-level authorization.
- [ ] Investigate SSPP and mass assignment.
- [ ] Review authentication and JWT handling.
- [ ] Inspect CORS, rate limiting, and error messages.
- [ ] Record reproducible evidence and remediation.

## 15. References

- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [PortSwigger API Testing](https://portswigger.net/web-security/api-testing)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)

---

**Disclaimer:** For CTFs, lab environments, and authorized security testing only.