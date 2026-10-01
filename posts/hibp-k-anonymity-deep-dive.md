---
title: "Check Passwords Against Breaches with HIBP k-Anonymity"
slug: "hibp-k-anonymity-deep-dive"
date: "2026-02-25"
hub: "security"
tags: ["tutorial","identity","cryptography","privacy"]
summary: "How Troy Hunt's Have I Been Pwned API uses mathematical k-anonymity to securely verify breached passwords without ever exposing the password or full hash."
readTime: "6 min read"
coverAccent: "from-emerald-900 via-slate-900 to-green-950"
---

## The Problem: How to Verify Passwords Privately

If a user signs up on your website, you want to ensure they haven't chosen a password found in credential dumps. But sending their plain-text password (or even full hash) to a third party creates a catastrophic security hazard.

Troy Hunt solved this dilemma by introducing the **k-Anonymity Pwned Passwords API**.

### How k-Anonymity Works

1. Compute the SHA-1 hash of the candidate password locally in client memory:
   ```
   password: "password123"
   SHA-1: CBFDAC6008F9CAB4083784CBD1874F76618D2A97
   ```
2. Split the 40-character hex string into two parts:
   - **Prefix (first 5 characters)**: `CBFDA`
   - **Suffix (remaining 35 characters)**: `C6008F9CAB4083784CBD1874F76618D2A97`
3. Query the HIBP API with ONLY the 5-character prefix:
   ```bash
   curl https://api.pwnedpasswords.com/range/CBFDA
   ```
4. The server returns a list of hundreds of hash suffixes that share that exact prefix, along with their breach count.
5. In client memory, compare your suffix against the returned list. If a match is found, you know the breach count without the API ever knowing which password or suffix you had!

```typescript
async function checkBreachedPassword(password: string): Promise<number> {
  // Step 1: Hash locally with Web Crypto
  const encoder = new TextEncoder();
  const data = encoder.encode(password);
  const hashBuffer = await crypto.subtle.digest('SHA-1', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const fullHash = hashArray.map(b => b.toString(16).padStart(2, '0')).join('').toUpperCase();

  const prefix = fullHash.slice(0, 5);
  const suffix = fullHash.slice(5);

  // Step 2: Query API with prefix only
  const response = await fetch(`https://api.pwnedpasswords.com/range/${prefix}`);
  const text = await response.text();

  // Step 3: Match locally
  for (const line of text.split('\n')) {
    const [hashSuffix, count] = line.trim().split(':');
    if (hashSuffix === suffix) {
      return parseInt(count, 10);
    }
  }
  return 0; // Not breached
}
```

This mathematical beauty is embedded directly in Nectarpost's free **Password Strength & Breach Checker** tool.
