---
title: "Designing a Zero-Friction Kiosk Without Giving Up on Security"
category: Application Security / Full-Stack Engineering
date: 2026-02 to 2026-03
reading_time: 9 minutes
---

# Designing a Zero-Friction Kiosk Without Giving Up on Security

*Built for an internal team at an AI infrastructure startup. Details that could identify the employer, the specific third-party systems involved, or any individual are intentionally generalized or omitted below — the security reasoning and debugging is what's described in detail.*

**TL;DR:** Replaced a manual, slow attendance-logging workflow with a walk-up tablet kiosk backed by a small backend service integrating with an existing internal system. The interesting engineering wasn't the happy path — it was making a deliberately frictionless, no-login kiosk defensible: a threat-modeling pass that found two real vulnerabilities before deployment, a security control that quietly broke a reliability feature for a while without anyone noticing, and correctness bugs that hid behind a UI that looked fine.

## Context

A manual, one-record-at-a-time internal attendance process was slow enough to be a genuine daily time cost. I replaced it with a walk-up kiosk: a tablet in the office lists active team members with a status indicator; a person taps their name, then clocks in or out; a small backend service performs that action against an existing internal system on their behalf, using a single service credential. No individual authenticates directly — that's a deliberate design choice, covered below.

## Threat-Modeling a "Simple" Kiosk

Before building the security layer, I traced what actually travels over the wire rather than assuming a simple app has a simple threat model. That exercise found two real problems in the initial design:

1. **Traffic ran unencrypted on the internal network.** Anyone positioned to observe that segment could read every request and the data coming back in responses.
2. **The endpoint that listed team members had no authentication at all.** Anyone on the network could pull names and live status with a single unauthenticated request.

Both were real, exploitable gaps — the value was in actually tracing the wire protocol instead of reasoning abstractly about the app's intent.

## The Deliberate Tradeoff: No Individual Login, By Design

The obvious fix for the authentication gap is "make people log in." I didn't do that, on purpose. A login screen reintroduces exactly the friction that motivated building the kiosk in the first place — the whole point was walk up, tap, done.

Instead, I used a shared-secret pattern: the client sends a pre-shared key on every request, checked server-side before any action is taken. This was a real, explicit tradeoff, documented as such rather than left implicit: **the system runs on an honor system for individual identity.** Nothing server-side stops someone from tapping a colleague's name instead of their own. For an internal kiosk on a controlled network, sitting behind that shared-secret gate, that was an accepted risk — and the write-up is explicit that this tradeoff would not be acceptable for a system with a different threat model.

**What was hardened instead of individual identity:**
- Every state-changing endpoint required the shared key; only a liveness-check endpoint stayed open.
- An administrative preview endpoint was restricted at the code level to local-host requests only, making it structurally unreachable from the network at all, rather than relying on a second secret that could leak the same way the first one could.
- Standard security headers on every response (clickjacking protection, MIME-sniffing protection, a content-security policy).
- Per-client rate limiting at both the application and reverse-proxy layer.
- The service ran as a non-root container user, so a compromise of the app wouldn't hand over root on the host.
- All secrets were environment-provided, never hardcoded, and excluded from version control.
- The offline queue (below) used parameterized queries, not string-built SQL.

## The Bug That Broke the Safety Net Silently

The core resilience feature was a local queue: if the upstream system was unreachable, an action was saved locally instead of lost, with a background job retrying the sync periodically. After deployment, that safety net was **silently non-functional**, for reasons that never once surfaced as a visible error.

**Cause 1 — the process model.** The queue's initialization and the background retry job both lived inside a code path that only executes when a script is run directly — which never happens when a production-grade WSGI server hosts the app instead, which is how it was actually deployed. The queue was never initialized and the retry job never ran, with nothing on the surface indicating that anything was wrong. Fixed by moving that startup work to run unconditionally at import time, regardless of how the process was launched.

**Cause 2 — file permissions, compounding the first bug.** Once the background job was actually running, the queue still failed with a low-level "can't open database file" error. The non-root container user — the same hardening described above — had no write access to the directory it was mounted against. The fix that made the container more secure (drop root) was exactly what had silently broken the feature meant to keep the system reliable during an outage — a clear illustration of why a security control and a reliability feature need to be tested together, not assumed independent of each other.

**Verification, not assumption:** once both causes were fixed, I proved the whole failover path end-to-end with a deliberate, induced-outage test — intentionally broke connectivity to the upstream system, performed an action and confirmed it queued locally, restored connectivity, and confirmed the queue drained back to empty on its own. That's the difference between "I fixed the bug" and "I confirmed the fix actually closes the loop under the exact condition it exists to handle."

## Correctness Bugs Hiding Behind "It Looks Right on Screen"

Two time-handling bugs shipped because the part a user actually sees was correct, which masked bugs in the part they don't see:

- **Generated reports showed times that were off by several hours.** The tablet's own display was already correct, because the browser itself converts a stored timestamp to local time automatically — but a downstream reporting path used the raw stored value directly, with no conversion. A feature can look completely correct in the one place a non-technical stakeholder actually checks it, while being wrong everywhere else it's used.
- **Period-boundary calculations used the wrong timezone reference**, which meant an event near a boundary could be attributed to the wrong period entirely — a correctness bug with real downstream consequences, not a cosmetic one. Fixed by converting boundary values to the correct reference timezone before using them in a query, rather than assuming a single timestamp field behaved consistently everywhere it was read.

## Finding

The recurring theme across this build: **the system's own visible behavior told a consistent, reassuring story while a real problem sat underneath it** — a dead background thread with no startup error, a queue that looked stuck but was actually just waiting out its own retry interval (a red herring that had to be ruled out before finding the real bugs), a display that looked right while the data behind it was wrong. Trusting the surface-level signal instead of verifying the underlying mechanism was the actual failure mode every time, more than any single technology choice.

## Impact / Skills Demonstrated

- **Performed an actual threat-modeling pass** before deployment — traced real network behavior rather than reasoning abstractly, and found two concrete, exploitable gaps before they shipped.
- **Made an explicit, documented security tradeoff** (shared-secret kiosk auth over individual login) rather than either defaulting to "more security always" or leaving the gap unaddressed — and scoped precisely where that tradeoff is and isn't acceptable.
- **Diagnosed a security control that silently disabled a reliability feature** — recognizing that container-hardening and the failsafe queue needed to be tested together, since one's correct behavior broke the other's assumptions.
- **Distinguished "looks correct" from "is correct"** by chasing a time-handling bug past the one place it was already fixed (the client display) to find where it was still broken (server-side reporting and query logic).
- **Verified resilience empirically**, with a real induced-outage test proving the failover path end-to-end, rather than trusting that the code "should" work.
- **Practiced credential hygiene under pressure** — recognizing when a secret had been exposed during troubleshooting and rotating it immediately, rather than treating exposure as a hypothetical risk.

**Technologies:** Python/Flask-class web backend, embedded SQL datastore, WSGI production serving, containerized deployment with non-root hardening, REST integration with an internal system of record, transactional email for alerting, rate limiting, security headers, timezone-safe datetime handling.
