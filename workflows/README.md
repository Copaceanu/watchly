# Workflows

The 8 n8n workflows that make up Watchly. Import each JSON into n8n via **Import from File** (or paste).

| File | Webhook / Trigger | Role |
|---|---|---|
| `watchly-main.json` | `POST /webhook/watchly-inbound` | router: extract text/image, conversation memory, classify, dispatch, reply |
| `watchly-scraping.json` | `POST /webhook/watchly-scraping` | product link → name + price (universal JSON-LD/meta parser, via ScrapingBee) |
| `watchly-search.json` | `POST /webhook/watchly-search` | description → best-matching eMAG product (model-aware picker) |
| `watchly-image-recognize.json` | `POST /webhook/watchly-image-recognize` | photo → product + variants (vision) |
| `watchly-price-alert.json` | `POST /webhook/watchly-price-alert` | save alert to Google Sheets + seed history |
| `watchly-realism-check.json` | `POST /webhook/watchly-realism-check` | is the target price realistic? |
| `watchly-history.json` | `POST /webhook/watchly-history` | record a price point → builds price history |
| `watchly-notify.json` | Schedule (every 6h) | re-check active alerts → notify + mark triggered |

## How to export from n8n (to (re)generate these files)
- **All at once:** Workflows page → select the `watchly-*` workflows → **Download**.
- **One by one:** open a workflow → **⋮ menu → Download** → save as `<name>.json` here.

## How to import
- n8n → **Import from File** → pick the JSON. Then attach credentials on the HTTP/Sheets nodes and activate.

> The exported JSON contains **no secrets** — credentials are referenced by name/ID and must be configured in your own n8n. See [`../.env.example`](../.env.example).
