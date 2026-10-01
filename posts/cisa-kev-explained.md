---
title: "CISA Known Exploited Vulnerabilities Explained: How to Prioritize and Act"
slug: "cisa-kev-explained"
date: "2026-03-20"
hub: "security"
tags: ["tutorial","cisa","threat-intel","vulnerability"]
summary: "Why CVSS scores alone mislead security teams, and how the CISA KEV catalog delivers evidence-based patch prioritization for enterprise defenders."
readTime: "10 min read"
coverAccent: "from-emerald-950 via-slate-900 to-teal-950"
---

## The Failure of CVSS Prioritization

Every year, over 25,000 Common Vulnerabilities and Exposures (CVEs) are published. A significant percentage receive a CVSS score of 9.0 or higher ("Critical"). If an IT security team attempts to patch every critical CVSS vulnerability immediately, they inevitably suffer alert fatigue and burnout.

Empirical telemetry reveals that **less than 4% of all published vulnerabilities are ever actively weaponized in the wild**.

This is why the Cybersecurity and Infrastructure Security Agency established the **Known Exploited Vulnerabilities (KEV) catalog**.

```
                 ┌──────────────────────────────────────┐
                 │        All CVEs (~250,000+)          │
                 │                                      │
                 │   ┌──────────────────────────────┐   │
                 │   │  CVSS 9.0+ Critical (~40,000)│   │
                 │   │                              │   │
                 │   │   ┌──────────────────────┐   │   │
                 │   │   │ CISA KEV Catalog     │   │   │
                 │   │   │ (~1,300 CVEs)        │   │   │
                 │   │   │ ACTIVELY EXPLOITED   │   │   │
                 │   │   └──────────────────────┘   │   │
                 │   └──────────────────────────────┘   │
                 └──────────────────────────────────────┘
```

### Criteria for Inclusion in CISA KEV

A vulnerability is added to the KEV catalog if and only if:
1. It has an assigned CVE ID.
2. There is **reliable evidence of active exploitation in the wild** (e.g. ransomware campaigns, state-sponsored intrusion, or automated botnets).
3. A clear remediation or vendor patch exists.

### Programmatic Ingestion via Public JSON API

CISA publishes the entire KEV catalog as a machine-readable JSON feed updated in real time:

```bash
# Fetch the official public CISA KEV JSON dataset
curl -s https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json | jq '.vulnerabilities[:3]'
```

Each entry contains:
- `cveID`: The CVE identifier (e.g., CVE-2024-3400)
- `vendorProject`: The affected vendor (e.g., Palo Alto Networks)
- `product`: The software or appliance (e.g., PAN-OS)
- `vulnerabilityName`: A plain-text description of the flaw
- `dateAdded`: Timestamp when exploitation was verified
- `dueDate`: The statutory compliance deadline for federal agencies to remediate
- `knownRansomwareCampaignUse`: Flag indicating observed ransomware extortion

### Action Plan for Defensive Engineers

1. **Daily Ingestion**: Ingest the KEV feed automatically at 06:00 UTC.
2. **Asset Cross-Referencing**: Match affected software CPE strings against your asset management database (e.g., Wazuh, Qualys, or Tenable).
3. **Emergency Sprints**: Treat any internet-facing asset matching a newly added KEV entry as an emergency remediation ticket with a strict 48-hour SLA.
