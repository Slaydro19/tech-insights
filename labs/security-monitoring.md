---
title: "From Raw Logs to Actionable Alerts: Building a SIEM Pipeline That Doesn't Cry Wolf"
category: Homelab / Security Operations
reading_time: 5 minutes
---

# From Raw Logs to Actionable Alerts: Building a SIEM Pipeline That Doesn't Cry Wolf

## Context

A SIEM that generates alerts nobody trusts is worse than no SIEM: analysts stop reading them. I built a Splunk-based monitoring pipeline alongside Snort IDS in my homelab specifically to work through that problem, getting from raw log ingestion to alerts worth acting on, including the tuning work that most tutorials skip.

## Method

- **Centralized log ingestion first, correlation second.** Got multi-source logs (network, host, application) into Splunk with consistent parsing before attempting any correlation logic. Correlation queries against inconsistently-normalized data produce false confidence, not real detection.
- **Ran controlled attack simulations** (Metasploit, Infection Monkey) against the monitored environment specifically to validate that the detection pipeline actually caught what it was supposed to catch, testing the detector, not just the target.
- **Tuned Snort rules iteratively against false-positive volume**, not just detection coverage. A rule that fires on every legitimate scan is functionally useless even if it also catches the malicious ones, because it trains analysts to ignore it.
- **Built dashboards around specific operational questions** (e.g., "show me anomalous auth patterns in the last 24h") rather than generic "everything" dashboards that require an analyst to already know what they're looking for.

## Finding

The gap between "logs are being collected" and "an analyst can actually act on an alert" is almost entirely tuning work: false-positive reduction, log normalization, and dashboard design aimed at a specific question. That gap is invisible in a demo (where everything is clean, synthetic data) and is exactly where a real SOC lives.

## Skills Demonstrated

- SIEM deployment and log correlation (Splunk): ingestion, normalization, multi-source correlation
- IDS rule tuning (Snort) with false-positive reduction as an explicit, measured goal, not just adding more rules
- Validating detection capability through controlled attack simulation, rather than assuming coverage from configuration alone
- Designing alerting/dashboards around specific analyst questions, informed by incident-triage and escalation workflow design

**Tools:** Splunk (SIEM), Snort (IDS/IPS), Metasploit and Infection Monkey (controlled attack simulation), Wazuh, Security Onion, TheHive.
