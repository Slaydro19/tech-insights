---
title: "An 8-Category Framework for Auditing LLM-Driven Backend Services"
category: Security Audit / Application Security
date: 2025-08
reading_time: 5 minutes
---

# An 8-Category Framework for Auditing LLM-Driven Backend Services

## Context

I ran third-party and internal security audits against several services at an AI infrastructure startup, including a schema/validation library used as the entry point for LLM-driven "intent" processing — a service that took loosely-structured planning output from an LLM and validated it into a structured execution plan before anything downstream acted on it.

Services shaped like this have an unusual risk profile: the "untrusted input" isn't just the HTTP request, it's also whatever the LLM itself decided to produce, which means the attack surface includes prompt injection and hallucinated/malformed structured output — categories a conventional REST API audit checklist doesn't cover.

## Method

I built and applied a repeatable audit framework in eight categories, each producing evidence-backed findings rather than a pass/fail checklist result:

1. **Attack surface mapping** — every entry point that accepts external or LLM-generated input, including fields with no server/DB of their own (e.g., a pure schema-validation library still has an attack surface: anything that trusts its output without re-validating).
2. **Attack vector enumeration** — SSRF, IDOR, broken access control, and indirect/direct prompt injection assessed against the actual data flow, not a generic checklist applied blindly.
3. **Trust boundary identification** — specifically, where output from one trust domain (LLM-generated structured data) crosses into a domain that assumes validated input, and whether validation is actually enforced at that boundary or just assumed.
4. **Security control inventory** — what's already in place (schema validation, type constraints, allow-lists) versus what's assumed to be handled elsewhere.
5. **Detection and response readiness** — whether a successful exploitation of a finding would actually be observable (logged, alertable) or would pass through silently.
6. **Live verification** — every finding was reproduced through actual testing, not just static code reading. A finding that can't be demonstrated live is a hypothesis, not a finding.
7. **Severity rating and remediation options** — findings included multiple fix approaches with explicit trade-offs (e.g., a quick mitigation vs. a structural fix), so the owning team could make an informed call rather than getting a single prescriptive patch.
8. **Cross-repo follow-through** — because a validation gap in one shared library can propagate into every service that imports it, findings triggered audits of downstream consumers, not just the originating repo.

## Finding

One representative finding: a data model accepted caller-supplied tenant/user identity fields without validating them against the authenticated session's actual identity — meaning a caller could, in principle, supply identity fields that didn't match who they actually were. I explicitly avoided treating the "add a server-side check" fix as adequate on its own without understanding *every* call site that constructed this model, since a partial fix that only covers the obvious entry point leaves the same vulnerability reachable through a less obvious one — the difference between a durable fix and a band-aid.

## Impact / Skills Demonstrated

- Adapted a conventional security-audit framework (attack surface / vectors / trust boundaries / controls) to account for LLM-specific risk categories (prompt injection, hallucinated structured output) that a generic web-app checklist misses.
- Insisted on live reproduction over static analysis alone — a discipline that surfaces false positives before they reach a team's backlog and gives real evidence for prioritization.
- Evaluated fixes for completeness across all call sites, not just the one that triggered the finding, avoiding partial fixes that leave the same class of bug reachable elsewhere.
- Presented findings with multiple remediation paths and explicit trade-offs rather than a single mandated fix, respecting that the owning team has context the auditor doesn't.

**Frameworks referenced:** OWASP LLM Top 10, MITRE ATLAS (adversarial threat landscape for AI systems), applied alongside a conventional attack-surface/trust-boundary audit structure.
