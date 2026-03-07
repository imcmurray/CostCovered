---
title: "Permissions-Policy Header: What It Does and How to Set It"
date: 2026-03-07
draft: false
tags: ["security", "headers", "tutorial", "guardscan", "permissions-policy"]
categories: ["Security Guides"]
summary: "The Permissions-Policy header controls which browser features your site can access. Here's the full list of directives, what they do, and how to configure them on Nginx, Apache, and Cloudflare."
ShowToc: true
---

## Your Site Might Be Giving Away Access to the Camera

Here's something most web developers don't think about: by default, any JavaScript running on your page can request access to the user's camera, microphone, geolocation, and a long list of other sensitive browser APIs.

That includes JavaScript you didn't write. Third-party analytics scripts, ad networks, compromised CDN libraries, injected browser extensions -- they all run in your page's context, and they all inherit your page's permissions.

The **Permissions-Policy** header lets you lock that down. It tells the browser exactly which features your site is allowed to use, and which should be blocked outright -- even if a script tries to access them.

Think of it as a bouncer list for browser APIs: if a feature isn't on the list, it doesn't get in.

## What Is Permissions-Policy?

Permissions-Policy is an HTTP response header that controls which browser features and APIs can be used on your page (and in any embedded iframes).

It replaced the older `Feature-Policy` header in 2020. The syntax changed (from space-separated to a structured header format), but the goal is the same: give site owners fine-grained control over which powerful APIs are available.

When a feature is blocked by Permissions-Policy, the browser silently denies the API call. A script calling `navigator.geolocation.getCurrentPosition()` will get a permission error. A script calling `navigator.mediaDevices.getUserMedia()` for the camera will be blocked. No prompt, no fallback -- just denied.

## Why It Matters

Even if your site never uses the camera or microphone, there are good reasons to set this header:

**Defense in depth.** If a third-party script is compromised or an XSS vulnerability is exploited, the attacker's code can't access sensitive APIs you've blocked. It's another layer of protection that costs nothing to add.

**Iframe control.** Without Permissions-Policy, embedded iframes inherit the ability to request permissions. An ad iframe could prompt your users for microphone access. The header lets you control what iframes can do.

**Signal intent.** Setting the header tells browsers, security scanners, and auditors that you've consciously decided which features your site needs. It moves you from "everything is allowed by default" to "only what we need is allowed."

**Compliance.** Some security frameworks and audits look for Permissions-Policy as part of a defense-in-depth posture.

## The Full List of Directives

Here's every feature you can control with Permissions-Policy. The directive name goes on the left, and you assign an allowlist on the right.

### Sensitive Device Access

| Directive | What It Controls |
|-----------|-----------------|
| `camera` | Access to video input devices (webcam) |
| `microphone` | Access to audio input devices |
| `geolocation` | Access to the user's geographic location |
| `display-capture` | Ability to capture the screen or parts of it |
| `midi` | Access to MIDI devices (musical instruments, controllers) |
| `usb` | Access to USB devices via WebUSB |
| `serial` | Access to serial ports via Web Serial |
| `bluetooth` | Access to Bluetooth devices via Web Bluetooth |
| `hid` | Access to HID (human interface) devices |

### Payment and Credentials

| Directive | What It Controls |
|-----------|-----------------|
| `payment` | Use of the Payment Request API |
| `publickey-credentials-create` | Creating new WebAuthn/passkey credentials |
| `publickey-credentials-get` | Using existing WebAuthn/passkey credentials |
| `identity-credentials-get` | Use of the Federated Credential Management API |

### Media and Display

| Directive | What It Controls |
|-----------|-----------------|
| `autoplay` | Whether media can auto-play without user interaction |
| `fullscreen` | Whether the page can use the Fullscreen API |
| `picture-in-picture` | Whether the page can use Picture-in-Picture mode |
| `screen-wake-lock` | Whether the page can prevent the screen from dimming |
| `speaker-selection` | Whether the page can enumerate and select audio output devices |

### Performance and Behavior

| Directive | What It Controls |
|-----------|-----------------|
| `sync-xhr` | Whether synchronous XMLHttpRequests are allowed |
| `document-domain` | Whether `document.domain` can be set (legacy same-origin workaround) |
| `encrypted-media` | Use of the Encrypted Media Extensions API (DRM) |
| `web-share` | Whether the page can use the Web Share API |
| `gamepad` | Access to the Gamepad API |
| `gyroscope` | Access to the Gyroscope sensor |
| `accelerometer` | Access to the Accelerometer sensor |
| `magnetometer` | Access to the Magnetometer sensor |
| `ambient-light-sensor` | Access to the ambient light sensor |

### Client Hints (Optional)

| Directive | What It Controls |
|-----------|-----------------|
| `browsing-topics` | Whether the Topics API (Privacy Sandbox) is available |
| `attribution-reporting` | Whether the Attribution Reporting API is available |
| `compute-pressure` | Whether the Compute Pressure API is available |
| `local-fonts` | Whether the page can enumerate local fonts |
| `storage-access` | Whether embedded content can request storage access |
| `window-management` | Whether the page can manage windows across screens |

Not every browser supports every directive. Chrome tends to lead, with Firefox and Safari implementing a subset. Unknown directives are silently ignored, so it's safe to include them.

## Syntax Explained

Each directive takes an **allowlist** that specifies who can use the feature. The syntax uses structured headers format:

### Deny everyone (most restrictive)

```
camera=()
```

Empty parentheses means no one can use this feature -- not your page, not any iframe. This is what you want for features you don't use.

### Allow same-origin only

```
camera=(self)
```

Your page and same-origin iframes can use the feature, but cross-origin iframes cannot.

### Allow specific origins

```
camera=(self "https://trusted-app.example.com")
```

Your page and the specified origin can use the feature. Note that origins inside the parentheses are quoted (except `self` and `*`).

### Allow everyone (least restrictive)

```
camera=*
```

Any page or iframe can use the feature. You rarely want this.

### Combining multiple directives

Separate directives with commas:

```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), autoplay=(self)
```

## Recommended Starter Policy

For most websites that don't use camera, microphone, or location features, this is a practical starting point:

```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), midi=(), display-capture=(), bluetooth=(), serial=(), hid=(), publickey-credentials-create=(), publickey-credentials-get=(), accelerometer=(), gyroscope=(), magnetometer=(), ambient-light-sensor=(), autoplay=(self), fullscreen=(self), picture-in-picture=(self)
```

This denies all sensitive device and hardware access, while allowing common media features (autoplay, fullscreen, picture-in-picture) for your own origin.

**Adjust for your needs.** If your site uses video calls, allow `camera=(self)` and `microphone=(self)`. If you use geolocation, allow `geolocation=(self)`. The key is to deny everything you don't need and explicitly allow what you do.

A shorter version that covers the most important directives:

```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
```

This is the minimum recommended set. It blocks the most sensitive features and is easy to remember.

## How to Set It

### Nginx

Add the header to your `server` block:

```nginx
server {
    # ... your existing config ...

    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
}
```

The `always` parameter ensures the header is sent even on error responses. If you're using `add_header` in a `location` block, be aware that Nginx doesn't inherit `add_header` directives from parent blocks -- you'll need to repeat them.

### Apache

Add the header in your VirtualHost config or `.htaccess`:

```apache
Header always set Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()"
```

Make sure `mod_headers` is enabled: `sudo a2enmod headers && sudo systemctl restart apache2`

### Cloudflare

Cloudflare's dashboard path for adding response headers has changed recently. Here's the current process:

1. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com) and select your domain
2. In the left sidebar, click **Rules**
3. You'll land on the **Rules Overview** page
4. Click the **Create rule** button
5. Select **Response Header Transform Rule**
6. Give the rule a name (e.g., "Add Permissions-Policy header")
7. Under **When incoming requests match**, set the field to match all traffic (or a specific hostname)
8. Under **Then**, choose **Set static** for the header modification
9. Set **Header name** to `Permissions-Policy`
10. Set **Value** to `camera=(), microphone=(), geolocation=(), payment=()`
11. Click **Deploy**

The header will be added at Cloudflare's edge to all matching responses.

### Other Platforms

**Vercel** -- Add to your `vercel.json`:

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Permissions-Policy",
          "value": "camera=(), microphone=(), geolocation=(), payment=()"
        }
      ]
    }
  ]
}
```

**Netlify** -- Add to your `netlify.toml` or `_headers` file:

```toml
[[headers]]
  for = "/*"
  [headers.values]
    Permissions-Policy = "camera=(), microphone=(), geolocation=(), payment=()"
```

**Next.js** -- Add to your `next.config.js`:

```javascript
module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Permissions-Policy',
            value: 'camera=(), microphone=(), geolocation=(), payment=()',
          },
        ],
      },
    ];
  },
};
```

## Common Mistakes

**Using Feature-Policy instead of Permissions-Policy.** Feature-Policy is the old name with different syntax (`camera 'none'` instead of `camera=()`). Modern browsers expect Permissions-Policy. Some older guides still reference Feature-Policy -- don't follow them.

**Wrong syntax.** The most common syntax error is using spaces instead of commas between directives, or forgetting the parentheses. Each directive must have a value in parentheses (or `*`):

```
# Wrong
Permissions-Policy: camera 'none', microphone 'none'

# Wrong
Permissions-Policy: camera; microphone

# Correct
Permissions-Policy: camera=(), microphone=()
```

**Setting it as a meta tag.** Unlike some other security headers, Permissions-Policy cannot be set via `<meta>` tags. It must be an HTTP response header.

**Forgetting iframes.** If you embed third-party content in iframes and they need certain permissions, you'll also need to add the `allow` attribute to the iframe tag:

```html
<iframe src="https://maps.example.com" allow="geolocation"></iframe>
```

The iframe `allow` attribute and the Permissions-Policy header work together -- the iframe can only use features that are allowed by both.

## Verify Your Header

Want to check if your site has Permissions-Policy configured correctly? Run a free scan at [GuardScan](https://guardscan.com) -- it checks this header along with a dozen other security settings and tells you exactly what's missing.
