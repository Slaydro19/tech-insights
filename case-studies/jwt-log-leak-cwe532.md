---
title: "Finding and Closing a Two-Pipe Credential Leak in Application Logs"
category: Vulnerability Research / Application Security
cwe: CWE-532 (Insertion of Sensitive Information into Log File)
date: 2025-08
reading_time: 5 minutes
---

# Finding and Closing a Two-Pipe Credential Leak in Application Logs

## Context

During a security review of a Python (FastAPI) backend service at an AI infrastructure startup, I investigated how the service authenticated WebSocket connections. Browsers cannot set custom headers on a WebSocket handshake, so the service — like many WS-based APIs — passed the session's JWT as a query parameter on the connection URL (`?token=<JWT>`).

That pattern is a known anti-pattern: anything that ends up in a URL tends to end up in places URLs get logged. The question was whether this service actually leaked it, and if so, how badly.

## Method

I traced every logging path the request would pass through before reaching the application's business logic:

1. **The web server's own access logging** (the ASGI server's request-line logger) — logs the full request line, including query string, by default.
2. **The application's own structured logging middleware** — a custom `RequestLoggingMiddleware` that logged request metadata (including query parameters) as structured JSON for observability.

A prior remediation attempt had shipped a fix for pipe 1 only, on the assumption — stated directly in that PR's description — that the application's own logging was "path-only and unaffected." I didn't take that claim at face value; I read the middleware's actual field list and confirmed it captured `query_params` verbatim.

To verify impact rather than just reason about it abstractly, I checked live pod logs in the development cluster and confirmed a real, valid `platform:admin`-scoped JWT was recoverable in plaintext from both log streams via `kubectl logs`.

## Finding

**Two independent logging pipes captured the same secret**, not one:

- **Pipe 1 (access log):** the ASGI server's request-line logger, emitting the full URL including `?token=...`.
- **Pipe 2 (application log):** the app's own structured JSON logger, which independently serialized `query_params` into its output — a path the earlier fix attempt had explicitly (and incorrectly) ruled out.

Sealing only one pipe would have left a live, high-privilege session token recoverable from centralized logging — meaning anyone with log read access (a broader population than "people who can read the database" in most orgs) could impersonate an admin session.

## Fix

Both pipes needed independent remediation, sharing one source of truth so they couldn't drift apart again:

```python
# Single shared list — every redaction point reads from here,
# so there's no way for one pipe to fall out of sync with another.
SENSITIVE_KEYS = ("token", "password", "secret", "key", "authorization")
```

- **Pipe 1:** a `logging.Filter` registered on the access logger, scrubbing sensitive query params before any downstream formatter sees them — registered early enough in the filter chain that no other filter could see the raw value first.
- **Pipe 2:** a dict-comprehension redaction step inside the request-logging middleware itself, applied to `query_params` before they're serialized to the structured log line.
- A third, previously-hardcoded redaction list elsewhere in the request-body logger was repointed at the same shared `SENSITIVE_KEYS` constant, closing a possible future drift point.

**Verification:** re-tested against live logs post-fix; the token field rendered as `token=***REDACTED***` in both the access log and the structured app log, using a synthetic JWT (never a real one) to avoid reintroducing the same class of leak in test artifacts or shell history.

## Impact / Skills Demonstrated

- **Didn't trust an existing fix's own claims** — verified the "unaffected" logging path by reading the actual code path instead of accepting the prior PR's reasoning.
- **Root-caused to the shared-state level**, not just the two symptomatic call sites — the fix removed a whole class of future drift (a third redaction list existed elsewhere and was consolidated at the same time).
- **Verified in a live environment**, not just via code review — confirmed both the vulnerable state and the fixed state against real pod logs.
- Practiced safe verification hygiene: synthetic credentials only, to avoid the exact failure mode under test.

**Vulnerability class:** CWE-532. **OWASP mapping:** relevant to A09:2021 – Security Logging and Monitoring Failures (sensitive data exposure via logs) and A02:2021 – Cryptographic Failures (secret-in-transit-via-URL as the root design issue).
