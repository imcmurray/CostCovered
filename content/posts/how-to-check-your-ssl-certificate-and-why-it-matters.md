---
title: "How to Check Your SSL Certificate (And Why It Matters)"
date: 2026-02-23
draft: false
tags: ["ssl", "tls", "security", "certificates", "https"]
categories: ["Security Guides"]
summary: "Your SSL certificate is the padlock in your browser's address bar. Here's how to check if it's valid, when it expires, and what to do if it's broken."
ShowToc: true
---

Your SSL/TLS certificate is what puts the padlock in your browser's address bar. It encrypts the connection between your visitors and your server, protecting passwords, credit cards, and personal data in transit.

When it expires or is misconfigured, browsers show scary warnings that drive visitors away. Google also uses HTTPS as a ranking signal, so a broken certificate can hurt your SEO.

Here's how to check your SSL certificate and what to look for.

## The Quick Way: Use an Online Scanner

The fastest way to check your SSL certificate is with a free online tool. Enter your domain and get a report in seconds.

**[GuardScan](https://guardscan.dev)** checks your SSL/TLS certificate alongside HTTP headers, DNS records, and cookie security -- all in one scan. You'll get a letter grade and specific findings for each category.

Other dedicated SSL checkers include [Qualys SSL Labs](https://www.ssllabs.com/ssltest) for deep protocol analysis and [SSL Shopper's SSL Checker](https://www.sslshopper.com/ssl-checker.html) for quick expiry checks.

## What to Check For

### 1. Certificate Validity

The most basic check: is your certificate currently valid? Certificates have a "not before" and "not after" date. If the current date falls outside that range, the certificate is invalid and browsers will reject it.

Most certificates are issued for 90 days (Let's Encrypt) or 1 year (commercial CAs). If you're not auto-renewing, set a calendar reminder at least two weeks before expiry.

### 2. Certificate Expiry Date

An expired certificate is the number one cause of those "Your connection is not private" warnings. According to a 2025 study, over 3% of the Alexa top 100,000 sites had expired or misconfigured certificates at any given time.

To check manually in your browser:
1. Click the padlock icon in the address bar
2. Click "Connection is secure" (or "Certificate")
3. Look for the "Valid to" or "Expires" date

Or from the command line:

```bash
echo | openssl s_client -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates
```

This will show you the `notBefore` and `notAfter` dates.

### 3. TLS Protocol Version

Your server should support **TLS 1.2** and **TLS 1.3**. Older protocols (TLS 1.0, TLS 1.1, SSL 3.0) have known vulnerabilities and are deprecated by all major browsers.

TLS 1.3 is faster and more secure -- it reduces the handshake from two round trips to one, which means faster page loads on top of better security.

### 4. Certificate Chain

Your certificate needs to chain back to a trusted root certificate authority (CA). If intermediate certificates are missing, some browsers and devices will reject the connection even if the certificate itself is valid.

This is a common issue when installing certificates manually. Your CA will provide the intermediate certificates -- make sure you install them.

### 5. Subject Alternative Names (SANs)

The SAN field lists which domains the certificate covers. If your certificate is for `example.com` but your site is served at `www.example.com`, visitors to the `www` subdomain will see a certificate error.

Most modern certificates include both the bare domain and the `www` subdomain. Wildcard certificates (`*.example.com`) cover all subdomains.

## How to Check Your SSL Certificate from the Command Line

### Check expiry date

```bash
echo | openssl s_client -connect yourdomain.com:443 2>/dev/null \
  | openssl x509 -noout -enddate
```

### Check the full certificate details

```bash
echo | openssl s_client -connect yourdomain.com:443 2>/dev/null \
  | openssl x509 -noout -text
```

### Check the certificate chain

```bash
echo | openssl s_client -connect yourdomain.com:443 -showcerts 2>/dev/null
```

### Check TLS version

```bash
openssl s_client -connect yourdomain.com:443 -tls1_3 2>/dev/null
```

If this connects successfully, the server supports TLS 1.3.

## What to Do If Your Certificate Is Broken

### Expired certificate

Renew it. If you're using Let's Encrypt, run `certbot renew`. If you're using a commercial CA, log into their dashboard and reissue. Better yet, set up auto-renewal so this never happens again.

### Wrong domain

You need a new certificate that includes the correct domain(s). If you're using Let's Encrypt:

```bash
certbot certonly -d yourdomain.com -d www.yourdomain.com
```

### Missing chain certificates

Download the intermediate certificates from your CA and add them to your server configuration. The exact steps depend on your web server (Apache, Nginx, etc.).

### Old TLS version

Update your server's TLS configuration to disable TLS 1.0 and 1.1. For Nginx:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

For Apache:

```apache
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
```

## Automate Your SSL Monitoring

Checking manually is fine for a quick audit, but you should automate ongoing monitoring. Options include:

- **[GuardScan](https://guardscan.dev)** -- scan your site for free and get a shareable report covering SSL, headers, DNS, and cookies
- **Uptime monitoring services** like Uptime Robot or Pingdom that alert you before certificate expiry
- **Let's Encrypt + certbot** with auto-renewal and a cron job

The worst time to discover your SSL certificate has expired is when a customer emails you about it. Set up monitoring now.

## Key Takeaways

- Check your SSL certificate regularly -- expiry is the most common issue
- Use TLS 1.2 or 1.3, disable older versions
- Make sure your certificate chain is complete
- Verify your SANs cover all domains you serve
- Automate renewal and monitoring so you never get caught off guard

**Want to check your site right now?** Run a free scan at [guardscan.dev](https://guardscan.dev) -- you'll get your SSL grade plus security headers, DNS, and cookie analysis in seconds.
