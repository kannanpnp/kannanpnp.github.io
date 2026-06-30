# Risk2Safe — 7-Day Trial Gate (Signup-Wall Mode)

This is the simplified flow you confirmed: nobody sees anything on app.risk2safe.com
until they fill in the trial form. Once submitted, they get 7 days of full access to
everything (your tool catalog page + every individual tool). After 7 days, every
request redirects to a "contact us" page with a pre-filled email for enterprise
enquiries.

The 7-day check happens entirely on Cloudflare's servers (in a KV store), not in the
visitor's browser — so clearing cookies or using incognito does not reset the clock.

## Files in this package

| File | What it is | Where it goes |
|---|---|---|
| `signup/index.html` | The trial signup form — this becomes `app.risk2safe.com/` | GitHub Pages |
| `worker/trial-expired.html` | "Trial ended, contact us" page | GitHub Pages |
| `worker/worker.js` | The gatekeeper logic | Cloudflare (NOT GitHub) |
| `worker/wrangler.toml` | Worker configuration | Stays on your machine |
| Your existing tool repos (RA Studio, HAZOP Studio, etc.) | Unchanged | Stay on GitHub Pages exactly as they are |

---

## ⚠️ Before you test: this won't work until Part B is done

If you open `signup/index.html` directly on GitHub Pages (or as a local file) and
click "Start 7-day trial" before finishing Part B below, you'll see an error like
`Unexpected token '<', "<html> <he"... is not valid JSON`. **This is expected, not a
bug.** GitHub Pages only serves static files — there's no `/api/start-trial`
endpoint there at all, so the request falls through to a 404 page, and the form's
JSON parser chokes on the `<` at the start of that HTML page.

The form has been updated to catch this case and show a plain message instead
("the trial system isn't connected yet on this domain…"), but the real fix is
completing **Part B — Cloudflare steps**, which is what actually makes
`/api/start-trial` exist and respond with real JSON. Until then, the signup page
will render correctly and look right, it just can't process a submission yet.

If you want to verify the form logic works before committing to the full DNS
migration, see "Testing locally before going live" near the bottom of this file.

---



### A1. Replace your homepage with the signup form

1. Open your `kannanpnp.github.io` repository on GitHub
2. Open `index.html` in the file browser (top right pencil icon to edit, or click
   the file then "Edit this file")
3. Select all existing content, delete it
4. Paste in the full contents of `signup/index.html` from this package
5. Scroll down, write a commit message like "Add trial signup form", click
   **Commit changes**

### A2. Add the trial-expired page

1. In the same repo, click **Add file → Create new file**
2. Name it exactly `trial-expired.html` (root of the repo, same level as `index.html`)
3. Paste in the full contents of `worker/trial-expired.html`
4. Commit

### A3. Where does your discipline-grouped tool catalog page go?

You'll still want the nicely designed page listing HAZOP Studio, RA Studio, BowTie
Studio etc. (the one we built earlier with the five pillars write-up) — it just
shouldn't be the very first thing people see anymore, since the signup form takes
that spot now.

Add it as a second page in the same repo:

1. **Add file → Create new file** → name it `catalog.html`
2. Paste in the discipline-grouped landing page content (the version with the
   animated top border, five pillars, and tool cards)
3. Commit

This becomes reachable at `app.risk2safe.com/catalog` — which is exactly where the
signup form's "Go to the toolkit →" link points after someone completes the trial
form, so the flow connects correctly.

### A4. Your tool repos — no changes needed

`risk2safe-ra-studio`, `https-risk2safe.github.io-HAZoP`, `bowtie`, etc. stay exactly
as they are, with GitHub Pages already enabled on each, same as before.

That's everything on the GitHub side. The remaining steps are all on Cloudflare and
are what actually enforces the 7-day cutoff — without them, the signup form would
just collect emails with no real gate behind it.

---

## Part B — Cloudflare steps (this is what makes the gate real)

### B1. Move risk2safe.com's DNS to Cloudflare

Cloudflare Workers only run on domains it manages DNS for.

1. Sign up free at https://dash.cloudflare.com
2. "Add a Site" → enter `risk2safe.com` → Cloudflare scans your existing DNS records
3. It gives you 2 nameservers (e.g. `aida.ns.cloudflare.com`, `vince.ns.cloudflare.com`)
4. Log into your current registrar (wherever you bought risk2safe.com) and replace
   the nameservers there with Cloudflare's two
5. Wait for propagation — Cloudflare emails you once active (often a few hours, up
   to 24–48h). Your existing GitHub Pages CNAME records carry over automatically, so
   nothing breaks during the wait.

### B2. Create the KV namespace (this is your trial database)

1. Cloudflare dashboard → Workers & Pages → KV → **Create a namespace**
2. Name it `TRIALS`
3. Copy the namespace ID shown
4. Open `worker/wrangler.toml` in this package and paste that ID in place of
   `REPLACE_WITH_YOUR_KV_NAMESPACE_ID`

### B3. Deploy the Worker

On your machine, with Node.js installed:

```bash
npm install -g wrangler
wrangler login
cd worker
wrangler secret put SIGNING_SECRET
# paste any long random string when prompted — generate one with:
#   openssl rand -hex 32

wrangler deploy
```

Optional, if you want a notification (Slack/email) every time someone signs up:

```bash
wrangler secret put NOTIFY_WEBHOOK
# paste a Zapier/Make/Slack incoming webhook URL
```

That's it — once deployed, the Worker automatically intercepts every request to
`app.risk2safe.com`, on every path, and enforces the rule: no valid trial → signup
form only; valid trial → everything; expired trial → the contact page.

---

## How to verify it's working

1. Visit `app.risk2safe.com` in a private/incognito window — you should see only the
   signup form, nothing else reachable
2. Fill in the form and submit — you should land on the success view with your trial
   start/expiry dates shown
3. Click "Go to the toolkit" — `catalog.html` and every tool repo should now load
4. To test expiry without waiting 7 real days: open the Cloudflare dashboard → KV →
   TRIALS → find your `trial:<youremail>` key → edit the `start` value to a timestamp
   8 days in the past → refresh the catalog page → you should be redirected to
   `/trial-expired`

## Managing trials manually (no code needed)

Because the trial record lives in Cloudflare KV, you can do all of this directly from
the dashboard, anytime:

- **Extend someone's trial** — edit their `trial:<email>` record's `start` value to a
  more recent timestamp
- **Revoke access early** — delete their `trial:<email>` key
- **See every lead you've ever captured** — browse keys prefixed `lead:` (these never
  expire, even after the 7-day trial record does), or export via:
  ```bash
  wrangler kv:key list --namespace-id=<your-namespace-id> --prefix="lead:"
  ```

## Post-trial feedback (star rating + merits/demerits/improvements)

The `/trial-expired` page now shows two panels stacked together: the existing
"contact us" CTA, and a feedback form underneath it. The contact CTA is NOT gated
behind submitting feedback — someone can email you about enterprise pricing whether
or not they fill in the survey, so you never lose a hot lead over an unanswered form.

The feedback form asks for:
- **A 1–5 star rating** (required — click a star, hover to preview)
- **Merits** — what worked well (optional)
- **Demerits** — what didn't (optional)
- **Improvements** — what to build or fix next (optional)

It identifies the visitor automatically via their existing trial cookie (the same one
set when they signed up), so there's no second login or email re-entry required.

Every submission is stored permanently in KV under a `feedback:<timestamp>:<email>`
key, separate from the (expiring) trial record, alongside the rating and all three
text fields plus their name/company pulled from their original signup. View or export
the same way as leads:

```bash
wrangler kv:key list --namespace-id=<your-namespace-id> --prefix="feedback:"
```

If you've set `NOTIFY_WEBHOOK`, you'll also get a Slack/email ping the moment someone
submits feedback, with the rating and all three answers included in the message.

## Testing locally before going live (optional)

If you want to confirm the signup → trial → expiry → feedback logic works correctly
before doing the full DNS migration, you can run the Worker on your own machine:

```bash
cd worker
wrangler dev
```

This starts a local server (usually at `http://localhost:8787`) running your actual
Worker code against a local test KV store — no real DNS or deployment needed yet.

To point the signup form at it temporarily: open `signup/index.html`, find the line
`fetch('/api/start-trial', {`, and change it to
`fetch('http://localhost:8787/api/start-trial', {`. Do the same in
`worker/trial-expired.html` for the `/api/submit-feedback` call. **Remember to
change both back to their relative paths (`/api/start-trial` and
`/api/submit-feedback`) before deploying for real** — in production, the Worker and
the pages share the same domain, so relative paths are correct and required.

## What I can't do for you

DNS nameserver changes, creating the Cloudflare KV namespace, and running
`wrangler deploy` all require your own account login — I don't have access to your
domain registrar, Cloudflare account, or GitHub account. Everything above is written
so you can do it yourself step by step. If any step throws an error, paste me the
exact message and I'll help you debug it.
