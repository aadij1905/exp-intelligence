# Exp Intelligence

AI-assisted storefront review pipeline for Shopify stores: an extractor app pulls
store data, an analytics service normalizes and scores it, an AI service turns
that into suggestions (with code patches), and a review dashboard lets a PO
approve/reject them for developers to pick up.

## Services

| Service            | Folder             | Port   | Purpose                                              |
| ------------------ | ------------------- | ------ | ----------------------------------------------------- |
| Shopify extractor   | `shopify-pp`         | 3000   | Installs on a store, extracts data, syncs to Analytics |
| Analytics Service   | `analytic service`   | 4000   | Normalizes/ingests store data, serves analytics API    |
| AI Service          | `ai service 2`       | 5001   | Generates suggestions + code patches from analytics    |
| Review Hub          | `review-hub`         | 5173   | React dashboard — PO review + developer view           |

Requests flow: `shopify-pp` → `analytic service` → `ai service 2` → `review-hub`.
`review-hub` also falls back to an embedded demo dataset if the other services
aren't running, so it's demoable on its own.

## Prerequisites

- Node.js 18+ and npm

## Quick start

Install dependencies once per service:

```bash
cd "analytic service" && npm install
cd "../ai service 2" && npm install
cd ../shopify-pp && npm install
cd ../review-hub && npm install
```

Set up environment variables (secrets are gitignored, so copy the examples and
fill in your own keys):

```bash
cp "ai service 2/.env.example" "ai service 2/.env"      # add your GROQ/MISTRAL/CEREBRAS/GEMINI keys
cp shopify-pp/.env.example shopify-pp/.env               # add your Shopify app client id/secret
```

`analytic service` needs no env vars to run locally (defaults to port 4000).

Then start each service in its own terminal, in this order:

```bash
# 1. Analytics Service — http://localhost:4000
cd "analytic service" && npm start

# 2. AI Service — http://localhost:5001
cd "ai service 2" && npm start

# 3. Shopify extractor — http://localhost:3000
cd shopify-pp && npm start

# 4. Review Hub — http://localhost:5173
cd review-hub && npm run dev
```

Open `review-hub` at http://localhost:5173 and log in (see
[`review-hub/README.md`](review-hub/README.md) for demo credentials and the
full PO/developer flow). Service endpoints the Review Hub talks to are
configured in [`review-hub/src/lib/config.js`](review-hub/src/lib/config.js).

## Per-service docs

- [`review-hub/README.md`](review-hub/README.md) — dashboard flow, personas, endpoint config
