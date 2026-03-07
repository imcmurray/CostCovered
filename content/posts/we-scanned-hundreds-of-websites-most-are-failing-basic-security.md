---
title: "We Scanned Hundreds of Websites. Most Are Failing Basic Security."
date: 2026-03-07
draft: false
tags: ["security", "data", "guardscan", "web-security", "research"]
categories: ["Security Guides"]
summary: "We analyzed 150+ security scans from GuardScan. Only 8% scored an A. The majority landed at C or D. Here's what's going wrong and how to fix it."
ShowToc: true
---

## We Built a Scanner. Then We Looked at the Data.

[GuardScan](https://guardscan.dev) is a free security scanner we built to check websites for common misconfigurations — security headers, SSL, DNS, cookies, CORS, mixed content, redirects, and technology stack. Each site gets a score from 0 to 100, mapped to a letter grade from A to F.

After 150+ scans run between January and March 2026 — roughly 40% SaaS and business sites, 30% e-commerce, and 30% blogs, portfolios, and other sites — we pulled the aggregate data. The results weren't pretty, and they're consistent with what [Mozilla Observatory](https://observatory.mozilla.org) and [SecurityHeaders.com](https://securityheaders.com) report across the wider web.

Here's how they break down:

{{< rawhtml >}}
<div style="max-width: 480px; margin: 1.5em auto;">
<svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Pie chart showing grade distribution: A 8%, B 17%, C 42%, D 23%, F 10%" style="width: 100%; height: auto;">
  <g transform="rotate(-90 100 100)">
    <circle cx="100" cy="100" r="45" fill="none" stroke="#fbbf24" stroke-width="90" stroke-dasharray="118.75 163.99" stroke-dashoffset="-70.69"/>
    <circle cx="100" cy="100" r="45" fill="none" stroke="#fb923c" stroke-width="90" stroke-dasharray="65.03 217.71" stroke-dashoffset="-189.44"/>
    <circle cx="100" cy="100" r="45" fill="none" stroke="#f87171" stroke-width="90" stroke-dasharray="28.27 254.47" stroke-dashoffset="-254.47"/>
    <circle cx="100" cy="100" r="45" fill="none" stroke="#4f8cff" stroke-width="90" stroke-dasharray="48.07 234.68" stroke-dashoffset="-22.62"/>
    <circle cx="100" cy="100" r="45" fill="none" stroke="#34d399" stroke-width="90" stroke-dasharray="22.62 260.12" stroke-dashoffset="0"/>
  </g>
  <g font-size="14" font-family="system-ui, -apple-system, sans-serif">
    <rect x="210" y="40" width="16" height="16" rx="3" fill="#34d399"/><text x="232" y="53" fill="currentColor" font-weight="600">A — 8%</text>
    <rect x="210" y="65" width="16" height="16" rx="3" fill="#4f8cff"/><text x="232" y="78" fill="currentColor" font-weight="600">B — 17%</text>
    <rect x="210" y="90" width="16" height="16" rx="3" fill="#fbbf24"/><text x="232" y="103" fill="currentColor" font-weight="600">C — 42%</text>
    <rect x="210" y="115" width="16" height="16" rx="3" fill="#fb923c"/><text x="232" y="128" fill="currentColor" font-weight="600">D — 23%</text>
    <rect x="210" y="140" width="16" height="16" rx="3" fill="#f87171"/><text x="232" y="153" fill="currentColor" font-weight="600">F — 10%</text>
  </g>
</svg>
</div>
{{< /rawhtml >}}

**The most common grade is C.** Nearly a quarter of sites scored a D, and one in ten outright failed. Only 8% earned an A — fewer than the sites that scored an F.

These aren't obscure hobby sites. They include business websites, SaaS products, e-commerce stores, and popular blogs — sites with paying customers and real traffic. For context, Mozilla Observatory's ongoing scans show average scores around 50-60/100 with CSP missing on the vast majority of sites, so these numbers aren't an outlier.

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

Each category scores 0 to 100. The overall score is a weighted average — security headers carry the most weight (~30%), followed by SSL and email security (~15% each), with cookies, CORS, DNS, mixed content, and redirects making up the rest. The final score maps to a letter grade: A (90-100), B (80-89), C (70-79), D (60-69), F (below 60).

An A isn't an unreasonable bar. It just means you've properly configured the standard security controls that every modern website should have in place.

## What's Dragging Scores Down

Three categories account for the vast majority of lost points.

### 1. Missing Security Headers

This is the biggest problem by far. Most websites return almost no security headers beyond basic HSTS.

The headers that are almost universally missing:

- **Content-Security-Policy (CSP)** — Prevents XSS and data injection attacks. Setting it up requires effort, so most sites skip it entirely.
- **Permissions-Policy** — Controls which browser features (camera, microphone, geolocation) your site can use. [Almost nobody sets it](/posts/permissions-policy-header-what-it-does-and-how-to-set-it/).
- **Cross-Origin-Opener-Policy (COOP)**, **Cross-Origin-Embedder-Policy (COEP)**, and **Cross-Origin-Resource-Policy (CORP)** — These isolation headers protect against Spectre-class side-channel attacks and enable powerful features like `SharedArrayBuffer`. Most developers haven't heard of them.

Note: if you still see `X-XSS-Protection` in your headers, it's safe to remove — all modern browsers have deprecated it in favor of CSP. And `Referrer-Policy` should be set to at least `strict-origin-when-cross-origin` (the browser default since 2021), though many sites still don't set it explicitly.

The irony is that adding most of these headers is straightforward — it's usually a few lines in your web server config or a middleware in your framework. The problem is awareness, not difficulty.

For a full walkthrough, see our [security headers guide](/posts/how-to-check-your-website-security-headers/).

### 2. DNS Email Security Gaps

Most websites have a domain that can be spoofed for phishing emails. The three records that prevent this:

- **SPF** — Declares which servers can send email for your domain. Many sites have it, but often with syntax errors or overly permissive configurations (`+all`).
- **DMARC** — Tells receiving servers what to do with emails that fail SPF/DKIM checks. The majority of sites either have no DMARC record or set it to `p=none`, which means "report but don't block" — effectively useless against active spoofing. In 2026, `p=none` should only be a temporary stepping stone; aim for `p=quarantine` at minimum, ideally `p=reject`.
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

**Security Headers:** Set CSP, HSTS (with long max-age), X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, COOP, COEP, and CORP. Drop `X-XSS-Protection` if you still have it — it's deprecated in all modern browsers. This is the hardest category to ace because CSP requires careful tuning to avoid breaking your site.

**SSL/TLS:** Valid certificate, not expiring within 30 days, TLS 1.2 or 1.3, strong cipher suite. Most sites already pass this.

**DNS Security:** DNSSEC enabled, CAA records set. CAA records are a one-time DNS addition that restricts which certificate authorities can issue certificates for your domain.

**Email Security:** Valid SPF record (not using `+all`), DMARC with `p=reject` or `p=quarantine` (not `p=none`), DKIM configured. This requires working with your email provider but is a one-time setup.

**Cookie Security:** All cookies have `Secure`, `HttpOnly` (where applicable), and `SameSite` attributes. Usually a framework configuration change.

**CORS:** Either no CORS headers (safest default) or properly scoped `Access-Control-Allow-Origin` with specific origins. Never `*` with credentials.

**Mixed Content:** No HTTP resources on HTTPS pages. Fix any `http://` URLs in your HTML/CSS/JS.

**Redirects:** HTTP redirects to HTTPS. No redirect loops. Reasonable chain length.

None of these are exotic requirements. They're all documented best practices that have existed for years. The gap isn't knowledge — it's prioritization. Security configuration is invisible when it works, so it gets deprioritized in favor of features.

## How to Fix Your Score

Start by scanning your site with [GuardScan](https://guardscan.dev), [Mozilla Observatory](https://observatory.mozilla.org), or [SecurityHeaders.com](https://securityheaders.com) — pick whichever you prefer. Once you know what's missing, here's a prioritized roadmap:

### 5 Minutes

- Add `X-Content-Type-Options: nosniff` — one line, zero risk of breaking anything.
- Add `X-Frame-Options: DENY` (or `SAMEORIGIN` if you embed your own site in iframes).
- Set `Referrer-Policy: strict-origin-when-cross-origin` explicitly.

### 30 Minutes

- Enable **HSTS** with a long `max-age` (at least 6 months) and `includeSubDomains`. Make sure all subdomains support HTTPS first.
- Tighten your **SPF** record — remove `+all` if present, switch to `~all` or `-all`.
- Add a **DMARC** record with at least `p=quarantine`. [Step-by-step guide here](/posts/how-to-check-spf-dmarc-records-email-security/).
- Audit your **cookies** for `Secure`, `HttpOnly`, and `SameSite` flags. Most frameworks have configuration options for this.

### 1+ Hours

- Set **Permissions-Policy** to disable browser features you don't use (camera, microphone, geolocation). [Setup guide here](/posts/permissions-policy-header-what-it-does-and-how-to-set-it/).
- Deploy **CSP in report-only mode** first (`Content-Security-Policy-Report-Only`), monitor for breakage, then enforce. This is the single highest-impact header, but on complex apps with third-party scripts or legacy CMS setups, expect testing cycles — possibly days or weeks of tuning. [Full header guide here](/posts/how-to-check-your-website-security-headers/).
- Review your **CORS policy** — make sure you're not reflecting arbitrary origins or using wildcards with credentials. [CORS guide here](/posts/cors-misconfiguration-how-to-check-and-fix/).
- Set up **DKIM** with your email provider and upgrade DMARC to `p=reject`.

For simple static sites and modern stacks, you can knock out most of this in an afternoon. For apps with heavy JavaScript, third-party integrations, or strict compliance requirements, budget more time — especially for CSP. Either way, the ROI is significant: these are the lowest-effort, highest-impact defenses against phishing and client-side attacks.

## Scan Your Site

The data is clear: the bar for web security is low, and most sites aren't clearing it. The good news is that the fixes are well-documented and — for most of the checklist above — straightforward.

Pick a scanner and run it on your domain today:

- **[GuardScan](https://guardscan.dev)** — free, no signup, covers all eight categories above with fix-it guides
- **[Mozilla Observatory](https://observatory.mozilla.org)** — Mozilla's well-established scanner with detailed scoring methodology
- **[SecurityHeaders.com](https://securityheaders.com)** — quick header-focused check by Scott Helme

Whichever tool you use, the important thing is to actually look at the results and act on them. The web gets more secure one site at a time.
