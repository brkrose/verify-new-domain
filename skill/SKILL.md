---
name: verify-new-domain
description: Use when you register or ship a new domain (especially via Vercel) and need it added to Google Search Console + Bing Webmaster Tools. Triggers on phrases like "set up search console", "add to GSC", "verify with Google/Bing", "submit sitemap", or any time a new domain is going live.
---

# verify-new-domain

## Overview

Wraps `bin/verify-new-domain` — a Python CLI that, given a domain on Vercel DNS, adds the necessary DNS records and registers the domain with **Google Search Console** and **Bing Webmaster Tools** via their APIs, then submits the sitemap.

Per-domain runtime: ~90 seconds (mostly DNS-propagation waits).

## When to use

- A new domain just got pointed at Vercel and you want it indexed.
- User says "set up search console for X", "add X to Google + Bing", "submit sitemap for X".

**Do NOT use for:**
- Domains not on Vercel DNS (script edits Vercel DNS specifically).
- Subdomains where the apex is already verified (sc-domain property covers all subdomains).

## Quick reference

```bash
# one-time setup
verify-new-domain setup-google    # OAuth dance, stores refresh token at ~/.config/verify-domain/google.json
verify-new-domain setup-bing      # paste API key, stores at ~/.config/verify-domain/bing.json

# per new domain
verify-new-domain example.com
verify-new-domain example.com --sitemap https://example.com/custom-sitemap.xml
verify-new-domain example.com --skip-bing   # google only
```

See the project README for the full one-time setup steps (Google Cloud Console + Bing Webmaster).

## What it does per domain

1. **Google** (fully automated):
   - Calls Site Verification API → gets a `google-site-verification=...` token.
   - Adds it as a TXT record on the apex via `vercel dns add`.
   - Waits 30s, retries verification up to 4× with 20s backoff.
   - Adds `sc-domain:<domain>` to Search Console.
   - Submits sitemap (`https://<domain>/sitemap.xml` by default).
2. **Bing** (semi-automated — Bing's REST API doesn't expose the per-site CNAME token):
   - `AddSite` for `https://<domain>/` via API.
   - Opens the Bing verify page in browser; pick "Add CNAME record to DNS", grab the auth code, then `vercel dns add <domain> <auth-code> CNAME verify.bing.com` and click Verify.
   - **Fallback**: if you have Google Search Console linked to Bing (Settings → Google search console accounts), the site auto-imports + verifies within 24h — no manual step needed.

Idempotent: skips Vercel records whose value already exists.

## Common gotchas

- **No `refresh_token` returned** during `setup-google` → revoke prior grant at https://myaccount.google.com/permissions and rerun.
- **Verification fails 4× in a row** → DNS hasn't propagated. Rerun after a few minutes; the script is idempotent.
- **Sitemap doesn't exist yet** → submission accepted but unprocessed. Deploy a sitemap, then rerun.

## Files (after setup)

- `~/.config/verify-domain/google.json` — client_id, client_secret, refresh_token (chmod 600)
- `~/.config/verify-domain/bing.json` — api_key (chmod 600)
