---
title: "Multi-Zone Firewall Design: Segmentation as the Actual Control"
category: Homelab / Network Security
reading_time: 5 minutes
---

# Multi-Zone Firewall Design: Segmentation as the Actual Control

## Context

A flat home network gives you almost none of the constraints a real enterprise network security review cares about. I built a multi-zone pfSense environment (LAN / WAN / DMZ, plus isolated VLANs for offensive-security testing) specifically to practice designing and defending network segmentation, not just clicking through a firewall's rule UI.

## Method

- **Designed the zone architecture first, rules second.** Before writing any firewall rule, I mapped which zones should never be able to initiate contact with which others (e.g., DMZ services should never be able to reach the internal LAN, only respond to what reached them) — the segmentation model, not the rule list, is the actual security control.
- **Applied least-privilege as a default-deny baseline**, then added explicit allow rules per service — rather than starting from an open policy and trying to lock it down after the fact, which reliably leaves gaps.
- **Coordinated firewall rules with VPN policy** so remote-access traffic terminated into the correct zone rather than being implicitly trusted as "internal" once it crossed the VPN boundary.
- **Extended the same segmentation logic to host-based firewalling** (Windows Defender Firewall) for defense in depth — a host inside an already-segmented zone still shouldn't trust every other host in that zone by default.

## Finding

The recurring failure mode I was testing against wasn't "a rule is misconfigured" — it was "the zone model itself has an implicit trust assumption nobody wrote down." A rule set can be internally consistent and still be wrong if the underlying segmentation design assumed the wrong thing about which zones should trust which.

## Skills Demonstrated

- Multi-zone network design (LAN/WAN/DMZ) with segmentation as the primary control, not an afterthought
- Least-privilege, default-deny rule design
- Coordinating firewall policy with VPN and host-based firewall layers for defense in depth
- Documenting rule intent, not just rule syntax, so a rule set stays maintainable by someone other than the person who wrote it

**Tools:** pfSense, Windows Defender Firewall, PowerShell (automation/management).
