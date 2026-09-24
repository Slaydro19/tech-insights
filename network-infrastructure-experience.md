# Enterprise Network & Data Center Infrastructure Experience

During an internship at an AI infrastructure startup, I gained hands-on exposure to enterprise-grade network and data center equipment, complementing my cybersecurity coursework and homelab work with real production-adjacent context.

## Hands-On Exposure

- **Next-generation firewalls**: physical placement, perimeter defense strategy, and how NGFW threat-prevention features fit into a broader network security posture
- **Enterprise switching**: high-density (48-port class) Layer 2/3 switches, stacking for scalability, and port/VLAN planning in a multi-vendor environment
- **Rack-mounted server infrastructure**: rack layout, cable management, and redundant power planning in a live data center

## What I Took Away

**Enterprise network design**
- Multi-tier architecture (core / distribution / access) and how it differs from the flatter networks I run at home
- Security zone segmentation (DMZ vs. internal) applied to real hardware, not just diagrams
- Redundant paths and failover as a design requirement, not an afterthought

**Multi-vendor operations**
- Coordinating across firewall, switch, and server vendors in one environment
- Documentation and labeling standards that make an infrastructure maintainable by someone other than the person who built it
- Physical security and access-control practices for a controlled data center environment

**Why it matters for security work**
- Physical infrastructure exposure gives network security findings (segmentation gaps, exposed management interfaces, overly broad firewall rules) real-world grounding instead of purely theoretical context
- Understanding how traffic actually flows through physical firewall, switch, and server tiers makes it easier to reason about where a control failure would actually matter

---

*This experience runs alongside my [homelab work](README.md#homelab), where I replicate similar segmentation and monitoring patterns (Proxmox/pfSense, VLANs, Wazuh/Security Onion) in an environment I fully control and can document in depth.*
