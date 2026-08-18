# SOP: Setup and Deployment - Exp Intelligence

This document covers how to install requirements, set up all four services,
register the Shopify Extractor as an app on the Shopify Partner Dashboard, and
deploy everything to production.

The system has four services and data flows one way:

```
shopify-pp  ->  analytic service  ->  ai service 2  ->  review-hub
(extractor)     (analytics)           (AI suggestions)   (dashboard)
port 3000       port 4000             port 5001          port 5173
```

---

## 1. Requirements

Install these before doing anything else.

| Requirement | Why | How to get it |
|-------------|-----|---------------|
| Node.js 18 or newer, plus npm | All four services run on Node | https://nodejs.org |
| Git | Clone the repo and its submodules | https://git-scm.com |
| Chromium (via Playwright) | Analytics service takes store screenshots | Installed by a command in Step 3 |
| cloudflared (optional) | Expose local services over HTTPS for testing | `brew install cloudflared` |
| Shopify Partner account | Create and host the extractor app | https://partners.shopify.com/signup |
| A Shopify development store | Test installs safely | Create it inside the Partner Dashboard |
| AI provider API keys | AI service needs at least one | Groq, Gemini, Mistral, or Cerebras dashboards |

AI provider key sign-up pages:
- Groq: https://console.groq.com
- Gemini: https://aistudio.google.com/apikey
- Mistral: https://console.mistral.ai
- Cerebras: https://cloud.cerebras.ai

You only need one working AI key to start. Gemini is the default in `.env.example`.

---

## 2. Get the code

The four services are git submodules, so clone with `--recurse-submodules`:

```bash
git clone --recurse-submodules <repo-url>
cd "gl exp intelligence"
```

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

---

## 3. Install and configure each service

Run these from the project root.

### 3a. Analytics Service (port 4000)

```bash
cd "analytic service"
npm install
npx playwright install chromium
cp .env.example .env
```

No keys are required for local dev. The defaults in `.env` work as-is.

### 3b. AI Service (port 5001)

```bash
cd "../ai service 2"
npm install
cp .env.example .env
```

Open `ai service 2/.env` and set at minimum:

```
AI_PROVIDER=gemini
GEMINI_API_KEY=<your key>
ANALYTICS_SERVICE_URL=http://localhost:4000
SHOPIFY_APP_URL=http://localhost:3000
PORT=5001
```

Switch `AI_PROVIDER` to `groq`, `mistral`, or `cerebras` if you prefer, and fill
in that provider's key instead.

### 3c. Shopify Extractor (port 3000)

```bash
cd ../shopify-pp
npm install
cp .env.example .env
```

Generate an encryption key for storing tokens:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Open `shopify-pp/.env` and set:

```
SHOPIFY_CLIENT_ID=<from Partner Dashboard, Step 5>
SHOPIFY_CLIENT_SECRET=<from Partner Dashboard, Step 5>
SHOPIFY_SCOPES=read_reports,read_themes
APP_URL=<your public https url, see Step 5/7>
ANALYTICS_SERVICE_URL=http://localhost:4000/api/analytics/ingest
ENCRYPTION_KEY=<the 32-byte hex you just generated>
PORT=3000
```

For local testing, `APP_URL` must be a public HTTPS URL (Shopify cannot reach
`localhost`). Get one with a tunnel:

```bash
cloudflared tunnel --url http://localhost:3000
```

Copy the printed `https://<random>.trycloudflare.com` URL into `APP_URL`, and use
that same URL in the Partner Dashboard (Step 5).

### 3d. Review Hub (port 5173)

```bash
cd ../review-hub
npm install
```

The service URLs it talks to are set in `review-hub/src/lib/config.js`:

```
AI_URL         -> AI service       (http://localhost:5001 for local)
ANALYTICS_URL  -> Analytics service (http://localhost:4000 for local)
SHOPIFY_APP_URL -> Extractor        (http://localhost:3000 for local, or the deployed URL)
```

For a fully local run, point `AI_URL` and `ANALYTICS_URL` at localhost. This is a
browser app, so if you run the services on a different machine, use public HTTPS
URLs (tunnels or deployed URLs), not localhost.

---

## 4. Run everything locally

Start each service in its own terminal, in this order, and leave them running:

```bash
# Terminal 1
cd "analytic service" && npm start        # http://localhost:4000

# Terminal 2
cd "ai service 2" && npm start            # http://localhost:5001

# Terminal 3
cd shopify-pp && npm start                # http://localhost:3000

# Terminal 4
cd review-hub && npm run dev              # http://localhost:5173
```

Open http://localhost:5173 and log in with `admin` / `admin123`.

Each service should print that it is listening on its port with no red errors.

---

## 5. Register the Shopify Extractor as an app (Partner Dashboard)

Do this once. It gives you the Client ID and Client secret and tells Shopify
which URLs to trust.

1. Sign in at https://partners.shopify.com, go to **Apps**, click **Create app**,
   then **Create app manually**. Name it (for example `gl_extractor`).
2. Open the app's **Overview** or **Client credentials** page. Copy the
   **Client ID** into `SHOPIFY_CLIENT_ID` and the **Client secret** into
   `SHOPIFY_CLIENT_SECRET` in `shopify-pp/.env`.
3. Go to **Configuration** and set:
   - **App URL**: your public HTTPS base URL (same value as `APP_URL`), for
     example `https://shopify-extractor-production.up.railway.app`.
   - **Allowed redirection URL(s)**: your base URL plus `/auth/callback`, for
     example `https://<domain>/auth/callback`. This must match exactly or OAuth
     fails.
4. Under **API scopes** / **Protected customer data access**, request the scopes
   in `SHOPIFY_SCOPES`: `read_reports` and `read_themes`.
5. Under **Compliance webhooks** (also called GDPR or mandatory webhooks), set all
   three to your domain:
   - `customers/data_request` -> `https://<domain>/webhooks/customers/data_request`
   - `customers/redact` -> `https://<domain>/webhooks/customers/redact`
   - `shop/redact` -> `https://<domain>/webhooks/shop/redact`
6. Save.

The `app/uninstalled` webhook is registered automatically by the app after each
install, so you do not configure it here.

### Faster alternative: push config with the Shopify CLI

`shopify-pp/shopify.app.toml` already contains the app name, client id, scopes,
redirect URL, and the three compliance webhook URLs. Instead of clicking through
the dashboard, edit every URL in that file to point at your domain, then run:

```bash
cd shopify-pp
npm install -g @shopify/cli @shopify/app   # once
shopify app deploy
```

This applies the configuration to the app automatically.

---

## 6. Install the app on a store

Once the extractor is deployed and the Partner Dashboard is configured, install
it on a store by opening:

```
https://<your-domain>/auth?shop=THEIR-STORE.myshopify.com
```

Or send the merchant to `https://<your-domain>/` and let them type their
`*.myshopify.com` domain into the install page. They click **Install**, approve
the scopes, and the app stores their encrypted token.

Trigger a data sync for an installed store:

```bash
curl -X POST "https://<your-domain>/api/sync?shop=THEIR-STORE.myshopify.com" \
  -H "Content-Type: application/json" -d '{}'
```

---

## 7. Deploy to production

Each service deploys independently. The reference host is Railway
(https://railway.app); Render, Fly.io, or a VPS also work.

### 7a. Deploy the Shopify Extractor (needs a persistent disk)

1. Push `shopify-pp` to GitHub.
2. In Railway: **New Project -> Deploy from GitHub repo**, select the repo.
   Railway auto-detects Node, runs `npm install`, then `npm start`.
3. Add a **Volume** (service -> **Volumes -> New Volume**) mounted at `/data`.
   This keeps installed shops across redeploys.
4. Set **Variables**:
   ```
   SHOPIFY_CLIENT_ID=<from Partner Dashboard>
   SHOPIFY_CLIENT_SECRET=<from Partner Dashboard>
   SHOPIFY_SCOPES=read_reports,read_themes
   APP_URL=https://<your-service>.up.railway.app
   ANALYTICS_SERVICE_URL=https://<your-analytics-service>/api/analytics/ingest
   DB_PATH=/data/shops.db
   ENCRYPTION_KEY=<32-byte hex>
   ```
   Leave `PORT` unset; Railway injects it.
5. Generate a domain (**Settings -> Networking -> Generate Domain**). Copy it into
   `APP_URL` and into the Partner Dashboard (App URL, redirect URL, and all three
   compliance webhook URLs). They must all match this domain.
6. Redeploy. Visit `https://<domain>/` to confirm the install page loads.

Critical: `DB_PATH` must point at the mounted volume (`/data/shops.db`). If it
points at the app directory, every redeploy wipes all installed shops and their
tokens. Also, any domain change means updating `APP_URL` and all matching Partner
Dashboard URLs, or OAuth and webhooks break.

### 7b. Deploy the Analytics Service (needs the Playwright image)

This service uses a Dockerfile based on the official Playwright image so Chromium
is available.

1. Push `analytic service` to GitHub.
2. In Railway: **New Project -> Deploy from GitHub repo**. Railway detects the
   `Dockerfile` and builds it.
3. Set variables (all optional for a basic run):
   ```
   PORT=<Railway injects this>
   PUBLIC_URL=https://<your-analytics-service>
   ```
   For persistent screenshot storage in production, also set the five R2 variables
   (`R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`,
   `R2_PUBLIC_URL`) together. Without them, screenshots use local disk, which is
   ephemeral on Railway.
4. Generate a domain. This URL is what the extractor's `ANALYTICS_SERVICE_URL` and
   the Review Hub's `ANALYTICS_URL` must point to.

### 7c. Deploy the AI Service

1. Push `ai service 2` to GitHub.
2. In Railway: **New Project -> Deploy from GitHub repo**. Auto-detects Node.
3. Set variables:
   ```
   AI_PROVIDER=gemini
   GEMINI_API_KEY=<your key>
   CODE_PROVIDER=gemini
   ANALYTICS_SERVICE_URL=https://<your-analytics-service>
   SHOPIFY_APP_URL=https://<your-extractor>
   PORT=<Railway injects this>
   ```
4. Generate a domain. This URL is what the Review Hub's `AI_URL` must point to.

Note: free-tier AI keys hit rate limits. `TUNNELS.md` documents running the AI and
Analytics services locally behind Cloudflare tunnels to avoid this during demos.

### 7d. Deploy the Review Hub (static site)

The Review Hub is a Vite React app that builds to static files.

1. Edit `review-hub/src/lib/config.js` so `AI_URL`, `ANALYTICS_URL`, and
   `SHOPIFY_APP_URL` point at the deployed service domains from 7a to 7c.
2. Build:
   ```bash
   cd review-hub && npm install && npm run build
   ```
   Output goes to `review-hub/dist`.
3. Deploy `dist` to any static host (Netlify, Vercel, Cloudflare Pages, or Railway
   static). On Railway, use a static-site service pointing at the build output.

### Deployment order

Deploy in dependency order so each service can reach the one before it:

```
Analytics Service  ->  AI Service  ->  Shopify Extractor  ->  Review Hub
```

After all four are live, set each service's URL variable to point at the deployed
domain of the service before it, then redeploy any service whose variables changed.

---

## 8. Verify a full run

1. All four services are deployed and reachable over HTTPS.
2. Partner Dashboard App URL, redirect URL, and webhook URLs all match the
   extractor's live domain.
3. Install the app on a development store (Step 6).
4. Open the deployed Review Hub, log in as `admin` / `admin123`, enter the store
   domain, and click **Sync to Dashboard**.
5. The header should show data source `ai` (not `demo`), and suggestions should
   appear. Approve one, log in as `dev1` / `dev123`, and confirm the approved
   suggestion shows a code patch.

If the header shows `demo`, the AI or Analytics service is unreachable; recheck the
URLs in `config.js` and that both services are running.
