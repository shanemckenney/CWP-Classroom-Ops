# SECURITY.md — CWP Classroom Ops

This document describes the security controls applied to this project
under a Secure Software Development Lifecycle (SSDLC) approach.
It is intended as both a reference for maintainers and a teaching
artifact for CWP students.

---

## Threat Model

| Actor         | Capability                                    | Motivation              |
|---------------|-----------------------------------------------|-------------------------|
| Student       | Browser DevTools, JS console, network tab     | Inflate score, see PIN  |
| External user | Finds Firebase URL in source or GitHub        | Read data, vandalize DB |
| Curious peer  | Views page source on GitHub                   | Extract instructor PIN  |

---

## Controls Applied

### [SEC-01] Instructor PIN — SHA-256 Hash Only
**Threat:** PIN visible in source code → student accesses instructor dashboard.  
**Control:** The plaintext PIN is never stored in source. Only a SHA-256 hex
digest is embedded. Comparison uses the Web Crypto API (`crypto.subtle.digest`).
A constant-time XOR loop prevents timing-based extraction.  
**Residual risk:** SHA-256 is not a password hashing algorithm (no salt, no
iterations). For a production system, use bcrypt or Argon2 server-side.
For a classroom tool with a short-lived PIN, this is an acceptable control.

### [SEC-02] XSS Prevention — Zero innerHTML on Untrusted Data
**Threat:** Callsign crafted as `<script>` or `<img onerror=...>` injected
via Firebase into instructor dashboard.  
**Control:** All Firebase-sourced values (callsigns, scores) are written
exclusively via `textContent` and `createElement`/`appendChild`.
Two helper functions (`safeText`, `safeMake`) enforce this pattern.
No template literals are injected into `innerHTML` for external data.

### [SEC-03] Content Security Policy (CSP)
**Threat:** Injected scripts, unauthorized outbound connections.  
**Control:** A `<meta http-equiv="Content-Security-Policy">` tag restricts:
- `script-src` to self and `gstatic.com` (Firebase SDK CDN) only
- `connect-src` to Firebase database domains only
- `object-src`, `base-uri`, `form-action` all locked to `none`
**Note:** CSP via meta tag does not cover all vectors (e.g. HTTP headers
are stronger). For GitHub Pages, this is the available option.

### [SEC-04] No Global Scope Exposure
**Threat:** `window.__CWP__` global exposes Firebase config, SDK refs,
and PIN to any script running in the page.  
**Control:** All application logic runs inside an ES module
(`<script type="module">`). Modules are scoped and do not pollute `window`.
The Firebase app instance, database refs, and PIN hash are not accessible
from the browser console.

### [SEC-05] Callsign Validation — Defense in Depth
**Threat:** Path traversal or injection via malformed callsign used as a
Firebase key (Firebase keys cannot contain `.`, `#`, `$`, `/`, `[`, `]`).  
**Control:** Enforced at two layers:
1. **Client-side:** `sanitizeCallsign()` strips invalid characters and
   enforces the regex `/^[A-Z0-9_\-]{2,20}$/` before any Firebase write.
2. **Server-side:** Firebase Rules `.validate` enforces the same regex
   on every write. Client-side bypass does not help an attacker.

### [SEC-06] Score Cap — Client and Server
**Threat:** Student calls `pushPartial(true)` from the console with an
arbitrary score value.  
**Control:**
1. **Client-side:** `Math.min(score, MAX_SCORE)` applied before every write.
   `MAX_SCORE` is computed from the question bank at runtime.
2. **Server-side:** Firebase Rules enforce `score <= 450` (the maximum
   possible for a 3-question bank). Adjust this ceiling in the rules
   when your question bank changes.

### [SEC-07] Submit Lock — Firebase Rules
**Threat:** Student resubmits after refreshing to re-attempt and inflate score.  
**Control:** Firebase Rules include:
```
".write": "!data.exists() || (data.child('submitted').val() !== true)"
```
Once `submitted === true` is written, no further writes to that player node
are accepted. This is enforced server-side regardless of client behavior.

### [SEC-08] Firebase Config Isolation
**Threat:** Firebase API key visible in source is sometimes misunderstood
as a secret. While Firebase client keys are not secrets by design, they
should not be attached to `window` unnecessarily.  
**Control:** Config lives in module scope only. It is not attached to any
global and is not accessible from DevTools console.  
**Note:** Firebase client API keys are intended to be public. Security is
enforced by Firebase Rules, not by hiding the key. This is documented in
Firebase's own security guidance.

### [SEC-09] Subresource Integrity (SRI) — Partial
**Threat:** CDN compromise injects malicious code via Firebase SDK.  
**Control:** ES module import maps with SRI are not yet universally
supported across browsers. The Firebase SDK is loaded from
`gstatic.com` (Google-controlled, HTTPS-enforced).  
**Residual risk:** If SRI is required, migrate to a local npm build
pipeline rather than CDN imports.

---

## Firebase Rules Deployment

Paste `firebase-rules.json` into:  
**Firebase Console → Realtime Database → Rules tab → Edit**

Key rules:
- `control` node: readable by anyone, writable by no one (instructor writes
  through the same open DB but this limits surface area)
- `players/$callsign`: callsign format validated, score capped at 450,
  no rewrite after `submitted === true`
- `answers.$qIndex`: each answer field type-validated server-side

---

## Changing the Instructor PIN

1. Open any browser console and run:
```js
crypto.subtle.digest('SHA-256', new TextEncoder().encode('YOURNEWPIN'))
  .then(b => console.log(
    [...new Uint8Array(b)].map(x => x.toString(16).padStart(2,'0')).join('')
  ))
```
2. Copy the 64-character hex string output.
3. Replace the `PIN_HASH` constant in `cwp-classroom-ops.html`.

---

## Known Residual Risks

| Risk | Severity | Notes |
|------|----------|-------|
| Firebase Rules allow unauthenticated writes to `players/` | Medium | Required for anonymous classroom use. Mitigated by score caps and submit locks. |
| CSP via meta tag weaker than HTTP header | Low | GitHub Pages does not allow custom HTTP headers. Acceptable for this context. |
| No Firebase Authentication | Low | Adding Anonymous Auth would strengthen per-user identity but adds setup friction for a classroom tool. |
| SHA-256 PIN without salt | Low | Acceptable for short-lived classroom sessions. Do not reuse this PIN outside this tool. |

---

## SSDLC Phase Mapping

| Phase | Activity | Implemented |
|-------|----------|-------------|
| Requirements | Threat model documented | ✓ |
| Design | Defense-in-depth layering (client + server) | ✓ |
| Implementation | XSS-safe DOM API, hashed PIN, CSP | ✓ |
| Verification | Firebase Rules enforce server-side constraints | ✓ |
| Release | SECURITY.md, residual risk disclosure | ✓ |
| Maintenance | PIN rotation instructions documented | ✓ |
