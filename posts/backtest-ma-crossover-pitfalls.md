---
title: "Backtesting an MA Crossover, and Why Backtest Results Mislead"
slug: "backtest-ma-crossover-pitfalls"
date: "2026-03-05"
hub: "trading"
tags: ["backtesting","strategies","risk","math"]
summary: "Deconstructing moving average crossover strategies, look-ahead bias, curve-fitting traps, and how to evaluate realistic strategy performance."
readTime: "8 min read"
coverAccent: "from-orange-950 via-slate-900 to-amber-950"
---

## The Classic Golden Cross and Death Cross

The Simple Moving Average (SMA) crossover is the most famous algorithmic trading concept. When a fast moving average (e.g. 20-period) crosses above a slow moving average (e.g. 50-period), a buy signal is generated. When it crosses below, a sell or cash signal is triggered.

While conceptually straightforward, running this strategy naively often yields disappointing results in live markets.

### The 3 Major Backtesting Biases

1. **Look-Ahead Bias**: Using data point $t$ to calculate the signal, but recording the execution at price $t$ instead of $t+1$. In real life, you cannot execute at the closing price of the candle that determined the signal.
2. **Survivorship Bias**: Testing only on companies or tokens that currently exist today, ignoring the hundreds that went bankrupt or were delisted during the backtest timeframe.
3. **Over-Optimization (P-Hacking)**: Testing 200 combinations of moving averages (e.g. 17 & 43) until one shows a 400% return by pure historical coincidence.

### What Real Alpha Looks Like

A robust algorithmic strategy focuses on **Maximum Drawdown mitigation** and risk-adjusted return (Sharpe / Sortino ratio), rather than chasing extreme speculative returns.

Visit our **Interactive Client-Side Backtester** at `/trading/backtest` to run real SMA and RSI backtests directly in your browser with realistic commission, slippage, and equity curves.
