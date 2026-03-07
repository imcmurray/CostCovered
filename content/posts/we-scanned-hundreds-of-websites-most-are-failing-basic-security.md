---
title: "We Scanned Hundreds of Websites. Most Are Failing Basic Security."
date: 2026-03-07
draft: false
tags: ["security", "data", "guardscan", "web-security", "research"]
categories: ["Security Guides"]
summary: "We analyzed the results of hundreds of security scans from GuardScan. Zero sites scored an A. The majority landed at D. Here's what's going wrong and how to fix it."
ShowToc: true
---

## We Built a Scanner. Then We Looked at the Data.

[GuardScan](https://guardscan.dev) is a free security scanner we built to check websites for common misconfigurations — security headers, SSL, DNS, cookies, CORS, mixed content, redirects, and technology stack. Each site gets a score from 0 to 100, mapped to a letter grade from A to F.

After 150+ scans across a wide range of websites, we pulled the aggregate data. The results weren't pretty.

Here's how they break down:

| Grade | Share of Sites |
|-------|---------------|
| **A** | 8% |
| **B** | 17% |
| **C** | 42% |
| **D** | 23% |
| **F** | 10% |

**The most common grade is C.** Nearly a quarter of sites scored a D, and one in ten outright failed. Only 8% earned an A — fewer than the sites that scored an F.

These aren't obscure hobby sites. They include business websites, SaaS products, e-commerce stores, and popular blogs. Sites with paying customers and real traffic.

## How We Grade

GuardScan runs eight checks on every site:

1. **Security Headers** — CSP, HSTS, X-Frame-Options, Permissions-Policy, COOP, CORP, and more
2. **SSL/TLS** — Certificate validity, expiration, protocol version, cipher strength
3. **DNS Security** — DNSSEC, CAA records
4. **Email Security** — SPF, DMARC, and DKIM records
5. **Cookie Security** — Secure, HttpOnly, and SameSite flags on all cookies
6. **CORS Policy** — Whether cross-origin resource sharing is safely configured
7. **Mixed Content** — HTTP resources loaded on HTTPS pages
8. **Redirect Chain** — HTTP-to-HTTPS redirects, redirect loops, excessive hops

Each category scores 0 to 100. The overall score is a weighted average, mapped to a letter grade: A (90-100), B (80-89), C (70-79), D (60-69), F (below 60).

An A isn't an unreasonable bar. It just means you've properly configured the standard security controls that every modern website should have in place.

## What's Dragging Scores Down

Three categories account for the vast majority of lost points.

### 1. Missing Security Headers

This is the biggest problem by far. Most websites return almost no security headers beyond basic HSTS.

The headers that are almost universally missing:

- **Content-Security-Policy (CSP)** — Prevents XSS and data injection attacks. Setting it up requires effort, so most sites skip it entirely.
- **Permissions-Policy** — Controls which browser features (camera, microphone, geolocation) your site can use. [Almost nobody sets it](/posts/permissions-policy-header-what-it-does-and-how-to-set-it/).
- **Cross-Origin-Opener-Policy (COOP)** and **Cross-Origin-Resource-Policy (CORP)** — Newer headers that isolate your site from cross-origin attacks. Most developers haven't heard of them.

The irony is that adding these headers is straightforward — it's usually a few lines in your web server config or a middleware in your framework. The problem is awareness, not difficulty.

For a full walkthrough, see our [security headers guide](/posts/how-to-check-your-website-security-headers/).

### 2. DNS Email Security Gaps

Most websites have a domain that can be spoofed for phishing emails. The three records that prevent this:

- **SPF** — Declares which servers can send email for your domain. Many sites have it, but often with syntax errors or overly permissive configurations (`+all`).
- **DMARC** — Tells receiving servers what to do with emails that fail SPF/DKIM checks. The majority of sites either have no DMARC record or set it to `p=none`, which means "report but don't block" — effectively useless against active spoofing.
- **DKIM** — Cryptographic signing of outgoing emails. Requires coordination with your email provider, so it's frequently missing.

Without all three properly configured, anyone can send emails that appear to come from your domain. Your customers get phishing emails "from" you, and you never know.

See our [SPF and DMARC guide](/posts/how-to-check-spf-dmarc-records-email-security/) for setup instructions.

### 3. Cookie Security Flags

Websites that set cookies — which is nearly all of them — frequently miss the security attributes:

- **`Secure`** — Cookie only sent over HTTPS. Without this, cookies can be intercepted on insecure connections.
- **`HttpOnly`** — Cookie inaccessible to JavaScript. Without this, XSS attacks can steal session cookies.
- **`SameSite`** — Controls when cookies are sent with cross-site requests. Without this, CSRF attacks are easier to pull off.

Many frameworks set these by default now, but plenty of sites still run on older configurations or have manually created cookies that skip these flags.

## Why "HTTPS Is Enough" Is a Myth

Here's the thing — almost every site we scanned has HTTPS configured. SSL certificates are free (thanks, Let's Encrypt) and most hosting providers set them up automatically. That part is solved.

But HTTPS only encrypts the connection. It doesn't protect against:

- **Cross-site scripting (XSS)** — That's what CSP headers prevent
- **Clickjacking** — That's what X-Frame-Options and COOP prevent
- **CORS abuse** — That's what [proper CORS configuration](/posts/cors-misconfiguration-how-to-check-and-fix/) prevents
- **Email spoofing** — That's what SPF, DMARC, and DKIM prevent
- **Session hijacking** — That's what cookie security flags prevent
- **Feature abuse** — That's what Permissions-Policy prevents

HTTPS is the floor, not the ceiling. The real security posture of a website is determined by the layers above it — and that's where almost everyone fails.

## What an A Grade Actually Requires

Here's what a perfect score looks like across all eight categories:

**Security Headers:** Set CSP, HSTS (with long max-age), X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, COOP, and CORP. This is the hardest category to ace because CSP requires careful tuning to avoid breaking your site.

**SSL/TLS:** Valid certificate, not expiring within 30 days, TLS 1.2 or 1.3, strong cipher suite. Most sites already pass this.

**DNS Security:** DNSSEC enabled, CAA records set. CAA records are a one-time DNS addition that restricts which certificate authorities can issue certificates for your domain.

**Email Security:** Valid SPF record (not using `+all`), DMARC with `p=reject` or `p=quarantine`, DKIM configured. This requires working with your email provider but is a one-time setup.

**Cookie Security:** All cookies have `Secure`, `HttpOnly` (where applicable), and `SameSite` attributes. Usually a framework configuration change.

**CORS:** Either no CORS headers (safest default) or properly scoped `Access-Control-Allow-Origin` with specific origins. Never `*` with credentials.

**Mixed Content:** No HTTP resources on HTTPS pages. Fix any `http://` URLs in your HTML/CSS/JS.

**Redirects:** HTTP redirects to HTTPS. No redirect loops. Reasonable chain length.

None of these are exotic requirements. They're all documented best practices that have existed for years. The gap isn't knowledge — it's prioritization. Security configuration is invisible when it works, so it gets deprioritized in favor of features.

## How to Fix Your Score

If you've scanned your site on [GuardScan](https://guardscan.dev) and got a poor grade, here's the fastest path to improvement:

1. **Add security headers** — This will have the biggest impact. Start with the easy ones (X-Content-Type-Options, Referrer-Policy, Permissions-Policy) and work up to CSP. [Full guide here](/posts/how-to-check-your-website-security-headers/).

2. **Set up SPF, DMARC, and DKIM** — Three DNS records that prevent your domain from being used in phishing attacks. [Step-by-step instructions here](/posts/how-to-check-spf-dmarc-records-email-security/).

3. **Audit your cookies** — Check that all cookies have `Secure`, `HttpOnly`, and `SameSite` flags. Most frameworks have configuration options for this.

4. **Review your CORS policy** — If you're using CORS, make sure you're not reflecting arbitrary origins or using wildcards with credentials. [CORS guide here](/posts/cors-misconfiguration-how-to-check-and-fix/).

5. **Check Permissions-Policy** — A one-line header that restricts browser feature access. [Setup guide here](/posts/permissions-policy-header-what-it-does-and-how-to-set-it/).

Most of these changes take under an hour for a typical web application. The hardest part is knowing what to fix — and that's what GuardScan shows you.

## Scan Your Site

The data is clear: the bar for web security is low, and most sites aren't clearing it. The good news is that the fixes are well-documented and usually straightforward.

**[Scan your site for free at GuardScan](https://guardscan.dev)** — get your grade, see exactly what's missing, and follow the fix-it guides to improve your score. No signup required.
