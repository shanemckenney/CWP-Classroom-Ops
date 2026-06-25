# SECURITY.md — CWP Classroom Ops

This document describes the security controls applied to this project
under a Secure Software Development Lifecycle (SSDLC) approach.
It is intended as both a reference for maintainers and a teaching
artifact for CWP students reviewing real-world security decision-making.

---

## Threat Model

| Actor | Capability | Motivation |
|---|---|---|
| Student | Browser DevTools, JS console, network tab | Inflate score, access instructor dashboard |
| External user | Finds GitHub repo or Firebase URL | Read student data, vandalize DB, quota abuse |
| Curious peer | Views page source on GitHub Pages | Extract PIN, understand system behavior |

---

## Controls Applied

### [SEC-01] Instructor PIN — Deterrent Control (Plaintext, Module-Scoped)

**Threat:** Unauthorized student access to instructor dashboard.

**Control:** A plaintext PIN string is compared inside an ES module
(`<script type="module">`). Module scope means the variable is not
accessible from the browser console or `window` object — a student
would need to read the page source to find it.

**Conscious tradeoff:** This is not cryptographic authentication.
A determined student who reads source and finds the PIN string can
access the dashboard. The dashboard shows: aggregate submission
counts, per-question accuracy percentages, and a leaderboard of
callsigns and scores. It does **not** show individual answer
selections or any PII.

**Residual risk:** Low. The worst realistic outcome is a student
launching the session early or resetting scores — disruptive but
recoverable. Firebase Rules are the real security boundary.

**Why not SHA-256 hashing?** A previous version of this tool used
`crypto.subtle.digest` for PIN comparison. The operational overhead
of running a hash function in DevTools to rotate the PIN before
every cohort was incompatible with a 4-nights-a-week teaching
schedule. Pragmatic security decisions require weighing real costs
against realistic risks. This is documented, not hidden.

**Mitigation:** Rotate the PIN each cohort by editing one line.
Do not reuse the PIN outside this tool.

---

### [SEC-02] XSS Prevention — Zero innerHTML on Untrusted Data

**Threat:** Callsign crafted as `<script>alert(1)</script>` or
`<img src=x onerror=...>` injected via Firebase and rendered into
the instructor dashboard's leaderboard.

**Control:** Two helper functions enforce safe DOM rendering throughout:

```js
const safeText = (el, text) => { el.textContent = String(text); };

function safeMake(tag, cls, text) {
  const el = document.createElement(tag);
  if (cls)  el.className = cls;
  if (text !== undefined) el.textContent = String(text);
  return el;
}
```

All Firebase-sourced values (callsigns, scores, reflection text)
are written exclusively via `textContent` or `appendChild`.
No template literals are inserted into `innerHTML` for external data.

**Verified locations:** Leaderboard (student view), leaderboard
(dashboard view), accuracy rows, feedback banners, standby screen,
completion screen, question bank preview panel.

---

### [SEC-03] Content Security Policy — REMOVED

**Status:** Deliberately removed in v3. Documented here as a closed
control with explanation.

**Original intent:** A `<meta http-equiv="Content-Security-Policy">`
tag was implemented to restrict script execution to known origins
and limit outbound connections to Firebase domains.

**Why it was removed:** Firebase Realtime Database uses long-polling
(`.lp?` URLs) as a transport fallback when WebSocket connections are
unavailable. Long-polling works by dynamically injecting `<script>`
tags that load from `https://[project]-default-rtdb.firebaseio.com`.
These requests are blocked by any `script-src` policy that does not
explicitly whitelist the Firebase database domain.

Adding the database domain to `script-src` defeats the purpose of
the policy — it would allow script execution from a domain that
accepts user-controlled data, which is precisely the attack surface
CSP is meant to restrict.

Additionally, GitHub Pages does not support custom HTTP response
headers. A CSP delivered via `<meta>` tag is materially weaker than
a server-enforced `Content-Security-Policy` header: it cannot protect
against certain injection vectors (including `<base>` tag attacks in
some parsers) and is bypassable by content injected before the meta
tag is parsed.

**Residual risk:** Medium. Without CSP, a successful XSS payload
could execute scripts from arbitrary origins. This risk is mitigated
by [SEC-02] (zero innerHTML on untrusted data), which prevents the
injection vector that would make XSS possible in the first place.
Defense-in-depth: the DOM API controls are the primary XSS defense;
CSP would have been a secondary layer.

**What a real fix looks like:** Migrate hosting to a platform that
supports custom HTTP headers (Cloudflare Pages, Netlify, Vercel).
Set `Content-Security-Policy` as a response header. Use Firebase's
WebSocket transport exclusively by ensuring the deployment environment
does not block WSS connections, eliminating the long-polling fallback.

---

### [SEC-04] No Global Scope Exposure

**Threat:** Firebase config, SDK references, and PIN accessible from
the browser console via `window` object inspection.

**Control:** All application logic runs inside `<script type="module">`.
ES modules do not expose their scope to `window`. The Firebase app
instance, database references, and `INSTRUCTOR_PIN` are not reachable
from DevTools console.

**Note on Firebase client config:** Firebase API keys are not secrets
by design. They identify the project but do not grant access.
Security is enforced by Firebase Rules, not by hiding the config.
This is documented in Google's own Firebase security guidance.

---

### [SEC-05] Callsign Validation — Defense in Depth

**Threat:** Firebase key injection via malformed callsign.
Firebase keys cannot contain `.`, `#`, `$`, `/`, `[`, `]`.
A malformed key could corrupt the database structure.

**Control:** Two-layer enforcement:

1. **Client-side** — `sanitizeCallsign()` strips all non-matching
   characters and tests against `/^[A-Z0-9_\-]{2,20}$/` before
   any Firebase write.

2. **Server-side** — Firebase Rules `.validate` enforces the same
   regex on every incoming write, regardless of client behavior.

Client-side bypass does not help an attacker because the server
rule is the authoritative check.

---

### [SEC-06] Score Cap — Client and Server

**Threat:** Student calls Firebase write directly from DevTools
console with `score: 999999`.

**Control:**

1. **Client-side** — `Math.min(score, MAX_SCORE)` applied before
   every write. `MAX_SCORE` is computed at runtime from the
   question bank: `(mc/scenario × 150) + (reflection × 50)`.

2. **Server-side** — Firebase Rules enforce `score <= 1500`
   (maximum for a 10-question bank). The mathematical ceiling
   scales with question count and is enforced regardless of
   what the client sends.

---

### [SEC-07] Submit Lock — Firebase Rules

**Threat:** Student refreshes and resubmits to improve their score
after seeing the correct answers revealed.

**Control:** Firebase Rule on the player node:

```json
".write": "!data.exists() || (data.child('submitted').val() !== true)"
```

Once `submitted: true` is written, no further writes to that
player node are accepted by Firebase. This is enforced server-side
regardless of what the client sends.

---

### [SEC-08] Firebase Config Isolation

**Control:** Firebase config object is declared inside an ES module
and never attached to `window` or any global. It is visible in
page source (as any client-side config must be for a static app),
but is not programmatically accessible from the console.

---

### [SEC-09] Question Bank Validation

**Threat:** Malformed or malicious JSON written to Firebase via the
instructor paste panel, causing the student-facing app to crash or
behave unexpectedly.

**Control:** Client-side validation runs before any Firebase write.
The `validateBank()` function checks:

- Root object structure and required fields
- `ticketMode` is exactly `"opening"` or `"exit"`
- `timerSeconds` is a number between 5 and 120
- `questions` is a non-empty array of 1–10 items
- Each question has a valid `type` field
- MC/scenario questions have 2–4 non-empty options
- `answer` index is within bounds of the options array

Write to Firebase is blocked until validation passes. The Apply
button is disabled until Validate returns clean.

---

## Firebase Rules Summary

See `firebase-rules.json`. Key constraints:

- `control` node: world-readable, not writable by clients
- `questionBank` node: world-readable, not writable by clients
- `players/$callsign`: callsign format validated; score capped at 1500; no rewrite after `submitted === true`
- Individual answer fields: type-validated server-side

---

## Residual Risk Summary

| Risk | Severity | Mitigation |
|---|---|---|
| No Content Security Policy | Medium | [SEC-02] zero innerHTML on untrusted data is the primary XSS defense; CSP removal documented with full rationale above |
| PIN visible in page source to determined student | Low | Module scope limits casual discovery; rotate per cohort; dashboard exposes no sensitive data |
| Firebase Rules allow unauthenticated writes | Medium | Required for anonymous classroom use; score caps, submit lock, and validation limit abuse surface |
| No Firebase Authentication | Low | Anonymous auth would strengthen identity but adds setup friction; Firebase Rules compensate |
| CDN dependency on gstatic.com | Low | Google-controlled, HTTPS-enforced; acceptable for classroom deployment context |

---

## SSDLC Phase Mapping

| Phase | Activity | Status |
|---|---|---|
| Requirements | Threat model documented | ✓ |
| Design | Defense-in-depth layering (client + server) | ✓ |
| Implementation | XSS-safe DOM API, module scope isolation, input validation | ✓ |
| Verification | Firebase Rules enforce server-side constraints | ✓ |
| Release | SECURITY.md, residual risk disclosure, README | ✓ |
| Maintenance | PIN rotation documented; CSP removal rationale documented | ✓ |

---

## Version History of Security Controls

| Version | Change | Reason |
|---|---|---|
| v1 | SHA-256 hashed PIN via Web Crypto API | Maximum PIN secrecy |
| v2 | Plaintext PIN, module-scoped | Operational overhead of hash rotation incompatible with nightly classroom use |
| v2 | CSP meta tag added | Defense against script injection |
| v3 | CSP meta tag removed | Firebase long-polling transport blocked by script-src; meta tag CSP insufficient without server header support; removal documented as residual risk |
| v3 | Question bank moved to Firebase | Eliminates nightly file edits and GitHub push cycle |
| v3 | [SEC-09] Question bank validation added | Prevents malformed JSON from reaching Firebase or crashing student app |
