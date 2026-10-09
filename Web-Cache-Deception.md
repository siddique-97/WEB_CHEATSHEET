# Burp Suite — Web Cache Deception

**Difficulty:** Apprentice → Expert  
**Tool:** Burp Suite Community / Professional  
**Environment:** PortSwigger Web Security Academy

## 1. What is Web Cache Deception?

Web Cache Deception (WCD) occurs when a cache and an origin server interpret the same URL differently. An attacker may cause a cache to store a victim's sensitive, personalized response under a URL that the cache considers static or cacheable.

**Core idea:** Make the origin return a private response while making the cache believe the URL is safe to store.

## 2. Testing workflow

1. Log in to your own authorized test account.
2. In **Proxy → HTTP history**, identify a sensitive endpoint, such as `/my-account`.
3. Send the request to **Repeater** and record the normal response.
4. Test path mapping, delimiters, and normalization differences.
5. Identify cache rules by testing likely static extensions or directory prefixes.
6. Compare responses and cache headers across repeated requests.
7. In a designated Academy lab, validate whether a victim's response can be cached.
8. Document the vulnerable URL pattern, impact, and remediation.

## 3. Important response indicators

| Indicator | Meaning |
|---|---|
| `X-Cache: miss` | Often indicates the response was not served from an existing cache entry. |
| `X-Cache: hit` | Often indicates the response came from the cache. |
| `Cache-Control: max-age=30` | Indicates a freshness lifetime of 30 seconds, subject to cache behavior. |
| `404 Not Found` | May indicate that the origin did not recognize the modified path. |
| `200 OK` | The request succeeded, but this alone does not prove a vulnerability. |

**Important:** Header names and behavior vary by implementation. Confirm caching with repeated requests and response-body comparisons.

## 4. Path mapping discrepancy

**Lab:** Exploiting path mapping for web cache deception  
**Difficulty:** Apprentice

The origin server maps additional path segments to the same underlying endpoint, while the cache treats the full URL as a separate cacheable resource.

Test in Repeater:

```http
GET /my-account HTTP/2
```

Then try:

```http
GET /my-account/abc HTTP/2
GET /my-account/abc.js HTTP/2
```

**Look for:**
- The modified path still returns personalized account data.
- A static extension such as `.js` triggers cache behavior.
- Repeating the request changes `X-Cache` from `miss` to `hit`.

**Lesson:** The origin may map `/my-account/abc.js` to `/my-account`, while the cache treats it as a static file URL.

## 5. Path delimiter discrepancies

**Lab:** Exploiting path delimiters for web cache deception  
**Difficulty:** Practitioner

Different components may interpret characters such as `;` and `?` differently.

Use a baseline request:

```http
GET /my-accountabc HTTP/2
```

Test delimiter candidates in Burp Intruder:

```http
GET /my-account§§abc HTTP/2
```

Replace the payload position with candidate characters from the authorized lab's delimiter list. Disable automatic URL encoding when testing literal delimiters.

Then test whether a static extension triggers caching:

```http
GET /my-account;wcd.js HTTP/2
```

**Look for:**
- A delimiter causes the origin to return the account page.
- The cache does not treat that delimiter as a path boundary.
- The resulting URL is cached under a static-extension rule.

**Lesson:** A delimiter discrepancy can make one URL identify two different resources to the cache and origin.

## 6. Origin server normalization

**Lab:** Exploiting origin server normalization for web cache deception  
**Difficulty:** Practitioner

The origin resolves encoded path segments, while the cache may apply a different normalization process.

Example test paths:

```http
GET /aaa/..%2fmy-account HTTP/2
GET /resources/aaa HTTP/2
GET /resources/..%2fmy-account HTTP/2
```

**Testing approach:**
1. Check whether the origin resolves the encoded dot-segment.
2. Identify a cacheable directory, such as `/resources`.
3. Compare normal and modified paths.
4. Repeat requests to confirm cache behavior.

**Lesson:** If the origin resolves a path to a private endpoint but the cache applies a static-directory rule to the original path, personalized content may be cached.

## 7. Cache server normalization

**Lab:** Exploiting cache server normalization for web cache deception  
**Difficulty:** Practitioner

Here, the cache normalizes paths in a way that differs from the origin.

Example normalization test:

```http
GET /aaa/..%2fresources/YOUR-RESOURCE HTTP/2
```

Compare it with:

```http
GET /resources/..%2fYOUR-RESOURCE HTTP/2
```

**Look for:**
- A modified path produces a cache miss followed by a cache hit.
- Moving the encoded dot-segment changes the caching behavior.
- The response body and origin status help distinguish the cache rule from origin routing.

**Lesson:** Test normalization both before and after a suspected cacheable directory. A cache hit alone is not enough to identify the exact rule.

## 8. Exact-match cache rules

**Lab:** Exploiting exact-match cache rules for web cache deception  
**Difficulty:** Expert

An exact-match rule may cache a particular filename, such as `/robots.txt`, rather than every file in a static directory.

Start with a known endpoint:

```http
GET /my-account HTTP/2
```

Test a known cacheable filename:

```http
GET /robots.txt HTTP/2
GET /aaa/..%2frobots.txt HTTP/2
```

Then investigate how the origin interprets delimiters and how the cache normalizes paths.

**Look for:**
- The exact filename is cached.
- An encoded dot-segment changes the cache's interpretation of the URL.
- A crafted path can return personalized content while matching the cache's filename rule.

**Lesson:** Exact-match rules require identifying the filename the cache recognizes and how path normalization affects that match.

## 9. Burp Suite quick checklist

- [ ] Capture the normal authenticated request in Proxy.
- [ ] Send it to Repeater.
- [ ] Record the normal response body and headers.
- [ ] Test extra path segments.
- [ ] Test static extensions such as `.js` and `.css`.
- [ ] Test likely cacheable directory prefixes.
- [ ] Use Intruder to test delimiter candidates.
- [ ] Test encoded dot-segments such as `..%2f`.
- [ ] Compare normalization before and after a cacheable prefix.
- [ ] Repeat requests and compare cache headers.
- [ ] Check whether responses contain user-specific information.
- [ ] Confirm impact only within an authorized lab.
- [ ] Record the working URL pattern and remediation.

## 10. Common mistakes

- Assuming every `200 OK` response is cached.
- Treating `X-Cache: hit` as proof of a security vulnerability.
- Forgetting that browsers may interpret characters such as `#` before sending a request.
- Testing only one URL normalization pattern.
- Ignoring cache freshness and previously stored responses.
- Confusing an origin routing result with a cache rule.
- Testing against real users or systems without authorization.

## 11. Remediation

For developers and defenders:

- Avoid caching authenticated or personalized responses by default.
- Configure cache keys and origin routing to use consistent URL normalization.
- Apply cache rules to validated routes rather than ambiguous filename patterns.
- Prevent sensitive responses from being stored by shared caches.
- Use appropriate directives, such as `Cache-Control: private, no-store`, where sensitive responses must not be cached.
- Test CDN, reverse-proxy, framework, and origin behavior together.

## References

- [PortSwigger — Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [PortSwigger — Web Cache Deception Labs](https://portswigger.net/web-security/web-cache-deception#labs)
- [PortSwigger — CSRF](https://portswigger.net/web-security/csrf)
- [PortSwigger — Delimiter List](https://portswigger.net/web-security/web-cache-deception/wcd-lab-delimiter-list)

---

**Disclaimer:** For educational use and authorized security testing only. Perform exploitation and victim-simulation steps exclusively in PortSwigger Academy or systems for which you have explicit permission.