---
title: "Free Threat Intel Sources Every Analyst Should Know"
slug: "free-threat-intel-sources"
date: "2026-03-11"
hub: "security"
tags: ["threat-intel","soc","feeds","defensive"]
summary: "Curated high-signal, zero-cost intelligence feeds covering malicious IPs, command & control infrastructure, and malware hashes."
readTime: "7 min read"
coverAccent: "from-teal-950 via-slate-900 to-slate-950"
---

## High-Fidelity Intelligence Without Enterprise License Fees

Commercial threat intel feeds can cost tens of thousands of dollars annually. Yet some of the most reliable and immediate indicators of compromise (IOCs) originate from academic research projects, CERT teams, and community honeypots.

### 1. abuse.ch Ecosystem

Operated by the Bern University of Applied Sciences:
- **URLhaus**: High-frequency collection of verified malware distribution sites. Essential for DNS sinkholing and web gateway blocklists.
- **ThreatFox**: Curated IOCs (IP addresses, domain names, hashes) tied directly to malware families like Lumma, RedLine, and QakBot.
- **Feodo Tracker**: Tracks active Command & Control (C2) servers for banking trojans and loader botnets.

### 2. AlienVault Open Threat Exchange (OTX)

The largest open threat intel community in the world. Allows defensive analysts to subscribe to community "pulses" detailing campaign-specific TTPs and IOCs.

### 3. CISA KEV & NVD

The standard reference catalogs for vulnerability management. NVD provides the technical vulnerability score (CVSS), while CISA KEV provides real-world exploitation confirmation.

### 4. AbuseIPDB

Community-driven database of IP addresses engaged in brute force, port scanning, and automated credential stuffing.

### Defanging Best Practices

When sharing threat indicators with team members or in documentation, **always defang URLs and IPs** so they cannot be accidentally clicked or pre-fetched by corporate mail clients:
- `http://` becomes `hxxp://`
- `.` becomes `[.]`
- `@` becomes `[@]`
