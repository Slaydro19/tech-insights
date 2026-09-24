---
title: "IPsec vs. OpenVPN: Testing Two VPN Stacks Against the Same Threat Model"
category: Homelab / Network Security
reading_time: 5 minutes
---

# IPsec vs. OpenVPN: Testing Two VPN Stacks Against the Same Threat Model

## Context

Rather than deploy one VPN technology and call it done, I stood up both IPsec (IKEv2, certificate-based auth) and OpenVPN in my homelab specifically to compare them under the same threat model — server deployment, client configuration, and adversarial testing against both.

## Method

- **Built out a PKI for certificate-based authentication** on the IPsec side, rather than relying on pre-shared keys, and validated the full IKE_SA_INIT / IKE_AUTH exchange with Wireshark to confirm the negotiated parameters (AES-256, correct key-exchange behavior) matched what the config claimed.
- **Ran adversarial scenarios against both stacks**: ARP-poisoning/MITM traffic-interception attempts, and VPN-password strength/attack assessment — treating "we deployed a VPN" as a claim to test, not a conclusion.
- **Analyzed ESP (Encapsulating Security Payload) traffic** to confirm actual encryption in transit, rather than trusting the client's "connected, secure" status indicator.
- **Extended into the human layer**, not just the protocol layer: reviewed how VPN-related phishing and social-engineering attempts typically target credential capture, and implemented SPF/DKIM/DMARC on VPN-related communications as a complementary control — a VPN's cryptography doesn't help if the credential was phished before the tunnel was ever established.

## Finding

The most useful comparison wasn't "which VPN is more secure" in the abstract — both, configured correctly, hold up to the traffic-interception testing I ran. The more useful finding was operational: certificate-based IPsec auth removes an entire class of credential-phishing risk that a password-based OpenVPN setup doesn't, at the cost of more complex certificate lifecycle management. That's a real trade-off, not a clear winner.

## Skills Demonstrated

- IPsec (IKEv2) and OpenVPN server/client deployment and configuration, including PKI setup for certificate-based auth
- Validating cryptographic claims empirically (packet-level verification) rather than trusting UI status indicators
- Adversarial testing methodology: MITM/ARP-poisoning scenarios, credential-attack assessment
- Recognizing that VPN security includes the human/credential layer, not just the protocol layer — and implementing complementary controls (SPF/DKIM/DMARC) accordingly

**Protocols/Tools:** IPsec (IKEv2), OpenVPN, Wireshark (ESP/IKE analysis), PowerShell.
