# Setup playbook: from zero to verified tracking

Follow the phases in order. Each phase lists the tool calls, what to check, and what to tell the customer.
Group tools are called as `{"action": "<action>", "site_id": N, ...}` (see [mcp-tools.md](mcp-tools.md)).

## Phase 0: intake (ask only what the tools cannot find out)

Ask in one short message, in the customer's language. Skip anything you already know.

| Ask | Why |
|---|---|
| Website address | `list_sites` / `create_site` |
| What counts as success: sales, leads/forms, bookings, sign-ups, calls? | Decides the events |
| Which ad platforms do you advertise on (Facebook/Instagram, TikTok, Google Ads, …)? | Decides the destinations |
| Where does payment or booking happen: on your site, or on Stripe/PayPal/Calendly/another domain? | Off-site → checkout integration or cross-domain |
| When should a sale count: when paid, when the order is placed (cash on delivery), or only when delivered? | `purchase_on` of the store/checkout integration |
| Should refunds and cancellations also be reported to the ad platforms? (Ask only when a store/checkout webhook is set up) | `send_lifecycle` per destination (off by default) |
| Is any Facebook/TikTok/Google tracking already installed (theme, plugin, app, GTM, sales channel)? | Duplicate-pixel risk |
| Do you need a cookie-consent banner (EU/UK visitors)? | `consent_mode` |
| Should I publish when done, or do you want to review first? | `publish_version` vs preview link |

Credentials (Pixel ID, access token, etc.) are asked for in Phase 4, per platform, with exact
"where to find it" steps from [ad-platforms.md](ad-platforms.md).

Never ask for the TrackFlow password, and never ask the customer to click through the TrackFlow
dashboard for something a tool can do.

## Phase 1: site and install

1. `sites → list_sites`. If the site is missing: `create_site {"name", "domain": "example.com", "platform"}`
   (`platform`: wordpress, woocommerce, shopify, webflow, framer, squarespace, wix, nextjs, custom).
2. `sites → get_site`. If `last_seen_at` is empty, the tag is not installed:
   - **Shopify:** the main install is the Customer Events code: `get_install_snippet {"mode":
     "shopify_pixel"}` → give the customer `code` and the steps from [install-cms.md](install-cms.md)
     §4.1 (Settings → Customer events → Add custom pixel → Save → Connect). Turn on what it should send
     (`auto_pageview`, `ecommerce_datalayer`; extras via `shopify_events`). Add the theme tag (`auto`) too
     only for custom click/scroll/form tracking (it also writes the cart attributes for the webhook).
     Done when `list_events` shows the store's events (`get_site` shows `shopify_pixel_detected: true` only after 3 different visitors, so a dev store with one tester may never show it; tracking doesn't depend on it).
   - Other sites: `install → get_install_snippet`. Choose the mode:
     - `standard`: classic tag, JS function `trackflow(...)`. Simplest for sites where you will add
       on-page code.
     - `auto` (default): stealth path or custom domain, **random JS function name** in the `global`
       field. Better against ad blockers. Use that name in every on-page snippet.
   - Give the customer the exact steps for their CMS from [install-cms.md](install-cms.md), with the
     returned `html` pasted verbatim. One tag per site, inside `<head>`, on every page.
   - After they confirm, check `get_site` again (`last_seen_at` set) and `reports → list_events`.
3. Run the duplicate-pixel check from [install-cms.md](install-cms.md) section 15.

## Phase 2: understand the site

1. `tag_manager → get_tagging_reference` once per session (vocabulary: variable types, triggers,
   operators, actions, recipes).
2. `tag_manager → inspect_site` with `url` for each page type: home, a product page, cart, checkout,
   thank-you/order-received, contact/lead page, booking page. Note:
   - forms (fields, builder, suggested selector),
   - CTA buttons and links,
   - product data (JSON-LD / OpenGraph),
   - dataLayer pushes (GA4 ecommerce events?),
   - existing tags/pixels,
   - checkout/payment links and their providers, iframes/embeds,
   - suggested recipes.
3. `list_tags`, `list_tag_triggers`, `list_variables`, `get_container`: what already exists. Don't
   rebuild something that exists; update it.

## Phase 3: plan, then build in the draft

State the plan in one or two sentences ("I'll track product views, add-to-cart, checkout and purchases
with the order total, and send them to Facebook and TikTok"). Then pick the building blocks:

| Business | What to build |
|---|---|
| Ecommerce with GA4 ecommerce dataLayer (inspect_site shows `view_item`, `add_to_cart`, `purchase` pushes) | `update_site settings.ecommerce_datalayer=true` (built-in bridge, Purchase id `purchase_<transaction_id>`). Don't also apply the `ecommerce` recipe. |
| Shopify | Customer Events code (Phase 1) with `ecommerce_datalayer=true` (+ `auto_pageview`): product, cart, checkout and purchase from Shopify's own data, `purchase_<order id>`. No ecommerce recipe. Order webhook as backup (Phase 5). |
| Ecommerce without a dataLayer (WooCommerce, custom) | `apply_recipe ecommerce` (pass selectors / `currency` as options), then check in preview that value, currency and order id resolve. WooCommerce: thank-you snippet + order webhook ([install-cms.md](install-cms.md) §3). |
| Lead generation (forms) | `apply_recipe lead_form` or specific `form_submit` tags; `auto_forms=true` to capture email/phone for match quality (`auto_forms` alone never sends a Lead). |
| Newsletter | `apply_recipe newsletter` (event `Lead`, or `CompleteRegistration` / `Subscribe` via `event_name`). |
| Bookings (Calendly / Cal.com) | `apply_recipe booking` (+ Calendly checkout integration for server-side confirmation). |
| Page views for retargeting | `update_site settings.auto_pageview=true` (also covers SPA route changes). |
| Button / link clicks, scroll, video, thank-you page | `create_tag` with the matching trigger. Cookbook in [tag-manager.md](tag-manager.md) section 10. |
| Hosted checkout on another domain | `update_site settings.cross_domains` + container setting `decorate_checkouts`, plus a checkout integration. |
| Forms or bookings in third-party iframes (Typeform, Jotform, Tally, HubSpot, Zoho, GoHighLevel) | `decorate_checkouts` with those providers + a hidden field `tf_vid` in the form, then the tool's webhook ([visitor-id.md](visitor-id.md)). |

Rules while building:
- Event names: Meta-style standard names, case-sensitive (`Purchase`, `Lead`, `AddToCart`,
  `InitiateCheckout`, `ViewContent`, `CompleteRegistration`, `Schedule`, `Contact`, `Subscribe`, …).
  TrackFlow maps them to every platform ([event-parameters.md](event-parameters.md)).
- Purchases: numeric `value`, ISO `currency`, `order_id`, event id `purchase_<order id>`.
- Read the `warnings` array in every create/update result (e.g. unknown variable names) and fix them.
- Build bottom-up: `create_variable` → `create_tag_trigger` (or inline trigger) → `create_tag`.

## Phase 4: connect the ad platforms

For each platform the customer uses:

1. `destinations → list_platforms` (field names, which fields each mode needs).
2. Ask the customer for exactly those values, with the "where to find it" steps from
   [ad-platforms.md](ad-platforms.md). Ask for one platform at a time.
3. `add_destination {"type": "meta", "mode": "...", "config": {...}}`.
   Mode: `both` = browser tag + server (best), `browser` only, or `server` only (when another pixel stays
   installed). Without `mode` it is derived from the fields you pass (`both` only when the server fields
   are there too). The result shows `ready` and `missing`.
4. `test_destination`: proves the server credentials work. Fix and repeat until it passes.
5. For a test run in the platform's own test tool, set its test field (Meta `test_event_code`, Google
   Ads `validate_only`, GA4 `debug`). **Write down that you did it**: it must be removed before go-live,
   because while it is set *all* server events of that destination go to test mode.
6. Google Ads: server uploads need a Google sign-in. After `add_destination`, send the customer the
   `google_ads_sign_in_link` URL (single use, 15 minutes); a self-made refresh token is the only
   alternative. Each Google Ads conversion needs its own conversion action/label mapping.

## Phase 5: server-side sources (when the sale or lead happens elsewhere)

See [integrations.md](integrations.md).

- **Store backup (Shopify, WooCommerce)**: always add the order webhook next to the browser Purchase. It
  recovers purchases the browser missed (closed tab, ad blocker, payment redirects) and reports refunds
  and cancellations. Shopify: topics `orders/create`, `orders/paid`, `orders/fulfilled`,
  `orders/cancelled`, `refunds/create`. WooCommerce: Order created + Order updated. Same event id
  `purchase_<order id>` as the thank-you page / Customer Events code, so nothing counts twice.
- **Hosted checkout / payment / course / booking platform** (Stripe, PayPal, Paddle, Lemon Squeezy,
  Gumroad, ThriveCart, SamCart, Razorpay, ClickFunnels 2.0, systeme.io, Teachable, Kajabi, Whop,
  Calendly): `list_checkout_providers` → ask for the provider's signing secret →
  `create_checkout_integration` with `config.purchase_on` (`paid` default; cash on delivery:
  `created_cod`; delivered-only: `completed`) → give the customer the returned `webhook_url` and
  **every** event in `events` (including refunds/cancellations) → `test_checkout_payload` with a real
  sample → `get_checkout_log`.
- **Visitor-id passthrough** for every server source, so the sale joins the ad click
  ([visitor-id.md](visitor-id.md)): `decorate_checkouts` for links/embeds/form iframes; the WooCommerce
  snippet; server code for Stripe sessions, PayPal, SSLCommerz, Razorpay; bKash/Nagad report from the
  merchant's server via `/api/v1/events`.
- **Refunds and cancellations** are recorded automatically but reach ad platforms only with
  `send_lifecycle` on a destination. Explain what each platform does with them
  ([lifecycle-events.md](lifecycle-events.md)) and turn it on only for destinations the customer
  chooses. Cash-on-delivery stores: `purchase_on: "created_cod"` + `send_lifecycle` + mark refused
  orders Cancelled.
- **CRM** (HubSpot, Pipedrive, GoHighLevel, Salesforce, Zoho, generic): `list_crm_providers` →
  `create_crm_integration` with a stage map (e.g. "Qualified" → `Lead`, "Won" → `Purchase`) →
  `test_crm_payload`. With an `api_token` it starts polling immediately.
- **Offline / phone / in-store sales**: `import_offline_conversions` with `dry_run: true` first, check
  the result, then import for real; or `create_offline_source` for a recurring CSV / sheet URL.
- **Any other tool with webhooks**: `create_webhook_integration` with a `mapping` that includes
  `order_id_path` (required for Purchase; dedup + refunds) or `event_id_path`, `vid_path` if the tool
  carries `tf_vid`, `status_path` + `status_map` for refunds/cancellations, and a `signature` scheme + `secret`
  matching how the tool signs (else `verified: false`). Check with `test_webhook_payload` before giving
  out the URL. Cal.com: `provider: "cal"` adds cancellations and no-shows.

## Phase 6: preview and test

1. `publishing → create_preview_link {"url": "<page>"}`. Open it in a **normal** browser (headless
   browsers are blocked as bots) or give it to the customer. Events from it are marked test and the
   draft container is used.
2. Walk the funnel: page → product → add to cart → checkout → (test order if the customer agrees) /
   submit the form.
3. `reports → list_events {"test": "only"}` and `get_event {"event_row_id": <id>}` for each step.
   Check: event name, value is a number, currency, order id, content ids, user data present on
   lead/purchase, one event per action (no duplicates).
4. `send_test_event` for a server-side check through every destination.
5. Remember: test events reach only destinations that have a test field set; others show `skipped`.
   Browser pixels in a preview fire live.

## Phase 7: go-live checklist

Tick every line before `publish_version`:

- [ ] Customer asked for (or approved) publishing.
- [ ] Each event fires once per action in preview, with correct name, value, currency, order id.
- [ ] No duplicate pixel for any connected platform (or that destination is Server only).
- [ ] Every destination `ready`, `test_destination` passed.
- [ ] Test fields (`test_event_code`, `validate_only`, `debug`) removed again, unless the customer
      explicitly wants to stay in test mode.
- [ ] `consent_mode` matches the customer's legal needs.
- [ ] `dedup`, `block_bots` stay on.
- [ ] Shopify: Customer Events code connected (`shopify_pixel_detected` after real visits), no channel app
      sending the same pixel ids.
- [ ] Store/checkout webhooks registered with **all** listed topics (incl. refunds/cancellations), secret
      set (`signature_verification: on`), `purchase_on` as agreed, a test delivery in `get_checkout_log`.
- [ ] Visitor-id passthrough in place for each server source (`test_checkout_payload` → `matched_by` is
      `vid` or `order` for a realistic sample).
- [ ] `send_lifecycle` decided **per destination with the customer** (default off); if on, they know what
      each platform does with refunds/cancellations.

Then `publishing → publish_version {"notes": "What changed and why"}`. Visitors get it on their next
page load.

## Phase 8: verify live

1. A few minutes after publishing: `list_events` (newest first). Statuses:
   `delivered` good · `partial` some destinations failed · `failed` · `blocked` (rule/bot/reload) ·
   `no_destination` (nothing connected) · `browser_only` (browser pixels only, no server delivery set up).
2. `get_event` on a sample of each event name: platform HTTP responses (Meta `events_received: 1`).
3. Fix credentials and use `retry_event_delivery` for failed events (careful: it re-sends to every
   destination; GA4, Taboola and Outbrain don't dedup).
4. After 24–48 h: `get_match_quality` and `get_overview`. Raise match quality with `auto_forms`,
   `identify` calls on login/checkout, or user data in purchase tags.
5. Webhook sources: `get_checkout_log` → `matched_by` per order (mostly `vid` / `order` is good;
   many `email` / `landing_url` / `none` → fix the passthrough); refunds show as `OrderRefunded` /
   `OrderCancelled` in `list_events` and as `refunds` / `net_revenue` in `get_overview`.
6. Problems: [audit-playbook.md](audit-playbook.md).

## Phase 9: hand-off message to the customer

Keep it short and non-technical:

1. **What is tracked:** each event in plain words, and which platforms receive it.
2. **What you still need to do,** if anything: paste the tag, add the DNS record, register the webhook.
   Give exact steps.
3. **How to check it yourself:** for example, "In Facebook Events Manager → Test events you'll see
   Purchase with the order amount."
4. **Optional next steps:** a first-party tracking domain for ad-blocker resistance
   ([first-party-domain.md](first-party-domain.md)), consent mode, offline conversions.

Never include secrets in the summary. Confirm credentials only by their last 4 characters.
