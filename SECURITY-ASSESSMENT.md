# Security assessment — ceseagencia.com

A hands-on security review of the CeSeA Agenc.ia website, from reconnaissance
through remediation to verification in production.

| Field | Detail |
|-------|--------|
| **Target** | `https://ceseagencia.com` — Next.js 15 marketing site + lead-capture backend |
| **Type** | White-box review — full source access (site built by the author) |
| **Standard** | OWASP Top 10 (2021), OWASP Secure Headers Project |
| **Date** | 16 July 2026 |
| **Outcome** | 3 findings, all remediated and verified live, plus additional hardening |

---

## Executive summary

The data layer was already solid: schema validation, rate limiting, output
encoding and server-side secret handling were all in place. What the site lacked
was an HTTP security-header baseline, and it carried two exposure
misconfigurations.

I reviewed the source and the running site, confirmed each issue by hand,
remediated all of them, and verified the fixes against the production
deployment. The main change is the header layer: the site went from no security
response headers to a strict, first-party Content-Security-Policy and a full
header baseline, applied without breaking a single page. It also closed two
exposure issues: an open image-optimiser proxy, and a set of internal decks that
were reachable by URL.

| # | Finding | Severity | OWASP 2021 | Status |
|---|---------|----------|------------|--------|
| 1 | No HTTP security headers (CSP, HSTS, framing, MIME…) | Medium | A05 | ✅ Remediated |
| 2 | Internal client-facing decks reachable by URL, no access control | High | A01 | ✅ Remediated |
| 3 | Image optimiser configured as an open proxy | Medium | A10 / A05 | ✅ Remediated |

Two lower-value items were folded into hardening rather than filed as findings —
a framework-disclosure header and missing `Retry-After` on throttled responses
(both below).

---

## Scope & methodology

**In scope:** the public marketing site, its two lead-capture route handlers,
the static internal decks, and the application/build configuration.

**Out of scope:** infrastructure and network penetration testing, the hosting
control plane, and third-party providers.

**Approach:** manual source review plus a heuristic static pass mapped to the
OWASP Top 10 (2021), followed by runtime verification against a production
build — response headers inspected on the wire, CSP violations captured in a
headless browser across representative pages, and access-control behaviour
exercised with and without credentials. Two scanner alerts were triaged as false
positives (documented at the end).

---

## Findings

### 1 · No HTTP security headers — Medium — A05 (Security Misconfiguration)

The site returned none of the standard security response headers: no
Content-Security-Policy, no HSTS, no `X-Frame-Options`/`frame-ancestors`, no
`X-Content-Type-Options`, no `Referrer-Policy`, no `Permissions-Policy`. That
left it without defence-in-depth against cross-site scripting, clickjacking,
MIME-type sniffing and protocol downgrade.

**Remediation.** Added a full header baseline in the framework configuration,
including a strict, first-party CSP. The site loads no third-party scripts,
analytics, embeds or CDN fonts (fonts are self-hosted), so the policy locks down
to `'self'`. `script-src` keeps `'unsafe-inline'` for one reason: the marketing
pages are statically generated, so there is no per-request response on which to
mint a nonce, and the framework's inline bootstrap scripts change hash between
builds, which makes a hash-based `script-src` brittle to maintain. It is a
deliberate trade-off with low residual risk here — the site renders no
user-controlled HTML, and its structured-data blocks are serialised with `<`
escaped so they cannot break out of their script context. `'unsafe-eval'` is not
allowed.

Deployed policy (verified live):

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline';
  style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self';
  connect-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self';
  frame-ancestors 'none'; upgrade-insecure-requests
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), browsing-topics=()
Cross-Origin-Opener-Policy: same-origin
```

**Verification.** Confirmed on the production domain: the full header set is
present, exactly **one** CSP header per response (the reverse proxy injects none
of its own), and no CSP violations while loading the home, contact, case, sector
and methodology pages in a headless browser.

### 2 · Internal decks reachable by URL — High — A01 (Broken Access Control)

A set of internal, client-facing proposal pages was served as static files under
a predictable path, with no access control and no `robots` restriction. The
content was confidential business material; anyone who knew or guessed a URL
could read it, and crawlers were not told to stay away.

**Remediation — defence in depth:**

- `robots` disallow **and** an `X-Robots-Tag: noindex, nofollow` response header,
  so the pages are never indexed.
- A **middleware access gate**. The pages are served only when the request
  carries a valid token (`?k=…`). On success the gate sets a short-lived
  `HttpOnly`, `Secure`, `SameSite=Lax` cookie so the page's own assets load;
  every other request gets a flat `404`, so the resource's existence is never
  confirmed and the path cannot be enumerated. The gate **fails closed**: with no
  token configured in the environment, it serves nothing.

**Verification.** Six cases exercised: no token → `404`; valid token → `200` +
cookie; asset with cookie → `200`; asset without cookie → `404`; wrong token →
`404`; the rest of the site → unaffected.

### 3 · Image optimiser as an open proxy — Medium — A10 (SSRF) / A05

The framework's image optimiser was configured to accept **any** HTTPS host (a
wildcard remote pattern). All imagery on the site is first-party, so this served
no purpose and effectively made the optimiser an open image proxy — a
bandwidth-abuse vector, and a limited server-side request forgery surface, since
the server would fetch arbitrary remote URLs on request.

**Remediation.** Removed the remote pattern entirely. The optimiser now processes
local assets only.

---

## Hardening applied

Two lower-value items were fixed in the same pass:

- **Framework disclosure.** Responses advertised the stack through `X-Powered-By`,
  a free fingerprint for an attacker. Header removed.
- **`Retry-After` on throttled responses.** Both lead-capture endpoints already
  rate-limit by IP and return `429`; they now also send `Retry-After` so
  well-behaved clients back off instead of hammering.

---

## Controls already in place

The data layer was in good shape before this review, and it is worth recording:

- **Strict input validation** (Zod) on both endpoints — typed fields, bounded
  lengths, and an explicit consent gate; malformed input is rejected with `400`.
- **Per-IP rate limiting** returning `429` on both mutating endpoints.
- **Output hygiene** — every dynamic value is HTML-escaped; transactional email is
  normalised to numeric entities for safe cross-client rendering; JSON-LD is
  serialised with `<` escaped to prevent script-context breakout.
- **Server-side secrets** — the privileged datastore key never reaches the client
  bundle; credentials live only in the environment.
- **Graceful degradation** — notification email, confirmation email and datastore
  writes run independently, so one failing dependency never drops a lead.

---

## Residual risk & recommendations

- **Rotate** the transactional-email credential and manage it through the
  platform's secret store (operational; owner action).
- Move rate-limit state to a **shared store** (e.g. Redis) if the app scales
  beyond a single instance, so limits hold across replicas.
- If the framework's inline scripts gain stable per-build hashes, or the site
  moves to per-request rendering, tighten `script-src` to a **hash- or
  nonce-based** policy and drop `'unsafe-inline'`.
- Keep the application **source repository private** — it documents deployment
  internals that do not belong in public.

---

## Triaged false positives

- The static pass flagged every `dangerouslySetInnerHTML`. Each one renders
  JSON-LD structured data built from static, first-party content and serialised
  with `<` escaped — not attacker-controlled HTML. Left as-is (it is the
  framework's standard pattern) and now additionally covered by the CSP.
- The static pass reported the lead endpoints as un-rate-limited. They **are**
  rate-limited; the heuristic simply doesn't recognise the in-memory
  implementation. Confirmed by reading the code and by observing `429` responses.

---

Assessed and remediated by **Daniel Brosed** — developer & security auditor
[danielbrosed.com](https://danielbrosed.com)

_Findings were remediated and verified against the production deployment.
Methodology aligned to the OWASP Top 10 (2021)._
