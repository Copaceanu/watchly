# Watchly — Handoff (status complet pentru continuare/Codex)

**Ce este:** asistent WhatsApp de monitorizare prețuri pentru România, construit ca workflow-uri n8n.
**n8n cloud:** https://copaceanu.app.n8n.cloud — proiect personal `S2sl2PDmeaRKDXRp`.
**Data:** 2026-06-06.

## Fluxul
WhatsApp (WasenderAPI) → webhook `watchly-main` → extrage text/poză → LLMok (gpt-5.4-mini) clasifică în `scraping | search | price_alert | realism_check | chat` (scope strict: doar produse/prețuri/reduceri) → cheamă sub-workflow-ul potrivit → răspunde pe WhatsApp.

## Workflow-uri (toate ACTIVE)
| Workflow | ID | Webhook | Rol |
|---|---|---|---|
| watchly-main | `jS3QWdKobxvPa3Al` | `/webhook/watchly-inbound` | router principal (20 noduri) |
| watchly-scraping | `VcXok7kpQrttnmyC` | `/webhook/watchly-scraping` | link → nume+preț (JSON-LD) |
| watchly-search | `vi5bTKNbzGDvhsiY` | `/webhook/watchly-search` | descriere → produs+preț (eMAG search, top rezultat) |
| watchly-image-recognize | `gv3jV7JLnAzjpE2a` | `/webhook/watchly-image-recognize` | poză → produs + variante (gpt-5.4-mini vision) |
| watchly-history | `eTi9WtXjayG9Q9K4` | `/webhook/watchly-history` | istoric 24 luni de pe istoric-preturi.info → tab price_history |
| watchly-price-alert | `vV4yBEFyJkOTeAe6` | `/webhook/watchly-price-alert` | salvează alertă în Google Sheets + seed istoric |
| watchly-realism-check | `zLDlkMWg80mwtRup` | `/webhook/watchly-realism-check` | „e realist prețul?" (folosește istoricul real) |
| watchly-notify | `tRuWllN9jfbmZtOF` | cron 6h | reverifică prețuri + alertă + marchează triggered |

URL intern de bază: `https://copaceanu.app.n8n.cloud/webhook/<path>`.

## Credențiale (în n8n, tip + id)
- **LLMok** (httpBearerAuth) id `zB1qUzfyhkmlfbJ2` — endpoint `https://burn.llmok.app/v1/chat/completions`, model **`gpt-5.4-mini`** (NU `gpt-4o` — nu există pe cheie). Cheie: `<LLMOK_API_KEY>`.
- **WasenderAPI** (httpBearerAuth) id `YBysESALRaJUegYU` — send `https://www.wasenderapi.com/api/send-message`, body `{to, text}`. Cheie funcțională: `<WASENDER_API_KEY>` (e și sessionId; valabilă DOAR cât sesiunea WhatsApp e Connected). Plan plătit €6.
- **Google Sheets** (googleSheetsOAuth2Api) id `iqZLGUcix3E0VxAJ`. Spreadsheet `Watchly` id `<GOOGLE_SHEET_ID>`. Tab-uri: `alerts` (phone|product_url|product_name|current_price|target_price|created_at|status) și `price_history` (product_name|source|date|price|fetched_at).
- **ScrapingBee** (httpQueryAuth, Name=`api_key`) — RECREAT de user (id nou). Cheie: `<SCRAPINGBEE_API_KEY>`. Mod folosit: `render_js=false`, `premium_proxy=true`, `country_code=ro` (eMAG blochează IP-uri datacenter cu AWS WAF 511; premium proxy trece).

> ⚠️ Toate cheile de mai sus au fost expuse în chat → de rotit după hackathon.

## Limitări / capcane importante (LECȚII)
1. **MCP NU poate atașa credențiale HTTP** (httpBearerAuth/httpQueryAuth) pe noduri httpRequest — nici setNodeCredential, nici addNode. Trebuie făcut din UI de user (Credential dropdown + Save workflow). Funcționează și persistă.
2. **MCP NU poate scrie valori secrete** de credențial. User le introduce în UI. La câmpuri secrete goale n8n trimite `__n8n_BLANK_VALUE__`.
3. **WhatsApp 429 la rafală**: WasenderAPI/WhatsApp limitează trimiterile rapide de pe un număr (anti-spam) — chiar pe plan plătit. La uz uman normal nu se simte. Mitigare: pe nodurile Send → Settings → **Retry On Fail** ON (wait ~5000ms).
4. eMAG search NU are JSON-LD; prețuri în `product-new-price">` (`&#46;`=mii, `<sup>`=bani). Pagina de produs eMAG ARE JSON-LD `offers.price`.
5. model gpt-5.4-mini: fără `temperature` custom (poate refuza). Toate apelurile LLMok forțează model via override `{{ JSON.stringify(Object.assign({}, $json.payload, { model:'gpt-5.4-mini', temperature: undefined })) }}` sau direct în payload.

## CONFIRMAT că merge (stress test 36 mesaje + teste directe)
- Clasificare + rutare: 36/36 corecte (scraping/search/realism/chat/refuz off-topic).
- Scraping link→preț: 10/10 (iPhone 17 Pro 6599.99).
- Search descriere→produs+preț: ex. „macbook m4 8 core cpu 8 core gpu" → MacBook Air M4 4799 lei; TV/boxe/căști/Samsung OK.
- Poză→produs Apple: recunoscut (MacBook Pro/Air, iPad).
- Salvare alertă Google Sheets: OK. Trimitere WhatsApp: OK (la pas uman).

## BUG HUNT (2026-06-06) — TOATE REZOLVATE
- ✅ **FIXAT — search relevanță** (watchly-search/Extract Candidates): scorează candidații după potrivirea cuvintelor din descriere; scor 0 → `found:false`. (nonsens → "niciun rezultat relevant"; macbook m4 → MacBook Air M4 4799 ✓)
- ✅ **FIXAT — istoric (Bug #2)**: istoric-preturi.info era nefiabil (POST /search, pagină JS). REPROIECTAT: `watchly-history` acum ia prețul curent prin `watchly-scraping` (eMAG, merge) → scrie un punct {date,price} în tab-ul `price_history` și întoarce `{points:[{date,price}], lowest_price, count}`. Sheet-ul acumulează istoric REAL în timp (cronul scrie la 6h). Verificat: întoarce 6599.99 + scrie în sheet.
- ✅ **FIXAT — parser universal (Bug #3)**: `watchly-scraping/Extract Price` rescris să prindă JSON-LD offers.price + offers.lowPrice + AggregateOffer + meta og:price:amount/product:price:amount + itemprop=price + tipar eMAG `"price":{"current":N}` + generic `"price":N`. Acoperă mult mai multe magazine (eMAG confirmat 6599.99). [Altex/Flanco netestate cu URL real — URL-ul de test era 404.]
- ✅ **realism** acum pe date reale (folosește punctul din history): „ținta 5000 e sub minimul 6599.99, sugerez 6299, ~24 săpt".
- ✅ **cron watchly-notify** rulat manual (exec 156): citește alerte → refresh istoric (scrie în sheet) → verifică preț → IF corect (nu trimite când preț>țintă). Funcțional.
- ✅ poză non-produs → `recognized:false`; ✅ search normal OK.
- Cosmetic rămas: nume produs la search = slug cu liniuțe (se poate înfrumuseța cu un pas LLM/Code, opțional).

## Memorie conversatie (adaugat 2026-06-07)
watchly-main (acum 22 noduri) are memorie per numar via `$getWorkflowStaticData('global').mem[phone]` (fara DB/credentiale): nod **Load Memory** (dupa Has Image? false) injecteaza `prior_context` in Build Decision Request; LLMok combina contextul cu mesajul nou (ex. context "macbook m4" + "cel cu 8 nuclee gpu" => description "macbook m4 8 nuclee gpu"); nod **Save Memory** (intre Parse Decision si Route by Type) salveaza summary-ul (produsul/descrierea) per numar, pass-through la $json ca Switch sa mearga. La workflow_type chat pastreaza contextul anterior. VERIFICAT live (exec 196). watchly-search are si Build Query (curata cuvinte de umplutura inainte de cautare eMAG) + filtrare pe cip Apple (m1-m5, exclude alte cipuri, alege cea mai ieftina varianta potrivita).

## Fix model-disambiguation (2026-06-07)
watchly-search/Extract Candidates: era potrivire substring (bug: "16" se potrivea in "16e"). Acum potrivire pe CUVINTE INTREGI (word-set per nume), + negatii (regex `(nu|fara|not|without) <cuvant>` => exclude candidatii cu acel cuvant), + filtru cip Apple m1-m5, tie-break cea mai ieftina. PUR COD, fara nod AI/credential. Verificat: "iphone 16" => iPhone 16 (nu 16e), "iphone 16 nu 16e" => iPhone 16, "iphone 16e" => iPhone 16e, "macbook m4" => M4 Air. watchly-search are din nou DOAR 4 noduri (Webhook, Build Query, Fetch eMAG Search, Extract Candidates) — nodurile LLM-pick au fost scoase. Rafinari (2026-06-07): potrivire hibrida (exact pt tokenuri scurte ca "16"; prefix pt coduri >=5 chr ca "1000xm5"=>"1000xm5b"); necesita >=1 token SPECIFIC (non-generic) potrivit, altfel found:false (ex. "iphone 26" inexistent => negasit, nu mai da iPhone XR). Verificat 7/7 cazuri: iphone 26=negasit, iphone16!=16e, 16e exact, sony wh-1000xm5 corect, macbook m4, iphone15.

## Cum continui
- Test direct fără WhatsApp: `POST /webhook/watchly-scraping {product_url, phone_number}`, `/watchly-search {description}`, `/watchly-image-recognize {image_data}`, `/watchly-history {product_name,product_url}`, `/watchly-realism-check {product_name,product_url,target_price}`. responseMode lastNode → răspunsul vine direct.
- Debug: n8n Executions per workflow (get_execution includeData).
- Pentru a folosi: oricine trimite mesaj pe numărul WhatsApp Watchly primește răspuns automat.
