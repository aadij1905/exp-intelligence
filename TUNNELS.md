# Cloudflare tunnels for local dev

Why you'd want this: the Railway deployments of `ai service 2` and
`analytic service` are on free-tier plans with real rate limits (we hit
Groq's daily token cap and Gemini's "high demand" 503s during testing).
Running those two services **locally** and exposing them through a
Cloudflare quick tunnel lets `review-hub` (and anything else) reach them
over a real HTTPS URL without deploying, while `shopify-pp` (the Shopify
extractor/theme-code source) stays on its Railway deployment as-is — it
doesn't need a tunnel for this workflow.

## Prerequisite

```bash
brew install cloudflared
```

No Cloudflare account or login needed — `cloudflared tunnel --url ...` uses
Cloudflare's free "quick tunnel" product, which mints a random
`https://<random-words>.trycloudflare.com` URL on the spot.

## Step by step

**1. Start the local services first**, each in its own terminal tab (they
must keep running — closing the tab or letting the machine sleep kills
them):

```bash
cd "analytic service" && npm start   # http://localhost:4000
cd "ai service 2" && npm start       # http://localhost:5001
```

**2. Start a tunnel per service**, each in its own terminal tab:

```bash
cloudflared tunnel --url http://localhost:4000   # for analytic service
cloudflared tunnel --url http://localhost:5001   # for ai service 2
```

Each prints a line like:

```
2026-07-13T18:46:12Z INF |  https://led-performances-suggestions-councils.trycloudflare.com  |
```

Copy that URL — you need it in the next step.

**3. Wire the URLs into every service that talks to them:**

| Tunnel for       | Paste the URL into                                          | Variable / field           |
| ----------------- | ------------------------------------------------------------ | --------------------------- |
| analytic service   | `ai service 2/.env`                                           | `ANALYTICS_SERVICE_URL`     |
| analytic service   | `review-hub/src/lib/config.js`                                 | `ANALYTICS_URL`             |
| ai service 2        | `review-hub/src/lib/config.js`                                 | `AI_URL`                    |

`SHOPIFY_APP_URL` (in `ai service 2/.env` and `review-hub/src/lib/config.js`)
stays pointed at the deployed extractor —
`https://shopify-extractor-production.up.railway.app` — no tunnel needed
there.

**4. Restart `ai service 2` after editing `.env`.**
`dotenv` only loads `.env` once, at process startup — editing the file while
`npm start` is already running does nothing until you kill and restart it:

```bash
lsof -tiTCP:5001 -sTCP:LISTEN | xargs kill
cd "ai service 2" && npm start
```

`review-hub`'s Vite dev server picks up `config.js` edits on save (HMR), no
restart needed there.

## Gotchas learned the hard way

- **Tunnel URLs are single-use.** Every time `cloudflared tunnel --url ...`
  (re)starts, it mints a brand-new random URL — it does not reconnect to the
  old one. If a tunnel process dies and you restart it, you must re-paste
  the new URL everywhere in the table above.
- **All four processes (2 servers + 2 tunnels) are independent and
  foreground-ish** — none of them are daemons/services that survive a
  terminal closing or the machine sleeping. If everything suddenly stops
  responding at once, it's almost always "a terminal got closed" or "the
  Mac slept," not an application bug — check `lsof -iTCP:4000` /
  `-iTCP:5001` and `ps aux | grep cloudflared` before debugging the code.
- **Syncing via the deployed `shopify-pp` does NOT populate your local
  tunneled `analytic service`.** The deployed extractor's own
  `ANALYTICS_SERVICE_URL` (set in its Railway environment, not anything in
  this repo) points at the deployed `analytic service`, not your tunnel.
  Real syncs triggered through the production extractor land in Railway's
  analytics data, not your local instance's. To get fresh crawled data into
  your local tunneled instance, either run `shopify-pp` locally too (with
  *its* `ANALYTICS_SERVICE_URL` pointed at `http://localhost:4000/api/analytics/ingest`),
  or seed the local instance's cache directly.
- **`ai service 2`'s own `ANALYTICS_SERVICE_URL` can just be
  `http://localhost:4000`** instead of the tunnel URL, if both services run
  on the same machine — the tunnel is only required for a *browser*
  (`review-hub`) to reach them from outside that machine's `localhost`.
  Using the tunnel URL for server-to-server calls works too, just adds an
  extra network hop.
