# Risk2Safe — 7-Day Trial Gate: Setup Guide

This package gives you a real, server-enforced 7-day trial for app.risk2safe.com.
Visitors fill in a form, get 7 days of access, and after that get redirected to a
"contact us for enterprise pricing" page. The check happens entirely on Cloudflare's
servers — clearing cookies or using incognito will NOT reset the trial, because the
authoritative record lives in Cloudflare KV (a server-side database), not the browser.

## What you're deploying

- `signup/index.html`      → the trial signup form (Name, Email, Company, Phone, Purpose, User count)
- `worker/worker.js`        → the Cloudflare Worker that issues trials and gates /tools/*
- `worker/trial-expired.html` → the page shown once 7 days are up
- `worker/wrangler.toml`    → Worker configuration

## Step 1 — Move risk2safe.com's DNS to Cloudflare (one-time, ~15 min + propagation)

Cloudflare Workers only run on domains whose DNS Cloudflare manages.

1. Sign up at https://dash.cloudflare.com (free plan is enough)
2. Click "Add a Site" → enter `risk2safe.com`
3. Cloudflare scans your existing DNS records and shows you a copy — review it
4. Cloudflare gives you 2 nameservers (e.g. `aida.ns.cloudflare.com`, `vince.ns.cloudflare.com`)
5. Log into your current registrar (GoDaddy/Namecheap/wherever risk2safe.com is registered)
   and replace the existing nameservers with Cloudflare's two
6. Wait for propagation (Cloudflare will email you once it's active — usually a few hours,
   can take up to 24-48h)

Your existing GitHub Pages CNAME records carry over automatically in this migration, so
your tool subdomains keep working throughout.

## Step 2 — Create the KV namespace (your trial database)

1. In the Cloudflare dashboard: Workers & Pages → KV → "Create a namespace"
2. Name it `TRIALS`
3. Copy the namespace ID it gives you
4. Paste that ID into `worker/wrangler.toml` in place of `REPLACE_WITH_YOUR_KV_NAMESPACE_ID`

## Step 3 — Install Wrangler (Cloudflare's CLI) and deploy

On your machine (requires Node.js installed):

```bash
npm install -g wrangler
wrangler login          # opens a browser to authorize your Cloudflare account
cd worker
wrangler secret put SIGNING_SECRET
# When prompted, paste any long random string — e.g. generate one with:
#   openssl rand -hex 32

wrangler deploy
```

Optional — if you want a Slack/email ping every time someone starts a trial:

```bash
wrangler secret put NOTIFY_WEBHOOK
# paste a Zapier/Make/Slack incoming webhook URL
```

## Step 4 — Upload the signup and trial-expired pages

These are static HTML, so they go wherever the rest of app.risk2safe.com is hosted
(your `kannanpnp.github.io` repo, or wherever you point app.risk2safe.com):

- `signup/index.html` → becomes the homepage visitors see at `app.risk2safe.com/`
- `worker/trial-expired.html` → upload as `app.risk2safe.com/trial-expired`

## Step 5 — Route your existing tools under /tools/

Right now your tools live at separate repos like
`kannanpnp.github.io/risk2safe-ra-studio/`. For the Worker to gate them, requests need
to come through paths starting with `/tools/` on app.risk2safe.com.

The cleanest way: keep your tool repos exactly as they are, and add a small
`_redirects` or proxy rule so:

```
app.risk2safe.com/tools/ra-studio/*      → kannanpnp.github.io/risk2safe-ra-studio/*
app.risk2safe.com/tools/hazop/*          → kannanpnp.github.io/https-risk2safe.github.io-HAZoP/*
app.risk2safe.com/tools/bowtie/*         → kannanpnp.github.io/bowtie/*
... etc for each tool
```

This is a few extra lines in the Worker (a routing table) — I've left this as a clearly
marked extension point in `worker.js` (`handleToolAccess`, the final `fetch(request)` call)
rather than hardcoding it, since I don't yet know your final URL structure decisions
(e.g. whether you're keeping the discipline-grouped landing page as the public homepage
and only gating the tool launches themselves, or gating from the very first click).

## How the trial actually works (so you can sanity-check it)

1. Visitor fills the form → Worker stores a record in KV: `{name, email, company, phone,
   purpose, userCount, start: <timestamp>}`, keyed by their email
2. Worker sets a signed cookie containing their email + start time
3. Every request to `/tools/*` is intercepted by the Worker, which:
   - Reads the cookie, verifies its signature (rejects tampered cookies)
   - Looks up the AUTHORITATIVE record in KV by email (not just trusting the cookie)
   - Checks if `now - start > 7 days`
   - If valid: passes the request through to the real tool
   - If expired or missing: redirects to `/trial-expired`
4. Because the source of truth is server-side KV, you can also manually extend, revoke,
   or whitelist any trial directly from the Cloudflare dashboard (KV → TRIALS → search
   by `trial:<email>`) without touching code — useful for VIP prospects or extensions
   you grant personally.

## Viewing your leads

Every signup is also stored permanently under a `lead:<timestamp>:<email>` key (separate
from the expiring trial record), so you have a durable list of everyone who's ever
signed up, even after their 7-day KV record expires. You can export these via:

```bash
wrangler kv:key list --namespace-id=<your-namespace-id> --prefix="lead:"
```

or browse them in the dashboard under KV → TRIALS.

## What I could not do for you

I don't have access to your domain registrar, Cloudflare account, or GitHub account —
DNS nameserver changes, KV namespace creation, and `wrangler deploy` all require you to
authenticate as yourself. Everything above is written so you (or any developer) can run
it step by step. If you get stuck on a specific step, paste me the exact error and I'll
help you debug it.
