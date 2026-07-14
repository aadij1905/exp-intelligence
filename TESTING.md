# API testing reference (Postman)

Endpoints across all three backend services, for manual testing. No auth
headers needed on any of these — each service looks up what it needs
(tokens, cached data) server-side by the `shop`/`storeId` param.

**Test store:** `my-store-mkct5tzv.myshopify.com`

**Production base URLs:**
| Service | Base URL |
|---|---|
| Shopify extractor (`shopify-pp`) | `https://shopify-extractor-production.up.railway.app` |
| Analytics Service (`analytic service`) | `https://analytics-service-production-6d80.up.railway.app` |
| AI Service (`ai service 2`) | `https://ai-service-production-b7c5.up.railway.app` |

**Local dev base URLs** (when running each with `npm start`/`npm run dev`):
| Service | Local URL |
|---|---|
| Shopify extractor | `http://localhost:3000` |
| Analytics Service | `http://localhost:4000` |
| AI Service | `http://localhost:5001` |
| Review Hub | `http://localhost:5173` |

---

## Shopify extractor (`shopify-pp`)

**Install / reinstall a store**
```
GET /auth?shop=my-store-mkct5tzv.myshopify.com
```
Not a JSON API — open in a browser, approve the install screen.

**Shop status**
```
GET /api/shop?shop=my-store-mkct5tzv.myshopify.com
```
Returns `shopDomain`, `lastSyncedAt`.

**Theme code extraction** — the real theme files that render one page
```
GET /api/debug/theme-code?shop=my-store-mkct5tzv.myshopify.com&page=/
```
Other `page` values to try: `/products/<handle>`, `/collections/<handle>`,
`/cart`, `/pages/<handle>`, `/blogs/<blog>/<article-handle>`.
Response: `themeName`, `template`, `templateSuffix` (non-null only if that
resource has a Shopify "alternate template" assigned), `truncated`,
`files[]` (`key`, `bytes`, `content`).

**Raw analytics query output** — runs the 10 ShopifyQL queries, returns rows
directly (does NOT forward to the Analytics Service, unlike `/api/sync`)
```
GET /api/debug/queries?shop=my-store-mkct5tzv.myshopify.com
```
Response: `queriesRun`, `metrics`, `errors` (per-query failure messages, if
any), `raw` (rows per query name). Rows will be empty arrays if the store has
no real traffic/orders in the query's lookback window (7d for most, 30d for
Core Web Vitals) — that's expected on a low-traffic dev store, not a bug.

**Sync** — production path: runs the queries AND forwards results to the
Analytics Service (triggers ingest + crawl there)
```
POST /api/sync?shop=my-store-mkct5tzv.myshopify.com
```
Optional param: `websiteUrl=<url>` to override the auto-detected storefront
domain. Returns only `metrics`/`errors` — use `/api/debug/queries` instead if
you want to see the actual row data.

---

## Analytics Service (`analytic service`)

**Health check**
```
GET /health
```
Response includes `storesWithRealData` (list of `storeId`s with ingested
data) and `crawlerAvailable`.

**Ingest** — normally called by `shopify-pp`'s `/api/sync`, but callable
directly if you want to POST a custom payload
```
POST /api/analytics/ingest
Body (JSON): { "storeId": "...", "raw": { ... }, "websiteUrl": "..." }
```
`raw` must match the shape `extractAll()` produces in `shopify-pp`
(`overview`, `sales`, `pages`, `trafficSources`, `devices`,
`funnelBreakdown`, `checkoutConversion`, `performanceLCP`, `performanceCLS`,
`performanceINP` — each an array of row objects). Responds immediately;
if `websiteUrl` is non-empty, the crawler runs asynchronously afterward.

**Status** — ingestedAt, query metrics, crawler status
```
GET /api/analytics/status?storeId=my-store-mkct5tzv.myshopify.com
```
`crawlerStatus` will be `not_triggered` / `running` / `complete` / `failed`.

**Full report** — normalized data + detected flags
```
GET /api/analytics/report?storeId=my-store-mkct5tzv.myshopify.com
```

**Section-specific views** (each includes a plain-English `summary` plus the
relevant slice of `flags`)
```
GET /api/analytics/overview?storeId=my-store-mkct5tzv.myshopify.com
GET /api/analytics/traffic?storeId=my-store-mkct5tzv.myshopify.com
GET /api/analytics/devices?storeId=my-store-mkct5tzv.myshopify.com
GET /api/analytics/pages?storeId=my-store-mkct5tzv.myshopify.com
GET /api/analytics/funnel?storeId=my-store-mkct5tzv.myshopify.com
```
All of these fall back to generated mock data if no real ingest has happened
yet for that `storeId` (`dataSource: "mock"` in the response vs `"shopify"`).

---

## AI Service (`ai service 2`)

**Health check**
```
GET /health
```

**Generate a full report + suggestions** — pulls normalized data from the
Analytics Service internally, then prompts an LLM
```
POST /report/generate?storeId=my-store-mkct5tzv.myshopify.com
Body (JSON, optional): { "mode": "comprehensive" }
```
`mode` is `"comprehensive"` (20-25 suggestions) or `"quick"` (top 3). Add
`?mock=true` to skip the LLM and return canned mock suggestions instead
(useful for testing without burning API credits).

**Fetch a previously generated report** (cached in-memory by `storeId`)
```
GET /report?storeId=my-store-mkct5tzv.myshopify.com
GET /report/pm?storeId=my-store-mkct5tzv.myshopify.com
```
`/report/pm` strips the `analysis` array (which carries `codePatch`) — the
PO-facing view.

**Analyze pre-fetched data directly** — bypasses the Analytics Service, you
supply the normalized data yourself
```
POST /analyze
Body (JSON): {
  "normalized": { ... },
  "flags": [ ... ],
  "crawlerRan": false,
  "mode": "comprehensive",
  "storeId": "..."
}
```

**Generate a full code patch for one accepted suggestion**
```
POST /code/generate
Body (JSON): {
  "item": {
    "title": "...",
    "category": "...",
    "affectedPage": "...",
    "issue": "...",
    "recommendation": "..."
  }
}
```
As of today this does **not** pull real theme code from `shopify-pp` — the
prompt assumes the Dawn theme. See `THEME_CODE_INTEGRATION_PLAN.md` for the
planned fix.

**Suggestion accept/reject tracking** (in-memory, per `storeId`)
```
POST /suggestions/respond
Body: { "storeId": "...", "suggestionId": "...", "action": "accept", "title": "..." }

GET /suggestions/status?storeId=my-store-mkct5tzv.myshopify.com

DELETE /suggestions/status?storeId=my-store-mkct5tzv.myshopify.com
```

---

## Suggested test order (full pipeline, end to end)

1. `shopify-pp` → `GET /api/shop` — confirm the store is installed.
2. `shopify-pp` → `POST /api/sync` — pulls fresh ShopifyQL data, forwards to Analytics Service.
3. `analytic service` → `GET /api/analytics/status` — confirm `ingestedAt` is recent and check `crawlerStatus`.
4. `analytic service` → `GET /api/analytics/report` — inspect normalized data + flags.
5. `ai service` → `POST /report/generate` — generate suggestions from that data.
6. `ai service` → `GET /report` — fetch the cached result.
7. `ai service` → `POST /code/generate` — generate a full patch for one suggestion (currently Dawn-generic, not store-specific — see integration plan).
