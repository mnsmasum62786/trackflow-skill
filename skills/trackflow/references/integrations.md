# TrackFlow integrations: checkout webhooks, CRMs, offline conversions, mapped webhooks, Google sign-in

Server-side sources create events without a browser. They go through the same pipeline as browser
events (blocking rules, dedup, attribution, delivery to every enabled destination's server API). All
tools are in the `integrations` group (`{"action": "...", "site_id": N, ...}`), except
`google_ads_sign_in_link` (group `destinations`).

| Source | Use for | Origin in `list_events` | Verified? | Tools |
|---|---|---|---|---|
| Checkout integration | Store backup and payments/bookings: Shopify, WooCommerce, Stripe, PayPal, Lemon Squeezy, Paddle, Gumroad, ThriveCart, SamCart, Calendly, Razorpay, ClickFunnels 2.0, systeme.io, Teachable, Kajabi, Whop | `webhook` | Provider signature (Gumroad: seller_id; SamCart/Teachable/Kajabi: optional `?secret=`) | `list_checkout_providers`, `create/update/delete_checkout_integration`, `list_checkout_integrations`, `test_checkout_payload`, `get_checkout_log` |
| CRM integration | Pipeline stages → events (HubSpot, GoHighLevel, Pipedrive, Salesforce, Zoho, generic) | `crm` | Provider signature or shared secret | `list_crm_providers`, `create/update/delete_crm_integration`, `list_crm_integrations`, `test_crm_payload`, `sync_crm_integration`, `get_crm_log` |
| Offline conversions | Store/phone/back-office sales: CSV, batch API, Google Sheet / CSV URL on a schedule | `offline` | API key (batch API) | `import_offline_conversions`, `list_offline_imports`, `create/update/delete_offline_source`, `list_offline_sources`, `sync_offline_source` |
| Mapped ("custom") webhook | Any app that POSTs JSON (Typeform, Tally, Cal.com, Zapier, Razorpay without the preset, a backend) | `webhook` | Optional: TrackFlow signature, the app's own HMAC, header secret, Basic auth | `create/update/delete_webhook_integration`, `list_webhook_integrations`, `test_webhook_payload` |

Rule of thumb: for stores and payment providers always prefer a **checkout integration** (signature
check, purchase stage, order dedup, refunds and cancellations, alternate-id memory) over a mapped webhook;
for CRMs use a **CRM integration**. In the dashboard, Shopify and WooCommerce checkout integrations sit
under **Integrations → Store backup**, the other providers under **Payments & checkout**, form/booking
tools under **Forms & booking**, and the mapped webhook under **Offline & custom**.

Besides the Purchase, checkout integrations and mapped webhooks create **lifecycle events** (refunds,
cancellations, chargebacks, renewals, booking cancellations, no-shows). They never count as conversions
and reach a destination only when its `send_lifecycle` is on: [lifecycle-events.md](lifecycle-events.md).

---

## 1. How server events reach the ad platforms (stitching + dedup)

### Visitor matching (attribution)

For every server-side event TrackFlow picks the visitor in this order: passthrough visitor id (`vid`) →
**order binding** (an earlier event of the same order id, e.g. the thank-you page Purchase) → provider ids
remembered from earlier webhooks of the order (`provider_ref`) → hashed email → hashed phone → ad click
ids in the order's landing URL (`landing_url`) → `none`. A passed-through vid counts only if that visitor
exists on the site.

A matched event **inherits the visitor's stored click ids** (`fbc`, `fbp`, `gclid`, `gbraid`, `wbraid`,
`ttclid`, `msclkid`…), IP, user agent and earlier hashed identifiers, so the platform can tie it to the
ad click. Results show `vid` + `matched_by` (`vid`, `order`, `provider_ref`, `email`, `phone`,
`landing_url`, `none`). CRM and offline results use `vid` / `email` / `phone`.

How to get the visitor id into each kind of checkout (automatic decoration, Shopify cart attributes,
hidden `tf_vid` fields, WooCommerce/Stripe/PayPal/SSLCommerz/Razorpay/bKash/Nagad code):
[visitor-id.md](visitor-id.md).

### Dedup

| Source | event_id | Effect |
|---|---|---|
| Checkout integration | `purchase_<order id>` (other event names: `<eventname lowercased>_<order id>`; Calendly: `calendly_<invitee uuid>`). Lifecycle ids: [lifecycle-events.md](lifecycle-events.md) | Same id as the browser thank-you Purchase (`purchase_{{order_id}}` tag, the GA4 dataLayer bridge `purchase_<transaction_id>`, the Shopify Customer Events code) → counted once. The order id is used as sent (trimmed), e.g. `purchase_#1001` stays as is; ids written by older versions in sanitised form (`purchase__1001`) are still recognised. Alternate ids of one payment (Stripe session/payment intent/invoice, PayPal order/capture, Shopify numeric id vs order name) are remembered, so a later webhook is a `duplicate`. Redeliveries with the provider's delivery id (Razorpay, Whop, ClickFunnels) are dropped (`duplicate_delivery`). |
| CRM | `crm_<integration id>_<object id>_<stage>` | Each stage of each deal counts once (webhook retries, poll after webhook). |
| Offline | `offline_<order id>`, else a hash of the row | Same order in a later file / twice in one file → `duplicate`. |
| Mapped webhook | `event_id_path` value (+ `event_id_prefix`), else `purchase_<order id>` from `order_id_path` (other event names: `<name>_<order id>`), else a hash of integration + raw body | Without an order/event id only byte-identical retries dedup. |

Server dedup key = `event_id` + `event_name` per site. **The browser and server copies only dedup if
both the id and the event name are identical.** Make the thank-you tag's order id equal the id the
provider's webhook uses (table in §2 "Thank-you `transaction_id`").

A webhook that arrives after the browser already tracked the order is a `duplicate`, but it still
**updates TrackFlow's order record** (verified value, items, contact, visitor, status) that later refunds
and cancellations are measured against.

### action_source (Meta)

| Source | action_source |
|---|---|
| Checkout Purchase | `website` when a user agent is known (payload or matched visitor), else `system_generated` |
| Lifecycle events | `system_generated` (Meta: `website` only when the event carries a user agent and URL) |
| Mapped webhook | `website`, or `system_generated` when no URL is mapped |
| CRM | `config.action_source`, default `system_generated` |
| Offline | row `action_source`, default `physical_store` (rows with a Meta `lead_id` → `system_generated`) |

---

## 2. Checkout integrations

Webhook URL format: `https://<trackflow-host>/hooks/checkout/co_<token>` (returned as `webhook_url`).

| `provider` | Secret to ask for (`secret`) | Webhook events to select | Visitor passthrough | Thank-you `transaction_id` that dedups |
|---|---|---|---|---|
| `shopify` | Webhook signing secret (Settings → Notifications → Webhooks) or app client secret | `orders/create`, `orders/paid`, `orders/fulfilled`, `orders/cancelled`, `refunds/create` (one webhook each; `purchase_on` decides which one sends the Purchase) | Cart attributes `_tf_vid`, `_tf_fbc`, `_tf_fbp`, `_tf_gclid` (written by the theme tag); landing-URL click ids are read too | Order id (`checkout.order.id`; the Customer Events code uses it), not the name `#1001` (unless `order_id_field`) |
| `woocommerce` | Webhook secret (WooCommerce → Settings → Advanced → Webhooks) | Two webhooks: **Order created** and **Order updated** (paid, completed, cancelled, refunded all arrive as updated) | Order meta `tf_vid` (+ `tf_fbc`, `tf_fbp`; older `_tf_vid` read) from the snippet in visitor-id.md; order-attribution entry URL, IP, UA read too | Order id `$order->get_id()` (or `order_id_field: "number"`) |
| `stripe` | Endpoint signing secret `whsec_…` (5-min tolerance) | `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `charge.refunded`, `charge.dispute.created`, `invoice.paid`, `customer.subscription.deleted`; optional `payment_intent.succeeded` | `client_reference_id=<vid>` (Payment Links accept it in the URL; Buy Button / Pricing Table `client-reference-id`) or `metadata.tf_vid`; PaymentIntents `metadata.tf_vid`; subscriptions `subscription_data.metadata.tf_vid` | Checkout Session id (`cs_…`, `{CHECKOUT_SESSION_ID}`) |
| `paypal` | **Webhook ID** (not a secret key) | `CHECKOUT.ORDER.COMPLETED`, `PAYMENT.CAPTURE.COMPLETED`, `PAYMENT.CAPTURE.REFUNDED`, `PAYMENT.CAPTURE.REVERSED`, `CUSTOMER.DISPUTE.CREATED`, `PAYMENT.SALE.COMPLETED`, `BILLING.SUBSCRIPTION.CANCELLED` | Orders v2 `purchase_units[].custom_id=<vid>`; subscriptions `custom_id` | PayPal order id (`onApprove data.orderID`) |
| `lemonsqueezy` | Signing secret you set | `order_created`, `order_refunded`, `subscription_payment_success`, `subscription_cancelled`, `subscription_payment_refunded` | `?checkout[custom][tf_vid]=<vid>` | Order id |
| `paddle` | Notification destination secret `pdl_ntfset_…`; optional `api_key` to look up the customer email | `transaction.completed`, `adjustment.created`, `adjustment.updated`, `subscription.canceled` | `Paddle.Checkout.open({customData:{tf_vid}})` | Transaction id `txn_…` |
| `gumroad` | Optional: your `seller_id` | Ping (sales, recurring charges, refunded / disputed sales) | Product URL `?tf_vid=<vid>` | `sale_id` |
| `thrivecart` | Secret word (`thrivecart_secret`) | `order.success`, `order.refund`, `order.subscription_payment`, `order.subscription_cancelled` | `?passthrough[tf_vid]=<vid>` | `order_id` |
| `samcart` | Optional shared secret, appended as `?secret=…` to the Notify URL | Order, Refund, Partial Refund, Subscription Charged, Subscription Canceled | `?tf_vid=<vid>` (decoration adds `utm_content`, also read) | Order id |
| `calendly` | Webhook signing key (3-min tolerance) | `invitee.created`, `invitee.canceled`, `invitee_no_show.created` | Link/embed `?salesforce_uuid=<vid>` or `utm_content`/`utm_term` | Event id `calendly_<invitee uuid>` (= the `booking` recipe) |
| `razorpay` | Webhook secret (Dashboard → Accounts & Settings → Webhooks) | `payment.captured` (or `order.paid`), `refund.processed`, `payment.dispute.created`, `subscription.charged`, `subscription.cancelled` | `notes.tf_vid` (order or Checkout.js) | Razorpay order id (`order_…`), else payment id |
| `clickfunnels` | Webhook endpoint secret (600-s tolerance) | `one-time-order.completed` (or `order.completed`), `one-time-order.refunded` / `order.refunded`, `subscription.invoice.paid`, `subscription.canceled` | None documented: order binding + email/phone (contact custom attribute `tf_vid` read if present) | Order id |
| `systeme` | Webhook secret | `SALE_NEW`, `SALE_CANCELED` | None: email/phone + order binding | Order id |
| `teachable` | Optional shared secret (`?secret=…` on the URL) | `Sale.created` (or `Transaction.created`), `Transaction.refunded`, `Sale.subscription_canceled` | None: email + order binding | Sale id |
| `kajabi` | Optional shared secret (`?secret=…` on the URL) | Purchase / `payment.succeeded` | None: email + order binding | Transaction id |
| `whop` | Webhook secret (Standard Webhooks) | `payment.succeeded`, `refund.created`, `dispute.created`, `membership.updated` (also `membership.deactivated`) | Checkout configuration `metadata.tf_vid` | Payment id (`pay_…`) |

`list_checkout_providers` returns this live (`docs`, `secret_label`, `api_label`, `signed`, `events`,
`purchase_stages`, `passthrough`, and the allowed `purchase_on` values). Some payload paths of the newer
presets (ClickFunnels, systeme.io, Teachable, Kajabi, Whop, SamCart/ThriveCart lifecycle notifications,
Calendly no-shows) could not be verified against public samples: **run `test_checkout_payload` with a real
sample from the provider before relying on them.**

### When to send Purchase (`config.purchase_on`)

| Value | Purchase is sent when | Good for |
|---|---|---|
| `paid` (default) | Payment is confirmed (Shopify `orders/paid` or financial status paid; Woo `processing`/`completed`; Stripe paid session) | Most stores |
| `created` | The order exists **with payment authorized or paid** (Shopify `authorized`; otherwise same as paid) | Providers that authorise first, capture later |
| `completed` | Fulfilled / completed (Shopify `orders/fulfilled`, Woo `completed`); providers without fulfilment treat it as paid | Only real, delivered sales (see the COD quality mode in lifecycle-events.md) |
| `created_cod` | The order is created, **even unpaid** (Shopify `orders/create`, Woo `pending`/`on-hold`); a later cancellation sends `OrderCancelled` | Cash-on-delivery stores |

Stage-aware providers: Shopify, WooCommerce, Stripe (async payments), Lemon Squeezy (order status). For the
others the webhook itself is the paid signal. The Purchase's **event time is the time of the chosen stage**
(paid / fulfilled / created), falling back to the order's creation time. Every order webhook updates the
order record even when it doesn't send the Purchase (result `ignored` with "Purchase is sent when …").

### `config` keys

`config` replaces the saved config as a whole on update: send the full object.

| Key | Providers | Meaning |
|---|---|---|
| `purchase_on` | all (matters for stage-aware ones) | `paid` (default), `created`, `completed`, `created_cod` (see above). Other values are rejected. |
| `event_name` | all | Event created (default `Purchase`; Calendly `Schedule`). |
| `accept_test` | all | `true` = also track test/sandbox payments and refunds (default: ignored). |
| `order_id_field` | stripe (`payment_intent`), shopify (`name` / `order_number`), woocommerce (`number`) | Which id becomes the order id. |
| `default_currency`, `default_country` | all | Used when the payload has none. |
| `renewals` + `renewal_event_name` | stripe | Legacy: send `invoice.paid` renewals as that (conversion) event instead of the `SubscriptionRenewed` lifecycle event. |
| `subscription_event_name` | stripe | Rename first subscription checkouts. |
| `payment_intents` | stripe | Use every `payment_intent.succeeded`, not only those with `metadata.tf_vid`. |
| `paid_statuses` | woocommerce | Default `["processing","completed"]`. |
| `track_invoice_payments` | razorpay | Also track Razorpay Invoices payments (subscriptions come from `subscription.charged`). |
| `sandbox` | paddle, razorpay | Mark events as test / use the sandbox API. |
| `value_divisor` | thrivecart | Divide totals (ThriveCart sends hundreds). |
| `event_types` | samcart | Notification types that create the Purchase (default `["Order"]`). |
| `vid_field`, `value` | calendly | Tracking field holding the vid; fixed value per booking. |

Built-in handling: amounts in minor units are converted (zero- and three-decimal currencies respected);
Stripe unpaid sessions wait for `async_payment_succeeded` (under `paid`); Shopify refunds are summed from
successful refund transactions; cancelled/failed Woo orders become `OrderCancelled`. A Purchase with no
vid, email, phone, IP or click id is ignored.

### Setup steps (MCP)

1. Ask the customer: which provider, when a sale should count (usually paid; cash on delivery →
   `created_cod`), currency if the provider doesn't send one, and whether test payments should count.
2. `list_checkout_providers` → read `secret_label`, `events`, `purchase_stages`.
3. `create_checkout_integration {"site_id": N, "provider": "stripe", "config": {"purchase_on": "paid"}}`
   → give the customer `webhook_url` and tell them to select **exactly** the listed events (including the
   refund/cancel ones) in the provider dashboard.
4. Providers that issue the secret after the endpoint exists (Stripe, Shopify…): ask them to paste it,
   then `update_checkout_integration {"integration_id": X, "secret": "<pasted>"}`. Confirm only the last
   4 characters. `signature_verification` reads `on`, `off` (signed provider without a secret: **anyone
   with the URL can post orders**, always set it), `secret check` (Gumroad/ThriveCart/SamCart/Teachable/
   Kajabi with a secret) or `not verified (secret URL only)`.
5. Stitching: [visitor-id.md](visitor-id.md) (`decorate_checkouts` + publish, or the code for that
   checkout).
6. `test_checkout_payload {"integration_id": X, "payload": {...}, "headers": {"x-shopify-topic": "refunds/create"}}`
   → every event it would create (Purchase or lifecycle) with `status`, `order_id`, `event_id`, `value`,
   `partial` / `remaining_value` for refunds, `vid`, `matched_by`, `action_source`, `identifiers`,
   `click_ids`, `event_time`, or the reason it would be ignored. Signature not checked, nothing stored.
   Topic headers: `x-shopify-topic`, `x-wc-webhook-topic`, `x-webhook-event` (systeme.io).
7. Refunds/cancellations reach ad platforms only with `send_lifecycle` on the destination: ask the
   customer ([lifecycle-events.md](lifecycle-events.md)).
8. After a real payment: `get_checkout_log` and `list_events {"origin": "webhook"}`.

```json
{"action": "test_checkout_payload", "site_id": 12, "integration_id": 3,
 "payload": {"type": "checkout.session.completed", "livemode": false,
   "data": {"object": {"id": "cs_test_123", "payment_status": "paid", "amount_total": 4900,
     "currency": "usd", "client_reference_id": "VISITOR_ID",
     "customer_details": {"email": "buyer@example.com"}, "created": 1767225600}}}}
```

(`livemode: false` returns `ignored` unless `accept_test: true`; that is expected.)

### Log and result statuses

| Where | Status | Meaning |
|---|---|---|
| Per result | `created` | Event ingested (`match_score`, `blocked` if a rule blocked it). |
| | `duplicate` | Same payment/event already tracked (alternate id, or browser thank-you page / Shopify pixel / earlier webhook with the same event id). The order record is still updated (`enriched`). |
| | `ignored` | `reason`: not a used event type, test mode, not at the `purchase_on` stage yet, no tracked Purchase for a refund, no order id, no identifier. |
| | `would_create` / `would_skip` | Dry run only. |
| Log entry (`get_checkout_log`, last 100) | `ok` / `duplicate` / `ignored` | Summary of the delivery's results, with an `events` list (`created: OrderRefunded refund_… 20 USD`), `delivery_id`, results. |
| | `duplicate` + `delivery_id` | Provider redelivery of an event already processed. |
| | `rejected` | Signature failed (provider got HTTP 400; `last_error` set). |
| | `error` | Processing failed (HTTP 500). |

---

## 3. CRM integrations

Webhook URL: `https://<trackflow-host>/hooks/crm/crm_<token>`.

| `provider` | Inbound | `secret` | `api_token` (enables lookups + polling) |
|---|---|---|---|
| `hubspot` | Workflow "Send a webhook" (properties JSON) or app webhook subscriptions (ids only) | App client secret (X-HubSpot-Signature v1/v2/v3, 5-min window) | Private app token, scopes `crm.objects.deals.read`, `crm.objects.contacts.read`. Polls deals by `hs_lastmodifieddate`. |
| `gohighlevel` | Workflow "Webhook" action (opportunity events) | Optional shared secret (`?secret=` or `X-TrackFlow-Secret`) | Private Integration token (`opportunities.readonly`, `contacts.readonly`) **+ `config.location_id`** |
| `pipedrive` | Webhooks v2/v1, deal events | Optional HTTP basic auth stored as `user:password` | Personal API token; optional `config.company_domain` |
| `salesforce` | Outbound Messages (SOAP; TrackFlow sends the `<Ack>`) or Flow HTTP callouts (JSON) | Organization ID (15/18 chars) or shared secret | No polling |
| `zoho` | Workflow rule webhook; name params `email, phone, first_name, last_name, stage, amount, currency, id` | Optional `secret` parameter | No polling |
| `generic` | Any JSON/form POST (Zapier, Make, n8n, lead-ads relay): `id, stage/status, email, phone, first_name, last_name, name, city, state, zip, country, value/amount, currency, event_time, tf_vid, lead_id` | Optional: `X-TrackFlow-Secret`, `?secret=`, body `trackflow_secret`, or `X-Signature` = hex HMAC-SHA256 of the raw body | No polling |

Bad/missing secret → HTTP 401 and a `rejected` log entry. Slow processing: after 4 s the CRM gets
`{accepted: true, processing: true}` and lookups finish in the background.

### Stage map

Default (stage keys compared lowercase, punctuation removed: `Closed Won` = `closedwon`):

| Stage | Event |
|---|---|
| `won`, `closedwon` | `Purchase` with the deal value |
| `qualified`, `qualifiedlead`, `qualification` | `QualifiedLead` |
| HubSpot `qualifiedtobuy`, `salesqualifiedlead`, `marketingqualifiedlead` | `QualifiedLead` |
| HubSpot `appointmentscheduled` | `Schedule` |
| Salesforce `closedwon` (or `IsWon = true`) | `Purchase` with the amount |

```json
{"action": "create_crm_integration", "site_id": 12, "provider": "hubspot",
 "secret": "<client secret the customer pasted>", "api_token": "<private app token>",
 "config": {
   "stage_map": {"qualifiedtobuy": "QualifiedLead", "presentationscheduled": "Schedule",
                 "closedwon": {"event_name": "Purchase", "value": true}},
   "keep_defaults": false, "default_currency": "EUR", "poll_minutes": 15}}
```

- A custom `stage_map` **replaces** the defaults unless `keep_defaults: true`.
- Value is sent only for `Purchase` or rules with `value: true` (`is_conversion` can also be set per rule).
- Unmapped stages → `ignored` ("Stage … is not mapped"). Payloads without a stage: set `config.event_name`
  (+ `include_value`) to create one event per payload.
- Custom pipelines (HubSpot, Pipedrive, GoHighLevel) often send **internal stage ids**; map those ids.

Other `config`: `mapping` (dot paths: `email_path`, `phone_path`, `first_name_path`, `last_name_path`,
`name_path`, `city_path`, `state_path`, `zip_path`, `country_path`, `value_path`, `currency_path`,
`stage_path`, `object_id_path`, `lead_id_path`, `vid_path`, `external_id_path`, `event_time_path`,
`value_divisor`), `default_country`, `action_source`, `poll_minutes` (min 5, default 15), `poll: false`,
`backfill_days`, `location_id` (GHL), `company_domain` (Pipedrive), `public_url` (HubSpot, when it calls
a different public URL than TrackFlow sees).

Events need an email, phone, external id, Meta lead id or visitor id; otherwise `ignored` (add the API
token so TrackFlow can look the contact up). Visitor id fields read automatically: `tf_vid`,
`trackflow_vid`, `trackflow_visitor_id`, `visitor_id` (top level, `customData`, `properties`).

**Meta lead ads (Conversions API for CRM):** a payload field `lead_id`, `leadgen_id`, `fb_lead_id`,
`facebook_lead_id`, `meta_lead_id` or `lead_ads_id` (or `mapping.lead_id_path`) is sent to Meta unhashed
with `action_source: system_generated`. The customer must also set up the CRM integration for the
dataset in Meta Events Manager.

### Setup steps (MCP)

1. Ask: CRM name, which stages mean what (qualified, booked, won), currency, and whether they can
   create a private-app/API token (recommended: fills in contact details + polling).
2. `list_crm_providers` → `default_stage_map`, `secret_label`, `api_label`, `pollable`, `mapping_fields`.
3. `create_crm_integration` → give the customer `webhook_url` and where to paste it.
4. `test_crm_payload {"integration_id": X, "payload": {...}}` (XML string for Salesforce) → per record
   `would_create` / `ignored` with `stage`, `event_name`, `event_id`, `value`, `identifiers`, `vid`,
   `matched_by`, `lead_id`. (Dry run: no lookups, no events.)
5. With a token: `sync_crm_integration` to poll now (`fetched`, `created`, `results`).
6. `get_crm_log` (kinds `webhook` / `poll` / `test`; statuses `ok`, `ignored`, `rejected`, `error`).

---

## 4. Offline conversions

### Fields (CSV columns are auto-detected; override with `mapping: {field: "Header"}`)

| Field | Recognised headers (case/punctuation ignored) |
|---|---|
| `email`, `phone` | email, email address, mail, customer email, em · phone, phone number, mobile, tel, telephone, cell, ph |
| `name`, `first_name`, `last_name` | name, full name, customer name, contact name · first name, fname, given name, first, fn · last name, lname, surname, family name, last, ln |
| `city`, `state`, `zip`, `country` | city, town, ct · state, province, region, st · zip, zip code, postcode, postal code, zp · country, country code |
| `external_id` | external id, customer id, user id, client id, crm id, contact id |
| `event_name` | event, event name, conversion, conversion name, conversion type |
| `value`, `currency` | value, amount, revenue, total, price, order total, order value, conversion value, sale amount · currency, currency code |
| `order_id` | order id, order, order number, transaction id, invoice, invoice id, invoice number, receipt, receipt id, sale id |
| `event_time` | event time, date, time, datetime, timestamp, order date, conversion time/date, purchase date, sale date, created at |
| `action_source` | action source, channel, sales channel |
| `lead_id`, `vid` | lead id, fb/meta/facebook lead id · vid, visitor id, tf vid |
| click ids | gclid, gbraid, wbraid, fbc, fbp, fbclid (converted to fbc), ttclid, msclkid |

`defaults`: `event_name` (Purchase), `currency` (USD), `action_source` (physical_store; also
phone_call, system_generated, email, chat, website, app, other), `country`, `date_format` (`mdy` default
or `dmy` for slash dates). Dates: ISO 8601, `YYYY-MM-DD[ HH:mm[:ss]]`, Unix s/ms, slash dates; no zone =
UTC; date-only = 12:00 UTC.

### Validation

| Rejected | Flagged (imported with warning) |
|---|---|
| No identifier (email, phone, external id, lead id, vid or click id) | Invalid email when another identifier exists |
| Invalid event name (`^[A-Za-z][\w ]{0,63}$`) | Older than 7 days (non-store) / 62 days (physical_store): Meta won't accept |
| Value not a number / negative; **Purchase without value** | Lead id not 10–20 digits |
| Currency not 3-letter ISO | No event time (upload time used) |
| Unknown action source; unreadable/future date; **older than 90 days** | |
| Duplicate order id within the file | |

Other platform windows (older rows are `skipped` per destination with "Too old for <Platform> (max N
days)"): GA4 72 h, TikTok/Pinterest/Reddit/OpenAI 7 days, Snapchat 37 days, LinkedIn/Google Ads/Microsoft
90 days ([lifecycle-events.md](lifecycle-events.md) §3). Taboola and Outbrain can't use offline
rows (deliveries `skipped` with the reason).

### Ways in

| Way | Tool / endpoint | Limits |
|---|---|---|
| CSV text | `import_offline_conversions {"csv": "...", "mapping": {}, "defaults": {}, "dry_run": true}` | Header row required; comma/semicolon/tab; ≤50,000 rows. **Run `dry_run: true` first** (status per row: `valid` / `rejected` / `duplicate` + warnings), then without. |
| Scheduled sheet / CSV URL | `create_offline_source {"url": "...", "mapping": {}, "defaults": {}, "interval_minutes": 60}` | Interval 15–1440 (default 60). Imports only rows added since the last sync (row cursor). |
| Server batch API | `POST /api/v1/offline-events` with `Authorization: Bearer <API key>` | ≤1000 events/request; body `{site_key, defaults, events:[…], validate_only}` |
| History | `list_offline_imports` | accepted / rejected / duplicates / flagged / matched + first 100 problem rows |

**Public URL requirement (sources):** the URL must be reachable without login from the public internet
(private/internal hosts are refused at save time). Google Sheets: Share → "Anyone with the link" (or
File → Share → Publish to web → CSV). Edit links are converted to the CSV export URL (keeping `gid`). An
HTML response (a login page) fails with an explanatory error. Don't repeat a private sheet link back in
full. Deleted rows → the next sync restarts from row 1 (order-id dedup keeps it safe);
`update_offline_source reset_cursor: true` or a new `url` also restarts.

```json
{"action": "import_offline_conversions", "site_id": 12, "dry_run": true,
 "csv": "Order ID,Email,Phone,Amount,Currency,Date\nINV-1042,buyer@example.com,+15555550100,1290,USD,2026-10-20 14:05",
 "defaults": {"event_name": "Purchase", "action_source": "phone_call"}}
```

Batch API body (developer integration; placeholders only):

```json
{"site_key": "tf_xxxxxxxx", "defaults": {"currency": "EUR", "action_source": "phone_call"},
 "events": [{"event_name": "Purchase", "order_id": "INV-1042", "value": 1290,
             "event_time": "2026-10-20T14:05:00Z", "email": "buyer@example.com",
             "phone": "+15555550101", "gclid": "GCLID_VALUE"}],
 "validate_only": true}
```

### What to ask the customer

Which event (Purchase, QualifiedLead…), currency, where the sale happened (store / phone / back office →
action_source), date format of their sheet (US `mdy` or `dmy`), and that every row has an order id and
email or phone (plus the country code on phones).

---

## 5. Mapped ("custom") webhooks

`create_webhook_integration {"event_name": "Purchase", "provider": "generic", "mapping": {...}, "secret": "...", "signature": {...}}`
→ returns `webhook_url` (`https://<trackflow-host>/hooks/<token>`). `provider` is a label (`stripe`,
`calendly`, `typeform`, `shopify`, `woocommerce`, `cal`, `tally`, `generic`); the mapping does the work,
except `cal` (Cal.com cancellations/no-shows, below). Closed schema: unknown keys are rejected.
`update_webhook_integration` replaces `mapping` and `signature` as a whole; the URL stays the same.

### Mapping (dot paths into the JSON body)

| Key | Meaning |
|---|---|
| `email_path`, `phone_path`, `name_path`, `first_name_path`, `last_name_path` | Contact (hashed by TrackFlow) |
| `value_path`, `value_divisor` (100 for cents), `currency_path`, `default_currency` | Order value |
| `order_id_path` | **Order id. Required** for new `Purchase` webhooks and whenever `status_path` / `status_map` / `refund_amount_path` is used. Event ids become `purchase_<order id>` (matches the browser tag and checkout integrations), `refund_<order id>_<refund id>`, `cancel_<order id>`; it also links refunds to the order's visitor. Saving without it → 400 with the reason. Existing webhooks keep working until you add lifecycle mapping. |
| `event_id_path`, `event_id_prefix` | Explicit event id (wins over `order_id_path`); e.g. prefix `purchase_` |
| `status_path` + `status_map` | Order status and what each value means: `{"paid": "purchase", "refunded": "refund", "partially_refunded": "refund", "cancelled": "cancel", "disputed": "chargeback", "pending": "ignore"}`. Values: `purchase` (creates `event_name`), `refund` (`OrderRefunded`), `cancel` / `chargeback` (`OrderCancelled`), `ignore`. Without a map, common statuses work (paid, completed, succeeded, approved, captured → purchase; refunded, partially_refunded → refund; cancelled, voided → cancel; chargeback, disputed → chargeback; pending, failed, processing → ignore); unmapped statuses are ignored. A `status_map` needs `status_path`. |
| `refund_amount_path` | Refunded amount for partial refunds (`value_divisor` applies; without it the rest of the order is refunded) |
| `refund_id_path` | The refund's own id (keeps two refunds of the same amount apart) |
| `vid_path` (= `visitor_path`) | TrackFlow visitor id passed through (hidden field `tf_vid`, checkout metadata, Shopify `_tf_vid`) |
| `landing_url_path` | Landing page URL; its ad click ids are used when no visitor matches |
| `time_path` | When it happened (ISO, s or ms; default now) |
| `url_path`, `fbc_path`, `fbp_path` | Page URL, Meta cookies |
| `filter_path` + `filter_value` | Accept only bodies where the path equals the value (others → `ignored: filtered`) |

Refunds and cancellations follow the checkout rules: only for orders whose Purchase was tracked, nothing
after a full refund or cancellation, retries deduplicated ([lifecycle-events.md](lifecycle-events.md)).
Visitors are matched by `vid_path`, then the order, email/phone, then click ids in `landing_url_path`
([visitor-id.md](visitor-id.md)).

**Cal.com** (`provider: "cal"`, `event_name: "Schedule"`): the `BOOKING_CANCELLED` trigger becomes
`BookingCancelled` (`bookingcancel_<payload.uid>`) and `BOOKING_NO_SHOW_UPDATED` (an attendee marked
no-show) becomes `BookingNoShow` (`noshow_<uid>`) automatically, regardless of the filter. Select those
triggers in Cal.com too.

### Verification (`secret` + `signature`)

`secret` (8–200 chars) is checked as `signature.scheme` says; returned masked with `secret_set`, omit to
keep, `""` to clear.

| `signature.scheme` | Check | Fields |
|---|---|---|
| `trackflow` (default) | `X-TrackFlow-Signature: sha256=<hex HMAC-SHA256 of the raw body>`, or the secret itself in `X-Webhook-Secret` / `?secret=` (Zapier, Make). No secret = not verified | – |
| `hmac` | The app's own HMAC signature in `header`; any one of several comma/space-separated signatures may match | `header`*, `algorithm` (`sha256`/`sha1`), `encoding` (`hex`/`base64`), `signed` (`body` or `timestamp.body`, the latter needs `timestamp_header`; timestamps > 5 min off are rejected), `prefix` (e.g. `sha256=`, stripped) |
| `header_secret` | The secret in a header | `header` (default `X-Webhook-Secret`) |
| `basic` | HTTP Basic auth; secret = `user:password` (or only the password) | – |
| `none` | Not verified: anyone with the URL can post | – |

`hmac`, `header_secret` and `basic` require a secret. Failed checks → `401`. `list_webhook_integrations`
shows `signature`, `verified` (false = anyone with the URL can post) and `warnings` (no order/event id,
not verified): relay them to the customer.

### Dry run: `test_webhook_payload`

```json
integrations {"action": "test_webhook_payload", "site_id": 12, "integration_id": 4,
  "payload": "{\"order\":{\"id\":\"1042\",\"status\":\"refunded\",\"total\":\"49.00\"}}",
  "headers": {"X-Signature": "…"},
  "mapping": {"order_id_path": "order.id", "status_path": "order.status", "value_path": "order.total"}}
```

Returns the `signature` check (`ok`, `verified`, `scheme`, `reason`) and every event it would create
(`kind`, `event_name`, `event_id`, `custom_data`, `vid` + `matched_by`, `conversion`, `duplicate`,
`would`: store and send / skip / reject) or `ignored` + `reason`. Pass the **raw JSON string** as `payload`
to test an HMAC signature; `mapping` tries an unsaved mapping. Nothing is stored or sent.

---

## 6. Google Ads sign-in (OAuth)

Google Ads server-side delivery (Data Manager API) needs a refresh token; TrackFlow gets it through
"Sign in with Google" instead of a hand-made token.

1. `add_destination` for `google_ads` with the customer's Ads fields (see the destinations reference).
2. `google_ads_sign_in_link {"site_id": N, "destination_id": D}` → `url` + `note`. Send the link to the
   customer: they open it, sign in with the Google account that has access to the Ads account (or
   manager account) and allow the **Data Manager** permission. The link works **once** and expires in
   **15 minutes**.
3. `list_destinations` → refresh token saved (masked), `has_token: true` → `test_destination`.

Uses TrackFlow's managed OAuth app, or the destination's own `oauth_client_id` / `oauth_client_secret`
if set (then the Google Cloud project needs the Data Manager API enabled and a "Web application" client
with TrackFlow's redirect URI from the note). `update_destination` with `refresh_token: null` disconnects.

| Error shown | Fix |
|---|---|
| `redirect_uri_mismatch` | Add the redirect URI from the note to the customer's OAuth client. |
| `invalid_client` / `unauthorized_client` | Wrong client id/secret, or not a "Web application" client. |
| `access_denied` | Cancelled; consent screen in Testing → add the account as test user. |
| `no_refresh_token` | Remove TrackFlow at the Google account permissions page, sign in again. |
| `scope_missing` | Tick the Data Manager permission; enable the API on own clients. |
| `state_expired` / `state_used` / `invalid_grant` | Create a new link. |

---

## 7. Gotchas

| Symptom | Cause / fix |
|---|---|
| Purchases counted twice (thank-you tag + webhook) | Event ids differ. Thank-you tag must use `purchase_<same order id as the webhook>` (Stripe: session id `cs_…`; Shopify: numeric order id, not `#1001`; PayPal: order id; WooCommerce: `$order->get_id()`). Or keep only one source. |
| Webhook events have no ad attribution (`matched_by` email/phone/landing_url/none) | No passthrough vid and no order binding. Set up the passthrough for that checkout ([visitor-id.md](visitor-id.md)), make the thank-you page send the Purchase with the order id, or capture the email earlier (form `identify`). |
| Test payments not tracked | Expected: test/sandbox mode ignored unless `config.accept_test: true` (log reason says so). |
| All webhooks `rejected` | Wrong secret type (PayPal needs the **Webhook ID**; Stripe needs the endpoint's `whsec_`, a different one per endpoint), or the secret was rotated. Mapped webhook: wrong `signature` scheme/header (`test_webhook_payload` shows the reason). |
| Shopify / WooCommerce log full of `duplicate` | Normal: several topics per order (create, paid, fulfilled) share `purchase_<order id>`; only the `purchase_on` stage sends, the rest enrich the order record. |
| Orders `ignored` "Purchase is sent when …" | The order hasn't reached the `purchase_on` stage (e.g. unpaid COD order under `paid`). Choose `created_cod` for cash-on-delivery stores. |
| Refund/cancel `ignored` "No tracked Purchase for order …" | The order's Purchase was never tracked (or under another order id). Align ids; refunds of untracked orders are deliberately not sent. |
| Refunds recorded but not on the ad platform | `send_lifecycle` is off for that destination (delivery `skipped`), the platform can't take them (Snapchat, Pinterest, Quora; Google Ads full refunds), or too old. See [lifecycle-events.md](lifecycle-events.md). |
| Mapped webhook rejected on save: `order_id_path is required` | New Purchase webhooks and any status/refund mapping need `mapping.order_id_path`. |
| CRM stage changes ignored | Stage not in `stage_map` (often internal ids), or custom map without `keep_defaults`. Check `get_crm_log` reasons. |
| CRM events ignored "No email, phone…" | Webhook carries ids only: add `api_token`, or include contact fields / `mapping`. |
| Offline rows rejected | Purchase without value, rows >90 days, bad currency, no identifier. Always `dry_run` first. |
| Sheet sync errors "Got an HTML page" | Sheet not public; share as "Anyone with the link" or publish to web as CSV. |
| Mapped webhook duplicates | No `order_id_path` / `event_id_path`. |
| Server events never reach a destination | Destination in browser-only mode or missing server fields → event status `no_destination` (see reports-and-audit-data.md); or the event is older than the platform accepts ("Too old for …"). |
