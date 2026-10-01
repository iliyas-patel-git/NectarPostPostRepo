---
title: "Position Sizing and Risk Basics: Fixed Fractional vs Kelly Criterion"
slug: "position-sizing-risk-basics"
date: "2026-02-10"
hub: "trading"
tags: ["risk","math","portfolio","sizing"]
summary: "The mathematical proof showing that bet size matters more than entry accuracy when preserving trading capital over long time horizons."
readTime: "8 min read"
coverAccent: "from-amber-900 via-slate-900 to-stone-950"
---

## The Gambler's Ruin Problem

Even if a trader possesses a genuine 60% win rate with a 1:1 risk-to-reward ratio, risking 20% of the account on every trade results in an over **90% probability of ruin** within 100 trades.

Consecutive losses of 4 or 5 trades occur regularly in normal distribution curves. Recovering from drawdowns is mathematically asymmetric:
- A **10% loss** requires an **11.1% gain** to break even.
- A **20% loss** requires a **25% gain** to break even.
- A **50% loss** requires a **100% gain** to break even.
- An **80% loss** requires a **400% gain** to break even.

### The Fixed Fractional Model

The industry gold standard for defensive risk management is the **Fixed Fractional Method**:
- Risk a fixed percentage $f$ (typically $1.0\%$ to $2.0\%$) of current account equity per position.
- As account equity grows, position sizes gradually increase.
- During a losing streak, position sizes automatically shrink, dampening the dollar drawdown.
