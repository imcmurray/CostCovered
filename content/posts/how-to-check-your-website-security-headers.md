---
title: "How to Check Your Website's Security Headers (And Why You Should)"
date: 2026-02-22
draft: false
tags: ["security", "headers", "tutorial", "guardscan"]
categories: ["Technical"]
summary: "HTTP security headers are your website's first line of defense. Here's what they are, why they matter, and how to check yours in 10 seconds."
ShowToc: true
---

## What Are HTTP Security Headers?

Every time someone visits your website, your server sends back HTTP headers along with the page content. Most of these are mundane — content type, cache settings, server info. But a handful of them are specifically designed to protect your visitors from attacks.

These are your **security headers**, and most websites are missing them entirely.

A 2025 study of the top 1 million websites found that fewer than 15% set a Content Security Policy header. Only 25% use Strict-Transport-Security. These are basic protections that take minutes to add and can prevent entire categories of attacks.

## The Security Headers That Matter

Here are the nine headers every website should have, ordered by impact:

### 1. Strict-Transport-Security (HSTS)

Tells browsers to only connect to your site over HTTPS. Without it, users can be downgraded to HTTP through man-in-the-middle attacks — even if you have an SSL certificate.

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**What happens without it:** An attacker on the same Wi-Fi network can intercept the initial HTTP request before the redirect to HTTPS, stealing session cookies or injecting content.

### 2. Content-Security-Policy (CSP)

Controls which resources — scripts, styles, images, fonts — the browser is allowed to load on your page. This is the single most effective header against XSS (cross-site scripting) attacks.

```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
```

**What happens without it:** If an attacker finds a way to inject a `<script>` tag into your page (through a form field, URL parameter, or stored data), the browser will execute it with no questions asked. CSP blocks unauthorized scripts from running.

### 3. X-Content-Type-Options

Prevents browsers from guessing ("MIME-sniffing") the content type of a response. Without it, a file you serve as plain text could be interpreted as JavaScript and executed.

```
X-Content-Type-Options: nosniff
```

This is a one-line header. There is no reason not to set it.

### 4. X-Frame-Options

Prevents your site from being embedded in an iframe on another domain. This defends against clickjacking — where an attacker overlays your site with invisible elements to trick users into clicking things they didn't intend to.

```
X-Frame-Options: DENY
```

Or use `SAMEORIGIN` if you need iframes on your own domain.

### 5. Referrer-Policy

Controls how much URL information is sent when users click links on your site. Without it, the full URL (including query parameters, which might contain tokens or user data) gets sent to third-party sites.

```
Referrer-Policy: strict-origin-when-cross-origin
```

### 6. Permissions-Policy

Restricts which browser features (camera, microphone, geolocation, payment API) your site can use. Even if you don't use these features, setting this header prevents injected scripts from accessing them.

```
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### 7. Cross-Origin-Opener-Policy (COOP)

Isolates your browsing context from cross-origin windows. This prevents Spectre-style side-channel attacks where a malicious page can read data from your page's process.

```
Cross-Origin-Opener-Policy: same-origin
```

### 8. Cross-Origin-Resource-Policy (CORP)

Prevents other origins from loading your resources. Without it, any site can embed your images, scripts, or API responses.

```
Cross-Origin-Resource-Policy: same-origin
```

### 9. X-XSS-Protection

A legacy header that enabled the browser's built-in XSS filter. Modern browsers have removed this filter (CSP replaced it), but the recommended practice is to explicitly disable it:

```
X-XSS-Protection: 0
```

Setting it to `1` can actually introduce vulnerabilities in older browsers.

## Check Your Headers in 10 Seconds

You can manually inspect headers using browser dev tools (Network tab → click a request → Headers), but that's slow and you have to interpret the results yourself.

We built [GuardScan](https://guardscan.dev) to do this instantly. Enter your URL, and in under a second you get:

- A letter grade (A+ through F) for your overall security posture
- A breakdown of every security header — present, missing, or misconfigured
- Plain-English explanations of what each finding means
- Scores for SSL/TLS, DNS configuration, and cookie security too

It's free, no signup required, and you can scan any public site.

## How to Add Missing Headers

The fix depends on your server or hosting platform.

### Nginx

Add to your `server` block:

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Resource-Policy "same-origin" always;
add_header X-XSS-Protection "0" always;
```

### Apache

Add to your `.htaccess` or virtual host config:

```apache
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
Header always set Content-Security-Policy "default-src 'self'; script-src 'self'"
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"
Header always set Cross-Origin-Opener-Policy "same-origin"
Header always set Cross-Origin-Resource-Policy "same-origin"
Header always set X-XSS-Protection "0"
```

### Cloudflare

If your site is behind Cloudflare, you can add security headers using Transform Rules (free plan) or a Cloudflare Worker (free tier available).

### Vercel / Netlify / Next.js

Most modern hosting platforms let you set headers in a config file. Check your platform's docs for custom headers — it's usually a `vercel.json`, `netlify.toml`, or `next.config.js` addition.

## After Adding Headers: Verify

After making changes, scan your site again to confirm the headers are being set correctly. Misconfigured headers (like a CSP that's too permissive) can be worse than no header at all because they give a false sense of security.

[Scan your site now](https://guardscan.dev) — it takes one second and might save you from a breach.
