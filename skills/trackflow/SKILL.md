---
name: trackflow
description: Set up, audit and fix website conversion tracking with TrackFlow (trackflow.webanalyticssolution.com) through its MCP server or REST API. Covers the install tag, Meta Pixel + Conversions API, TikTok, Google Ads, GA4, Snapchat, Pinterest, LinkedIn and other ad platforms, browser + server deduplication, event parameters, the built-in tag manager, first-party tracking domains, checkout/CRM/offline conversions, refunds and cancellations, visitor-id passthrough, CMS installs (Shopify Customer Events, WordPress/WooCommerce, Wix, Webflow, Squarespace, Next.js) and tracking audits. Use whenever the user mentions TrackFlow, its MCP tools, or asks to track conversions, pixels or CAPI on a site connected to TrackFlow.
---

# TrackFlow

TrackFlow runs a website's conversion tracking: one browser tag plus server-side delivery to ad and
analytics platforms (Meta, TikTok, Google Ads, GA4, LinkedIn, Snapchat, Pinterest, Reddit, X, Microsoft,
and more). Every event gets one shared `event_id`, so the browser pixel copy and the server (Conversions
API) copy are deduplicated by the platform.

TrackFlow is **AI-first**. The customer talks to you in plain language, like instructing an employee, and
you do the work end to end with the tools. Only things *outside* TrackFlow are the customer's job:
pasting the install snippet into their site, creating one DNS record, or registering a webhook URL in
Stripe or a CRM. For those, tell them exactly what to paste and where, in short numbered steps.

## 1. Connect

| | |
|---|---|
| MCP server (Streamable HTTP) | `https://trackflow.webanalyticssolution.com/mcp` |
| Sign-in | OAuth 2.1 (the app opens a TrackFlow sign-in page; dynamic client registration, PKCE) |
| Or API key | Header `Authorization: Bearer <API key>`. Keys are created in the dashboard: **Connect AI → API keys**. |
| Setup guides per AI app | Dashboard → **Connect AI** (`/app/connect-ai`) |

If no TrackFlow tools are available, tell the user to open **Connect AI** in the TrackFlow dashboard,
pick their AI app and follow the steps there. Never ask them for their TrackFlow password.

### How the tools are called

Tools are **grouped**. Call a group tool with an `action` plus that action's parameters:

```json
tag_manager  {"action": "create_tag", "site_id": 12, "name": "Lead - contact form", ...}
reports      {"action": "list_events", "site_id": 12, "test": "only", "limit": 20}
destinations {"action": "help", "of": "add_destination"}
```

| Group | Actions (summary) |
|---|---|
| `sites` | list_sites, get_site, create_site, update_site, delete_site, clear_site_data, load_sample_data |
| `reports` | get_overview, get_event_flow, list_events, get_event, get_attribution, list_journeys, get_journey, get_match_quality, send_test_event, retry_event_delivery |
| `destinations` | list_platforms, list_destinations, add_destination, update_destination, test_destination, remove_destination, google_ads_sign_in_link |
| `tag_manager` | get_tagging_reference, inspect_site, variables, triggers, tags, list_recipes, apply_recipe, update_container_settings, get_container |
| `publishing` | create_preview_link, list_preview_links, delete_preview_link, publish_version, list_versions, rollback |
| `conversions` | get_conversion_events, set_conversion_events, delete_conversion_event, restore_conversion_event |
| `integrations` | checkout webhooks (stores, payments, courses, bookings), CRMs, mapped webhooks (+ `test_webhook_payload`), offline conversions (CSV / sheets) |
| `install` | get_install_snippet, get_proxy_config, domain_status, add_domain, verify_domain, remove_domain, update_first_party_settings, blocking rules |
| `account` | get_account, update_account, list_api_keys, create_api_key, revoke_api_key |

- `{"action": "help", "of": "<action>"}` returns an action's full description and JSON schema. **Call it
  before the first use of any write action** you have not used in this conversation.
- Missing required fields are rejected. The `sites`, `reports`, `conversions` and `account` actions,
  blocking rules, mapped webhooks, `list_preview_links`, `delete_preview_link` and
  `google_ads_sign_in_link` also reject unknown or misspelled parameters with an error that lists the
  valid ones; the other actions ignore or forward extra keys. Read the error, fix the call, retry
  once. Account-level actions (`list_sites`, `get_account`, `list_api_keys`, `create_api_key`) do
  **not** take `site_id`.
- Parameter *types* are not strictly checked: a wrong type can silently fall back to a default. Date
  ranges are `from` / `to` in **Unix milliseconds** (`to` exclusive, max 366 days) or `days`; an ISO
  date string is ignored and the default window is used.
- `get_event` takes `event_row_id` = the numeric `id` from `list_events`, not the `event_id` string.
- The old per-operation tool names (`list_sites`, `create_tag`, …) still work when called directly,
  but `tools/list` shows only the 9 group tools.
- Full parameter reference: [references/mcp-tools.md](references/mcp-tools.md).

## 2. Golden rules

1. **New sites track nothing.** `auto_pageview`, `auto_forms` and `ecommerce_datalayer` are off and there
   are no tags. Switch on only what the customer asked for. Never enable tracking they did not want.
2. **Inspect before you build.** Run `inspect_site` on each relevant page (home, product, cart, checkout,
   thank-you, contact). It reports forms, buttons, product data, the dataLayer, existing pixels and
   checkout providers. Base every tag on what is really on the page.
3. **One source per event.** If the site already has a Meta/TikTok/Google pixel (theme, plugin, app,
   GTM) that fires the same events, the platform will count them twice. Tell the customer and remove or
   disable the old events before you publish, or keep the old pixel and don't add the same events here.
4. **Draft → preview → publish.** Tag-manager changes go into a draft. Nothing reaches visitors until
   `publish_version`. Publish when the customer asked for the change (or approved it). If they want to
   review first, give them a `create_preview_link` URL and a short summary, and don't publish.
   `rollback` makes any earlier version live again and, by default, resets the draft to it
   (`restore_draft: false` keeps unpublished edits).
5. **Know what is live immediately.** Tag-manager changes wait for `publish_version`, but these act at
   once: `update_site`, adding/updating/removing destinations, blocking rules, `rollback`,
   `update_first_party_settings` (`rotate_paths` breaks the installed snippet), new integration webhook
   URLs. These **send to ad platforms right away**: `retry_event_delivery`, `sync_crm_integration`,
   `import_offline_conversions` (use `dry_run: true` first), `create_offline_source` (first pull within
   about a minute), and a CRM integration with an `api_token` (starts polling).
6. **Test every destination.** After `add_destination` run `test_destination`. After publishing, verify
   real events with `list_events` / `get_event` (exact payloads and the platform's HTTP response).
   Test events (preview links, `send_test_event`) reach **only destinations with a test field set**
   (Meta `test_event_code`, Google Ads `validate_only`, GA4 `debug`, …); others record them as skipped.
   While a test field is set, **all** server events of that destination go to test mode, so remove it
   before go-live. Headless/automated browsers are blocked as bots: test in a normal browser.
7. **Real values only.** Purchases need a numeric `value`, an ISO-4217 `currency` and a unique
   `order_id`. A value without a valid ISO currency gets the site's `default_currency` (set it with `update_site`), else **USD**. Values are numbers in major units
   (`129.90`, not `"$129.90"` or `12990`). Use the event id `purchase_<order id>` on every source
   (tag, dataLayer, checkout integration) so one order counts once. Never hard-code a value. See
   [references/event-parameters.md](references/event-parameters.md).
8. **No PII in the wrong place.** Email, phone and names go only in user-data fields (TrackFlow hashes
   them server-side). Only user data is hashed: custom properties are forwarded **as sent**, so PII
   there reaches the platform unhashed. Never put PII in event names, custom properties or URLs.
9. **Secrets.** Customers paste Pixel IDs, access tokens, API secrets and webhook secrets into the chat.
   Pass them straight into the tool fields. Never repeat a full secret back; tool results mask them, so
   confirm only the last 4 characters. One-time values the customer must use (new API key, webhook
   URLs, install snippet, proxy config) are returned in full: hand those over.
10. **Refunds and cancellations are opt-in per destination.** Store and checkout webhooks create
   `OrderRefunded`, `OrderCancelled`, `SubscriptionRenewed`, … automatically; they never count as
   conversions and only reach a platform when its `send_lifecycle` is on (off by default). Turning it on
   changes what the platform reports and optimises on: explain the per-platform effect and only do it
   when the customer asks ([references/lifecycle-events.md](references/lifecycle-events.md)).
11. **Per-platform event choice without touching tags.** To send different events to different
   platforms ("everything to Meta, only Purchase and Lead to TikTok"), set the destination's
   `event_filter` (`only` / `except`) and `event_list` with `update_destination`. It applies to the
   browser pixel and the server copy; excluded deliveries show `Not in this destination's event list`.
12. **Server orders need the visitor id.** A webhook sale is credited to the ad click only when TrackFlow
   can find the visitor (`matched_by`: `vid` > `order` > `provider_ref` > `email` > `phone` >
   `landing_url` > `none`). Set up the passthrough for every server source
   ([references/visitor-id.md](references/visitor-id.md)).
13. **Ask before you destroy.** `delete_site` and `clear_site_data` cannot be undone (they need
   `confirm_domain`). Removing a destination, integration, domain or API key stops data or access.
   Confirm first unless the customer asked for exactly that; prefer `enabled: false` to deleting.
14. **Plain language.** The customer is usually not technical. Explain what you did and what it means
    for their ads ("Facebook will now see every purchase with its amount"), not tool names.

## 3. Setup workflow (summary)

Full step-by-step version with checklists: [references/setup-playbook.md](references/setup-playbook.md).

1. **Find or create the site.** `sites → list_sites` (or `create_site` with `domain` and `platform`).
   `get_site` shows settings and whether the tag has been seen (`last_seen_at`).
2. **Install the tag** if `last_seen_at` is empty: `install → get_install_snippet` and give the customer
   the exact steps for their CMS ([references/install-cms.md](references/install-cms.md)).
   Stealth is on by default, so modes `auto` / `stealth` / `custom_domain` / `proxy` return a tag whose
   JavaScript function has a **random name** (the `global` field in the result) and `window.trackflow`
   does not exist. Use that name in any on-page code, or use mode `standard` to keep `trackflow(...)`.
   Event names are case-sensitive Meta-style standard names (`Purchase`, `Lead`, `AddToCart`); TrackFlow
   translates them for every other platform.
   **Shopify:** the main install is the Customer Events code (`get_install_snippet` mode `shopify_pixel`,
   pasted under Settings → Customer events → Add custom pixel); it covers checkout and thank-you pages,
   sends what `auto_pageview` / `ecommerce_datalayer` / `shopify_events` allow, and `get_site` shows
   `shopify_pixel_detected`. The theme tag is optional (custom click/form tags, cart attributes). With both
   installed, each page view is still sent once and funnel events come only from the pixel.
3. **Understand the site:** `inspect_site` per page type, `get_tagging_reference`, then `list_tags`,
   `list_tag_triggers`, `list_variables` for what already exists.
4. **Agree the plan** in one or two sentences: which events, from which pages, to which platforms.
5. **Build in the draft.** Use the site settings for broad tracking (`auto_pageview`, `auto_forms`,
   `ecommerce_datalayer` via `update_site`), a recipe (`apply_recipe`: ecommerce, lead_form, booking,
   newsletter), or specific `create_variable` → `create_tag_trigger` → `create_tag`.
   ([references/tag-manager.md](references/tag-manager.md))
6. **Connect platforms** with the customer's credentials: `list_platforms` → `add_destination` →
   `test_destination`. ([references/ad-platforms.md](references/ad-platforms.md))
7. **Server-side sources** when the sale or lead happens off-site, and as a backup for store purchases:
   checkout webhooks (choose `purchase_on`: paid / created / completed / `created_cod` for cash on
   delivery; register the refund and cancel events too), CRM, mapped webhooks (`order_id_path`,
   `signature`, `test_webhook_payload`), offline conversions
   ([references/integrations.md](references/integrations.md)). Pass the visitor id through
   ([references/visitor-id.md](references/visitor-id.md)); decide `send_lifecycle` per destination with
   the customer ([references/lifecycle-events.md](references/lifecycle-events.md)).
8. **Preview and test:** `create_preview_link` (events are marked test), then `list_events`
   with `test: "only"`, `get_event`, `send_test_event`.
9. **Publish** (`publish_version` with notes) as the customer instructed.
10. **Verify live** with `list_events` / `get_event`, fix and `retry_event_delivery` if a delivery failed.
11. **Optional hardening:** first-party tracking domain
    ([references/first-party-domain.md](references/first-party-domain.md)), blocking rules, consent mode,
    cross-domain funnels.
12. **Report back** in plain language: what is tracked, where it goes, what the customer still has to do.

## 4. Audit workflow (summary)

Full version with a symptom → cause → fix table:
[references/audit-playbook.md](references/audit-playbook.md) and
[references/reports-and-audit-data.md](references/reports-and-audit-data.md).

1. `get_site` (settings, tag seen?, Shopify `shopify_pixel_detected`), `list_destinations` (ready /
   missing fields / mode / test mode left on? `send_lifecycle` as agreed?).
2. `inspect_site` on key pages: duplicate pixels, missing dataLayer, forms and checkout providers.
3. `get_overview` and `list_events` (filter `status: "failed"`, `"no_destination"`, `"browser_only"`).
4. `get_event` on samples: parameters, user data, platform responses.
5. `get_match_quality`: which user-data fields are missing and how to raise them. Webhook sources:
   `get_checkout_log` `matched_by` mix, refund/cancel events arriving, `refunds` / `net_revenue` in
   `get_overview`, deliveries skipped as "Too old for <Platform>".
6. `list_tags` / `get_container` / `list_versions`: draft vs live, unused or duplicate tags.
7. Write findings as: problem → impact on ads → fix → who does it (you or the customer). Fix what
   the tools can fix once the customer agrees.

## 5. Reference map

| Need | File |
|---|---|
| Every MCP action, parameter and return value | [references/mcp-tools.md](references/mcp-tools.md) |
| HTTP API, server-side event ingest, auth | [references/rest-api.md](references/rest-api.md) |
| Browser tracker commands (`trackflow(...)`), SPA, cookies, click IDs | [references/tracker-js.md](references/tracker-js.md) |
| Install per CMS / framework, ecommerce events per platform | [references/install-cms.md](references/install-cms.md) |
| Event names, parameters, user data, what to send and what not to send | [references/event-parameters.md](references/event-parameters.md) |
| Per ad platform (and the `custom` webhook destination for any other platform): credentials, modes, gotchas | [references/ad-platforms.md](references/ad-platforms.md) |
| Tags, triggers, variables, recipes, versions, preview | [references/tag-manager.md](references/tag-manager.md) |
| Checkout webhooks (Purchase stage `purchase_on`), CRM, offline conversions, mapped webhooks, third-party tools | [references/integrations.md](references/integrations.md) |
| Linking server orders to the ad click: `matched_by`, `tf_vid` passthrough per checkout, WooCommerce/Stripe/PayPal/SSLCommerz/Razorpay/bKash/Nagad code | [references/visitor-id.md](references/visitor-id.md) |
| Refunds, cancellations, renewals, no-shows: `send_lifecycle`, what each platform does, max event age, net revenue, cash on delivery | [references/lifecycle-events.md](references/lifecycle-events.md) |
| Custom tracking domain, ad-blocker resistance, blocking rules | [references/first-party-domain.md](references/first-party-domain.md) |
| Reading reports, event statuses, match quality | [references/reports-and-audit-data.md](references/reports-and-audit-data.md) |
| Step-by-step setup checklist | [references/setup-playbook.md](references/setup-playbook.md) |
| Step-by-step audit checklist | [references/audit-playbook.md](references/audit-playbook.md) |

Read only the files the task needs. Before writing tags for a platform you have not used yet in this
conversation, read its section in `ad-platforms.md` and the event table in `event-parameters.md`.
