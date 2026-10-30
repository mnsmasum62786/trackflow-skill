# TrackFlow reports and audit data

What the reporting tools return and how to read them when auditing a setup. Tools are in the `reports`
group (`{"action": "...", "site_id": N, ...}`) unless noted. All are read-only except
`send_test_event`, `test_destination` and `retry_event_delivery`.

| Action | Params | Answers |
|---|---|---|
| `get_overview` | `days` (1–365, default 7) or `from`+`to` (Unix **ms**, `to` exclusive, ≤366 days), `conv` | Is tracking working, delivery health, what converts |
| `get_event_flow` | `days` or `from`+`to` | Sources → event names → destinations with ok/error/skipped counts and last error |
| `list_events` | `name`, `status`, `origin`, `q`, `conversions_only`, `conv`, `test` (`only`/`exclude`), `limit` (default 25, max 200), `offset` | The event log |
| `get_event` | `event_row_id` (= `id` from list_events, not `event_id`) | Full event + every delivery payload and platform response |
| `retry_event_delivery` | `event_row_id` | Re-send to all enabled destinations with current settings |
| `get_attribution` | `days` (default 30) / range, `model` (`last` default, `first`), `conv` | Conversions & revenue by source, campaign, event, device, landing page |
| `list_journeys` / `get_journey` | range, `all`, `conv` / `vid` | Visitor timelines |
| `get_match_quality` | `days` / range | Score 0–10 per event name + identifier coverage |
| `send_test_event` | `event_name` (default Lead), `value` | One server test event through the pipeline |
| `test_destination` (group `destinations`) | `destination_id`, `event_name`, `value` | Same, then waits and returns each destination's answer |
| `inspect_site` (group `tag_manager`) | `url` | What is on a page and can be tracked |
| `get_conversion_events` (group `conversions`) | – | Which event names count as conversions in TrackFlow reports |

`conv`: `"site"` (default, the site's conversion list) or a destination id (that destination's own
list). It only changes TrackFlow's reporting, never what platforms receive. New sites count `Purchase`
and `Lead` as conversions; `set_conversion_events` changes that (and re-marks past events).

---

## 1. get_overview

| Field | Meaning |
|---|---|
| `site` | Site with resolved settings, `last_seen_at` (last time the tracker loaded/reported; empty = tag never seen). |
| `range` | `{from, to, days, custom}` |
| `totals.events` | **Every** event in the period, including blocked and test events. |
| `totals.sessions`, `visitors`, `conversions`, `revenue` | Real traffic only: **excludes blocked and test events**. Revenue = sum of `value` on conversion events. |
| `totals.delivered`, `failed`, `partial`, `blocked` | Event counts by status. |
| `totals.undelivered` | Count of status `no_destination`. (`browser_only` and `pending` are not broken out here; use `list_events status` or `get_event_flow`.) |
| `totals.match` | Average match score of all events. |
| `prev` | `events`, `conversions`, `sessions` for the previous period of equal length. |
| `series` | `[{t, events, conversions}]` per hour (≤2 days) or day (`bucket` ms). |
| `top_conversions` | `[{name, n}]` top 5 conversion event names. |
| `top_sources` | `[{name, n}]` top 6 sources by sessions. |
| `recent_events` | Last 8 events of any kind: `id, event_name, status, origin, url, created_at, match_score`. |
| `recent_conversions` | Last 8 real conversions: `value, currency, source, vid`. |
| `destinations` | `[{id, type, pixel_id, enabled, has_token, mode, ready}]`. `has_token` = server API usable; `ready` = nothing missing for its `mode` (`both` / `browser` / `server`). |

## 2. get_event_flow

`totals` (events, blocked, no_destination, pending, delivered, partial, failed), `sources`
(`browser`, `server`, `webhook`, `offline`, `crm`, `api` with counts and blocked), `traffic` (top 5
browser sources), `events` (top 8 names + "Other"), `destinations` (each with `ok`, `error`, `skipped`,
`events`, `attempts`, `last_at`, `last_error {http_status, message, at}`, `server`, `browser_tag`,
`removed`), `links`. Counts use the **latest server attempt** per event and destination. Browser tags
fire in the visitor's browser and are **not logged server-side**: a destination flagged `browser_tag`
gets no counts for them.

## 3. list_events / get_event

`list_events` row: `id`, `event_id`, `event_name`, `origin`, `status`, `block_reason`, `url`, `source`,
`device`, `value`, `currency`, `is_conversion`, `match_score`, `is_test`, `created_at`; plus `total`,
`names` (every event name ever seen on the site) and `conv`. `q` searches URL, event id and source.

### Event status

| `status` | Meaning | Typical action |
|---|---|---|
| `pending` | Accepted, delivery running (normally <1 s). | Re-check; if stuck, `get_event`. |
| `delivered` | Every attempted destination accepted it (HTTP OK). | None. Accepted ≠ attributed: check the platform's own reports. |
| `partial` | At least one destination OK, at least one error. | `get_event` → read the failing delivery. |
| `failed` | Every attempted destination returned an error. | `get_event` → fix credentials/fields → `retry_event_delivery`. |
| `blocked` | Stopped before delivery; `block_reason` says why. Never sent anywhere (browser tags skip bot UAs and reload PageViews too). Cannot be retried. | Expected for junk. Investigate if real. |
| `browser_only` | No destination sends server-side, but at least one browser tag is loaded: the pixels handled it. Normal for destinations in `browser` mode. | For server-side too, add server fields and switch mode to `both`. |
| `no_destination` | No destination attempted a server delivery and no browser tag applies: no enabled destination, or a **server-side event** (webhook, CRM, offline, API, test) while every destination is browser-only/missing server fields, or a **test event whose deliveries were all skipped**. | Add/enable a destination with server credentials; for tests set a test code (see §6). |

`block_reason` values: `Bot traffic` (UA matches bots, headless browsers, empty UA), `Page reload`
(PageView of a reload, setting `block_reloads`), `No marketing consent` (consent mode), `Rule: <type>
<pattern>` (a blocking rule; `list_blocking_rules` in group `install`).

`origin`: `browser` (tracker/tags), `server` (server events API and test events), `webhook` (checkout
integrations and mapped webhooks), `offline`, `crm`, `api`.

### get_event

`event`: `event_id`, `event_name`, `origin`, `vid`, `sid`, `url`, `referrer`, `source`/`medium`/
`campaign` (attributed), `ip`, `user_agent`, `device`, `user_data` (**SHA-256 hashes only**; keys like
`em`, `ph`, `fn`, `ln`, `ct`, `st`, `zp`, `country`, `external_id` plus platform variants `em_g`,
`em_r`, `em_gp`, `ph_e164`, `ph_last10`), `custom_data`, `click_ids` (`fbp`, `fbc`, `ttp`, `gclid`…),
`value`, `currency`, `is_conversion`, `match_score`, `status`, `block_reason`, `is_test`, `created_at`.

`deliveries`: one row **per attempt** (a retry adds rows): `type` (platform), `status` (`ok` / `error` /
`skipped`), `http_status`, `payload` (exactly what was sent; hashed identifiers only, never
credentials), `response` (platform reply, trimmed to 4000 chars).

Reading deliveries:

| You see | Usually means |
|---|---|
| `skipped` + "Test event: set this destination's test code / test mode…" | A test/preview event and the destination has no test mode set. Nothing was sent. |
| `skipped` with another reason | The platform can't use this event (e.g. GA4 events older than 72 h; AppLovin without page URL; Taboola/Outbrain without their click id). |
| Meta HTTP 400 "Invalid OAuth access token" / code 190 | Token wrong, expired, revoked, or for another business. New token → `update_destination` → retry. |
| Meta HTTP 400 mentioning the pixel/dataset id | Pixel id typo, or the token's system user has no access to that pixel. |
| Meta HTTP 400 about `event_time` | Event too old (Meta: 7 days; 62 for `physical_store`) — typical for offline/CRM backfills. |
| Meta HTTP 400 about parameters | Malformed custom data (e.g. value not numeric). Compare `payload` to the error. |
| TikTok HTTP 200 with non-zero `code` | Rejected; read `message`. |
| HTTP status 0 / timeout | Network error or 10 s timeout. `retry_event_delivery`. |
| `ok` but nothing in the platform's live reports | A test code / validate-only flag left on the destination (see diagnosis table). |

`retry_event_delivery` re-sends with the **same event id** (platforms dedup it, so resending a
delivered event is normally harmless) and returns the new `status`. Test events stay test on retry.

## 4. get_attribution, list_journeys, get_journey

`get_attribution` (excludes blocked and test events): `sources[{source, sessions, visitors,
conversions, revenue}]`, `campaigns[{source, campaign, conversions, revenue, sessions}]` (top 20),
`by_event[{source, event_name, n}]`, `devices[{name, sessions, conversions}]`,
`landing[{url, visitors, converted}]` (top 10). `model: "first"` uses the visitor's first source.

Source classification (first match wins): `fbclid` or Meta utm_source with paid medium → **Meta Ads**;
`ttclid` → **TikTok Ads**; `gclid`/`gbraid`/`wbraid` → **Google Ads**; `msclkid` → **Microsoft Ads**;
other `utm_source` → its value; no referrer or own domain → **Direct**; search engine → **Organic
search**; social network → **Organic social**; AI assistant → **AI assistants**; else the referring
domain. Campaign = `utm_campaign` (or `campaign_id`). Last touch: a Direct visit never overwrites an
earlier real source. Server events attribute to the matched visitor's last meaningful touch.

`list_journeys`: up to 100 visitors (converted only unless `all: true`): `vid`, `first_seen`,
`last_seen`, `first_source`, `last_source`, `first_campaign`, `landing_url`, `device`, `converted`,
`events`, `sessions`, `conversions` (names), `revenue`, `first_conv_at`.
`get_journey`: `visitor` (sources, landing, device, `identifiers` = which hashed keys are known, never
values) + every event (`event_name`, `origin`, `url`, `referrer`, `source`, `campaign`, `status`,
`value`, `currency`, `is_conversion`, `sid`, `created_at`). Use it to prove that a webhook/CRM/offline
conversion stitched to an ad-click visitor (same `vid`, earlier browser events with a paid source).

TrackFlow attribution is first-party; ad platforms use their own windows and view-through, so numbers
differ. Never present them as equal.

## 5. get_match_quality

Returns `events[{name, n, score, coverage}]`: `score` = average estimated match score (0–10), `coverage`
= % of that event's rows carrying each key (user-data keys incl. hash variants, and click ids such as
`fbc`, `fbp`, `gclid`, `ttp`). Uses the most recent 5000 non-blocked, non-test events in the range.

It is TrackFlow's **estimate** modelled on Meta's Event Match Quality, not Meta's number (that is in
Meta Events Manager). Weights (sum, capped at 10):

| Key | Weight | Source |
|---|---|---|
| `em` | 3.0 | Forms (`auto_forms`, `{{form_user}}`), `identify`, webhooks, CRM, offline |
| `ph` | 2.2 | Same; needs country code (or a country) to normalize |
| `fbc` | 1.6 | Visitor arrived from a Meta ad (`fbclid`) |
| `external_id` | 1.1 | Automatic from the visitor id on browser events, or your own id |
| `fbp` | 0.9 | Meta Pixel cookie (browser tag loaded) |
| `ip`, `ua` | 0.6 each | Request (server events inherit them from a matched visitor) |
| `fn`, `ln` | 0.4 each | Forms / payloads |
| `zp` | 0.3 | |
| `ct`, `st`, `country`, `db` | 0.2 each | |
| `ge` | 0.1 | |

Rough reading: anonymous browser event ≈ 3 (ip + ua + external_id + fbp); + email ≈ 6; + phone + fbc ≈ 9–10.
Unmatched server events without email/phone score very low. Identifiers captured once (form submit,
identify) are reused on that visitor's later events.

How to raise it: `user_data: "{{form_user}}"` on lead tags, `auto_forms: true` (if the customer wants
form capture), an `identify` action on checkout forms, email/phone mappings on webhooks and CRMs,
visitor-id passthrough (`decorate_checkouts`) so server events inherit click ids, phones with country code.

## 6. send_test_event and test_destination

Both create one **server-side test event** (`origin: server`, `is_test: 1`, URL
`https://<site domain>/trackflow-test`, test email, documentation IP, value in USD if given).

- Delivered **only to destinations with a test mode configured**: Meta/TikTok `test_event_code`, Google
  Ads `validate_only`, or the platform's other test field (`test_mode`, `test_id`, `validate`,
  `debug_mode`, any field in the platform's test section). Others record a `skipped` delivery, so with
  no test codes the event ends as `no_destination`. That proves the pipeline runs, not that credentials
  work.
- To prove credentials: set the platform's test code with `update_destination`, run `test_destination`,
  check the platform's Test Events screen, then **remove the test code** (`update_destination` with
  `""`) or live server events keep going to the test view.
- `test_destination` waits up to ~12 s and returns `event_id`, `event_row_id`, `status`, `match_score`,
  `deliveries[{type, status, http_status, response, payload}]` (filtered to `destination_id` if given),
  `destination {id, type, name, enabled, server_ready, mode}` and a `warning`/`note` when the
  destination is paused, browser-only, has no server API, or misses server fields. `status: pending` →
  call `get_event` later.
- A blocking rule matching the test (e.g. event name) returns `status: blocked`.
- Test events are excluded from overview business numbers, attribution and match quality.

## 7. inspect_site (group `tag_manager`)

Fetches one page **server-side** (no JavaScript executed) on the site's domain, its subdomains or its
`cross_domains` only; http(s), up to 5 redirects, 12 s timeout, first 3 MB. Private/internal hosts are
refused. HTTP ≥400 returns `{url, status, error}`. Bot-protected sites may answer 403/challenge pages.

| Field | Content |
|---|---|
| `status`, `url`, `title` | Final URL after redirects |
| `platform` | Detected: shopify, woocommerce, wordpress, elementor, wix, squarespace, webflow, framer, nextjs, bigcommerce, magento, gohighlevel |
| `existing_tags` | `gtm` (GTM-…), `ga4` (G-…), `meta_pixel` (ids from `fbq('init')`), `tiktok_pixel`, `trackflow` (bool; may miss first-party/stealth snippets) |
| `forms[]` (≤15) | `selector` (suggested), `id`, `action`, `method`, `builder` (cf7, elementor, gravity, wpforms, ninja, fluent, formidable, hubspot, webflow, squarespace, mailchimp, klaviyo, woocommerce_checkout, shopify_cart), `kind` (`lead`, `newsletter`, `search`, `login`, `add_to_cart`, `checkout`), `external`, `fields[{name, type, label, required}]`, `submit_text` |
| `embedded_forms` | Client-rendered builders found: hubspot, typeform, jotform, tally, calendly, cal, klaviyo, cf7, gravity, elementor, wpforms |
| `ctas[]` (≤40) | Buttons/links with CTA words or `tel:`/`mailto:`/`sms:`: `tag`, `text`, `href`, `selector`, `provider` |
| `json_ld_types`, `json_ld_product`, `json_ld_errors` | schema.org types; Product `{name, id, brand, category, price, currency, availability}`; unparsable blocks |
| `meta`, `microdata_price` | `og:*` / `product:*` tags; `itemprop=price` |
| `datalayer` | `pushes` (≤8 snippets), `events`, `gtag_events`, `ecommerce` (GA4 ecommerce present) |
| `checkout_links[]` | `{provider, href, text}`: stripe_payment_link, stripe_checkout, paypal, lemonsqueezy, gumroad, thrivecart, samcart, paddle, shopify_checkout, calendly, cal, typeform, jotform, tally, hubspot, kajabi, teachable, podia, kartra, clickfunnels, whatsapp, onsite_cart |
| `iframes[]`, `videos` | Embeds with provider; html5/youtube/vimeo counts |
| `suggestions[]` | Ready advice: which recipe, enable `ecommerce_datalayer`, `decorate_checkouts`, tel/WhatsApp click tags, video triggers |

Content rendered by JavaScript (SPAs, many Wix/Framer sections, popups, cart drawers) is invisible to
it: confirm selectors in preview. Inspect product, checkout, thank-you and contact pages separately.

---

## 8. Audit procedure (read-only first)

1. `get_site`: `last_seen_at`, settings (`auto_pageview`, `auto_forms`, `ecommerce_datalayer`,
   `consent_mode`, `block_bots`, `block_reloads`, `dedup` must be on, `cross_domains`).
2. `list_destinations` (group `destinations`): `enabled`, `mode`, `ready`, `missing`, `has_token`, any
   test code left on.
3. `get_overview days:30` and `get_event_flow`: volume, status mix, per-destination errors.
4. `list_events status:"failed"` / `"partial"` / `"no_destination"` / `"blocked"` → `get_event` on a few.
5. `list_events name:"Purchase"` (and Lead): value, currency, duplicates (same order in two rows),
   `origin` mix (browser + webhook for the same order?).
6. `get_match_quality`: score and `em`/`ph`/`fbc` coverage per conversion event.
7. `get_container which:"published"` + `list_tags` + `list_versions`: what is live, unpublished drafts,
   custom JS.
8. `inspect_site` on key pages: hard-coded pixels (`existing_tags`) that duplicate TrackFlow events.
9. `list_checkout_integrations` / `list_crm_integrations` / `list_offline_sources`: secrets set,
   `last_error`, logs.
10. Report findings ranked by revenue impact; propose fixes; change nothing until the customer agrees.

## 9. Diagnosis table

| Symptom | Likely cause | Fix |
|---|---|---|
| Zero events, `last_seen_at` empty | Snippet not installed, wrong site key, builder site not republished, caching/optimization plugin stripping it. | `get_install_snippet`; customer pastes it in `<head>` and republishes; check with `?tf_debug=1`. |
| Zero events, `last_seen_at` set | New sites track **nothing** by default: no tags, `auto_pageview` off. Or tags exist but are unpublished. Or consent mode waiting for consent. | Build tags / turn on the wanted setting; `get_container` → publish; check the CMP integration. |
| Only `blocked`, reason Bot traffic | Testing from a headless browser, script, uptime monitor. | Test in a normal browser. |
| `blocked`, Page reload | Normal: reload PageViews are dropped (`block_reloads`). | None. |
| `blocked`, No marketing consent | Consent mode on and visitor declined / custom integration sends `c:false`. | Expected; check the banner integration. |
| `blocked`, Rule: … | Blocking rule (office IP, staging URL, event name). | `list_blocking_rules`; disable if wrong. |
| `no_destination` | No enabled destination; or server-side events (webhook/CRM/offline/test) while every destination is browser-only or misses server fields; or test events with no test code. | Add destination / server credentials (`list_destinations` → `missing`), mode `both`; for tests set a test code. |
| `browser_only` | Destinations in browser mode. | Fine, unless server-side (CAPI) is wanted → add server fields, mode `both`. |
| `failed`, Meta HTTP 400 | Invalid/expired token, pixel id mismatch, token without access, event too old, bad parameter. | Read `get_event` response; `update_destination`; `retry_event_delivery`. |
| `failed`/`partial`, HTTP 0 | Timeout/network. | Retry; if persistent, check the endpoint field. |
| Many `pending` | Delivery in progress or stuck. | Re-check after a minute; `get_event`. |
| `delivered` but platform shows nothing live | Test code (`test_event_code`) or `validate_only` left on the destination: events go to Test Events / are not ingested. Or the event name isn't used by the platform's conversion settings. | Remove the test field with `update_destination`. |
| Low match quality | No email/phone on conversions; forms not passing `{{form_user}}`; webhooks/CRM without email mapping; server events not stitched to a visitor; phones without country code. | Add `user_data`, `auto_forms`, `identify`, webhook mappings, `decorate_checkouts`, country code. |
| Purchases without value | Value variable empty on that theme (selector/JSON-LD missing), dataLayer without `value`/items, webhook `value_path` missing. Missing currency defaults to USD. | Fix the variable (fallbacks, `transform:"price"`), check in preview with `get_event`; set currency. |
| Duplicate purchases | Hard-coded pixel also firing; thank-you tag without `event_id` (reloads); `event_id` template with an empty variable (random id); browser id ≠ webhook id; two tags for one action; `dedup` setting off. | Remove duplicate source; `event_id: "purchase_{{order_id}}"` with a resolving order id; align ids with the webhook; keep `dedup` on. |
| Conversions in TrackFlow ≠ Ads Manager | Different attribution windows/models, view-through, platform-side dedup, test events, conversion list differs. | Explain; compare like for like; check `get_conversion_events`. |
| Conversions count looks wrong in TrackFlow | `conversion_events` list doesn't contain the event (only Purchase + Lead by default), or sample data loaded. | `get_conversion_events` / `set_conversion_events` (reporting only); `clear_site_data` only with confirmation. |
| Preview events appear in Meta browser reports | Browser pixels fire for preview events too. | Use a platform test code during preview, or a staging site. |
| Tag change not visible | Draft unpublished, or config cache (~1 min). | `get_container` → `publish_version` (if approved) → reload after a minute. |
| Webhook purchases not attributed to ads | No vid passthrough and email unknown from the browser. | `decorate_checkouts` (+ publish) or pass `trackflow('vid')`; capture email earlier. |
