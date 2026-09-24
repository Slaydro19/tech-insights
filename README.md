# Jalen Salgado — Security & Infrastructure Portfolio

Case studies and technical write-ups from security research, QA methodology, and infrastructure engineering work. Each piece follows the same structure: context, method, finding, and the skills or impact it demonstrates — not a list of tools I've touched.

Company-specific details (internal repo names, endpoints, credentials, architecture diagrams) are intentionally omitted throughout, consistent with confidentiality obligations to past and current employers. Findings are described by vulnerability class and methodology, which is what's actually transferable.

## Case Studies

### Application & AI Security
- [Finding and Closing a Two-Pipe Credential Leak in Application Logs](case-studies/jwt-log-leak-cwe532.md) — CWE-532 log-leak finding and remediation, including catching an earlier fix attempt's incorrect assumption
- [An 8-Category Framework for Auditing LLM-Driven Backend Services](case-studies/security-audit-methodology.md) — security audit methodology adapted for LLM-specific risk (prompt injection, hallucinated structured output)

### QA & Testing Methodology
- [Building a Security-First QA Process for a Multi-Stage Data Pipeline](case-studies/security-testing-methodology.md) — V-Model testing with a mandatory threat-surface step and security escalation path

### Infrastructure & DevOps
- [Adding Non-Blocking Vulnerability Scanning to an Infrastructure Pipeline](case-studies/infrastructure-security-scanning.md) — Trivy-based CI/CD scanning designed for a team without dedicated security staff
- [Resolving Terraform Circular Dependencies: A Real-World AWS EKS Case Study](case-studies/terraform-eks-circular-dependency.md) — diagnosing and fixing a `terraform destroy` deadlock
- [Breaking the Karpenter Bootstrap Deadlock](case-studies/karpenter-bootstrap-deadlock.md) — distinguishing a scheduling issue from a configuration issue in EKS/Karpenter

## Homelab

Hands-on lab work in a segmented Proxmox/pfSense environment I fully control — used to practice the analysis and defense techniques referenced in the case studies above, with full documentation freedom since it's entirely my own infrastructure. The network design is adapted from a reference architecture by [Gerard O'Brien](https://www.youtube.com/watch?v=XIvn0ZDSmKA&list=PL3ljjyal211AbTqlxSo6CGBiVqsXw8wrp); the write-ups below cover my own build, debugging, and configuration work on top of it.

- [Building and Operating a Segmented Homelab](case-studies/homelab-network-lab.md) — VLAN/SIEM debugging traced hop by hop, a remote-access failure under time pressure, and containing an autonomous pentesting agent with a dedicated VLAN
- [Network Traffic Analysis: Finding What a Packet Capture Actually Proves](labs/network-security-analysis.md)
- [Multi-Zone Firewall Design: Segmentation as the Actual Control](labs/firewall-management.md)
- [IPsec vs. OpenVPN: Testing Two VPN Stacks Against the Same Threat Model](labs/vpn-security.md)
- [From Raw Logs to Actionable Alerts: Building a SIEM Pipeline That Doesn't Cry Wolf](labs/security-monitoring.md)

## Essays

- [Securing Agentic AI in Business: Why the TriSM Framework Matters](essays/trism-framework.md) — Trust, Risk, and Security Management framework applied to multi-agent LLM systems

## Other

- [Enterprise Network & Data Center Infrastructure Experience](network-infrastructure-experience.md) — hands-on exposure to enterprise firewall, switching, and server hardware during an internship

## About

Cybersecurity student (Spokane Falls Community College) and QA/Security Intern with hands-on experience across application security, LLM/agentic-system security, QA methodology, and infrastructure observability. Background summary and current focus areas: [linkedin.com/in/jalensalgado19](https://linkedin.com/in/jalensalgado19).

---
*Technical writing and case studies reflecting real project work, generalized to protect employer confidentiality while preserving the actual methodology and findings.*
