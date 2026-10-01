# Nectarpost — AI Engineering, Defensive Cybersecurity & Algo Trading

Nectarpost is a high-performance, local-first technical publication and suite of interactive browser tools built for software engineers, defensive SOC analysts, and quantitative traders.

---

## 1. Environment Variables

Configure these variables in your deployment settings (e.g. Vercel, Cloud Run, or `.env.local`):

| Variable | Description | Required? |
| :--- | :--- | :--- |
| `GEMINI_API_KEY` | Google Gemini API key for server-side chat streaming | Recommended |
| `XAI_API_KEY` | xAI Grok API key for alternative Grok chat completions | Optional |
| `XAI_MODEL` | Grok model name (e.g. `grok-beta` or `grok-2`) | Optional |
| `GITHUB_TOKEN` | Fine-grained Personal Access Token with read-only "Contents" scope | Optional (for private repo sync) |
| `GITHUB_OWNER` | GitHub account or organization owner | Optional |
| `GITHUB_REPO` | Private repository name containing `.md` posts | Optional |
| `GITHUB_BRANCH` | Branch name to fetch posts from (default `main`) | Optional |
| `REVALIDATE_SECRET` | Secret token to authenticate `/api/revalidate` webhook calls | Optional |
| `APP_URL` | Public canonical base URL (e.g. `https://nectarpost.com`) | Yes |

---

## 2. Content System & Adding a New Post

Posts are written in standard Markdown with YAML frontmatter:

```markdown
---
title: "Building a Zero-Trust Network Architecture with WireGuard"
slug: "zero-trust-wireguard-guide"
date: "2026-04-01"
hub: "security" # "ai" | "security" | "trading"
tags: ["tutorial", "networking", "wireguard"]
summary: "How to eliminate open listening ports and enforce cryptographic peer identity."
published: true
cover: "/images/covers/wireguard.png"
---

## Introduction

Body content in GitHub Flavored Markdown (GFM)...
```

- **File location in private repo:** `posts/[hub]/[slug].md`
- **Drafts:** Store unpublished drafts in `drafts/` (the site never accesses this directory).
- **Instant Revalidation:** Configure a GitHub Webhook on push to trigger `POST /api/revalidate` with `x-revalidate-secret`.

---

## 3. How to Rotate API Keys

1. **GitHub Fine-Grained Token:**
   - Go to GitHub -> Settings -> Developer settings -> Personal access tokens -> Fine-grained tokens.
   - Generate a new token scoped strictly to your private blog repo with read-only **Contents** permission.
   - Update `GITHUB_TOKEN` in your hosting console environment variables.
   - Revoke the old token.

2. **Google Gemini / xAI Grok Key:**
   - Generate a new key in Google AI Studio or xAI console.
   - Update `GEMINI_API_KEY` or `XAI_API_KEY` in environment variables.
   - Revoke previous key in the vendor dashboard.

---

## 4. Test Checklist

- [x] **Live Widgets Fallback:** CISA KEV, Hacker News, and crypto tickers gracefully fall back to cached data if external APIs rate-limit or fail.
- [x] **Chat Rate Limiting:** `/api/chat` enforces max 10 messages/minute and 50 messages/day per IP, with max 1,000 characters input validation.
- [x] **SSRF Protection:** `/api/security/headers` blocks localhost, loopback (`127.0.0.1`), RFC 1918 subnets (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), and AWS/GCP metadata endpoints (`169.254.169.254`).
- [x] **Client-Side Privacy:** JWT Decoder, Hash Generator, Multi-Format Encoder, CIDR Calculator, Password Checker, and Backtester execute 100% in client memory without data leaving the browser.
- [x] **Defensive Content Stance:** Threat indicators are rendered defanged (`hxxp://`, `[.]`, `[at]`), never as clickable links.
- [x] **SEO & Structured Data:** Includes `WebSite` JSON-LD with `SearchAction`, `Article` schemas, `FAQPage` schemas, `Person` schema, `sitemap.xml`, and `robots.txt`.
