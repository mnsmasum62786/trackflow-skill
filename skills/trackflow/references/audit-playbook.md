# Audit playbook: inspect a site's tracking and fix it

Use this when the customer asks "is my tracking working?", "audit my site", "why does Facebook show
fewer purchases than my shop?", or before any big change. An audit is **read-only** until the customer
agrees to the fixes.

Data meanings, event statuses and the full symptom → cause → fix table are in
[reports-and-audit-data.md](reports-and-audit-data.md). This file is the procedure and the report format.

## 1. Scope (one question, only if unclear)

Which site, and which goal matters most: sales, leads or bookings? Which ad platforms matter? Default
period: the last 30 days.

## 2. Collect evidence (read-only calls)

| # | Call | Look for |
|---|---|---|
| 1 | `sites → get_site` | `last_seen_at` (tag alive?), `auto_pageview` / `auto_forms` / `ecommerce_datalayer` (match what the customer wants?), `consent_mode`, `dedup` and `block_bots` on, `cross_domains` |
| 2 | `destinations → list_destinations` | each platform `enabled`, `mode`, `ready`, `missing` fields, **test fields left on** |
| 3 | `reports → get_overview {"days": 30}` | volume trend vs `prev`, conversions, revenue, failed / partial / blocked / no_destination counts |
| 4 | `reports → get_event_flow {"days": 30}` | per-destination `ok` / `error` / `skipped`, `last_error` |
| 5 | `reports → list_events` with `status` = `failed`, `partial`, `no_destination`, `blocked` | which events, which destinations, `block_reason` |
| 6 | `reports → get_event {"event_row_id": …}` on 2–3 samples per problem | the exact payload and the platform's response |
| 7 | `reports → list_events {"name": "Purchase"}` (and `Lead`, …) | numeric `value`, `currency`, one row per order, `origin` mix (browser + webhook for the same order must share the id) |
| 8 | `reports → get_match_quality {"days": 30}` | score per conversion event, `em` / `ph` / `fbc` coverage |
| 9 | `tag_manager → get_container` (published and draft), `list_tags`, `publishing → list_versions` | what is live, unpublished drafts, disabled or duplicate tags, custom JavaScript |
| 10 | `tag_manager → inspect_site` on home, product, checkout, thank-you, contact | hard-coded pixels / GTM that duplicate TrackFlow (`existing_tags`), forms or CTAs not tracked, dataLayer present but bridge off, checkout providers without an integration |
| 11 | `integrations → list_checkout_integrations`, `get_checkout_log`, `list_crm_integrations`, `get_crm_log`, `list_offline_sources`, `list_offline_imports` | signing secret set, `last_error`, rows rejected |
| 12 | `install → domain_status`, `list_blocking_rules` | first-party domain active?, rules blocking real traffic |
| 13 | `reports → get_attribution {"days": 30}` | conversions with source "Direct" that should be ads (lost click ids, missing cross-domain) |

## 3. Checklist (mark each ✅ / ⚠️ / ❌)

**A. Tag health**
- Tag installed and seen recently (`last_seen_at` within the last day on an active site).
- Only one TrackFlow tag per page; stealth/custom-domain tag in use where ad blockers matter.
- No headless-browser or office traffic polluting data (blocking rules for internal IPs if needed).

**B. Event coverage**
- Every business goal has an event: purchase / lead / booking / sign-up.
- Funnel events exist where ads optimise on them: `ViewContent`, `AddToCart`, `InitiateCheckout`.
- Off-site sales (Stripe, PayPal, Shopify checkout, Calendly, CRM) come in through an integration.

**C. Data quality**
- Purchases: numeric `value` > 0, correct ISO `currency` (not falling back to USD; set `default_currency`), unique `order_id`.
- Event names are the standard, case-correct names for standard actions.
- No PII in custom properties, event names or URLs.
- Products: `content_ids` match the catalog ids when the customer runs catalog/dynamic ads.

**D. Deduplication**
- Each platform receives each conversion once: no theme/plugin/GTM pixel firing the same event.
- Browser and server copies share `event_id` (and event name); purchases use `purchase_<order id>` on
  every source; no `event_id` template with a variable that can be empty.
- Site setting `dedup` is on.

**E. Delivery**
- No `failed` / `partial` in the last 7 days, or the cause is known and fixed.
- Server delivery set up (`mode: both`) for platforms with a Conversions API, unless the customer chose
  browser only.
- No test field left on a live destination.

**F. Match quality and attribution**
- Conversion events carry email/phone (forms, `identify`, webhooks, CRM mappings).
- `fbc` / click ids present for paid traffic; cross-domain and checkout decoration set for hosted
  checkouts so server conversions join the ad click.

**G. Privacy**
- `consent_mode` on when the customer has EU/UK visitors or a consent banner.
- Server-side sources only send consenting customers' data.

## 4. Report format (to the customer)

Plain language, most expensive problem first. For each finding:

```
❌ Purchases are counted twice on Facebook
   Why it matters: Facebook thinks your ads sell twice as much, so it optimises and reports wrongly.
   Cause: your theme also has a Facebook pixel that sends Purchase.
   Fix: remove the theme's pixel (I'll show you where), or I switch TrackFlow's Facebook connection
        to "server only".
   Who: you (theme setting) / me (one change)
```

End with a short summary: what works ✅, what needs fixing ❌/⚠️, and the fixes you can do right now
with the customer's OK.

## 5. Fix

1. Get the customer's OK for the list (or the parts they choose).
2. Apply the fixes with the tools, following [setup-playbook.md](setup-playbook.md) Phases 3–8: draft →
   preview → publish → verify. Fixes outside TrackFlow (remove a theme pixel, add a DNS record,
   register a webhook) are the customer's: give exact steps.
3. After fixing credentials, `retry_event_delivery` the failed events that matter (mind GA4, Taboola
   and Outbrain, which don't dedup on retry).
4. Re-run the evidence calls for the affected areas and report before/after.
