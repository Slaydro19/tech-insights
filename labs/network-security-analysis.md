---
title: "Network Traffic Analysis: Finding What a Packet Capture Actually Proves"
category: Homelab / Network Security
reading_time: 5 minutes
---

# Network Traffic Analysis: Finding What a Packet Capture Actually Proves

## Context

I run a segmented homelab (Proxmox + pfSense, isolated VLANs for offensive, defensive, and production traffic) as a controlled environment for practicing the kind of traffic analysis that matters in a real security review: not just "here's a packet capture" but "here's what this capture proves, and here's what it doesn't."

## Method

Working primarily in Wireshark and Nmap, I focused on a few recurring questions rather than tool feature tours:

- **Is this traffic actually protected, or does it just look protected?** Validated SSL/TLS implementations by inspecting the actual handshake and negotiated cipher suite, rather than assuming HTTPS-labeled traffic is automatically safe. Found and documented plaintext HTTP traffic that should have been encrypted, and cases of outdated cipher suites still being offered.
- **Can I tell the difference between a normal scan and reconnaissance?** Ran comparative traffic captures of ordinary network activity against Nmap scans at different intensities, to build an actual baseline for "what does suspicious network behavior look like on this network" instead of relying on a rule-of-thumb.
- **What does an unencrypted protocol actually expose?** Walked DNS-over-UDP traffic to concretely show what's visible to anyone positioned to observe it (query content, timing, no integrity protection), rather than treating "DNS is insecure" as an abstract fact.

## Finding

The most useful outcome wasn't a specific vulnerability. It was a repeatable baseline: a documented sense of what normal traffic looks like on this network, so that a deviation is something I can actually point to and explain, not just a gut feeling.

## Skills Demonstrated

- Deep packet inspection across TCP/IP, HTTP(S), DNS, and ARP, reading protocol behavior directly from captures rather than trusting labels
- SSL/TLS certificate and cipher-suite validation
- Building traffic baselines for anomaly detection, rather than pattern-matching against a generic "bad traffic" list
- Nmap-based reconnaissance and the corresponding blue-team perspective: what that reconnaissance looks like from the defender's side of the capture

**Tools:** Wireshark, Nmap, vulnerability scanners.
