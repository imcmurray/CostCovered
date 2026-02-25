---
title: "How to Check If Your Website Has SPF and DMARC Records (And Why Email Security Matters)"
date: 2026-02-24
draft: false
tags: ["dns", "spf", "dmarc", "email", "security"]
categories: ["Security Guides"]
summary: "SPF and DMARC records stop attackers from sending emails as your domain. Here's how to check yours in 10 seconds, and what to do if they're missing."
ShowToc: true
---

Someone is probably sending email from your domain right now. Not you -- someone pretending to be you.

Without SPF and DMARC records, anyone can send an email that looks like it came from `yourcompany.com`. Phishing attacks, fake invoices, password reset scams -- all using your domain name, all landing in your customers' inboxes.

The fix takes five minutes. Here's how to check if you're protected, and what to do if you're not.

## What Are SPF and DMARC?

### SPF (Sender Policy Framework)

SPF is a DNS record that says "these servers are allowed to send email for my domain." When someone receives an email claiming to be from your domain, their mail server checks your SPF record. If the sending server isn't on the list, the email gets flagged or rejected.

An SPF record looks like this:

```
v=spf1 include:_spf.google.com include:sendgrid.net -all
```

This says: Google and SendGrid can send email for us. Everyone else should be rejected (`-all`).

### DMARC (Domain-based Message Authentication, Reporting & Conformance)

DMARC builds on SPF (and DKIM) to tell receiving mail servers what to do when authentication fails. Without DMARC, a failing SPF check might still get delivered -- the receiving server doesn't know how strict you want to be.

A DMARC record looks like this:

```
v=DMARC1; p=reject; rua=mailto:dmarc-reports@yourdomain.com
```

This says: if an email fails authentication, reject it. And send us reports about it.

## Why This Matters Even If You Don't Send Email

Here's the part most people miss: **you need SPF and DMARC even if your domain doesn't send email.**

If `yourdomain.com` has no SPF record, an attacker can send emails as `ceo@yourdomain.com` and nothing will stop them. The receiving mail server has no way to know those emails aren't legitimate.

For domains that don't send email, you still want:

```
v=spf1 -all
```

This explicitly says "no server is authorized to send email for this domain." It's a one-line fix that blocks an entire category of attacks.

## How to Check Your Records

### The Quick Way

Scan your domain with [GuardScan](https://guardscan.dev). It checks SPF, DMARC, and DNSSEC as part of a full security scan, and shows you the actual record values.

### The Command-Line Way

Check your SPF record:

```bash
dig +short TXT yourdomain.com | grep "v=spf1"
```

Check your DMARC record:

```bash
dig +short TXT _dmarc.yourdomain.com
```

If either command returns nothing, you're missing that record.

### What Good Results Look Like

**SPF present and valid:**
```
"v=spf1 include:_spf.google.com ~all"
```

**DMARC present with enforcement:**
```
"v=DMARC1; p=quarantine; rua=mailto:dmarc@yourdomain.com"
```

### What Bad Results Look Like

**No SPF record:** The `dig` command returns nothing. Anyone can send email as your domain.

**SPF with `+all`:** This explicitly allows every server on the internet to send email as you. It's worse than having no record at all, because it looks intentional.

**DMARC with `p=none`:** The record exists but does nothing. It's monitoring mode -- useful temporarily while you're setting up, but not a real defense. Failed emails still get delivered.

## How to Add SPF and DMARC Records

### Step 1: Figure Out Who Sends Email for You

Before you add an SPF record, list every service that sends email using your domain:

- **Google Workspace** -- `include:_spf.google.com`
- **Microsoft 365** -- `include:spf.protection.outlook.com`
- **SendGrid** -- `include:sendgrid.net`
- **Mailchimp** -- `include:servers.mcsv.net`
- **Amazon SES** -- `include:amazonses.com`

Check your current email provider's documentation for their specific SPF include.

### Step 2: Create Your SPF Record

Add a TXT record to your domain's DNS:

**If you send email:**
```
v=spf1 include:_spf.google.com -all
```
Replace the `include:` with your actual email provider. Use `-all` (hard fail) to reject unauthorized senders.

**If you don't send email:**
```
v=spf1 -all
```

### Step 3: Create Your DMARC Record

Add a TXT record at `_dmarc.yourdomain.com`:

**Start with monitoring (recommended for new setups):**
```
v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com
```

This doesn't block anything yet, but sends you reports about who's sending email as your domain. Run this for a week to make sure your legitimate email still works.

**Then move to enforcement:**
```
v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@yourdomain.com
```

This sends failing emails to spam. Once you're confident, upgrade to `p=reject` to block them entirely.

### Where to Add These Records

The process depends on your DNS provider:

**Cloudflare:** DNS > Records > Add Record > Type: TXT

**Namecheap:** Advanced DNS > Add New Record > TXT Record

**Route 53:** Hosted Zones > your domain > Create Record > TXT

**GoDaddy:** DNS Management > Add > TXT

The name field is `@` for SPF (applies to the root domain) and `_dmarc` for DMARC.

## Common Mistakes

**Multiple SPF records.** You can only have one SPF record per domain. If you need multiple providers, combine them:

```
v=spf1 include:_spf.google.com include:sendgrid.net -all
```

Don't create two separate TXT records with `v=spf1`. This will break SPF entirely.

**Too many DNS lookups.** SPF has a limit of 10 DNS lookups. Each `include:` counts as one. If you're using many email services, you might hit this limit. Use an [SPF flattening tool](https://www.autospf.com/) to compress your record.

**Forgetting subdomains.** SPF and DMARC for `yourdomain.com` don't automatically cover `mail.yourdomain.com` or `app.yourdomain.com`. If you send email from subdomains, they need their own records. DMARC has an `sp=` tag for subdomain policy:

```
v=DMARC1; p=reject; sp=reject; rua=mailto:dmarc-reports@yourdomain.com
```

**Jumping straight to `p=reject`.** If your SPF record is misconfigured, a DMARC reject policy will block your own legitimate email. Always start with `p=none`, review the reports, then tighten.

## Check Your Domain Now

The fastest way to see where you stand is to run a free scan at [GuardScan](https://guardscan.dev). It checks SPF, DMARC, DNSSEC, plus HTTP headers, SSL certificates, and cookie security -- all in about 10 seconds.

If your SPF or DMARC records are missing, GuardScan flags them with a clear explanation of the risk. Beta users also get platform-specific fix instructions for setting up the records on Cloudflare, AWS, and other providers.

It takes five minutes to add these records. It takes much longer to recover from a phishing attack that used your domain.
