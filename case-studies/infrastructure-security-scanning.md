---
title: "Adding Non-Blocking Vulnerability Scanning to an Infrastructure Pipeline"
category: DevSecOps / CI-CD Security
date: 2025-07
reading_time: 5 minutes
---

# Adding Non-Blocking Vulnerability Scanning to an Infrastructure Pipeline

## Context

An EKS-based infrastructure pipeline at an AI infrastructure startup had no automated visibility into vulnerabilities in the container images and Terraform configs it deployed — issues were only found reactively, if at all. The team had no dedicated security function at the time, which meant any solution had to be genuinely zero-maintenance and impossible to accidentally turn into a deployment blocker, or it would get disabled the first time it caused friction.

## Method

- **Chose Trivy for combined image and IaC scanning**, since it could cover both container images (EKS core components, base OS images, GPU/ML runtime images) and Terraform configuration in one tool rather than stitching two separate scanners together.
- **Designed the integration to be explicitly non-blocking**: configured with `exit-code: 0` so a finding reports to GitHub's Security tab without ever failing a deployment. For a team without dedicated security staff, a scanner that can block shipping is a scanner that gets bypassed under time pressure — visibility without veto power was the deliberate trade-off.
- **Scoped what actually needed scanning**: EKS core components (CoreDNS, pause containers), the AWS base images and GPU/ML runtime images actually in use, and the Terraform configs deploying them — rather than scanning everything reachable, which just produces noise.
- **Set severity-based triage guidance** (Critical: fix immediately, High: within a week, Medium: next sprint, Low: excluded from automatic scans entirely) so findings had a clear, proportionate response instead of every finding looking equally urgent.
- **Scheduled scans on every push/PR to main plus a daily run**, so newly-disclosed CVEs in already-deployed images would surface even without a new commit triggering a scan.

## Finding

The main design risk wasn't technical — it was organizational. A security tool with no dedicated owner and the power to block deployments would either get disabled or would train the team to route around it. Making the tool strictly additive (report-only, severity-triaged, low-priority findings filtered out entirely) was what made it sustainable without a dedicated security team to run it.

## Impact / Skills Demonstrated

- Designed a CI/CD security control around the team's actual operating constraints (no dedicated security staff, cannot tolerate blocked deploys), not around an idealized "block everything critical" policy that wouldn't have survived contact with the team.
- Combined container-image and infrastructure-as-code scanning in one pipeline stage rather than treating them as separate concerns.
- Set explicit, proportionate severity-response guidance so the scanning output was actionable rather than just informational noise.
- Practiced right-sizing security tooling to organizational maturity — the correct amount of process for a startup without a security team is not the same as for one with a dedicated SOC.

**Tools:** Trivy, GitHub Actions, GitHub Security tab (code scanning), Terraform, Kubernetes/EKS.
