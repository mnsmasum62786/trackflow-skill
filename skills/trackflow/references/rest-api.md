# TrackFlow HTTP API reference

The HTTP endpoints an agent or developer can call directly. The MCP tools (see `mcp-tools.md`) are thin wrappers around the dashboard API documented here, so validation and error texts are identical. Prefer MCP when it is connected; use this API for server-to-server ingest, scripts, or clients without MCP.

Base URL: `https://trackflow.webanalyticssolution.com`

---

## 1. Access classes

| Class | How it authenticates | Endpoints |
|---|---|---|
| **Public** (no auth) | Site key in the body/query, or a secret token in the URL | `GET /t.js`, `POST /collect`, `GET /api/public/config`, `POST /hooks/:token`, `POST /hooks/checkout/:token`, `POST /hooks/crm/:token`, `GET /healthz`, OAuth + `/.well-known/*` endpoints |
| **API key** | `Authorization: Bearer YOUR_API_KEY` or `X-API-Key: YOUR_API_KEY` | `POST /api/v1/events`, `POST /api/v1/offline-events`, and **every dashboard route under `/api/…` listed in section 4** (they accept a dashboard session cookie *or* an API key) |
| **Dashboard session only** — do not call | Browser cookie from signing in | `/api/auth/*` (signup, login, logout, me, demo), `/api/admin/*` (TrackFlow staff only), `/oauth/consent` (browser approval page), `/api/google/oauth/callback` (Google redirect target) |
| **Internal** — do not call | Infrastructure | `/_edge/*`, the random stealth paths `/<slug>/…` (only reached through the install snippet / proxy) |

API keys (`tf_live_…`) are created in the dashboard (Connect AI → API keys) or with MCP `account → create_api_key`. A key has full access to its account, has no scopes and does not expire. OAuth access tokens issued to MCP clients are API keys too and work here.

### Responses and errors

- JSON everywhere. Errors: `{"error": "<message>"}` with an HTTP status (some add fields such as `missing`, `mode`, `used_by`, `field`).
- `401 {"error":"Not signed in"}` — dashboard route without a valid session/key. `/api/v1/*` say `Invalid or missing API key`.
- `403 {"error":"The demo workspace is read-only. Create a free account to make changes."}` — any non-GET on the public demo account. `403 This account is suspended. Contact TrackFlow support.`
- `404 {"error":"Site not found"}` — site id not in this account. Unknown `/api` path → `404 {"error":"Not found"}`.
- Request body limits: dashboard JSON 128 KB; `/api/v1/events` 64 KB; `/collect` 32 KB; checkout/CRM routes 1 MB; offline JSON 2 MB, CSV upload 10 MB; simple webhooks 512 KB.
- Times are Unix **milliseconds** unless stated.

---

## 2. Server-side ingest (API key)

### POST /api/v1/events — one server event

`Content-Type: application/json`. Creates an event with origin `server` through the full pipeline (blocking rules, visitor matching, dedup, hashing, delivery to every enabled server destination).

| Field | Type | Required | Notes |
|---|---|---|---|
| `site_key` | string | yes | The site's `site_key` (`tf_…`, from `list_sites` / `GET /api/sites`), must belong to the key's account. Not the numeric site id |
| `event_name` | string | yes | Cut to 64 chars. Use standard names (`Purchase`, `Lead`, `CompleteRegistration`, `Schedule`, `AddToCart`…) so platforms map them |
| `event_id` | string | recommended | ≤ 64 chars; random if absent. **Use the same id the browser sent** for the same action so browser + server deduplicate (in TrackFlow and on the platforms) |
| `visitor_id` | string | recommended | TrackFlow visitor id (cookie `_tf_vid`, or `trackflow('vid')` in the browser). Links the event to the visitor's journey and inherits their stored click ids, IP and user agent |
| `user_data` | object | recommended | Raw or already SHA-256-hashed (64 hex) values; normalized + hashed server-side. Keys: `em, ph, fn, ln, ct, st, zp, country, db, ge, external_id`; aliases accepted: `email, phone, mobile, first_name, last_name, city, state, zip, postal_code, country_code, dob, gender, user_id`; `name` is split into fn/ln. Phone: include the country code |
| `custom_data` | object | no | `value`, `currency` (ISO 4217, validated; missing/invalid → the site's `default_currency` (else USD) when value is set), `order_id`, `content_ids`, `contents[{id, quantity, item_price}]`, `content_type`, `content_name`, `num_items`… |
| `click_ids` | object | no | `fbc, fbp, gclid, gbraid, wbraid, msclkid, ttclid, ttp, …` |
| `fbc`, `fbp` | string | no | Top-level shortcut for Meta cookies |
| `url`, `referrer` | string | no | Page URL drives source/campaign attribution (UTMs, click ids) |
| `ip`, `user_agent` | string | no | The end user's, not your server's. Inherited from the visitor when `visitor_id` is known and these are blank |
| `is_conversion` | bool | no | Accepted but **ignored**: the site's conversion list decides |

Not supported on this endpoint: backdating (`event_time` is always "now"), `action_source` (always `website`), and a test flag. For backdated or offline conversions use `/api/v1/offline-events`; to test, use MCP `send_test_event` / `test_destination`.

Responses:
```json
{"ok": true, "blocked": false, "event_id": "order_10045", "match_score": 6.4}
{"ok": true, "duplicate": true, "event_id": "order_10045"}
{"ok": true, "blocked": true, "reason": "Rule: ip 203.0.113.*", "event_id": "…", "match_score": 3}
```
Errors: `401 Invalid or missing API key`, `404 Unknown site_key`, `400 event_name is required`. Delivery to platforms runs **after** the response; check results with `GET /api/sites/:siteId/events?q=<event_id>` or MCP `list_events` / `get_event`.

Dedup: with the site setting `dedup` on (default), a second event with the same `event_id` + `event_name` returns `duplicate: true` and is not stored or sent again.

### Recipe: send a server-side Purchase with curl

```bash
curl -X POST https://trackflow.webanalyticssolution.com/api/v1/events \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "site_key": "YOUR_SITE_KEY",
    "event_name": "Purchase",
    "event_id": "purchase_10045",
    "visitor_id": "VISITOR_ID_FROM_TF_VID_COOKIE",
    "url": "https://example.com/checkout/thank-you",
    "ip": "203.0.113.10",
    "user_agent": "Mozilla/5.0 (…)",
    "user_data": {"email": "customer@example.com", "phone": "+15551234567", "first_name": "Ana", "last_name": "Diaz", "country": "US"},
    "custom_data": {"value": 129.90, "currency": "USD", "order_id": "10045", "content_ids": ["SKU-1"], "content_type": "product"},
    "click_ids": {"fbc": "fb.1.1730000000000.AbCdEf", "fbp": "fb.1.1730000000000.123456789"}
  }'
```

Checklist: (1) `site_key`, not site id; (2) reuse the browser's event id (`purchase_<order id>` is what the dataLayer bridge and the ecommerce recipe use for Purchase); (3) pass `visitor_id` whenever you have it — it is the biggest match-quality win; (4) send the shopper's IP/UA, never your server's; (5) verify in the event log.

### POST /api/v1/offline-events — batch of offline / backdated conversions

`Content-Type: application/json`. Up to **1000 events per request**. Demo account → 403.

| Body field | Notes |
|---|---|
| `site_key` | required |
| `events` | array of records (also accepted: `data`, or a single record at top level when it has `event_name`) |
| `defaults` | `{event_name (default Purchase), currency (default USD), action_source (default physical_store), country, date_format: "mdy" (default) or "dmy"}` |
| `validate_only` | `true` = validate and match without creating events |
| `name` | label shown in the import history (default "API batch") |

Record fields: `event_name`, `order_id`, `event_time` (ISO 8601, `YYYY-MM-DD[ HH:mm[:ss]]`, Unix s or ms, `MM/DD/YYYY` or `DD/MM/YYYY` with `date_format: "dmy"`; no zone = UTC), `value`, `currency`, `action_source` (`physical_store, phone_call, system_generated, email, chat, website, app, other`), identifiers either flat (`email, phone, first_name, last_name, name, city, state, zip, country, external_id`) or in `user_data` (`em, ph, fn, ln, ct, st, zp, country, external_id`), `vid` or `visitor_id`, `lead_id` (numeric Meta lead id), click ids flat or in `click_ids` (`gclid, gbraid, wbraid, fbc, fbp, fbclid, ttclid, msclkid, …`), `custom_data`.

Validation per record: at least one identifier (email, phone, external id, lead id, visitor id or click id); `Purchase needs a value`; value numeric and ≥ 0; 3-letter currency; time not in the future and ≤ 90 days old (warnings when older than Meta accepts: 7 days, 62 for `physical_store`). Event id = `offline_<order_id>` (re-sending the same order is a `duplicate`), or a content hash without order id.

```bash
curl -X POST https://trackflow.webanalyticssolution.com/api/v1/offline-events \
  -H "Authorization: Bearer YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"site_key":"YOUR_SITE_KEY","defaults":{"currency":"USD","action_source":"phone_call"},
       "events":[{"event_name":"Purchase","order_id":"INV-2207","event_time":"2026-10-28 14:30","value":450,"email":"customer@example.com","gclid":"EXAMPLE_GCLID"}]}'
```
Response:
```json
{"import_id": 31, "summary": {"rows_total": 1, "accepted": 1, "rejected": 0, "duplicates": 0, "flagged": 0, "matched": 1},
 "results": [{"index": 0, "status": "imported", "event_id": "offline_INV-2207", "vid": "…", "matched_by": "email", "errors": [], "warnings": []}]}
```
`status`: `imported` (or `valid` with validate_only), `rejected`, `duplicate`. Errors: `401`, `404 Unknown site_key`, `400 events must be a non-empty array`, `400 Send at most 1000 events per request`.

---

## 3. Public endpoints (no auth)

| Method + path | Purpose | Notes |
|---|---|---|
| `GET /t.js` | Classic tracker script | Install via the snippet from `get_install_snippet`, not by hand |
| `GET /api/public/config?k=SITE_KEY[&preview=TOKEN]` | Config the tracker loads (browser tags without secrets, published container, settings) | Loading it marks the site as seen. Unknown key → 404 `unknown site` |
| `POST /collect` | Browser event collector used by the tracker | Body is JSON sent as `text/plain`: `{k: site_key, e: event_name, id: event_id, vid, sid, u: url, r: referrer, ud: user_data, cd: custom_data, ids: click_ids, rl: is_reload, c: consent, test}`. Rate limit 120/min per IP+site (`429 rate limited`). Bot filtering, reload blocking and consent apply only here. Do not use it for server-side events — use `/api/v1/events` |
| `POST /hooks/:token` | Simple mapped webhook (`create_webhook_integration`) | Any JSON. **No signature check**. Fields extracted with the integration's `mapping`. `filter_path`/`filter_value` mismatch → `{ok:true, ignored:true, reason:"filtered"}`. Disabled/unknown token → 404 `unknown webhook`. Response = ingest result |
| `POST /hooks/checkout/:token` | Checkout provider webhook (`create_checkout_integration`) | Signature verified when a secret is saved (bad signature → 400 with reason, logged as `rejected`). Response `{ok:true, results:[{status: created\|duplicate\|ignored, order_id, event_id, value, vid, matched_by, …}]}` |
| `POST /hooks/crm/:token` | CRM webhook (`create_crm_integration`) | JSON, form-encoded or Salesforce SOAP. Wrong secret → 401. Answers within 4 s (`{ok:true, accepted:true, processing:true}` if lookups are still running); Salesforce gets a SOAP Ack |
| `GET /healthz` | `{ok:true}` | |
| OAuth: `/.well-known/oauth-authorization-server`, `/.well-known/oauth-protected-resource`, `POST /oauth/register`, `GET /oauth/authorize`, `POST /oauth/token`, `POST /oauth/revoke` | MCP sign-in | See `mcp-tools.md` §1 |

Browser JavaScript API (after the snippet is installed; with stealth on, replace `trackflow` by the global from `get_install_snippet`):
`trackflow('track', 'Lead', {value: 49, currency: 'USD'}, {em: 'a@example.com'}, {id: 'lead_123'})` (name, custom_data, user_data, options.id = event id) · `trackflow('identify', {em, ph, fn, ln})` · `trackflow('page')` (manual SPA page view) · `trackflow('consent', true|false)` · `trackflow('vid')` (visitor id for checkout passthrough) · `trackflow('decorate', url)`. URL flags: `?tf_preview=TOKEN` / `?tf_preview=off`, `?tf_debug=1`.

---

## 4. Dashboard API (session cookie or API key)

All paths below are under `/api`. `:siteId` is the numeric site id. Every write is refused on the demo account.

### Account and keys
| Method + path | Body / query | Response |
|---|---|---|
| `PATCH /account` | `{name}` (≤80). Password change needs `{current_password, new_password (≥8)}` — leave to the dashboard | `{ok}` |
| `GET /keys` | — | `{keys:[{id, name, prefix, last_used_at, created_at}]}` |
| `POST /keys` | `{name}` (≤60) | `{key}` — full key, shown once |
| `DELETE /keys/:id` | — | `{ok}` (immediate revoke) |

### Sites
| Method + path | Body / query | Response / notes |
|---|---|---|
| `GET /sites` | — | `{sites:[{id, name, domain, site_key, platform, settings, last_seen_at, created_at}]}` |
| `POST /sites` | `{domain*, name, platform}` | `{site}`; 400 invalid domain, 409 duplicate |
| `PATCH /sites/:siteId` | `{name, domain, platform, settings{…}}` | `{site}`. `settings` merges; `cross_domains` replaces the list (string or array); `settings.conversion_events` is accepted here and re-marks past events |
| `DELETE /sites/:siteId` | — | `{ok}` — deletes everything, irreversible, no confirmation step on this route |
| `POST /sites/:siteId/clear-data` | — | `{ok}` — deletes all events + journeys, irreversible |
| `POST /sites/:siteId/demo-data` | — | `{ok}` — adds 14 days of sample data |

### Conversion lists (reporting only)
| Method + path | Body | Notes |
|---|---|---|
| `GET /sites/:siteId/conversions` | — | `{selected, events, hidden, destinations}` |
| `PUT /sites/:siteId/conversions` | `{events*: [names], destination_id?}` | Site list (re-marks history) or one destination's list. Google Ads → 400 |
| `DELETE /sites/:siteId/conversions/:name` | — | Custom names only |
| `POST /sites/:siteId/conversions/:name/restore` | — | Un-hide |

### Reports and event log
Window query for reports: `?days=1–365` (default 7) or `?from=<ms>&to=<ms>` (to exclusive, ≤ 366 days). `?conv=site|<destination id>`. Header `X-TZ-Offset` (minutes, like `getTimezoneOffset()`) aligns the overview's daily buckets.

| Method + path | Query | Response |
|---|---|---|
| `GET /sites/:siteId/overview` | days/from/to, conv | totals, prev, series, top_conversions, top_sources, recent_events, recent_conversions, destinations |
| `GET /sites/:siteId/events` | `status, name, origin, q, conversions=1, test=1\|0, conv, limit (1–200, default 50), offset` | `{events[], total, names[], conv}` |
| `GET /sites/:siteId/events/:eventRowId` | — | `{event, deliveries[{id, type, status, http_status, payload, response, created_at}]}` |
| `GET /sites/:siteId/events/deliveries` | `ids=1,2,3` (≤200), `full=1` | Latest delivery per destination per event + `destinations`, `browser_tags` |
| `POST /sites/:siteId/events/:eventRowId/retry` | — | `{ok, status}` — re-sends live; blocked → 400 `Event cannot be resent` |
| `POST /sites/:siteId/test-event` | `{event_name (default Lead), value}` | Ingest result. Flagged test: only destinations with a test code/test mode receive it |
| `GET /sites/:siteId/match-quality` | days/from/to | `{events[{name, n, score, coverage{}}]}` |
| `GET /sites/:siteId/attribution` | days/from/to, `model=last\|first`, conv | sources, campaigns, by_event, devices, landing |
| `GET /sites/:siteId/journeys` | days/from/to, `all=1`, conv | ≤100 journeys |
| `GET /sites/:siteId/journeys/:vid` | conv | visitor (identifier names only) + events |
| `GET /sites/:siteId/flow` | days/from/to, `test=1\|0` | sources → events → destinations with counts and last errors |

### Destinations
| Method + path | Body | Notes |
|---|---|---|
| `GET /platforms` | — | Every platform with modes and fields |
| `GET /sites/:siteId/destinations` | — | `{items}` secrets masked (`first 6 ••• last 4`) |
| `POST /sites/:siteId/destinations` | `{type*, config{…}, mode (both\|browser\|server), enabled, load_browser_pixel}` | 400 `{error, missing[keys], mode}` when fields are missing |
| `PATCH /sites/:siteId/destinations/:id` | same (type fixed) | Omitted/`""`/masked secret keeps the saved value; `null` deletes it |
| `DELETE /sites/:siteId/destinations/:id` | — | Deletes credentials |
| `GET /google/oauth/config` | — | `{managed, redirect_uri}` |
| `POST /sites/:siteId/destinations/:id/google/auth-url` | — | `{url, redirect_uri, managed}` — single-use Google sign-in link (15 min) |
| `GET /sites/:siteId/destinations/:id/google/status` | — | `{connected, email, managed, connected_at, last_error, last_error_at}` |

### Blocking rules and simple webhooks
| Method + path | Body | Notes |
|---|---|---|
| `GET/POST /sites/:siteId/blocking-rules` | POST `{type* (ip\|url\|user_agent\|referrer\|event), pattern*, note, enabled}` | 400 `Pick a rule type` / `Pattern is required` / invalid IP pattern |
| `PATCH/DELETE /sites/:siteId/blocking-rules/:id` | PATCH partial | 404 `Not found` |
| `GET/POST /sites/:siteId/integrations` | POST `{event_name*, provider, name, mapping{…}, enabled}` | Items include `token`; webhook URL = `<base>/hooks/<token>` |
| `PATCH/DELETE /sites/:siteId/integrations/:id` | PATCH partial (`mapping` replaced) | |

### Tag manager (draft) and publishing
| Method + path | Body / query | Notes |
|---|---|---|
| `GET /tagging/reference` | — | Vocabulary + recipes |
| `POST /sites/:siteId/inspect` | `{url}` | Page analysis (site's own domains only) |
| `GET/POST /sites/:siteId/variables` | POST `{name*, type*, …fields or config{}, transform, default, fallback, notes}` | 409 duplicate name; response has `warnings` |
| `PATCH/DELETE /sites/:siteId/variables/:idOrName` | PATCH partial | |
| `GET/POST /sites/:siteId/tags` | POST `{name*, trigger*, actions*, conditions, once, enabled, priority, notes}` | |
| `PATCH/DELETE /sites/:siteId/tags/:id` | PATCH partial | |
| `GET/POST /sites/:siteId/tag-triggers` | POST `{name*, trigger*{type,…}, notes}` | |
| `PATCH/DELETE /sites/:siteId/tag-triggers/:idOrName` | | DELETE → 409 `{error, used_by}` while in use |
| `POST /sites/:siteId/tag-triggers/:idOrName/duplicate` | — | |
| `GET /sites/:siteId/recipes` · `POST /sites/:siteId/recipes/:recipe/apply` | `{options:{…}}` | recipes: `ecommerce, lead_form, booking, newsletter` |
| `GET /sites/:siteId/container` | `which=draft\|published` | `unpublished_changes` flag |
| `PATCH /sites/:siteId/container/settings` | `{decorate_checkouts}` | Draft setting |
| `GET /sites/:siteId/versions` | — | Version history |
| `POST /sites/:siteId/versions` | `{notes}` | **Publishes the draft live** |
| `POST /sites/:siteId/versions/:n/rollback` | `{by: "version", notes, restore_draft}` | **Live.** `:n` is the internal version **id** unless the body has `by: "version"` (then it is the version number) |
| `GET/POST /sites/:siteId/preview` | POST `{url, hours (0.25–24, default 2)}` | Preview links |
| `DELETE /sites/:siteId/preview/:token` | — | |

Field rules and enums: see `mcp-tools.md` §8–9.

### Checkout, CRM and offline integrations
| Method + path | Body / query | Notes |
|---|---|---|
| `GET /checkout/providers` | — | Providers, secret labels, passthrough fields |
| `GET/POST /sites/:siteId/checkout/integrations` | POST `{provider*, name, secret, api_key, config{}, enabled}` | `webhook_url` = `<base>/hooks/checkout/<token>` |
| `PATCH/DELETE /sites/:siteId/checkout/integrations/:id` | PATCH partial; omitted/masked secret keeps, `""` clears | |
| `GET /sites/:siteId/checkout/integrations/:id/log` | — | Last 100 |
| `POST /sites/:siteId/checkout/integrations/:id/test` | `{payload, headers}` | Dry run |
| `GET /crm/providers` | — | Providers, default stage maps, mapping keys |
| `GET/POST /sites/:siteId/crm/integrations` | POST `{provider*, name, secret, api_token, config{stage_map, mapping, …}, enabled}` | `webhook_url` = `<base>/hooks/crm/<token>` |
| `PATCH/DELETE /sites/:siteId/crm/integrations/:id` | | |
| `GET /sites/:siteId/crm/integrations/:id/log` | — | |
| `POST /sites/:siteId/crm/integrations/:id/test` | `{payload}` | Dry run |
| `POST /sites/:siteId/crm/integrations/:id/sync` | — | Poll now (creates live events) |
| `GET /sites/:siteId/offline/imports` | `limit` (≤200, default 50) | Import history |
| `POST /sites/:siteId/offline/preview` | JSON `{csv, mapping, defaults}`, multipart (`file`, `mapping`, `defaults`), or raw `text/csv` (mapping/defaults as JSON query params) | Validation only. ≤ 50,000 rows |
| `POST /sites/:siteId/offline/import` | same | **Creates events** |
| `GET/POST /sites/:siteId/offline/sources` | POST `{url*, name, mapping, defaults, interval_minutes (15–1440, default 60), enabled}` | URL must be public http(s); first pull within ~1 min |
| `PATCH/DELETE /sites/:siteId/offline/sources/:id` | PATCH partial, `reset_cursor` | |
| `POST /sites/:siteId/offline/sources/:id/sync` | — | Pull now; 502 with the fetch error on failure |

### First-party domains and install snippet
| Method + path | Body / query | Notes |
|---|---|---|
| `GET /sites/:siteId/edge` | — | Settings, DNS target, domains, health. **Returns the proxy secret unmasked** — treat as a secret |
| `PATCH /sites/:siteId/edge` | `{stealth, global, rotate_paths, rotate_secret}` | Live; rotating breaks installed snippets/proxies |
| `POST /sites/:siteId/edge/domains` | `{host*}` | Returns the CNAME record to create; max 5 per site |
| `POST /sites/:siteId/edge/domains/:id/verify` | — | `{domain, dns}` |
| `DELETE /sites/:siteId/edge/domains/:id` | — | |
| `GET /sites/:siteId/edge/snippet` | `mode=auto\|custom_domain\|proxy\|stealth\|standard` | `{mode, src, global, html, notes}` |
| `GET /sites/:siteId/edge/proxy-config` | `platform=nginx\|apache\|cloudflare\|vercel\|nextjs\|netlify\|wordpress` | Ready-to-install config |

---

## 5. Gotchas

1. `/api/v1/events` needs `site_key` (`tf_…`), dashboard routes need the numeric `:siteId`. Mixing them gives `404 Unknown site_key` / `404 Site not found`.
2. Server events without `visitor_id`, IP/UA or identifiers have low match quality and lose attribution; capture `_tf_vid` in the browser and pass it through checkout/forms.
3. Reuse the browser's event id for the same action; otherwise platforms count it twice.
4. `is_conversion` in request bodies is ignored; set conversions with `PUT /sites/:siteId/conversions` (reporting only).
5. `/api/v1/events` cannot backdate or flag test events. Use `/api/v1/offline-events` (≤ 90 days, Meta accepts ≤ 7 / 62 days) or the test-event route.
6. Test events reach only destinations with a test code / test mode set; others log `skipped`.
7. Blocking rules apply to every source including server events; bot/reload/consent blocking only to browser events.
8. Delivery is asynchronous: a `200` from ingest means accepted, not delivered. Check the event log.
9. Rollback over REST uses the version **id** unless `by: "version"` is sent.
10. Deleting a site or clearing data over REST has no confirmation step — confirm with the customer before calling.
