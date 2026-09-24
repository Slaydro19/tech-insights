---
title: "Tracing a Silent VLAN Drop Through Proxmox, pfSense, and a SIEM Agent"
category: Homelab / Network Engineering
date: 2026-01
reading_time: 7 minutes
---

# Tracing a Silent VLAN Drop Through Proxmox, pfSense, and a SIEM Agent

*Network design adapted from a reference architecture by [Gerard O'Brien](https://www.youtube.com/watch?v=XIvn0ZDSmKA&list=PL3ljjyal211AbTqlxSo6CGBiVqsXw8wrp). The build, debugging, and fixes below are my own.*

**TL;DR:** A VM on a VLAN-tagged bridge failed to boot, then failed to get a DHCP lease, even though every visible pfSense and Proxmox setting was correct. Tracing the frame hop-by-hop with `tcpdump` found the real fault: stale tap-interface state on the firewall VM that a config review couldn't reveal — fixed with a VM restart, then made durable so it would survive a reboot. Getting the Kali VM's activity into the SIEM was a second, unrelated failure chain (agent packaging incompatibility, then a hand-edited XML syntax error), isolated and fixed independently. Full trace below.

## Context

I built a segmented cybersecurity homelab on Proxmox — a pfSense firewall/router, multiple VLANs for trust-zone separation, and a tool stack (Kali, Wazuh, Security Onion, TheHive, Cortex, and others) split across them — based on a published reference architecture I adapted to my own hardware and Proxmox version. Standing it up surfaced a chain of problems that had nothing to do with the reference design being wrong, and everything to do with details that don't show up in a walkthrough: bridge configuration defaults, VLAN tagging behavior, and how Proxmox's virtual networking actually moves a tagged frame from a VM to a firewall VM.

**The reference design** (Gerard O'Brien's original architecture and tool stack):

![Reference network and tool-stack design by Gerard O'Brien](images/homelab-reference-design-obrien.png)
*Reference design by [Gerard O'Brien](https://www.youtube.com/watch?v=XIvn0ZDSmKA&list=PL3ljjyal211AbTqlxSo6CGBiVqsXw8wrp) — not my own work, shown for context on what I built from.*

**My build** (IP scheme, VLANs, and DHCP ranges as actually deployed):

![My homelab network topology: Proxmox host, pfSense firewall, and 4 segmented VLANs](images/homelab-network-topology.png)

## The First Failure: a VM That Wouldn't Boot

The first sign of trouble was mundane — a VM on the tools VLAN failed to start, with Proxmox reporting no physical interface on the bridge and a bridge network script failure. A second VM on the same bridge, with no VLAN tag set, started fine. That comparison was the actual diagnostic: the difference wasn't the VM, it was the tag.

The bridge itself wasn't configured as VLAN-aware. Proxmox will silently let you assign a VLAN tag to a VM's virtual NIC without the underlying bridge actually being able to interpret VLAN tags at all — the tag gets attached, but the bridge has no policy for what to do with it, and that mismatch was enough to prevent the VM from starting cleanly. The fix was adding `bridge-vlan-aware yes` to the bridge's definition in `/etc/network/interfaces` and reloading networking. The VM booted.

**What I didn't do:** the easy way out was to just drop the VLAN tag and avoid the problem — the initial suggestion I got when troubleshooting this. I didn't take it, because dropping the tag would have defeated the actual point of the exercise: trust-zone segmentation. A working VM on the wrong network isn't a fix.

## The Second Failure: Booted, No IP

With the VM running, the next problem was that it couldn't obtain a DHCP lease. This forced a methodical check of every layer between the VM and the DHCP server, because the obvious suspects (DHCP scope, firewall rule, VLAN-to-interface mapping) all turned out to be correctly configured:

- DHCP was enabled on the correct VLAN interface with the right address range.
- The firewall rule permitting traffic on that VLAN already existed.
- The VLAN was correctly bound to the firewall's LAN-side trunk interface.

Every piece of pfSense's configuration matched what it should have been. That's the uncomfortable case in networking troubleshooting — when the configuration you can see is correct, and the problem is somewhere you haven't looked yet.

## The Long Packet Trace

At that point I stopped trusting configuration review and started tracing the actual frame, hop by hop, with `tcpdump` at each layer:

1. **On the firewall's LAN interface**, filtering for the VLAN: zero packets arriving. So the problem was upstream of the firewall.
2. **On the Proxmox bridge's VLAN table** (`bridge vlan show`): the VLAN wasn't actually registered as a member of the bridge — everything was showing as untagged, PVID 1 only. I added the VLAN to the bridge and to the VM's tap interface manually.
3. **Still nothing** — even after disabling Proxmox's own per-VM firewall and fully power-cycling the VM (not just a reboot).
4. **A methodology correction along the way:** I'd been running `tcpdump` on the firewall VM itself for a VLAN whose tap interface actually lives on the Proxmox host, not inside the VM. Once I moved the capture to the host side, the traffic became visible.
5. **On the VM's own tap interface:** DHCP requests were leaving the VM — untagged. The VM's config still specified the VLAN tag, but the frame leaving the tap wasn't carrying it.
6. **On the bridge itself:** the same frame now showed up correctly tagged as 802.1Q VLAN traffic. So bridge-level tagging was actually working.
7. **On the firewall's own tap interface:** nothing. The tagged frame reached the bridge and never arrived at the firewall's side of the connection.

That's a genuinely confusing state to be in — every individual link in the chain looked correct in isolation, but the frame still didn't complete the path. The eventual fix was restarting the firewall VM itself, which recreated its own tap interfaces with correct VLAN membership. The interfaces Proxmox had built for that VM at boot time simply hadn't picked up the VLAN configuration changes made after it was already running — a stale-state problem, not a config-correctness problem, which is why reading the config back gave no indication anything was wrong.

**Making it durable:** a manually-run `bridge vlan add` command doesn't survive a reboot, so once the fix was confirmed, I moved it into the bridge's persistent configuration (a `post-up` hook) so a future reboot wouldn't silently regress the whole VLAN back to the same failure.

## The Third Failure: An Agent That Wouldn't Ship Logs

With networking solid, the next gap was operational: the Kali VM's activity wasn't showing up in the SIEM at all. This split into two separate problems that had to be isolated one at a time rather than assumed to be the same root cause:

**Getting a Wazuh agent onto the firewall itself** (to capture firewall-level events, not just the Kali host) turned out to be its own multi-step failure:
- The platform-specific agent package from the OS vendor's repo failed with a library-version mismatch — the firewall's underlying OS release was newer than what the packaged agent expected.
- Downloading the agent package directly from the vendor also failed (access denied on every URL tried).
- Building the agent from source hit a missing compiler, then a missing system header with no available package to provide it — a dead end for that install path on that specific OS version.
- I also caught, before wasting time on it, that installing an older agent version to work around the packaging gap would have created a version mismatch against the SIEM manager it needed to report to — not worth trading one failure for a subtler one.

**The actual fix was to stop trying to install a full agent and use syslog forwarding instead** — a lighter integration the firewall supported natively. That path had its own failure: hand-editing the SIEM manager's XML configuration to add the syslog input introduced small, hard-to-spot syntax errors (a stray closing tag, a malformed attribute, a block placed outside its enclosing section). The service's own error output on restart was vague — effectively just "config didn't load" with a process name and a signal number, no line reference. Running an XML linter against the file directly pointed at the exact malformed lines, which the service's own logging never did.

## Finding

The pattern across every failure here was the same: **the configuration that was visible was correct, and the actual fault was in state that isn't visible from reading a config file** — a bridge that hadn't been told it was VLAN-aware, tap interfaces that had gone stale relative to a config change made after boot, an XML typo a generic error message couldn't localize. None of these were "the tutorial was wrong" problems. They were "the tutorial's environment had different defaults or state than mine" problems, which is a different and more useful thing to get good at diagnosing.

## Impact / Skills Demonstrated

- **Systematic layer-by-layer packet tracing** (`tcpdump` at each hop, `bridge vlan show` for switch-level state) to localize a fault to a specific link in a multi-hop path, rather than guessing at the most likely culprit.
- **Distinguishing configuration-correctness from runtime-state problems** — every config value was right, and the fix was a service restart to reconcile stale interface state, a failure mode that config review alone cannot catch.
- **Isolating compound failures instead of treating them as one problem** — the SIEM-visibility issue was actually two independent failures (agent packaging/build environment, then a hand-edited config syntax error) that needed separate root-causing rather than one fix.
- **Recognizing a dead-end path early** — catching a potential agent/manager version mismatch before installing it, and abandoning a source-build path that had no viable route forward on that OS version, rather than continuing to force a specific solution past the point it made sense.
- **Making fixes durable, not just working** — moving a manually-applied VLAN fix into persistent configuration so it would survive a reboot, instead of leaving a fix that only worked until the next restart.

**Technologies:** Proxmox VE (bridge/VLAN networking), pfSense, 802.1Q VLAN tagging, `tcpdump`, Wazuh (agent and manager), syslog, XML configuration debugging.
