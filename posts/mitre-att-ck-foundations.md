---
title: "Getting Started with MITRE ATT&CK: Enterprise TTPs for Detection Engineers"
slug: "mitre-att-ck-foundations"
date: "2026-02-18"
hub: "security"
tags: ["tutorial","mitre","detection","blue-team"]
summary: "A practical guide to mapping SIEM detection rules and threat hunting hypotheses to MITRE ATT&CK tactics and techniques."
readTime: "8 min read"
coverAccent: "from-cyan-950 via-slate-900 to-teal-950"
---

## Beyond Signature-Based Defense

Traditional antivirus software relied on file hashes and static string signatures. Adversaries quickly adapted by recompiling payloads, packing binaries, and rotating domain names.

The **MITRE ATT&CK (Adversarial Tactics, Techniques, and Common Knowledge)** framework provides a standardized language for classifying adversary behavior across the entire cyber attack lifecycle.

### The 14 Enterprise Tactics

1. **Reconnaissance**: Gathering target intelligence.
2. **Resource Development**: Acquiring infrastructure, domains, botnets.
3. **Initial Access**: Gaining an initial foothold (phishing, public-facing exploits).
4. **Execution**: Running malicious code on a target system.
5. **Persistence**: Maintaining access across reboots and account resets.
6. **Privilege Escalation**: Elevating permissions to SYSTEM or root.
7. **Defense Evasion**: Disabling AV, obfuscating scripts, deleting logs.
8. **Credential Access**: Dumping LSASS, scraping memory, brute forcing.
9. **Discovery**: Mapping the internal network and domain privileges.
10. **Lateral Movement**: Pivoting to adjacent servers via RDP, SSH, WinRM.
11. **Collection**: Gathering intellectual property, email archives, customer data.
12. **Command & Control**: Establishing covert communication channels.
13. **Exfiltration**: Stealing data over encrypted tunnels or cloud storage.
14. **Impact**: Encrypting systems with ransomware, manipulating operational controls.

Explore our interactive **MITRE ATT&CK Explorer** under the Cybersecurity hub to search techniques, view code execution methods, and examine mitigations.
