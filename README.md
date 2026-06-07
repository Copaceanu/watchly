# Watchly 🛒📉

**A WhatsApp price-tracking assistant for Romania, built entirely on n8n.**

Talk to it on WhatsApp like a human: send a product link, *describe* a product in words, or send a *photo* — Watchly finds it on Romanian retailers (eMAG & co.), tells you the price, judges whether your target price is realistic, and pings you when it drops.

> Built for a hackathon. Conversational, multi-turn, with memory.

---

## ✨ Features

| You send | Watchly does |
|---|---|
| a **product link** (eMAG/…) | scrapes name + current price (via a residential-proxy fetch to beat anti-bot) |
| a **description** — *"macbook m4 8 core gpu"*, *"iphone 16 nu 16e"* | searches eMAG, picks the **right model** (knows iPhone 16 ≠ 16e ≠ 16 Pro), returns price + link |
| a **photo** of a product | recognizes it (GPT-4-class vision) + lists likely variants |
| *"e realist să-l iau la 5000 lei?"* | judges if the target price is realistic vs. observed history |
| *"anunță-mă când ajunge la X"* | saves a price alert; a cron re-checks every 6h and notifies you |
| *"cel cu 256gb"* (follow-up) | **remembers the conversation** and resolves the reference |
| *"cât e 2+2?"* | politely refuses — **strict scope**: products, prices & discounts only |

---

## 🧠 How it works

```
WhatsApp ⇄ WasenderAPI ──► watchly-main (router)
                              │  extract text/image  ·  conversation memory (per phone)
                              │  LLM classify: scraping | search | price_alert | realism_check | chat
                              ├─► watchly-scraping      (link → price, universal JSON-LD/meta parser)
                              ├─► watchly-search        (description → best-matching product on eMAG)
                              ├─► watchly-image-recognize(photo → product + variants, vision)
                              ├─► watchly-price-alert   (save alert to Google Sheets + seed history)
                              ├─► watchly-realism-check (target price realistic? uses real history)
                              └─► watchly-history       (records price points → builds 2-yr history)
                          watchly-notify (cron 6h): re-check prices → alert + mark triggered
```

**8 n8n workflows.** See [`workflows/`](workflows/) for importable JSON and [`docs/HANDOFF.md`](docs/HANDOFF.md) for the full technical writeup.

### Smart product matching (no extra AI cost)
The search picker is pure code and handles the cases that trip up naive matching:
- **whole-word** matching → `16` ≠ `16e`
- **prefix** matching for long model codes → `1000xm5` ≈ `1000xm5b`
- **model number weighted** heavily → right model beats right capacity
- **qualifier exclusion** → `iphone 16` won't return `16 Pro Max`
- **negations** → `nu 16e`, `fără pro`
- **nonexistent → "not found"** → `iphone 26` doesn't invent a product

---

## 🧩 Tech stack
- **n8n** (cloud) — orchestration, all logic in workflows
- **WasenderAPI** — WhatsApp send/receive
- **LLMok** (`gpt-5.4-mini`) — classification, vision, realism (OpenAI-compatible gateway)
- **ScrapingBee** — residential proxy + JS to beat eMAG's anti-bot (AWS WAF)
- **Google Sheets** — alert + price-history storage

---

## 🚀 Setup (high level)
1. Import the JSON files from [`workflows/`](workflows/) into your n8n.
2. Create credentials (see [`.env.example`](.env.example) for what you need):
   - **LLMok** (Bearer), **WasenderAPI** (Bearer), **Google Sheets** (OAuth2), **ScrapingBee** (Query Auth, `api_key`).
   - Attach them on the HTTP/Sheets nodes (n8n UI).
3. Create a Google Sheet `Watchly` with two tabs:
   - `alerts`: `phone | product_url | product_name | current_price | target_price | created_at | status`
   - `price_history`: `product_name | source | date | price | fetched_at`
4. Activate all workflows.
5. Point your WasenderAPI inbound webhook to `…/webhook/watchly-inbound`.
6. Text the bot. 🎉

Full step-by-step + gotchas: [`docs/HANDOFF.md`](docs/HANDOFF.md).

---

## ⚠️ Notes & limits
- WhatsApp throttles bursts (anti-spam) even on paid plans — fine for real human-paced use.
- Price *history* is built forward from periodic checks (the past isn't reliably scrapeable).
- No secrets are committed; provide your own keys via credentials.

---

*Made with n8n + a lot of stress-testing.* 🤖
