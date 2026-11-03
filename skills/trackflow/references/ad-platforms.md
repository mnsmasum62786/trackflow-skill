# TrackFlow destinations (ad & analytics platforms)

One section per platform: credentials (config keys exactly as TrackFlow validates them), where the user finds each value,
tracking modes, what the browser tag and the server API do, dedup, when TrackFlow skips a side, and gotchas.
**[vendor]** = general platform knowledge, not enforced by TrackFlow — confirm in the platform's current UI/docs.
All IDs below are placeholders.

Platforms (MCP `type`): `meta` · `tiktok` · `google_ads` · `ga4` · `microsoft` · `snapchat` · `pinterest` · `linkedin` ·
`reddit` · `x` · `quora` · `taboola` · `outbrain` · `nextdoor` · `openai`

---

## 0. How destinations work in TrackFlow

### Modes and field need levels

| `mode` | Meaning |
|---|---|
| `both` (default, recommended) | TrackFlow loads the platform's browser tag **and** sends the same events server-side with the same event ID |
| `browser` | Tag only. Nothing server-side; events get status `browser_only` |
| `server` | Server API only; tag not loaded (use when the site already runs its own pixel you don't want to remove) |

Each field has `need`: `always`, `browser` (needed when the tag is used), `server` (needed for server delivery) or empty
(optional). `list_platforms` returns every field with `key`, `label`, `need`, `section` (`setup` / `mapping` /
`advanced` / `test`), `secret`, `why`, `help`, plus `modes`, `default_mode` and the dedup note.

Saving fails with one clear sentence when the chosen mode lacks a field, e.g.
`Meta: Conversions API access token is required for server-side tracking. Add it, or choose mode "browser" (browser tag only).`
A destination that is not ready for the server is **not attempted** (it never turns events into `failed`).

### MCP flow

1. `list_platforms` → read `fields[].need` for the mode the user wants; ask only for those.
2. `add_destination` `{site_id, type, mode, config:{…}}` → returns `missing` + `next_step`.
3. `update_destination`: only passed keys change; omit a secret or pass `""` to keep it; `""` clears a non-secret field;
   `null` deletes a secret (`refresh_token: null` disconnects Google sign-in). Pause with `enabled: false`.
4. Set the platform's **test field** (table in §0.4), `test_destination`, check `deliveries[].status/response/payload`.
5. **Remove the test field** before going live. `remove_destination` deletes credentials — ask the user first.

### Event statuses and delivery results

Event: `pending` → `delivered` (all attempted destinations OK) / `partial` / `failed` / `blocked` (bot, reload, consent,
blocking rule) / `browser_only` (only browser tags connected) / `no_destination`. Per-destination delivery: `ok`, `error`
(read `response`), `skipped` (with reason — expected for one-sided events, unmapped events, missing click IDs, test events
without a test field, events older than the platform accepts: "Too old for <Platform> (max N days)", lifecycle events
while `send_lifecycle` is off). No automatic retries (10 s timeout per call); `retry_event_delivery` re-sends manually.
Only exception: Google Ads value restatements are queued (`skipped` "Queued: …") and sent by an hourly sweep when due.

### Refunds and cancellations (`send_lifecycle`)

Every platform has the advanced checkbox `send_lifecycle` ("Send refunds and cancellations", **off by default**).
On: lifecycle events from webhooks (`OrderRefunded`, `OrderCancelled`, `SubscriptionRenewed`, `SubscriptionCancelled`,
`BookingCancelled`, `BookingNoShow`) are delivered: GA4 `refund`; Google Ads partial-refund restatement (full refunds
not possible); custom events on Meta, TikTok, Reddit, OpenAI, Microsoft and the custom webhook; Outbrain/Taboola named
conversions with a click id; LinkedIn, X, Nextdoor only when mapped; never Snapchat, Pinterest, Quora.
Set it only when the customer asks. Full table and per-platform max event ages:
[lifecycle-events.md](lifecycle-events.md).

### Which events a destination receives (`event_filter`, `event_list`)

By default every destination receives every event. Each destination can instead receive a chosen set:

| `event_filter` | Effect |
|---|---|
| `all` (default) | every event; a stored `event_list` is kept but ignored |
| `only` | only the names in `event_list` (exact, case-sensitive; `only` with an empty list is rejected: "Pick at least one event or choose All events") |
| `except` | every event except the names in `event_list` |

`event_list`: up to 100 names, each ≤ 64 chars (trimmed, deduped). Example:
`destinations {"action": "update_destination", "site_id": 12, "destination_id": 7, "config": {"event_filter": "only", "event_list": ["Purchase", "Lead"]}}`.

- Applies to both paths: the browser pixel of that platform is not fired for an excluded event (t.js and the
  Shopify Customer Events code), and the server delivery is `skipped` with reason
  `Not in this destination's event list` (every origin: browser, webhook, CRM, offline, API, tests, retries).
- The event itself is still recorded in TrackFlow and sent to the other destinations.
- Lifecycle events (refunds, cancellations…) are not affected; they follow `send_lifecycle`.
- Use it instead of deleting tags when the customer wants, for example, all funnel events on Meta but only
  Purchase and Lead on TikTok.

### Test fields

| Platform | Test field | Note |
|---|---|---|
| Meta / TikTok / Snapchat | `test_event_code` | Snapchat also `validate_only` |
| Google Ads | `validate_only` | nothing ingested |
| GA4 | `debug` | validation server, nothing recorded |
| Pinterest | `test_mode` | |
| Reddit | `test_id` | |
| Nextdoor | `test_event` | |
| OpenAI | `validate_only` | |
| Quora | `debug` | **events still processed live** |
| LinkedIn, X, Microsoft, Taboola, Outbrain | — | test events are skipped for these |

While a test field is set, **all** server events for that destination are test/validation-only.

### PageView and single-page apps (SPA)

TrackFlow's tracker patches `pushState`/`replaceState`/`popstate`/`#/` routes and sends a `PageView` with its own event ID
per route — **only if the site setting `auto_pageview` is on (off for new sites)**; otherwise PageViews come only from
container tags or `trackflow('page')`. Several vendor pixels fire their own ID-less page views on history changes, which
would arrive undeduplicated; TrackFlow turns those off or works around them:

| Platform | Vendor auto-SPA behaviour | What TrackFlow does |
|---|---|---|
| Meta | `fbevents.js` fires PageView on every history change | sets `fbq.disablePushState = true`; sends its own PageView (both sides, shared ID). `autoConfig` off unless `automatic_events` is checked |
| TikTok | `events.js` history observer | `historyObserver: false`; calls `ttq.page()` per route; server PageView skipped while tag loads |
| GA4 | gtag `config` sends page_view (+ enhanced measurement history events **[vendor]**) | configures the ID with `send_page_view: false` and sends `page_view` itself; if the site already configures the same `G-` ID, TrackFlow leaves page views to the site |
| Microsoft UET | auto SPA tracking | **keeps** UET's `enableAutoSpaTracking: true` (UET records page loads); TrackFlow's PageView is a no-op in the browser and skipped server-side while the tag loads |
| Quora | `qevents.js` pushState ViewContent | `qp.disablePushState = true`; PageView → ViewContent with `event_id`, browser only |
| Outbrain | `obtp.js` tracks PAGE_VIEW on every history change | sends PAGE_VIEW only for the landing page; vendor handles the rest (pixel only) |
| LinkedIn | Insight Tag records the page view on load | the first PageView's ID is set as `_linkedin_event_id`; later routes only fire if PageView is mapped to a conversion |
| X | `twq('config')` records Site Visit | Site Visit is browser-only; PageView sent only if mapped in `event_ids` |
| Pinterest | — | `pintrk('page')` + a `pagevisit` event carrying the ID |
| Taboola | — | `page_view` with `item-url` per route (SPA-safe), browser only |
| Snapchat, Reddit, Nextdoor, OpenAI | — | PageView sent as a normal tracked event with the shared ID |

### General gotchas (all platforms)

- Remove hand-installed copies of a pixel TrackFlow loads (theme code, plugins, GTM tags, Shopify/Wix channel apps):
  different event IDs → double counting. Alternative: put that destination in `server` mode.
- One pixel per page for Reddit: a second destination of the same type runs server-only.
  Quora `qp('track')` and Outbrain `obApi` report to **every** pixel/marketer ID on the page.
- Bot user agents and reloaded PageViews (settings `block_bots`, `block_reloads`) are skipped by both the tags and the server.
- With `consent_mode` on, nothing (cookies, pixels, events) runs until marketing consent; TrackFlow reads Cookiebot,
  CookieYes, OneTrust, Complianz, Google Consent Mode, or `trackflow('consent', true|false)`. Server/API events are not filtered.

---

## Meta (Facebook / Instagram) — `meta`

Pixel (`fbevents.js`) + Conversions API (Graph API `v21.0`, `POST /<pixel_id>/events`). Modes: both / browser / server.

| Key | Label | Need | Where to find | Validation |
|---|---|---|---|---|
| `pixel_id` | Pixel ID | always | Events Manager → Data sources → your pixel (dataset) | digits only, 6–20 (`^\d{6,20}$`; usually 15–16) |
| `access_token` | Conversions API access token (secret) | server | Events Manager → your pixel → Settings → Conversions API → Generate access token (admin on the dataset) | — |
| `test_event_code` | Test event code | test | Events Manager → Test events → Server (e.g. `TESTxxxxx`) | — |
| `automatic_events` | Meta automatic events (checkbox) | advanced | off by default; on lets the pixel add SubscribedButtonClick/microdata events (no event ID, browser only) | — |

- **Browser:** `trackSingle` for standard names / `trackSingleCustom` for others (only TrackFlow's pixel receives them,
  never other pixels on the site); options `{eventID}`. Advanced matching: em, ph, fn, ln, ct, st, zp, country
  (lowercased, Meta hashes in browser) + visitor ID as `external_id`. `fbq` accepts customer data only on the **first
  init with data**, so TrackFlow inits empty until real PII is known, then re-inits once. Consent → `fbq('consent', 'grant'|'revoke')`.
- **Server:** `event_name`, `event_time` (s), `event_id`, `action_source`, `event_source_url`, `user_data` (hashed em ph fn
  ln ct st zp country db ge external_id as arrays; raw `client_ip_address`, `client_user_agent`, `fbp`, `fbc`, `lead_id`),
  `custom_data` **verbatim**. CRM events with `lead_id` get `custom_data.event_source: "crm"`, `lead_event_source: "TrackFlow"`.
- **Dedup:** `event_name` + `event_id`; every event incl. PageView sent from both sides.
- **Gotchas:** `test_event_code` diverts all server events to Test Events. Server events older than 7 days are rejected by
  Meta (62 days for `physical_store`). **[vendor]** Event Match Quality improves most with em, ph, fbc, fbp, external_id;
  dedup pairs copies within 48 h; health/financial advertisers may be limited to restricted event/parameter sharing.

## TikTok — `tiktok`

Pixel (`events.js`, `ttq`) + Events API 2.0 (`business-api.tiktok.com/open_api/v1.3/event/track/`, header `Access-Token`).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `pixel_id` | Pixel code | always | Ads Manager → Tools → Events → Web events → your pixel (string like `CXXXXXXXXXXXXXXXXXXX`) |
| `access_token` | Events API access token (secret) | server | Pixel → Settings → Events API → Generate access token (shown once) |
| `test_event_code` | Test event code | test | Pixel → Test events |

- **Browser:** `ttq.instance(pixel).track(event, props, {event_id})`; `ttq.page()` for PageView; `ttq.identify` with email,
  phone (TikTok hashes) and visitor ID as `external_id`; consent → `grantConsent`/`revokeConsent`. `_ttp` cookie travels to the server copy.
- **Server:** `event_source: "web"`, `event_source_id` = pixel; `user`: hashed `email`, `phone`, `external_id` + `ttclid`,
  `ttp`, `ip`, `user_agent`; `properties`: `value`, `currency`, `contents[{content_id, quantity, price}]`, `content_type`,
  `query`, `order_id`. Delivered only if TikTok answers `code: 0`.
- **Dedup:** `event` + `event_id`. **PageView is browser-only while the pixel loads** (`ttq.page()` has no ID); in
  `server` mode PageView is sent as `Pageview`.
- **Gotchas:** `Purchase` becomes `CompletePayment`. Names/address are never sent to TikTok. `ttclid` must survive to the
  landing URL **[vendor]** (don't strip query params in redirects).

## Google Ads — `google_ads`

Google tag (`gtag.js`, `AW-…`) + **Data Manager API** (`datamanager.googleapis.com/v1/events:ingest`, OAuth). No developer
token needed. TrackFlow uses Data Manager because Google says the older click-conversion upload fails from June 15, 2026
for developer tokens that never uploaded before.

| Key | Label | Need | Where to find / format |
|---|---|---|---|
| `conversion_id` | Conversion ID (Google tag) | browser | Goals → Conversions → action → Tag setup → `AW-123456789` in the event snippet |
| `conversion_labels` | Conversion labels (browser) | browser (mapping) | `Event=label` lines/JSON; label = part after `/` in `send_to`. Events without a label are not sent by the tag |
| `customer_id` | Customer ID (server) | server | top right in Google Ads; **must be 10 digits** (`123-456-7890` accepted, stored as digits) |
| `conversion_actions` | Conversion action IDs (server) | server (mapping) | `Event=<numeric action ID>` (`ctId` in the action's URL). Unmapped events skipped |
| `refresh_token` | OAuth refresh token (secret) | server, filled by sign-in | **Sign in with Google** stores it (MCP: `google_ads_sign_in_link`, single-use, 15 min). Manual token needs scope `https://www.googleapis.com/auth/datamanager` |
| `oauth_client_id` / `oauth_client_secret` | own Google Cloud OAuth client | server, unless TrackFlow's Google app is enabled | Cloud console → APIs & Services → Credentials → OAuth client (Web application); enable the Data Manager API; redirect URI shown in the dialog |
| `login_customer_id` | Login (manager) customer ID | optional | MCC ID if access is via a manager account; 10 digits |
| `server_mode` | `click_import` (default) / `multi_source` | optional | see dedup |
| `merchant_id` | Merchant Center ID | optional | sends cart items as Merchant Center products |
| `consent` | `""` / `granted` / `denied` | optional | sets `adUserData`/`adPersonalization` on uploads; set it for EEA users |
| `validate_only` | Validate only (test) | test | |

- **Browser:** `gtag('config', 'AW-…', {allow_enhanced_conversions: true})`; per labelled event
  `gtag('event','conversion',{send_to:'AW-…/label', transaction_id, value, currency})` (currency defaults USD).
  Enhanced conversions: raw email, E.164 phone (only if it matches `+` and 10–15 digits) and address (only when fn + ln +
  zp + country are all known) via `gtag('set','user_data',…)`. gclid read from the tag goes to the server copy.
- **Server:** `transactionId` (order_id ‖ event_id), `eventTimestamp`, `eventSource` (WEB; click_import also IN_STORE /
  PHONE / MESSAGE from `action_source`), `adIdentifiers` gclid/gbraid/wbraid, hashed email (gmail-dot rule) + E.164 phone,
  `conversionValue`, `currency`, `eventDeviceInfo` (IP/UA), `cartData`. **No address** server-side.
- **Dedup (per conversion action):** `click_import` uploads go to a separate "Website (Import from clicks)" action, so
  browser-origin events with a label are left to the tag; server only uploads events the browser never saw (webhook,
  offline, CRM, API). `multi_source` (Google allowlist only) uploads to the tag's own action and Google merges by
  transaction ID (API value overrides, except during its 14-day trial).
- **Skips:** no mapped action; no gclid/gbraid/wbraid, no user data and no IP; older than 90 days.
- **Refunds (`send_lifecycle`):** a partial refund restates the uploaded Purchase (same transaction id and action) to
  the remaining value, ≥ 24 h after the conversion (queued until then) and ≤ 54 days after it. Full refunds,
  cancellations and chargebacks are skipped: the Data Manager API cannot retract conversions (that needs the Google
  Ads API). Renewals etc. only when mapped under Conversion action IDs.
- **Gotchas:** one label **and** one action ID per conversion action/event. Sign-in errors: `redirect_uri_mismatch`
  (add redirect URI), `invalid_client`, `unauthorized_client` (must be Web application), `access_denied` (add test user).
  **[vendor]** Enhanced conversions must be turned on in Google Ads conversion settings; EEA traffic needs Consent Mode v2 signals.

## Google Analytics 4 — `ga4`

`gtag.js` (`G-…`) + Measurement Protocol (`www.google-analytics.com/mp/collect`, EU: `region1.google-analytics.com`).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `measurement_id` | Measurement ID | always | Admin → Data streams → web stream (`G-XXXXXXXXXX`) |
| `api_secret` | Measurement Protocol API secret (secret) | server | Admin → Data streams → web stream → Measurement Protocol API secrets → Create |
| `region` | `global` / `eu` | optional | server events only |
| `consent` | `""` / `granted` / `denied` | optional | `ad_user_data`/`ad_personalization` on server events |
| `debug` | Validation mode (test) | test | `/debug/mp/collect`, records nothing |

- **Browser:** events → GA4 names (`purchase`, `generate_lead`, `sign_up`… custom → snake_case ≤ 40 chars) with
  `send_to`, value/currency (USD default), `transaction_id` (Purchase: order_id ‖ event_id), `items`, `search_term`;
  `user_data` email/phone via `gtag('set')`; `user_id` from `external_id`. Reads `client_id`/`session_id` for server copies.
- **Server:** `client_id` (from `_ga`, else derived from the hashed visitor ID), `timestamp_micros`, `user_data`
  (`sha256_email_address`, `sha256_phone_number`), `ip_override`, `user_agent`, params incl. `session_id`,
  `engagement_time_msec: 1`, `page_location`, `page_referrer`.
- **Dedup:** GA4 has none by event ID. **While the tag loads, the server skips every browser-origin event** ("Sent by the
  GA4 tag") and sends only webhook/offline/CRM/API events. Purchases share `transaction_id`, which GA4 dedupes.
  `browser` mode is a complete GA4 setup; `api_secret` matters only for server-originated events.
- **Refunds (`send_lifecycle`):** refunds, cancellations and chargebacks → GA4 `refund` event with `transaction_id` =
  order id and the refunded value (lowers that transaction's revenue); renewals and subscription/booking events →
  custom events. Within 72 h.
- **Gotchas:** server events > 72 h old are skipped. `/mp/collect` returns 2xx even for invalid payloads → test with
  `debug`. Don't retry delivered events. **[vendor]** user-provided data collection must be enabled in the property for
  user_data; mark events as key events in GA4 Admin; never send PII in params/URLs.

## Microsoft Advertising — `microsoft`

UET tag (`bat.js`, `uetq`) + UET Conversions API (`capi.uet.microsoft.com/v1/<tag_id>/events`, Bearer, `dataProvider: "TrackFlow"`).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `tag_id` | UET tag ID | always | Tools → UET tag → Tag ID (numeric) |
| `access_token` | Conversions API token (secret) | server | UET tag → Edit → Save and next → Set up tagging → Use Conversions API → Copy token |
| `event_actions` | Event action overrides | mapping | `Lead=submit_lead_form` — only if goals use custom actions (defaults GA4-style: `purchase`, `generate_lead`…) |
| `consent` | `""`/`granted`/`denied` | optional | `adStorageConsent` (Microsoft treats empty as granted) |

- **Browser:** UET with `enableAutoSpaTracking: true`; own queue if the site already runs a UET tag. Events:
  `uetq.push('event', action, {event_id, revenue_value, currency, ecomm_prodid, ecomm_pagetype, ecomm_totalvalue, pid})`;
  `pid` = raw email / E.164 phone (UET hashes). Consent → `ad_storage`.
- **Server:** `eventType: custom` (pageLoad for PageView), `eventId`, `eventName`, `eventTime`, `userData` (`anonymousId`
  = `_uetvid` or hashed visitor ID, hashed `em`/`ph`, `msclkid` without dashes, IP, UA), `customData` (value, currency,
  transactionId, searchTerm, items, pageType, ecommTotalValue).
- **Dedup:** `eventId` + same action name. PageView browser-only while the tag loads.
- **Skips:** > 90 days old; no anonymousId/em/ph/msclkid. **[vendor]** msclkid needs auto-tagging on in Microsoft Ads;
  goals must use the same event action names.

## Snapchat — `snapchat`

Snap Pixel (`snaptr`) + Conversions API v3 (`tr.snapchat.com/v3/<pixel_id>/events`, `/events/validate` for validation).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `pixel_id` | Pixel ID (UUID) | always | Ads Manager → Events Manager → your Snap Pixel |
| `access_token` | Conversions API token (secret) | server | Events Manager → Pixel → Conversions API → Generate token (or Business settings → Conversions API tokens) |
| `test_event_code` / `validate_only` | test | test | |

- **Browser:** `snaptr('track', NAME, {client_dedup_id, price, currency, item_ids, number_items, item_category,
  transaction_id, search_string, user_email, user_phone_number})` (pixel hashes PII). `_scid` → server `sc_cookie1`.
- **Server:** `event_id` (= client_dedup_id), `event_time` ms, `action_source` WEB / MOBILE_APP / OFFLINE, hashed user
  data arrays, `sc_click_id` (from `?ScCid=`), `custom_data` incl. `order_id`, `num_items` (string), `predicted_ltv`.
- **Dedup:** client_dedup_id = event_id, same pixel + name, 48 h; purchases also `transaction_id` = `order_id`.
- **Gotchas:** only Snap event names are accepted → custom events and Contact/Donate/etc. are **not sent** (use
  `CUSTOM_EVENT_1`–`5` as the TrackFlow event name if needed). `Lead`, `CompleteRegistration`, `SubmitApplication` all
  become `SIGN_UP`.

## Pinterest — `pinterest`

Pinterest Tag (`pintrk`) + Conversions API v5 (`api.pinterest.com/v5/ad_accounts/<ad_account_id>/events`, Bearer).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `pixel_id` | Tag ID | browser | Ads Manager → Conversions → Pinterest Tag → Tag ID |
| `ad_account_id` | Ad account ID | server | number in the Ads Manager URL (`ads.pinterest.com/advertiser/<id>/`) |
| `access_token` | Conversions access token (secret) | server | Ads Manager → Conversions → Conversions API → Generate new token (starts with `pina_`) |
| `test_mode` | Test mode | test | `?test=true` |

- **Browser:** `pintrk('load', id, {em})` (plain email for enhanced match), `pintrk('track', name, {event_id, value,
  currency, order_id, search_query, line_items, order_quantity})`. `_epik` → server `click_id`.
- **Server:** `event_name` (`checkout` for Purchase, `custom` for unknown), `action_source` web/offline, `event_time`,
  `event_id`, hashed user data arrays, `custom_data.value` as **decimal string**.
- **Dedup:** event_id + same name, first copy in 48 h kept. Server-only mode needs no Tag ID.

## LinkedIn — `linkedin`

Insight Tag (`lintrk`) + Conversions API (`api.linkedin.com/rest/conversionEvents`, header `LinkedIn-Version` default `202610`).

| Key | Label | Need | Where to find / format |
|---|---|---|---|
| `partner_id` | Insight Tag partner ID | browser | Campaign Manager → Analyze → Insight Tag → `_linkedin_partner_id` in the tag code |
| `conversion_ids` | Insight Tag conversion IDs | mapping (browser) | `Purchase=1234567` — conversions created with the **Event-specific** method |
| `access_token` | Conversions API access token (secret) | server | OAuth member token with `rw_conversions` (and `r_ads`) from a user with Campaign Manager access or higher; **expires after 60 days** |
| `conversion_rules` | Conversions API rule IDs | server (mapping) | `Purchase=104004` or `urn:lla:llaPartnerConversion:<id>` — rules whose data source is Conversions API |
| `api_version` | LinkedIn-Version | optional | `YYYYMM`; malformed → default. Versions sunset after ~1 year |

- **Browser:** tag loads with `_linkedin_event_id` = first PageView ID; mapped events fire
  `lintrk('track', {conversion_id, event_id})`. `li_fat_id` cookie / `?li_fat_id=` → server.
- **Server:** `conversion` (rule URN), `conversionHappenedAt` (ms), `eventId`, `user.userIds` (`SHA256_EMAIL`,
  `LINKEDIN_FIRST_PARTY_ADS_TRACKING_UUID`, `PLAINTEXT_IP_ADDRESS` **IPv4 only**), `userInfo` hashed first+last name
  (pair, version ≥ 202609), `externalIds`, `conversionValue {currencyCode, amount}` only with both value and currency.
- **Dedup:** same eventId on both sides; LinkedIn keeps the Insight Tag copy. Use **different** rules for tag and CAPI.
- **Skips:** no rule mapped; no email / li_fat_id / IPv4 / first+last name. Token expiry is the most common failure.

## Reddit — `reddit`

Reddit Pixel (`rdt`) + Conversions API v3 (`ads-api.reddit.com/api/v3/pixels/<pixel_id>/conversion_events`, Bearer).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `pixel_id` | Pixel ID | always | Ads Manager → Events Manager → your pixel (starts `a2_` or `t2_`) |
| `access_token` | Conversion access token (secret) | server | Events Manager → Conversions API → Generate Access Token (shown once) |
| `test_id` | Test ID | test | Events Manager → Event Testing |

- **Browser:** `rdt('init', id, {email, phoneNumber, useDecimalCurrencyValues: true})`; `rdt('track', Name,
  {conversionId, value, currency, itemCount, products, customEventName})`. Cookies `_rdt_uuid`, `_rdt_cid` → server.
- **Server:** `event_at` ms, `action_source` WEBSITE/PHYSICAL_STORE/APP/OTHER, `type.tracking_type` (CUSTOM +
  `custom_event_name` for others), `metadata` (conversion_id, order_id, value, currency, item_count, products), `user`
  (email em_r rule, phone E.164, external_id, IP, UA, uuid), `click_id`. Physical-store events carry no device/click IDs.
- **Dedup:** conversionId = `metadata.conversion_id`, same type/name, website channel, within 2 days.
- **Gotchas:** one Reddit pixel per page (second destination is server-only).

## X (Twitter) — `x`

X Pixel (`twq`) + Conversion API (`ads-api.x.com/12/measurement/conversions/<pixel_id>`, header `X-Pixel-Token` or OAuth 1.0a).

| Key | Label | Need | Where to find / format |
|---|---|---|---|
| `pixel_id` | Pixel ID | always | Ads Manager → Tools → Events Manager → your X Pixel event source ID |
| `event_ids` | Event IDs per event | server (mapping, also used by the pixel) | `Purchase=tw-<pixel>-<event>` from Events Manager |
| `access_token` | Conversion API access token (secret) | server | Events Manager → Install Pixel → Manual → Conversion API → Generate access token |
| `consumer_key`, `consumer_secret`, `access_token_secret` | OAuth 1.0a (secrets) | optional, advanced | only with your own Ads API app; **fill all three or none** (partial set = error) |

- **Browser:** `twq('config', pixel)` (automatic Site Visit, browser only); mapped events `twq('event',
  'tw-<pixel>-<event>', {conversion_id, value, currency, contents, email_address, phone_number})` (pixel hashes; phone
  must be E.164). `_twclid` / `_twpid` → server.
- **Server:** `conversion_time`, `event_id` (event part of the tw- ID), `conversion_id` (= TrackFlow event ID),
  `identifiers` (twclid, twpid, hashed_email, hashed_phone_number, ip_address+user_agent), `value` (2-dec string),
  `price_currency`, `contents`, `number_items`.
- **Dedup:** same Events Manager event ID + `conversion_id`. **Unmapped events are not sent from either side.**
- **Skips:** no twclid, email or phone and no IP+UA pair.

## Quora Ads — `quora`

Quora Pixel (`qp`) + Conversion API (`api.quora.com/ads/v0/conversion`, Bearer).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `pixel_id` | Pixel ID | browser | Ads Manager → Pixels & Events → Setup Pixel: id in `qp('init', '…')` |
| `account_id` | Ad account ID | server | numeric Quora Ads account ID (non-numeric → server skips) |
| `access_token` | Conversion API token (secret) | server | Ads Manager → Conversion API tab → Generate Token (does not expire) |
| `unmapped` | `generic` (default) / `skip` | mapping | what to do with events Quora has no label for |
| `debug` | Debug mode | test | **events are still processed** — not a safe test mode |

- **Browser:** `qp('init', id, {email})` (only on init), `qp('track', label, {event_id, value})`. `quora_qclid` → server.
- **Server:** `conversion` (`event_name`, `timestamp` **µs**, `event_id`, `click_id`, `value`), `user` (email em_gp, ip),
  `device` (user_agent, referer).
- **Gotchas:** PageView is browser-only. **Value is read in the account currency** (no currency sent). Phone/name/geo not
  sent. Server events need the qclid to be attributed.

## Taboola — `taboola`

Taboola Pixel (`_tfa`) + S2S postback (`GET trc.taboola.com/actions-handler/log/3/s2s-action`). No server credentials.

| Key | Label | Need | Notes |
|---|---|---|---|
| `account_id` | Account ID | browser | Realize → Tracking: numeric Account ID (top left) |
| `event_names` | Event names | mapping | `Purchase=make_purchase`; must equal the **case-sensitive** Event Name of the Realize conversion |
| `unmapped` | `send` / `skip` | mapping | other events: snake_case name or not sent |

- Defaults: `make_purchase`, `complete_registration`, `view_content`, `add_to_cart`, `start_checkout`, `lead`, `search`, `page_view`.
- **Server:** `click-id` (tblci), `name`, `revenue` + `currency` (only AUD BRL CAD CNY EUR GBP HKD ILS INR JPY KRW MXN NZD
  RUB SGD THB TRY USD ZAR), `quantity`, `orderid` (order_id ‖ event_id).
- **Dedup:** same name in the same click context, S2S copy kept; both send the same `orderid`.
- **Gotchas:** S2S needs `tblci` on the landing URL — add `tblci={click_id}` to the campaign tracking code; without it
  only the pixel counts. `page_view` browser only. Page view sends hashed email as `unified_id`.

## Outbrain — `outbrain`

Outbrain Pixel (`obApi`) + S2S postback (`GET tr.outbrain.com/unifiedPixel`). Modes: **both / server only** (no browser-only).

| Key | Label | Need | Notes |
|---|---|---|---|
| `marketer_id` | Marketer ID (OB_ADV_ID) | browser | Amplify → Conversions → Outbrain Pixel: `OB_ADV_ID` value |
| `event_names` | Conversion names | mapping | Amplify conversion names are case-sensitive; default = TrackFlow name |
| `unmapped` | `send` / `skip` | mapping | only matters when names are mapped |

- **No dedup key exists**, so each event goes one way: visitor with an Outbrain click ID → S2S only; without → pixel only.
  Add `OutbrainClickId={{ob_click_id}}` (or `obclid=`) to the campaign tracking code.
- **Server:** `ob_click_id`, `name`, `orderValue` + `currency` (USD CAD EUR GBP ILS AUD MXN BRL SEK SGD RUB NZD INR JPY PHP
  CHF MYR only), `orderId`, `timestamp` (if > 5 min old). PAGE_VIEW pixel only.

## Nextdoor — `nextdoor`

Universal Pixel (`ndp`) + Conversions API (`ads.nextdoor.com/v2/api/conversions/track`, Bearer, `partner_id: "TrackFlow"`).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `pixel_id` | Pixel ID (data source ID) | always | Ads Manager → Assets → Pixels |
| `access_token` | Conversions API token (secret) | server | Ads Manager → Business Settings → Ads API → Generate token |
| `custom_events` | Custom conversion slots | mapping | `Contact=1` — slots 1–10 |
| `client_id` | Advertiser ID | optional | deprecated; only if Nextdoor asks |
| `test_event` | Test event code | test | |

- Dedup: `event_id` on the same pixel/data_source_id. `?ndclid=` captured → `customer.click_id`.
- **Gotchas:** `order_value` is sent as `"USD12.34"` and only when currency is present. **Phone is never sent** (Nextdoor
  hashes 10 digits without country code).

## OpenAI (ChatGPT Ads) — `openai`

Measurement Pixel (`oaiq`) + Conversions API (`bzr.openai.com/v1/events?pid=<pixel_id>`, Bearer, `integration_source: "trackflow"`).

| Key | Label | Need | Where to find |
|---|---|---|---|
| `pixel_id` | Pixel ID | always | ChatGPT Ads Manager → Conversions tab (one data source for pixel + API) |
| `access_token` | Conversions API key (secret) | server | Ads Manager → Conversions tab → create a Conversions API key |
| `validate_only` | Validate only | test | checked, not saved |

- Events: page_viewed, contents_viewed, items_added, checkout_started, order_created, lead_created,
  registration_completed, appointment_scheduled, subscription_created, trial_started; others `custom` +
  `custom_event_name` (snake_case, ≤ 64).
- **Amounts are integers in minor units** (TrackFlow converts; JPY/KRW 0 decimals, KWD/BHD 3) and need a currency.
- Server: hashed emails/phones/external_ids/first/last names, `oppref` (click ref), `obref`, IP, UA; web events require a
  `source_url` (else `action_source: other`). Geo (country/city/region/postal) only from the browser.
- Dedup: `event_id` = `events[].id`, first copy kept. Server events > 7 days old skipped. Browser uses `measureSingle`
  so other OpenAI pixels on the site are unaffected.

## Custom destination (generic outbound webhook) — `custom`

Use it when the customer wants a platform that is **not** in the built-in list (Klaviyo, Criteo, an
in-house backend, a data warehouse endpoint, …). Server-side only (no browser tag), so mode is `server`.

| Field | Notes |
|---|---|
| `url` | **https only.** Localhost, private, loopback and link-local hosts are rejected at save time, and DNS is re-checked on every send. Redirects are not followed. |
| `method` | `POST` (default), `PUT` or `PATCH` |
| `headers` | Object of request headers (API keys, version headers). Values are secrets: stored server-side and returned masked. Updating `headers` **replaces the whole set**: send every header you want to keep. |
| `events` | Optional allow-list of event names; empty = every event. |
| `payload_template` | Optional JSON with `{{path}}` placeholders, e.g. `{{event_name}}`, `{{event_id}}`, `{{event_time}}`, `{{value}}`, `{{currency}}`, `{{order_id}}`, `{{url}}`, `{{ip}}`, `{{user_agent}}`, `{{custom_data.*}}`, `{{user_data.*}}`, `{{click_ids.*}}`, `{{contents}}`. Unresolved placeholders become `null`. Must be valid JSON (checked on save). Without a template TrackFlow sends its standard JSON. |
| `signing_secret` | Optional. Adds `X-TrackFlow-Signature: sha256=<hex HMAC-SHA256 of the raw body>` (plus `X-TrackFlow-Timestamp`, not part of the signature). |
| `test_mode` | Test field: while on, test events are delivered and **every** request carries `test: true` / `X-TrackFlow-Test: 1`. Turn it off before go-live. |

Behaviour: `event_id` is always in the body and in `X-TrackFlow-Event-Id`, so the receiver can dedup.
User data is only available as **SHA-256 hashes** (TrackFlow never stores raw PII). 10 s timeout per
attempt; 429/5xx/network errors retry twice, other 4xx fail immediately; 2xx = delivered.

How an agent sets it up: read the target platform's API docs, then
`destinations → list_platforms` (the `custom` entry includes an example), `add_destination`
`{"type": "custom", "config": {"url": "...", "headers": {...}, "payload_template": {...}, "events": ["Purchase"]}}`,
turn on `test_mode`, run `test_destination` and read the receiver's response in `get_event`, then turn
`test_mode` off.
