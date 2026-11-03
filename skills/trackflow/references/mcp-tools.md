# TrackFlow MCP tool reference

Complete reference for the TrackFlow MCP server: connection, auth, how grouped tools are called, every action with its parameters, what it returns, and what it changes. Tool names, parameter names, enums and error texts are copied from the server.

---

## 1. Connecting

| Item | Value |
|---|---|
| Endpoint | `https://trackflow.webanalyticssolution.com/mcp` |
| Transport | MCP Streamable HTTP, **POST only**, plain JSON responses (no SSE stream). `GET /mcp` → `405 {"error":"Use POST (MCP streamable HTTP)"}` |
| Protocol version | Server answers with the client's `protocolVersion`, default `2025-06-18` |
| JSON-RPC methods | `initialize`, `ping`, `tools/list`, `tools/call`. Batches (JSON arrays) are accepted. Notifications get `202`. Other methods → error `-32601 Method not found` |
| Request body limit | 256 KB per request (matters for `import_offline_conversions` CSV text) |
| Server instructions | `initialize` returns an `instructions` string (how to work, secrets policy, ask-first rules). Clients should show it to the model |

### Authentication (two ways)

1. **API key** — header `Authorization: Bearer YOUR_API_KEY` (or `X-API-Key: YOUR_API_KEY`). Keys look like `tf_live_…`, are created in the dashboard under **Connect AI → API keys** (or with `account → create_api_key`), and give full access to the whole account (no scopes). Keys do not expire; they stop working only when revoked.
2. **OAuth 2.1 "Sign in with TrackFlow"** — for MCP clients that support remote-server auth (Claude, Cursor, mcp-remote…). Fully automatic: the client gets a 401, discovers the endpoints, registers itself, opens the browser, the customer signs in (or signs up) and clicks **Allow**.

Without valid credentials `/mcp` answers HTTP **401** with:

```
WWW-Authenticate: Bearer resource_metadata="https://trackflow.webanalyticssolution.com/.well-known/oauth-protected-resource/mcp", scope="mcp"
```
```json
{"jsonrpc":"2.0","id":null,"error":{"code":-32001,"message":"Sign in to TrackFlow to connect (your MCP client opens the sign-in page), or send an API key from TrackFlow → Connect AI → API keys."}}
```

### OAuth discovery (only what a client needs)

| Endpoint | Purpose |
|---|---|
| `GET /.well-known/oauth-protected-resource` (also `…/mcp`) | `{resource: "<origin>/mcp", authorization_servers: ["<origin>"], scopes_supported: ["mcp"], bearer_methods_supported: ["header"]}` |
| `GET /.well-known/oauth-authorization-server` (also `…/mcp`, `/.well-known/openid-configuration`) | issuer, `authorization_endpoint` `/oauth/authorize`, `token_endpoint` `/oauth/token`, `registration_endpoint` `/oauth/register`, `revocation_endpoint` `/oauth/revoke` |
| `POST /oauth/register` | Dynamic client registration (RFC 7591). Public clients only (`token_endpoint_auth_method: "none"`). `redirect_uris`: 1–10 URIs, each https, `http://localhost`/`127.0.0.1`/`[::1]` (any port), or an app scheme (`cursor://…`) |
| `GET /oauth/authorize` | `response_type=code` only; **PKCE `S256` required** |
| `POST /oauth/token` | `grant_type=authorization_code` (code valid 5 min, single use) or `refresh_token` |
| `POST /oauth/revoke` | Revokes the connection (access + refresh token) |

Facts that matter:
- The OAuth **access token is an ordinary API key** (`tf_live_…`, shown in Connect AI as "`<client name>` (signed in)"). No `expires_in` is returned.
- Refreshing **rotates** both tokens: the old access token stops working immediately.
- If the key is deleted in the dashboard, refresh fails with `invalid_grant` "This connection was removed. Sign in again."
- The read-only public demo account cannot authorize an OAuth connection (the consent page sends it to the login page).

---

## 2. How the tools are exposed

`tools/list` returns **9 group tools**. Each group bundles several operations ("actions"):

| Group tool | Actions |
|---|---|
| `sites` | list_sites, get_site, create_site, update_site, delete_site, clear_site_data, load_sample_data |
| `reports` | get_overview, get_event_flow, list_events, get_event, get_attribution, list_journeys, get_journey, get_match_quality, send_test_event, retry_event_delivery |
| `destinations` | list_platforms, list_destinations, add_destination, update_destination, test_destination, remove_destination, google_ads_sign_in_link |
| `tag_manager` | get_tagging_reference, inspect_site, list_variables, create_variable, update_variable, delete_variable, list_tag_triggers, create_tag_trigger, update_tag_trigger, duplicate_tag_trigger, delete_tag_trigger, list_tags, create_tag, update_tag, delete_tag, list_recipes, apply_recipe, update_container_settings, get_container |
| `publishing` | create_preview_link, list_preview_links, delete_preview_link, publish_version, list_versions, rollback |
| `conversions` | get_conversion_events, set_conversion_events, delete_conversion_event, restore_conversion_event |
| `integrations` | list_checkout_providers, list_checkout_integrations, create_checkout_integration, update_checkout_integration, delete_checkout_integration, test_checkout_payload, get_checkout_log, list_crm_providers, list_crm_integrations, create_crm_integration, update_crm_integration, delete_crm_integration, test_crm_payload, sync_crm_integration, get_crm_log, list_webhook_integrations, create_webhook_integration, update_webhook_integration, test_webhook_payload, delete_webhook_integration, import_offline_conversions, list_offline_imports, list_offline_sources, create_offline_source, update_offline_source, sync_offline_source, delete_offline_source |
| `install` | get_install_snippet, get_proxy_config, domain_status, add_domain, verify_domain, remove_domain, update_first_party_settings, list_blocking_rules, create_blocking_rule, update_blocking_rule, delete_blocking_rule |
| `account` | get_account, update_account, list_api_keys, create_api_key, revoke_api_key |

95 actions in total.

### Call shape

```json
{"name": "tag_manager", "arguments": {"action": "create_tag", "site_id": 12, "name": "Thank-you → Lead", "trigger": {"type": "page_view"}, "conditions": {"var": "page_path", "op": "contains", "value": "/thank"}, "once": "page", "actions": [{"type": "send_event", "event_name": "Lead"}]}}
```

- `action` (required) picks the operation; all other parameters go **at the top level** next to it.
- Alternatively put them in `params: {…}`; nested `params` values override top-level ones.
- `{"action": "help", "of": "<action>"}` returns that action's full description and JSON schema. Omit `of` to get every action of the group. Use it whenever unsure.
- **Legacy names:** each action name also works as a tool name by itself (`tools/call` with `name: "list_sites"`), although it is not listed in `tools/list`.
- Results come back as one `text` content item holding pretty-printed JSON. Failures come back as a normal result with `isError: true` and the error text (not a JSON-RPC error), except an unknown tool name, which is JSON-RPC error `-32602 Unknown tool: <name>. Tools: sites, reports, …`.

### Argument validation (what the server checks)

| Check | Error text |
|---|---|
| Unknown action | `Unknown action "<x>" for <group>. Actions: …` |
| Missing required parameter (null or `""` counts as missing) | `<group> → <action> needs <names>. Parameters: {…schema…}` |
| Unknown parameter — **only for actions whose schema is closed** (see below) | `<group> → <action> does not take <names>. Parameters: <list>` |
| `help` with an action of another group | `"<x>" is not an action of <group>. Actions: …` |

**Closed schemas (unknown parameters rejected):** every action of `sites`, `reports`, `conversions`, `account`, the blocking-rule and simple-webhook actions, `list_preview_links`, `delete_preview_link`, and `google_ads_sign_in_link`. Consequences:
- Do **not** pass `site_id` to account-level actions with no parameters: `list_sites`, `get_account`, `list_api_keys`, `create_api_key`, `revoke_api_key`, `update_account` → error "does not take site_id".
- Do **not** pass `from`/`to` to tools that do not list them; a misspelt key is an error, not silently ignored.

**Open schemas (extra keys not rejected by the MCP layer):** `tag_manager`, `publishing` (except the two above), `destinations` (except `google_ads_sign_in_link`), checkout/CRM/offline actions, `install` domain/snippet actions. Extra keys there are either ignored or forwarded to the API, which validates them (e.g. a variable type with an unknown field → `Variable type dom_text has no field "x". Fields: …`).

Types, enums, minimum/maximum are **not** validated by the MCP layer; the API validates or clamps (e.g. `days` is clamped to 1–365, `limit` to 1–200). A wrong type usually falls back to a default silently (e.g. `from: "2026-10-01"` is not a number → the report silently uses `days`).

### Conventions

- `site_id` (integer) comes from `sites → list_sites`. `site_key` (`tf_…`) is the public key used in the install snippet and the server-side REST ingest; it is not the same as `site_id`.
- Report windows: `days` (1–365; default 7, or 30 for `get_attribution` and `list_journeys`) **or** an exact range `from` + `to` = **Unix timestamps in milliseconds**, `from < to`, `to` exclusive, at most 366 days (a longer range is cut to the last 366 days). Both must be given; otherwise `days` is used.
- Errors from the underlying API are returned as `<message> (HTTP <status>)`, e.g. `Site not found (HTTP 404)`.
- **Secrets in results are masked** (any key ending in secret, password, token, api_key… shows `••••••` + last 4 characters; webhook/preview `token` fields stay readable because they are part of URLs). One-time values the customer must use are returned in full: a new API key, webhook URLs, the install snippet, a proxy config.
- The public **demo account is read-only**: every write returns `The demo workspace is read-only. Create a free account to make changes. (HTTP 403)`.

---

## 3. Recommended call order

1. `sites → list_sites` → pick `site_id` (or `create_site`). Check `last_seen_at`: empty = tag not installed → `install → get_install_snippet` (Shopify: mode `shopify_pixel` for Customer Events).
2. Turn on only the automatic tracking the customer asked for: `sites → update_site settings {auto_pageview | auto_forms | ecommerce_datalayer}` (all **off** on new sites).
3. Understand the site: `tag_manager → inspect_site` (home, product, checkout, thank-you, contact pages), `get_tagging_reference`, `list_tags` / `list_tag_triggers` / `list_variables`.
4. Build in the **draft**: `create_variable` → `create_tag_trigger` (or inline triggers) → `create_tag`, or `apply_recipe`.
5. Connect platforms: `destinations → list_platforms` → `add_destination` → `test_destination` (Google Ads server uploads: `google_ads_sign_in_link`).
6. Server-side sources if needed: `integrations → create_checkout_integration` / `create_crm_integration` / `create_webhook_integration` / `import_offline_conversions` / `create_offline_source`. Refunds and cancellations from them reach a destination only after `update_destination config.send_lifecycle: true` (only when the customer asks).
7. Test: `publishing → create_preview_link` → customer opens it → `reports → list_events {test: "only"}` → `get_event`.
8. Publish when the customer wants it live: `publishing → publish_version` (never if they said they want to review first; then hand over the preview URL).
9. Verify live: `reports → list_events` / `get_event` / `get_event_flow`; fix credentials → `retry_event_delivery`.
10. Report: `get_overview`, `get_attribution`, `list_journeys`, `get_match_quality`, `conversions → get_conversion_events`.

---

## 4. Read-only vs write vs live-affecting

| Class | Actions |
|---|---|
| **Read-only** (no changes) | list_sites, get_site, get_overview, get_event_flow, list_events, get_event, get_attribution, list_journeys, get_journey, get_match_quality, list_platforms, list_destinations, get_tagging_reference, inspect_site (fetches a page of the customer's site), list_variables, list_tag_triggers, list_tags, list_recipes, get_container, list_preview_links, list_versions, get_conversion_events, all list_*/get_*_log in integrations, list_checkout_providers, list_crm_providers, test_checkout_payload, test_crm_payload, test_webhook_payload, import_offline_conversions with `dry_run: true`, get_install_snippet, get_proxy_config, domain_status, list_blocking_rules, get_account, list_api_keys |
| **Draft-only writes** (nothing changes on the site until `publish_version`) | create/update/delete_variable, create/update/duplicate/delete_tag_trigger, create/update/delete_tag, apply_recipe, update_container_settings |
| **Preview only** | create_preview_link (draft runs only in the browser that opens the link; events flagged test) |
| **Live immediately — website tracking** | publish_version, rollback, update_site (settings reach visitors within ~a couple of minutes), add/update/remove_destination (browser tags load / stop loading on next page view), update_first_party_settings (`rotate_paths` breaks the installed stealth/custom-domain snippet; `stealth`/`global` change the JS global name), remove_domain, delete_preview_link, create/update/delete_blocking_rule |
| **Live immediately — sends data to ad platforms** | retry_event_delivery (re-sends a stored event with current credentials), import_offline_conversions (without dry_run), sync_offline_source, create_offline_source (first pull runs automatically within about a minute, then every `interval_minutes`), sync_crm_integration, create/update_crm_integration with `api_token` (automatic polling starts), create_checkout_integration / create_crm_integration / create_webhook_integration (their URLs accept real webhooks right away) |
| **Sends test events** (platform test mode only) | send_test_event, test_destination |
| **Reporting-only writes** (never change what platforms receive) | set_conversion_events (also re-marks past events), delete_conversion_event, restore_conversion_event, load_sample_data |
| **Destructive / irreversible** — ask the customer first | delete_site (needs `confirm_domain`), clear_site_data (needs `confirm_domain`), remove_destination (deletes saved credentials), delete_checkout_integration, delete_crm_integration, delete_webhook_integration, delete_offline_source, remove_domain, revoke_api_key (current key needs `confirm_current: true`), rollback (by default discards unpublished draft edits). Prefer `enabled: false` to deleting |
| **Account** | update_account (name only), create_api_key (returns the key once) |

---

## 5. Group `sites`

### list_sites
Params: none (do not pass `site_id`).
Returns `{sites: [{id, name, domain, site_key, platform, settings{…}, last_seen_at, created_at}]}`. `last_seen_at` = last time the installed tracker loaded or sent a browser event (ms); empty = not installed.

### get_site
Params: `site_id`* (integer).
Returns `{site}` with all settings: `consent_mode, auto_pageview, auto_forms, ecommerce_datalayer, block_bots, block_reloads, dedup, cross_domains, conversion_events, allow_custom_js`, `site_key`, `last_seen_at`, `shopify_pixel_seen_at` and `shopify_pixel_detected` (true once the Shopify Customer Events code from `get_install_snippet` mode `shopify_pixel` has sent events from at least 3 different visitors within the last 14 days; the theme tag then leaves the events that code sends to it).
Error: `Site <id> not found. Use list_sites.`

### create_site
| Param | Type | Notes |
|---|---|---|
| `domain`* | string | Bare domain `example.com` (protocol/path stripped). Must match `^[a-z0-9.-]+\.[a-z]{2,}$` |
| `name` | string | Default: the domain (max 80) |
| `platform` | enum | `wordpress, woocommerce, shopify, webflow, framer, squarespace, wix, nextjs, custom` (default `custom`; only changes install instructions) |

Returns `{site}`. Defaults of a new site: `auto_pageview`, `auto_forms`, `ecommerce_datalayer` **off**, no tags; `dedup`, `block_bots`, `block_reloads`, `allow_custom_js` **on**; `consent_mode` off; `conversion_events: ["Purchase","Lead"]`.
Errors: `Enter a valid domain, e.g. example.com (HTTP 400)`; `You already have a site for example.com. Open it from the site switcher. (HTTP 409)`.

### update_site
| Param | Type | Notes |
|---|---|---|
| `site_id`* | integer | |
| `name` | string | Not empty |
| `domain` | string | Bare domain |
| `platform` | enum | as create_site |
| `settings` | object | Only the keys given change. Allowed keys below |

`settings` keys (all boolean except `cross_domains` and `default_currency`):

| Key | Meaning |
|---|---|
| `consent_mode` | Browser events only sent after marketing consent (`trackflow('consent', true)` or a detected CMP); blocked until then |
| `auto_pageview` | Automatic PageView on every load and SPA navigation, to every destination, shared event id |
| `auto_forms` | On form submit, read email/phone/name fields and attach them (hashed server-side) to the visitor for match quality |
| `ecommerce_datalayer` | Map GA4 ecommerce dataLayer pushes (view_item, add_to_cart, begin_checkout, purchase…) to standard events with value, products, order id |
| `block_bots` | Drop browser events from known bots |
| `block_reloads` | Ignore the PageView of a page reload |
| `dedup` | Store/send each `event_id` + event name once. Keep on |
| `allow_custom_js` | Run `custom_js` tag actions (default on). Off = every custom JS stops at once. Change only when the customer asks |
| `default_currency` | ISO 4217 code (e.g. `EUR`, `BDT`); used when an event has a value but no/invalid currency (else USD). Invalid codes are rejected. Set it for every non-USD shop |
| `cross_domains` | string array, e.g. `["checkout.example.net"]`; **replaces the whole list** |

Errors: `Use set_conversion_events to choose conversion events.` (if `settings.conversion_events` given); `Unknown setting(s): x. Allowed: …`; `Nothing to change: pass name, domain, platform or settings.`
Side effect: live for visitors within a couple of minutes (tracker config is cached briefly).

### delete_site
Params: `site_id`*, `confirm_domain`* (the site's domain exactly as `list_sites` shows it, after the customer confirmed).
Deletes the site with ALL data: events, journeys, destinations + credentials, tags, versions, integrations, domains. Installed tag stops working. Irreversible.
Error without/with wrong confirm: `Deleting a site cannot be undone. Ask the customer to confirm, then pass confirm_domain: "example.com" (the site's domain).`

### clear_site_data
Params: `site_id`*, `confirm_domain`*. Deletes every event and visitor journey (reports restart at zero). Settings, destinations, tags, rules, integrations are kept; nothing is removed from ad platforms. Irreversible.

### load_sample_data
Params: `site_id`*. Adds two weeks of sample events/journeys/conversions (never sent to platforms; they mix with real data in reports). Only on request; remove with `clear_site_data`.

---

## 6. Group `reports`

Shared params: `site_id`*; window `days` / `from` + `to` (see Conventions); `conv` = which conversion list decides "conversion" in this report: `"site"` (default) or a destination id as string (e.g. `"7"`) from `get_conversion_events.destinations`.

### get_overview
Params: `days` (default 7), `from`, `to`, `conv`.
Returns `site, conv, range{from,to,days,custom}, totals{events, sessions, visitors, conversions, revenue, refunds, net_revenue, cancelled_orders, delivered, failed, partial, blocked, undelivered, match}, prev{…previous period}, bucket, series[], top_conversions[], top_sources[], recent_events[], recent_conversions[], destinations[{id,type,pixel_id,enabled,has_token,mode,ready}]`. Business numbers exclude blocked and test events; status counts include everything. `refunds` = refunded amounts + cancelled order values of orders whose Purchase counted; `net_revenue` = revenue − refunds; `cancelled_orders` = distinct orders cancelled or charged back ([lifecycle-events.md](lifecycle-events.md)).

### get_event_flow
Params: `days` (default 7), `from`, `to` (both needed).
Returns sources (browser, server, webhook, offline, crm, api) → top event names → each destination with delivered / failed / skipped counts and last error. Counts use the latest server delivery attempt; browser tags are not logged server-side.

### list_events
| Param | Type | Notes |
|---|---|---|
| `name` | string | Exact event name |
| `status` | enum | `pending, delivered, partial, failed, blocked, no_destination, browser_only` |
| `origin` | enum | `browser, server, webhook, offline, crm, api` |
| `q` | string | Search URL, event id or source |
| `conversions_only` | boolean | Only events counted as conversions (by `conv`) |
| `conv` | string | as above |
| `test` | enum | `only` (preview/test events) or `exclude`; default both |
| `limit` | integer | 1–200, default 25 |
| `offset` | integer | ≥ 0 |

Returns `{events: [{id, event_id, event_name, origin, status, block_reason, url, source, device, value, currency, is_conversion, match_score, is_test, created_at}], total, names[], conv}`. Newest first.
Status meanings: `pending` delivery running; `delivered` all server destinations OK; `partial` some failed; `failed` all failed; `blocked` (bot, reload, no consent, blocking rule); `no_destination` nothing could be sent server-side (includes test events skipped by every destination); `browser_only` only browser tags are connected (handled in the browser, nothing server-side).

### get_event
Params: `site_id`*, `event_row_id`* = the numeric `id` from list_events (**not** the `event_id` string).
Returns `{event: {…, user_data (hashed), custom_data, click_ids}, deliveries: [{id, type, status, http_status, payload, response, created_at}]}` — the exact payload and platform response per destination. Delivery `status`: `ok`, `error`, `skipped` with the reason in `response` (e.g. test event on a destination without test mode, "Too old for <Platform> (max N days)", "Refunds and cancellations are off for this destination", Google Ads "Queued: … sending after <time>" — a queued delivery gets a second row when it is sent).

### retry_event_delivery
Params: `site_id`*, `event_row_id`*. Re-sends the stored event to every enabled destination with current settings; returns `{ok, status}`. **Live send** (a test event stays a test event). Blocked events: `Event cannot be resent (HTTP 400)`.

### get_attribution
Params: `days` (default 30), `from`, `to`, `model` (`last` default | `first`), `conv`.
Returns `{model, conv, totals{revenue, refunds, net_revenue, cancelled_orders}, sources[{source, sessions, visitors, conversions, revenue, refunds, net_revenue, cancelled_orders}], campaigns[] (top 20, same extra fields), by_event[], devices[], landing[] (top 10)}`. Refunds and cancellations are attributed to the source/campaign of the order's Purchase. Excludes blocked and test events.

### list_journeys
Params: `days` (default 30), `from`, `to`, `all` (boolean: include non-converters), `conv`.
Returns up to 100 `journeys[{vid, first_seen, last_seen, first_source, last_source, first_campaign, landing_url, device, converted, events, sessions, conversions, revenue, first_conv_at}]`.

### get_journey
Params: `site_id`*, `vid`* (visitor id from list_journeys or get_event).
Returns `{visitor{…, identifiers: ["em","ph",…]} (names only, never values), events[…]}`. 404 `Visitor not found`.

### get_match_quality
Params: `days` (default 7), `from`, `to`.
Returns `{events: [{name, n, score (0–10), coverage{em:%, ph:%, fbc:%, fbp:%, external_id:%…}}]}` over the latest 5,000 real (non-test, non-blocked) events. Score is TrackFlow's own estimate (weights: em 3.0, ph 2.2, fbc 1.6, external_id 1.1, fbp 0.9, ip 0.6, ua 0.6, fn/ln 0.4, zp 0.3 …, max 10).

### send_test_event
Params: `site_id`*, `event_name` (default `Lead`), `value` (number; currency USD).
Sends one server event (email `test@example.com`) through the full pipeline, flagged **test**. Returns ingest result `{ok, event_id, match_score, test:true}` (or `blocked`/`duplicate`).
Gotcha: test events are delivered **only to destinations that have a test mode configured** (Meta/TikTok/Snapchat `test_event_code`, Google Ads/OpenAI/Snapchat `validate_only`, Pinterest `test_mode`, Reddit `test_id`, GA4/Quora `debug`, Nextdoor `test_event`…). All others record `skipped` ("Test event: set this destination's test code / test mode to send test events (they are never sent live)"). Check with list_events / get_event.

---

## 7. Group `destinations`

Platform ids (from `list_platforms`): `ga4, google_ads, linkedin, meta, microsoft, nextdoor, openai, outbrain, pinterest, quora, reddit, snapchat, taboola, tiktok, x`. Always read `list_platforms` for the exact field keys.

Modes: `both` (browser tag + server API, recommended), `browser` (tag only, no server secret), `server` (server API only, no tag). Field `need` levels: `always`; `browser` (needed in both|browser); `server` (needed in both|server); `null` = optional (see its `why`). Only ask the customer for the fields the chosen mode needs.

### list_platforms
Params: none. Returns `{platforms: [{id, name, category, docs, browser_tag, server_api, modes, default_mode, dedup{key, note}, google_sign_in?{available, redirect_uri, note}, fields: [{key, label, need, required, section, why?, secret, browser, help, placeholder, filled_by_sign_in?, options?, type?}]}]}`. Every platform has the advanced checkbox `send_lifecycle` ("Send refunds and cancellations", default false); its `why` says exactly what that platform receives ([lifecycle-events.md](lifecycle-events.md)).

### list_destinations
Params: `site_id`*. Returns `{destinations: [{id, site_id, type, name, pixel_id, config (secrets masked), access_token (masked), has_token, test_event_code, mode, missing[labels], missing_keys[], ready, load_browser_pixel, enabled, browser, server, created_at}]}`.

### add_destination
| Param | Type | Notes |
|---|---|---|
| `site_id`* | integer | |
| `type`* | string | Platform id, e.g. `"meta"` |
| `config`* | object | Field keys exactly as in list_platforms, e.g. `{"pixel_id":"YOUR_PIXEL_ID","access_token":"YOUR_ACCESS_TOKEN"}` |
| `mode` | enum | `both, browser, server` (must be one of the platform's modes). Omitted: `both` when server fields are given, else `browser` |
| `enabled` | boolean | default true |
| `load_browser_pixel` | boolean | Legacy; prefer `mode` (false = `server`) |

Returns `{destination, next_step}`. `next_step` examples: `Ready. Run test_destination to prove the server credentials work.`; `Saved, but not ready: missing … for mode "both". Send the customer the google_ads_sign_in_link to connect Google.`; `Browser-only: TrackFlow loads the tag; nothing is sent server-side…`.
Errors: missing fields → API message plus `Ask the customer for: <Label> → config.<key> (<help>); … Their modes: both, browser, server.`; bad mode → `<Platform>: mode must be "both" or "browser" or "server".`; unknown type → `Unknown platform. Choose one of: …`.
Live effect: browser tag loads on the next page view (modes both|browser); server delivery starts for the next event.
`config.send_lifecycle` (refunds, cancellations, renewals, no-shows) stays off unless the customer asks for it.

### update_destination
Params: `site_id`*, `destination_id`*, `config`, `mode`, `enabled`, `load_browser_pixel`.
Rules: only config keys passed change; omit a secret or pass `""` to **keep** the saved secret; `""` on a non-secret field clears it; `null` on a secret deletes it (`refresh_token: null` disconnects Google sign-in). Switching `mode` requires that mode's fields. `config.event_filter` (`all` | `only` | `except`) + `config.event_list` (event names) choose which events this destination receives, on both the browser pixel and server delivery (excluded server deliveries are skipped with `Not in this destination's event list`; `only` with an empty list is rejected; lifecycle events are not filtered). `config.send_lifecycle: true/false` switches sending lifecycle events on/off (live at once; changes what the platform reports and optimises on, so only on the customer's request). Error `Destination <id> not found on site <site>. Use list_destinations.`

### test_destination
Params: `site_id`*, `destination_id` (optional: only report this one), `event_name` (default `Lead`), `value` (USD).
Sends a test event through the pipeline (**every** enabled destination receives it, in test mode only) and waits up to ~12 s. Returns `{event_id, event_name, match_score, event_row_id, status, destination{id,type,name,enabled,server_ready,mode}, deliveries[{type,status,http_status,response,payload}], warning?/note?}` or `status: "pending"` with `hint: "Delivery is still running — call get_event later."`.
Gotcha: set the destination's test code / test mode first (see send_test_event), otherwise delivery is `skipped`. Paused, browser-only, or not-ready destinations come back with an explaining `warning`/`note`.

### remove_destination
Params: `site_id`*, `destination_id`*. Permanently deletes it and its credentials. Ask first; prefer `update_destination enabled=false`.

### google_ads_sign_in_link
Params: `site_id`*, `destination_id`* (a Google Ads destination). Closed schema.
Returns `{url, note}`. The customer opens the link, signs in with a Google account that can access the Ads account and clicks Allow; TrackFlow stores the refresh token. **Single use, expires in 15 minutes.** Afterwards `list_destinations` shows `has_token: true` → run `test_destination`. If the destination uses its own OAuth client, its Google Cloud project needs the Data Manager API and the redirect URI from the note (`redirect_uri_mismatch` = missing URI).

---

## 8. Group `tag_manager` (draft container)

Everything here edits the **draft**. Nothing reaches visitors until `publishing → publish_version` (or a preview link). Write responses include `warnings` such as `Unknown variable "x" (create it or use a built-in).` — the item still saves; fix it.

### get_tagging_reference
Params: none. Returns the vocabulary: `variable_types`, `variable_common_fields`, `builtins`, `trigger_types`, `form_presets`, `conditions`, `action_types`, `triggers`, `tag_fields`, `container_settings`, `canonical_custom_data`, `recipes`. Read before building.

### inspect_site
Params: `site_id`*, `url` (page on the site; default home page).
Fetches the page server-side and returns `{status, url, title, platform[], existing_tags{gtm, ga4, meta_pixel, tiktok_pixel, trackflow, google_ads?, shopify_pixels?{meta, tiktok, custom_pixels}}, forms[{selector, id, action, method, builder, kind (search|newsletter|login|add_to_cart|checkout|lead), external, fields[], submit_text}], embedded_forms[], ctas[{tag, text, href, selector, provider}], json_ld_types, json_ld_product, meta, microdata_price, datalayer{pushes, events, gtag_events, ecommerce}, checkout_links[{provider, href}], iframes[], videos, suggestions[]}`.
On Shopify pages `shopify_pixels` lists the Meta pixel ids, TikTok pixel codes and the number of custom pixels found in the store's web-pixel config (channel apps), and `suggestions` recommend the Customer Events code and warn about those duplicates.
Limits: only the site's domain, its subdomains and `cross_domains`; http(s) only; private IPs refused; 12 s timeout, max 5 redirects, 3 MB. Server-rendered HTML only (content injected by JavaScript is not seen). Errors: `Only pages on example.com (or its subdomains / cross-domain list) can be inspected.`, `The page returned HTTP 404`.

### Variables
| Action | Params | Notes |
|---|---|---|
| `list_variables` | `site_id`* | `{items:[{id, name, type, …fields, notes, recipe, updated_at}]}` |
| `create_variable` | `site_id`*, `name`*, `type`*, `config` (type fields; may also be top level), `transform`, `default`, `fallback`, `notes` | Name `^[A-Za-z_][A-Za-z0-9_]{0,63}$`, not a built-in name. 409 if the name exists |
| `update_variable` | `site_id`*, `variable`* (id or name), `name`, `type`, `config`, `transform`, `default`, `fallback` | Fields merge into config; `null` removes a field. Changing `type` starts a fresh config |
| `delete_variable` | `site_id`*, `variable`* | 404 `Variable not found` |

Variable `type` enum: `dom_text, dom_attr, dom_value, url, cookie, js_var, local_storage, session_storage, datalayer, json_ld, meta_tag, open_graph, constant, regex, lookup, element_count, click_element, form_field, page_title, referrer, user_agent, stored, event_data, calc`. Required fields: dom_text `selector`; dom_attr `selector, attribute`; dom_value `selector`; cookie `cookie`; js_var `path`; local/session_storage `key`; datalayer `key`; json_ld `field`; meta_tag `meta`; open_graph `property`; constant `value`; regex `source, pattern`; lookup `source, map`; element_count `selector`; form_field one of `field|label|selector`; stored `bag`; calc `op, a, b`. `transform`: `trim, lowercase, uppercase, number, int, price, currency, digits, count, first, json` (string or list). Example: `{"name":"price","type":"dom_text","selector":".product .price","transform":"price"}`.

Built-ins (no need to create): `page_url, page_path, page_host, page_query, page_hash, page_title, referrer, user_agent, language, event, event_name, event_data, click_text, click_url, click_id, click_classes, click_element_tag, form_id, form_name, form_action, form_classes, form_signature, form_preset, form_label, form_user, form_field_count, scroll_percent, video_status, video_percent, video_title, video_url, video_provider, message_origin, visible_percent, timer_count, error_message, error_source, file_extension, link_host, history_source, copy_text, vid, sid, product_name, product_id, product_price, product_currency`.

### Triggers (reusable, like GTM)
| Action | Params | Notes |
|---|---|---|
| `list_tag_triggers` | `site_id`* | `{items:[{id, name, trigger{type,…}, notes, recipe, used_by[{id,name}], updated_at}]}` |
| `create_tag_trigger` | `site_id`*, `name`* (1–120, unique), `trigger`* `{type, …fields}`, `notes` | 409 if the name exists |
| `update_tag_trigger` | `site_id`*, `trigger_id`* (id or exact name), `name`, `trigger` (replaces definition), `notes` | **Changes every tag using it** |
| `duplicate_tag_trigger` | `site_id`*, `trigger_id`* | Copy named `<name> (copy)` |
| `delete_tag_trigger` | `site_id`*, `trigger_id`* | 409 `Used by N tag(s): …` while any tag uses it |

Trigger `type`: `page_view, history_change, dom_ready, window_loaded, click (selector|text|target required; text_match contains|equals|starts_with|regex; target any|link|button), form_submit (selector, preset auto|native|<preset id(s)>, wait_for auto|none|request|element|text|url_change, success_selector, success_text, success_url, request_url, timeout_ms default 10000), element_visible (selector*, threshold 0–1, repeat, min_ms), custom_event (name*, source any|dom|datalayer; name may be "/regex/"), timer (interval_ms* ≥ 1000, limit), scroll_depth (percents, default [25,50,75,90]), video (providers, statuses start|progress|pause|complete, percents), iframe_message (preset calendly|cal|typeform|tally|jotform|hubspot, origin, contains), copy (selector), file_download (extensions), outbound_link (exclude), error (contains)`. Unknown fields are rejected: `Trigger click has no field "x". Fields: …`.
Form presets: `cf7, elementor, gravity, wpforms, ninja, fluent, formidable, hubspot, webflow, framer, squarespace, wix, typeform, jotform, tally, calendly, cal`.

### Tags
| Action | Params | Notes |
|---|---|---|
| `list_tags` | `site_id`* | `trigger` returned resolved as `[{trigger_id, trigger_name, type, …fields}]` |
| `create_tag` | `site_id`*, `name`* (≤120), `trigger`*, `actions`*, `conditions`, `once` (`""`, `page`, `session`, `visitor`), `enabled`, `priority` (higher runs first), `notes` | |
| `update_tag` | `site_id`*, `tag_id`*, any of the above | Only given fields change; `trigger`/`conditions`/`actions` are replaced as a whole |
| `delete_tag` | `site_id`*, `tag_id`* | Prefer `update_tag enabled=false` |

- `trigger`: one item or a list; each item is `{trigger_id: N}` (saved trigger) or an inline `{type, …fields, trigger_name?}`. Inline items are saved as reusable triggers (an identical saved definition is reused). Any trigger in the list fires the tag.
- `conditions`: leaf `{var, op, value, case_sensitive?}` or `{all:[…]}` / `{any:[…]}` / `{not:…}`; an array means all. Ops: `equals, contains, starts_with, ends_with, regex, gt, gte, lt, lte, exists, in, matches_css`, prefix `not_` to negate. Aliases accepted: `eq, =, starts, ends, >, <, >=, <=, matches`. Case-insensitive by default.
- `actions` (run in order):

| Action type | Required | Optional |
|---|---|---|
| `send_event` | `event_name` (cut to 64) | `custom_data`, `user_data`, `is_conversion` (ignored for reports), `event_id` (template, e.g. `"purchase_{{order_id}}"`, makes thank-you pages idempotent) |
| `store` | `bag` (1–40 `[\w-]`) | `values`, `storage` (`"session"`), `ttl_minutes` (default 4320), `append`, `clear`, `replace` |
| `identify` | `user_data` | |
| `set_variable` | `name`, `value` | `persist: "session"` |
| `custom_js` | `code` (≤ 20,000 chars; body of `function(tf, event)`) | |

- Templates: any string may use `{{variable}}`; `"{{x}}"` alone keeps the raw type; dot paths allowed (`{{event_data.payload.email}}`).
- `custom_data` canonical keys: `value, currency, content_ids, contents[{id, quantity, item_price}], content_type, content_name, content_category, num_items, order_id, search_string, status, predicted_ltv`. `user_data` keys: `em, ph, fn, ln, ct, st, zp, country, external_id` or `"{{form_user}}"`.
- Errors: `Tag name is required.`, `trigger is required, e.g. {"type":"page_view"}.`, `actions must be a non-empty array, e.g. [{"type":"send_event","event_name":"Lead"}].`, `once must be "", page, session or visitor.`, `Unknown action type "x". Use one of: …`, `Action send_event has no field "x". Fields: …`, `Trigger "N" does not exist on this site (see list_tag_triggers).`

Example (form lead with match-quality data):
```json
{"action":"create_tag","site_id":12,"name":"Contact form → Lead","trigger":{"type":"form_submit","selector":"#contact-form"},"actions":[{"type":"send_event","event_name":"Lead","user_data":"{{form_user}}"}]}
```

### Recipes
| Action | Params | Notes |
|---|---|---|
| `list_recipes` | `site_id`* | `{items:[{id, title, description, options}]}` |
| `apply_recipe` | `site_id`*, `recipe`* (`ecommerce, lead_form, booking, newsletter`), `options` (object) | Adds variables + tags to the draft. Re-applying updates the recipe's own items. Existing same-name variables not owned by the recipe are kept unless `options.overwrite: true`. Returns `{recipe, variables_created/updated/kept, tags_created/updated, settings?, next}` |

Options: **ecommerce** — `currency` (fallback, default USD), `add_to_cart_selector`, `price_selector`, `product_path_regex`, `checkout_path_regex`, `thank_you_path_regex`, `total_selector`, `thank_you_total_selector`, `order_id_selector`, `place_order_selector` (ViewContent, AddToCart, InitiateCheckout, Purchase with `event_id purchase_<order id>`). **lead_form** — `event_name` (default Lead), `form_selector`, `exclude_regex`, `wait_for`, `success_selector`, `success_text`. **booking** — `event_name` (default Schedule); Calendly/Cal.com embeds, also turns on checkout decoration for `calendly, cal`. **newsletter** — `event_name` (default Lead), `form_selector`.

### update_container_settings
Params: `site_id`*, and any of:
- `decorate_checkouts`: `true`, `false`, or a list of `stripe, paypal, lemonsqueezy, gumroad, thrivecart, samcart, paddle, calendly, cal, typeform, jotform, tally, hubspot, zoho, gohighlevel, hidden_input`. Appends the TrackFlow visitor id to third-party checkout/booking/form links, embeds (Stripe Buy Button / Pricing Table `client-reference-id`, Typeform `data-tf-hidden`, Calendly inline widget) and iframes so their webhooks match the ad click.
- `decorate_reload_iframes`: boolean (default false). True also rewrites provider iframes that had already loaded (reloads them); otherwise only iframes added later / lazy `data-src` are decorated.
- `shopify_events`: extra Shopify Customer Events: `true` (all of `ViewCategory, ViewCart, AddShippingInfo`), a list of some, or `false`/`null` (none). The funnel events follow `ecommerce_datalayer` / published tags.
Draft settings → need `publish_version`.

### get_container
Params: `site_id`*, `which` (`draft` default | `published`). Draft returns `{which, container{vars, tags, settings}, published_version, unpublished_changes, has_custom_js, allow_custom_js}`. Use `unpublished_changes` to know whether a publish is pending.

---

## 9. Group `publishing`

| Action | Params | Returns / effect |
|---|---|---|
| `create_preview_link` | `site_id`*, `url` (page on the site/subdomain/cross-domain; default home), `hours` (default 2, clamped 0.25–24) | `{token (pv_…), url (…?tf_preview=TOKEN), expires_at, custom_js_runs, note}`. Opening the URL loads the **draft** for that browser session and flags every event `test`. `?tf_preview=off` leaves preview. Error: `Preview links only work on example.com (and its subdomains or cross-domains).` |
| `list_preview_links` | `site_id`* | Active links `{items:[{token, url, created_at, expires_at, custom_js_runs}]}` |
| `delete_preview_link` | `site_id`*, `token`* | Revokes immediately |
| `publish_version` | `site_id`*, `notes` (≤500) | **Live**: draft becomes a new version; visitors get it on next page load. Returns `{version, id, tags, variables, has_custom_js, custom_js_runs}`. Publish only when the customer asked for / approved going live |
| `list_versions` | `site_id`* | Up to 100 `{items:[{id, version, notes, live, tags, variables, has_custom_js, via (dashboard\|api\|system), created_at}], allow_custom_js}` |
| `rollback` | `site_id`*, `version`* (version **number** from list_versions), `notes`, `restore_draft` (default true) | **Live**: creates a new version with the old content. By default also resets the draft to it (unpublished edits lost; pass `restore_draft: false` to keep them). Returns `{ok, version, restored_from, draft_restored}`. 404 `Version not found` |

Custom JavaScript is published like everything else and runs while the site setting `allow_custom_js` is on; disabled tags are not published.

---

## 10. Group `conversions` (TrackFlow reporting only)

Never changes what platforms receive. Standard names: `Purchase, Lead, CompleteRegistration, Contact, Schedule, StartTrial, Subscribe, SubmitApplication, InitiateCheckout, AddPaymentInfo, AddToCart, AddToWishlist, CustomizeProduct, Donate, FindLocation, Search, ViewContent, PageView`.

| Action | Params | Notes |
|---|---|---|
| `get_conversion_events` | `site_id`* | `{selected, events[{name, conversion, standard, count, count_30d, last_at}], hidden, lifecycle_events[{name, count, count_30d, last_at}] (never conversions), destinations[{id, type, name, pixel_id, enabled, selectable, note, selected, events[{name, conversion, sent, failed, skipped, last_at, configured}]}]}` |
| `set_conversion_events` | `site_id`*, `events`* (full list, ≤200 names, each ≤64), `destination_id` | Without `destination_id`: replaces the site list and **re-marks past events**. With it: that destination's list (reports with `conv=<id>`). Google Ads is not selectable: `Google Ads decides which conversions count (Primary or Secondary)…`. Lifecycle names (`OrderRefunded`, `OrderCancelled`, `SubscriptionRenewed`, `SubscriptionCancelled`, `BookingCancelled`, `BookingNoShow`) are dropped and returned in `ignored` + `note`. Returns the list plus `changed` |
| `delete_conversion_event` | `site_id`*, `name`* | Custom names only (`<name> is a standard event and cannot be deleted. Switch it off instead.`). Removes from every list and hides it |
| `restore_conversion_event` | `site_id`*, `name`* | Un-hides; does not re-add to any list |

---

## 11. Group `integrations`

Common: create/update return the item with a `webhook_url` to give the customer; secrets are masked (`has_secret`, `has_api_key`/`has_api_token` booleans). For checkout and CRM updates, **omitting** `secret`/`api_key`/`api_token` (or sending back the masked value) keeps the saved one; sending `""` **clears** it (for a checkout integration that switches signature verification off), even though the tool text says "blank keeps". `config`, when passed, replaces the saved config as a whole, so send the full object. `integration_id` comes from the matching list action. These schemas are open (extra keys are forwarded and mostly ignored).

### Checkout webhooks (verified, order dedup, lifecycle)
Providers: `stripe, paypal, shopify, lemonsqueezy, paddle, gumroad, thrivecart, samcart, woocommerce, calendly, razorpay, clickfunnels, systeme, teachable, kajabi, whop`. Per-provider secrets, events and passthrough: [integrations.md](integrations.md) §2.

| Action | Params | Notes |
|---|---|---|
| `list_checkout_providers` | none | `{purchase_on:[…], providers:[{id, name, docs, secret_label, api_label, signed, events, purchase_stages, passthrough}]}` — read first; tell the customer to select exactly `events` |
| `list_checkout_integrations` | `site_id`* | Items with `webhook_url, purchase_on, signature_verification (on \| off \| secret check \| not verified (secret URL only)), received, events_created, last_received_at, last_error, passthrough, events, purchase_stages` |
| `create_checkout_integration` | `site_id`*, `provider`*, `name`, `secret`, `api_key`, `config` | Webhook URL is **live at once**. For providers that issue the secret only after the webhook exists (Stripe, Shopify…): create, give URL, then add `secret` via update. Unsigned SamCart/Teachable/Kajabi: optional shared secret that the URL must carry as `?secret=` |
| `update_checkout_integration` | `site_id`*, `integration_id`*, `name`, `secret`, `api_key`, `config`, `enabled` | `config` replaces the saved one: send the full object (e.g. `{"purchase_on":"created_cod", …}`) |
| `delete_checkout_integration` | `site_id`*, `integration_id`* | URL stops working. Ask first |
| `test_checkout_payload` | `site_id`*, `integration_id`*, `payload`* (object), `headers` (topic headers: `{"x-shopify-topic":"refunds/create"}`, `{"x-wc-webhook-topic":"order.updated"}`, `{"x-webhook-event":"SALE_NEW"}` for systeme.io) | Dry run: no signature check, nothing stored. `{results:[{status: would_create\|would_skip\|ignored\|duplicate, order_id, event_name, event_id, lifecycle?, value, currency, partial?, remaining_value?, original_value?, vid, matched_by (vid\|order\|provider_ref\|email\|phone\|landing_url\|none), action_source, identifiers, click_ids, event_time, purchase_on?, reason?}]}`: one result per event (Purchase or lifecycle) |
| `get_checkout_log` | `site_id`*, `integration_id`* | Last 100 deliveries: `ok, ignored, duplicate, rejected, error` with `verified`, `delivery_id`, an `events` summary and per-event results (incl. lifecycle events, visitor match, provider redeliveries) |

`config` keys used: `purchase_on` (`paid` default \| `created` = payment authorized or paid \| `completed` = fulfilled/completed \| `created_cod` = at order creation even unpaid; other values rejected), `event_name` (default Purchase; Calendly Schedule), `accept_test` (track test-mode payments), `order_id_field`, `default_currency`, `default_country`, `value_divisor`, `value`, `renewals` + `renewal_event_name` (legacy Stripe renewals as a conversion event), `subscription_event_name`, `paid_statuses`, `payment_intents`, `track_invoice_payments` (Razorpay), `sandbox`, `event_types` (SamCart), `vid_field`.
Behaviour: event id `purchase_<order id>` (other event names: `<lowercased name>_<order id>`; Calendly `calendly_<invitee uuid>`) — the same id the dataLayer bridge, the thank-you tag and the Shopify Customer Events code use, so browser + webhook count once. Purchase `event_time` = time of the `purchase_on` stage. Refunds, cancellations, chargebacks, renewals, subscription cancellations and booking cancellations/no-shows become `OrderRefunded` / `OrderCancelled` / `SubscriptionRenewed` / `SubscriptionCancelled` / `BookingCancelled` / `BookingNoShow` (never conversions; delivered only to destinations with `send_lifecycle`; [lifecycle-events.md](lifecycle-events.md)). **Without a `secret` signature verification is off** and any POST is accepted. Orders with no visitor id, email, phone, IP or click id are ignored. Test-mode payments are ignored unless `accept_test: true`.

### CRM integrations
Providers: `hubspot, gohighlevel, pipedrive, salesforce, zoho, generic`.

| Action | Params | Notes |
|---|---|---|
| `list_crm_providers` | none | `{providers:[{id, name, docs, secret_label, api_label, pollable, default_stage_map}], mapping_fields}` |
| `list_crm_integrations` | `site_id`* | Items with `webhook_url, pollable, received, events_created, last_polled_at, last_error` |
| `create_crm_integration` | `site_id`*, `provider`*, `name`, `secret`, `api_token`, `config` | `api_token` enables polling + contact lookups (HubSpot, Pipedrive, GoHighLevel; GoHighLevel also needs `config.location_id`) |
| `update_crm_integration` | `site_id`*, `integration_id`*, `name`, `secret`, `api_token`, `config`, `enabled` | `config` replaces the saved config as a whole (`stage_map` must be an object) |
| `delete_crm_integration` | `site_id`*, `integration_id`* | Webhook + polling stop; created events stay. Ask first |
| `test_crm_payload` | `site_id`*, `integration_id`*, `payload`* (object/array, or XML string for Salesforce) | Dry run: `{ok, records, results:[{status: would_create\|ignored, stage, event_name, event_id, value, lead_id, vid, matched_by, identifiers, reason?}]}` |
| `sync_crm_integration` | `site_id`*, `integration_id`* | Polls now (needs api_token). **Creates live events**. Error e.g. `HubSpot polling needs an API token`, `Sync already running` |
| `get_crm_log` | `site_id`*, `integration_id`* | Last 100 webhooks/polls |

`config` keys: `stage_map` (`{"<stage>": "EventName"}` or `{"<stage>": {"event_name":"Purchase","value":true,"is_conversion":true}}`; keys are matched lowercase-alphanumeric only, so `"Closed Won"` = `closedwon`), `keep_defaults` (merge with defaults instead of replacing them), `mapping` (`email_path, phone_path, first_name_path, last_name_path, name_path, city_path, state_path, zip_path, country_path, value_path, currency_path, stage_path, object_id_path, lead_id_path, vid_path, external_id_path, event_time_path`, plus `value_divisor`), `event_name` (used when the payload has no stage), `include_value`, `default_currency`, `default_country`, `action_source` (default `system_generated`), `poll` (false disables), `poll_minutes` (min 5, default 15), `backfill_days`, `location_id` (GoHighLevel), `company_domain` (Pipedrive), `public_url`.
Default stage map (all providers): `won`/`closedwon` → Purchase with value; `qualified`/`qualifiedlead`/`qualification` → QualifiedLead. HubSpot adds `qualifiedtobuy`, `salesqualifiedlead`, `marketingqualifiedlead` → QualifiedLead, `appointmentscheduled` → Schedule.
Behaviour: event id `crm_<integration id>_<object id>_<stage>`; a record without email, phone, external id, Meta lead id or visitor id is ignored. Meta lead ids found in the payload are sent for Conversions API for CRM. Polling starts from the integration's creation time (older deals only with `backfill_days`).

### Mapped ("custom") webhooks
Providers (label only, except `cal`): `stripe, calendly, typeform, shopify, woocommerce, cal, tally, generic`.

| Action | Params | Notes |
|---|---|---|
| `list_webhook_integrations` | `site_id`* | Items with `webhook_url`, `received`, `last_received_at`, `signature` (scheme), `secret` (masked) / `secret_set`, `verified` (false = anyone with the URL can post), `warnings` |
| `create_webhook_integration` | `site_id`*, `event_name`*, `provider` (default generic), `name`, `mapping`, `signature`, `secret`, `enabled` | Closed schema. Accepted POSTs become `event_name` (or a lifecycle event via `status_map`). 400 when a Purchase webhook has no `mapping.order_id_path`. Relay `warnings` |
| `update_webhook_integration` | `site_id`*, `integration_id`*, same fields | `mapping` and `signature` replaced as a whole; `secret` omitted keeps, `""` removes; URL unchanged |
| `test_webhook_payload` | `site_id`*, `integration_id`*, `payload`* (object, or the raw JSON string to check an HMAC), `headers`, `mapping` (try an unsaved mapping) | Dry run: `{signature{ok, verified, scheme, reason}, ignored?, reason?, events[{kind, status, event_name, event_id, event_time, vid, matched_by, custom_data, user_data_keys, click_ids, url, conversion, duplicate?, would}], note}`. Nothing stored or sent |
| `delete_webhook_integration` | `site_id`*, `integration_id`* | URL returns 404 afterwards |

`mapping` (closed object, all optional dot paths): `email_path, phone_path, name_path, first_name_path, last_name_path, value_path, value_divisor (number, 100 for cents), currency_path, default_currency, order_id_path, event_id_path, event_id_prefix, status_path, status_map, refund_amount_path, refund_id_path, vid_path (= visitor_path), landing_url_path, time_path, url_path, fbc_path, fbp_path, filter_path, filter_value`. **`order_id_path` is required** for new Purchase webhooks and whenever `status_path` / `status_map` / `refund_amount_path` is used (event ids `purchase_<order id>`, `refund_<order id>_…`, `cancel_<order id>`). `status_map` values: `purchase, refund, cancel, chargeback, ignore` (needs `status_path`). `provider: "cal"` turns Cal.com `BOOKING_CANCELLED` / `BOOKING_NO_SHOW_UPDATED` into `BookingCancelled` / `BookingNoShow`.
`signature`: `{scheme: trackflow (default) | hmac | header_secret | basic | none, header, algorithm (sha256|sha1), encoding (hex|base64), signed (body | timestamp.body), timestamp_header, prefix}`; `hmac`, `header_secret`, `basic` need a `secret` (8–200 chars). Details: [integrations.md](integrations.md) §5. Prefer checkout integrations for payment providers and CRM integrations for CRMs.

### Offline conversions
| Action | Params | Notes |
|---|---|---|
| `import_offline_conversions` | `site_id`*, `csv`* (text with header row), `mapping` (`{field: "Header"}`), `defaults` (`{event_name, currency, action_source, country, date_format: mdy\|dmy}`), `dry_run` | `dry_run: true` validates only. Without it **events are created and sent**. Returns `{import_id, mapping, summary{rows_total, accepted, rejected, duplicates, flagged, matched}, rows[problem rows]}`. Max 50,000 rows; via MCP the whole request must stay under 256 KB |
| `list_offline_imports` | `site_id`*, `limit` (default 20, max 200) | Import history with row errors |
| `list_offline_sources` | `site_id`* | Scheduled sources with `csv_url`, cursor, `last_status`, `last_error` |
| `create_offline_source` | `site_id`*, `url`* (Google Sheets link or published CSV URL, must be public by link), `name`, `mapping`, `defaults`, `interval_minutes` (15–1440, default 60) | **First pull runs automatically within about a minute** |
| `update_offline_source` | `site_id`*, `source_id`*, `name`, `url`, `mapping`, `defaults`, `interval_minutes`, `enabled`, `reset_cursor` | Changing `url` or `reset_cursor: true` re-reads from row 1 (rows with an order id are skipped as duplicates) |
| `sync_offline_source` | `site_id`*, `source_id`* | Pulls now; imports new rows (live) |
| `delete_offline_source` | `site_id`*, `source_id`* | Imported events stay. Ask first |

Fields (columns auto-detected): `email, phone, name, first_name, last_name, city, state, zip, country, external_id, event_name, value, currency, order_id, event_time, action_source, lead_id, vid, gclid, gbraid, wbraid, fbc, fbp, fbclid, ttclid, msclkid`. `action_source`: `physical_store` (default), `phone_call, system_generated, email, chat, website, app, other`.
Validation: needs an identifier (email, phone, external id, lead id, visitor id or click id); `Purchase needs a value`; 3-letter currency; event time not in the future and **≤ 90 days old** (warnings: Meta accepts ≤ 7 days, ≤ 62 for physical_store). Event id `offline_<order_id>` (re-imports dedup); rows without order id get a content hash id.

---

## 12. Group `install`

| Action | Params | Notes |
|---|---|---|
| `get_install_snippet` | `site_id`*, `mode` (`auto` default, `custom_domain`, `proxy`, `stealth`, `standard`, `shopify_pixel`) | `{mode, src, global, html, notes[]}`. `html` = the exact code for `<head>`. `auto` = custom domain if active, else stealth path. With stealth on the JS API is `window.<global>(…)` and `window.trackflow` is not defined. `shopify_pixel` returns `{mode, src, code, notes}`: `code` is plain JavaScript for Shopify admin → Settings → Customer events → Add custom pixel (loads from the custom domain when live; re-copy after adding one). The notes say which settings it currently sends |
| `get_proxy_config` | `site_id`*, `platform`* (`nginx, apache, cloudflare, vercel, nextjs, netlify, wordpress`) | Ready-to-install same-origin proxy config (contains the per-site proxy secret: hand it over as a whole, do not quote the secret alone) |
| `domain_status` | `site_id`*, `host` (optional filter) | `{settings{stealth, global, path_prefix, paths, proxy_secret (masked), cookies}, dns_target, domains[{id, host, status, cookie_domain, verified_via, dns, records, alternative_records, cert_expires_at, last_error, message}], edge, tracker}` |
| `add_domain` | `site_id`*, `host`* (subdomain of the site domain, e.g. `data.example.com`) | Returns the ONE CNAME record to create (Cloudflare: DNS only / grey cloud); `alternative_records` (A + TXT) only if CNAME is impossible. Max 5 per site. Errors: `Use a subdomain of example.com …`, `Use a dedicated subdomain such as data.example.com, not the site itself`, `This hostname is already connected (409)` |
| `verify_domain` | `site_id`*, `host` or `domain_id` | Checks DNS now; `pending_dns` → `verified` → `active` (TLS issued automatically, usually 2–5 min). Pending domains are also re-checked in the background |
| `remove_domain` | `site_id`*, `host` or `domain_id` | Tracking through that host stops. Switch the snippet first. Ask first |
| `update_first_party_settings` | `site_id`*, `stealth` (bool), `global` (identifier without tracking words, e.g. `qx7`), `rotate_paths` (bool), `rotate_secret` (bool) | Needs at least one key. `rotate_paths` breaks installed stealth/custom-domain snippets; `rotate_secret` breaks existing proxy configs; changing `stealth`/`global` breaks on-page code calling the old global |
| `list_blocking_rules` | `site_id`* | Rules with `hits` |
| `create_blocking_rule` | `site_id`*, `type`*, `pattern`*, `note`, `enabled` (default true) | Live at once, all sources. Blocked events are stored with status `blocked`, never delivered |
| `update_blocking_rule` | `site_id`*, `rule_id`*, `type`, `pattern`, `note`, `enabled` | |
| `delete_blocking_rule` | `site_id`*, `rule_id`* | Prefer `enabled: false` |

Blocking `type`: `ip` (exact, wildcard `203.0.113.*` or IPv4 CIDR `203.0.113.0/24`), `url` (wildcard `*/staging/*`, or substring when no `*`), `user_agent` / `referrer` (case-insensitive substring), `event` (exact event name).

Domain flow for ad-blocker resistance: `add_domain` → customer creates the CNAME → `verify_domain` until `active` → `get_install_snippet {mode:"auto"}` → customer replaces the old tag.

---

## 13. Group `account`

| Action | Params | Notes |
|---|---|---|
| `get_account` | none | `{account{id, name, email, plan, is_demo, created_at}, sites, api_keys}` |
| `update_account` | `name`* (max 80) | Password/email cannot be changed here |
| `list_api_keys` | none | `{keys:[{id, name, prefix, last_used_at, created_at, current}]}` — never the key itself |
| `create_api_key` | `name` (max 60, default "API key") | Returns the full `key` **once** (only a hash is stored). Use it for another AI client, a server integration, or the REST ingest API |
| `revoke_api_key` | `key_id`*, `confirm_current` | Immediate. Revoking the key this connection uses needs `confirm_current: true`, otherwise: `This is the key this AI connection uses: revoking it disconnects TrackFlow from this assistant. Pass confirm_current=true only if the customer explicitly wants that.` |

---

## 14. Common errors and fixes

| Error text (start) | Cause / fix |
|---|---|
| `Site N not found. Use list_sites.` / `Site not found (HTTP 404)` | Wrong `site_id` or another account's site |
| `… does not take site_id` | Account-level action; drop `site_id` |
| `… does not take from, to` | That report has no range parameters (check `help`) |
| `… needs site_id` / `needs <field>` | Required parameter missing or empty |
| `The demo workspace is read-only…` (403) | Demo account; customer must create a real account |
| `Unknown setting(s): …` | Only the listed site settings can be changed with update_site |
| `Use set_conversion_events to choose conversion events.` | Conversion list is not a site setting here |
| `… cannot be undone. Ask the customer to confirm, then pass confirm_domain: "…"` | delete_site / clear_site_data need the domain typed back after confirmation |
| `<Platform>: … required` + `Ask the customer for: …` | Destination fields missing for the chosen mode |
| `Destination N not found on site M. Use list_destinations.` | Wrong id |
| deliveries `skipped` "Test event: set this destination's test code / test mode…" | Add the platform's test field before testing |
| `Unknown variable "x" (create it or use a built-in).` (warning) | Template references a missing variable |
| `Used by N tag(s): …` (409) | Trigger still in use |
| `A variable/trigger named "x" already exists; update it instead.` (409) | Use the update action |
| `Preview links only work on example.com (and its subdomains or cross-domains).` | Preview URL must be on the site |
| `Webhook integration not found on this site. List them first to get a valid id.` | Wrong integration id |
| `mapping.order_id_path is required for purchase, refund and cancellation webhooks…` | Map the order id (e.g. `"order.id"`) |
| `mapping.status_map needs mapping.status_path…` / `status_map values must be purchase, refund, cancel, chargeback or ignore` | Fix the status mapping |
| `The hmac signature scheme needs a secret…` / `signature.header is required for hmac…` | Pass `secret` and the header the app signs in |
| `config.purchase_on must be one of: paid, created, completed, created_cod` | Checkout integration config |
| deliveries `skipped` "Refunds and cancellations are off for this destination" | Expected while `send_lifecycle` is off; turn on only on request |
| deliveries `skipped` "Too old for <Platform> (max N days)" | Event older than the platform accepts; nothing to fix for that event |
| HTTP 401 on `/mcp` | Missing/revoked key or expired OAuth connection: reconnect |
