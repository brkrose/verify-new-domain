# verify-new-domain

Register a Vercel-hosted domain with **Google Search Console** + **Bing Webmaster Tools** in one command.

```bash
verify-new-domain example.com
```

- Mints the Google verification token, adds it as a TXT record on Vercel, triggers verification, adds the `sc-domain` property to Search Console, submits your sitemap.
- Adds the site to Bing Webmaster Tools and opens the verify page for the CNAME step (Bing's REST API doesn't expose the per-site token).

Built for the typical "I just pointed a fresh domain at Vercel, get it indexed" workflow. ~90 seconds per domain.

## Requirements

- macOS or Linux
- Python 3.9+
- [`vercel` CLI](https://vercel.com/docs/cli), logged in to the account that owns the domain
- `curl`, `jq` (preinstalled on macOS)
- A Google account that owns Search Console
- A Microsoft account with Bing Webmaster Tools API access

## Install

```bash
git clone https://github.com/brkrose/verify-new-domain.git
sudo cp verify-new-domain/bin/verify-new-domain /usr/local/bin/
# or symlink it into a ~/bin on your PATH
```

If you use [Claude Code](https://claude.com/claude-code), drop the skill into place so Claude can invoke it:

```bash
mkdir -p ~/.claude/skills/verify-new-domain
cp verify-new-domain/skill/SKILL.md ~/.claude/skills/verify-new-domain/
```

## One-time setup — Bing (~5 min)

1. Sign in at https://www.bing.com/webmasters
2. Click the profile icon (top right) → **API access**
3. Click **Generate** and copy the API key
4. Run `verify-new-domain setup-bing` and paste it

## One-time setup — Google (~15 min)

1. Open https://console.cloud.google.com/ and pick or create a project
2. **APIs & Services → Library** → enable both:
   - "Site Verification API"
   - "Google Search Console API"
3. **APIs & Services → OAuth consent screen** → External → fill app name, your email twice, add scopes `siteverification` + `webmasters`, add your gmail as a test user
4. **APIs & Services → Credentials → Create Credentials → OAuth client ID → Desktop app**
5. Run `verify-new-domain setup-google`, paste Client ID + Secret, approve in the browser tab that opens

## Usage

```bash
verify-new-domain example.com
verify-new-domain example.com --sitemap https://example.com/custom-sitemap.xml
verify-new-domain example.com --skip-bing
verify-new-domain example.com --skip-google
```

## What it does per domain

**Google** (fully automated):
1. Site Verification API → get `google-site-verification=...` token
2. `vercel dns add <domain> @ TXT <token>` (skipped if already present)
3. Wait 30s, retry verification up to 4×
4. Add `sc-domain:<domain>` to Search Console
5. Submit sitemap

**Bing** (semi-automated):
1. `AddSite` for `https://<domain>/` via API
2. Opens the Bing verify page in browser
3. You pick "Add CNAME record to DNS", grab the auth code
4. Run `vercel dns add <domain> <auth-code> CNAME verify.bing.com`
5. Click Verify in the Bing UI

(If you have Google Search Console already linked to Bing under Settings → Google search console accounts, Bing will auto-import + verify the site within ~24h, so the manual step is optional.)

## Why Bing isn't fully automated

Bing's Webmaster Tools REST API exposes `AddSite`, `VerifySite`, `SubmitSitemap` — but not an endpoint to retrieve the per-site CNAME verification token. You can only get the token via the UI. PRs welcome if this changes.

## Files

After setup, credentials live in `~/.config/verify-domain/` (chmod 600):

- `google.json` — `{client_id, client_secret, refresh_token}`
- `bing.json` — `{api_key}`

Both files are gitignored — they live only on your machine.

## Credit

Built for the [Brooke Builds Stuff](https://brookebuildsstuff.com) portfolio workflow.

## License

MIT
