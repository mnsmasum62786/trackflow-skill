# TrackFlow integrations: checkout webhooks, CRMs, offline conversions, mapped webhooks, Google sign-in

Server-side sources create events without a browser. They go through the same pipeline as browser
events (blocking rules, dedup, attribution, delivery to every enabled destination's server API). All
tools are in the `integrations` group (`{"action": "...", "site_id": N, ...}`), except
`google_ads_sign_in_link` (group `destinations`).

| Source | Use for | Origin in `list_events` | Signed? | Tools |
|---|---|---|---|---|
| Checkout integration | Payments/bookings on Stripe, PayPal, Shopify, Lemon Squeezy, Paddle, Gumroad, ThriveCart, SamCart, WooCommerce, Calendly | `webhook` | Yes (per provider) | `list_checkout_providers`, `create/update/delete_checkout_integration`, `list_checkout_integrations`, `test_checkout_payload`, `get_checkout_log` |
| CRM integration | Pipeline stages → events (HubSpot, GoHighLevel, Pipedrive, Salesforce, Zoho, generic) | `crm` | Provider signature or shared secret | `list_crm_providers`, `create/update/delete_crm_integration`, `list_crm_integrations`, `test_crm_payload`, `sync_crm_integration`, `get_crm_log` |
| Offline conversions | Store/phone/back-office sales: CSV, batch API, Google Sheet / CSV URL on a schedule | `offline` | API key (batch API) | `import_offline_conversions`, `list_offline_imports`, `create/update/delete_offline_source`, `list_offline_sources`, `sync_offline_source` |
| Mapped webhook ("simple") | Any app that POSTs JSON (Typeform, Tally, Zapier, a backend) | `webhook` | **No** | `create/update/delete_webhook_integration`, `list_webhook_integrations` |

Rule of thumb: for payment providers always prefer a **checkout integration** (signature check, order
dedup, alternate-id memory) over a mapped webhook; for CRMs use a **CRM integration**.

---

## 1. How server events reach the ad platforms (stitching + dedup)

### Visitor matching (attribution)

For every server-side event TrackFlow picks the visitor in this order:

1. **Passthrough visitor id** (`vid`) found in the payload (Stripe `client_reference_id`, `metadata.tf_vid`,
   Shopify cart attribute `_tf_vid`, CRM field `tf_vid`, CSV column `vid`…). Used only if that visitor
   exists on the site (merchants' own ids in the same field are ignored).
2. Else the most recent visitor with the same **hashed email**, then **hashed phone**.
3. Else no visitor: the event still goes out with the hashed contact details it carries.

A matched event **inherits the visitor's stored click ids** (`fbc`, `fbp`, `gclid`, `gbraid`, `wbraid`,
`ttclid`, `msclkid`…), IP, user agent and earlier hashed identifiers, so the platform can tie it to the
ad click. Results show this as `vid` + `matched_by` (`vid`/`passthrough`, `email`, `phone`).

To get the visitor id into third-party checkouts, turn on link decoration (`update_container_settings
decorate_checkouts`, then publish; see tag-manager.md) or have the developer pass `trackflow('vid')` into
the provider's passthrough field.

### Dedup

| Source | event_id | Effect |
|---|---|---|
| Checkout integration | `purchase_<order id>` (other event names: `<eventname lowercased>_<order id>`; Calendly: `calendly_<invitee uuid>`) | Same id as the browser thank-you Purchase (`purchase_{{order_id}}` tag, or the GA4 dataLayer bridge `purchase_<transaction_id>`) → counted once. Alternate ids of one payment (Stripe session/payment intent/invoice, PayPal order/capture) are remembered, so the second webhook is a `duplicate`. |
| CRM | `crm_<integration id>_<object id>_<stage>` | Each stage of each deal counts once (webhook retries, poll after webhook). |
| Offline | `offline_<order id>`, else a hash of the row | Same order in a later file / twice in one file → `duplicate`. |
| Mapped webhook | value at `mapping.event_id_path`, else **random** | Without `event_id_path`, provider retries are double counted. |

Server dedup key = `event_id` + `event_name` per site. **The browser and server copies only dedup if
both the id and the event name are identical.** Make the thank-you tag's order id equal the id the
provider's webhook uses (table in §2 "Thank-you transaction_id").

### action_source (Meta)

| Source | action_source |
|---|---|
| Checkout | `website` when a user agent is known (payload or matched visitor), else `system_generated` |
| CRM | `config.action_source`, default `system_generated` |
| Offline | row `action_source`, default `physical_store` (rows with a Meta `lead_id` → `system_generated`) |

---

## 2. Checkout integrations

Webhook URL format: `https://<trackflow-host>/hooks/checkout/co_<token>` (returned as `webhook_url`).

| `provider` | Secret to ask for (`secret`) | Webhook events to select | Visitor passthrough | Thank-you `transaction_id` that dedups |
|---|---|---|---|---|
| `stripe` | Endpoint signing secret `whsec_…` (5-min tolerance) | `checkout.session.completed`, `checkout.session.async_payment_succeeded`; optional `payment_intent.succeeded`, `invoice.paid` | `client_reference_id=<vid>` (Payment Links accept it in the URL) or `metadata.tf_vid`; PaymentIntents: `metadata.tf_vid` | Checkout Session id (`cs_…`, `{CHECKOUT_SESSION_ID}`) |
| `paypal` | **Webhook ID** (not a secret key) | `CHECKOUT.ORDER.COMPLETED`, `PAYMENT.CAPTURE.COMPLETED` | Orders v2 `purchase_units[].custom_id=<vid>` | PayPal order id (`onApprove data.orderID`) |
| `shopify` | Webhook signing secret (Settings → Notifications → Webhooks) or app client secret | `orders/create` **or** `orders/paid` (pick one) | Cart attribute `_tf_vid` (optional `_tf_fbc`, `_tf_fbp`, `_tf_gclid`); landing-URL click ids are read too | Order id (`checkout.order.id`), not the order name `#1001` (unless `order_id_field`) |
| `lemonsqueezy` | Signing secret you set | `order_created` | `?checkout[custom][tf_vid]=<vid>` | Order id |
| `paddle` | Notification destination secret `pdl_ntfset_…`; optional `api_key` to look up the customer email | `transaction.completed` | `Paddle.Checkout.open({customData:{tf_vid}})` | Transaction id `txn_…` |
| `gumroad` | Optional: your `seller_id` | Ping (sale) | Product URL `?tf_vid=<vid>` | `sale_id` |
| `thrivecart` | Secret word (`thrivecart_secret`) | `order.success` | `?passthrough[tf_vid]=<vid>` | `order_id` |
| `samcart` | Optional shared secret, appended as `?secret=…` to the Notify URL | `Order` | `?tf_vid=<vid>` (decoration adds `utm_content`, which this integration also reads) | Order id |
| `woocommerce` | Webhook secret (WooCommerce → Settings → Advanced → Webhooks) | `order.created`, `order.updated` | Order meta `_tf_vid` (developer saves the `_tf_vid` cookie on checkout) | Order id |
| `calendly` | Webhook signing key (3-min tolerance) | `invitee.created` | Link/embed `?salesforce_uuid=<vid>` or `utm_content`/`utm_term` | Event id `calendly_<invitee uuid>` (= the `booking` recipe) |

`list_checkout_providers` returns this live (docs link, `secret_label`, `api_label`, `events`, `passthrough`).

### `config` keys

| Key | Providers | Meaning |
|---|---|---|
| `event_name` | all | Event created (default `Purchase`; Calendly `Schedule`). |
| `accept_test` | all | `true` = also track test/sandbox payments (default: ignored). |
| `order_id_field` | stripe (`payment_intent`), shopify (`name` / `order_number`), woocommerce (`number`) | Which id becomes the order id. |
| `default_currency`, `default_country` | all | Used when the payload has none. |
| `renewals`, `renewal_event_name` | stripe | Track `invoice.paid` subscription renewals (default off; name default Purchase). |
| `subscription_event_name` | stripe | Rename first subscription checkouts. |
| `payment_intents` | stripe | Use every `payment_intent.succeeded`, not only those with `metadata.tf_vid`. |
| `paid_statuses` | woocommerce | Default `["processing","completed"]`. |
| `sandbox` | paddle | Mark events as test / use the sandbox API. |
| `value_divisor` | thrivecart | Divide totals (ThriveCart sends hundreds). |
| `event_types` | samcart | Notification types to use (default `["Order"]`). |
| `vid_field`, `value` | calendly | Tracking field holding the vid; fixed value per booking. |

Built-in handling: Stripe amounts in minor units are converted (zero- and three-decimal currencies
respected); unpaid sessions wait for `async_payment_succeeded`; cancelled Shopify orders, refunded/
disputed Gumroad sales, unpaid Lemon Squeezy orders and Woo orders outside `paid_statuses` are skipped.
An order with no vid, email, phone, IP or click id is ignored.

### Setup steps (MCP)

1. Ask the customer: which provider, which event name (usually Purchase), currency if the provider
   doesn't send one, and whether test payments should count.
2. `list_checkout_providers` → read `secret_label` and `events`.
3. `create_checkout_integration {"site_id": N, "provider": "stripe", "config": {"event_name": "Purchase"}}`
   → give the customer `webhook_url` and the exact events to select in the provider dashboard.
4. Providers that issue the secret after the endpoint exists (Stripe, Shopify…): ask them to paste it,
   then `update_checkout_integration {"integration_id": X, "secret": "<pasted>"}`. Confirm only the last
   4 characters. **Without a secret, signature verification is off** (`signature_verification: "off"`):
   anyone who knows the URL can post orders. Always set it.
5. Stitching: `update_container_settings decorate_checkouts: ["stripe"]` (or the provider) + publish, or
   give the developer the passthrough instruction from the table.
6. `test_checkout_payload {"integration_id": X, "payload": {...}, "headers": {"x-shopify-topic": "orders/create"}}`
   → shows `would_create` with `order_id`, `event_id`, `value`, `currency`, `vid`, `matched_by`,
   `action_source`, `identifiers`, `click_ids` (signature not checked, nothing stored).
7. After a real payment: `get_checkout_log` and `list_events {"origin": "webhook"}`.

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
| | `duplicate` | Same payment already tracked (alternate id, or browser thank-you page / earlier webhook with the same event id). |
| | `ignored` | `reason`: not a used event type, test mode, unpaid/cancelled, no order id, no identifier. |
| | `would_create` | Dry run only. |
| Log entry (`get_checkout_log`, last 100) | `ok` / `duplicate` / `ignored` | Summary of the delivery's results. |
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

Other platform windows: GA4 72 h (older skipped for GA4), OpenAI 7 days, Microsoft 90 days; AppLovin,
Taboola, Outbrain can't use offline rows (deliveries `skipped` with the reason).

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

## 5. Mapped ("simple") webhooks

`create_webhook_integration {"event_name": "Lead", "provider": "typeform", "mapping": {...}}` → returns
`webhook_url` (`https://<trackflow-host>/hooks/<token>`). `provider` is only a label (`stripe`,
`calendly`, `typeform`, `shopify`, `woocommerce`, `cal`, `tally`, `generic`); the mapping does the work.

Mapping keys (dot paths into the JSON body): `email_path`, `phone_path`, `name_path`, `first_name_path`,
`last_name_path`, `value_path`, `value_divisor` (100 for cents), `currency_path`, `default_currency`,
`event_id_path` (**set it**: otherwise each retry is a new event), `visitor_path` (TrackFlow vid →
joins the journey), `url_path`, `fbc_path`, `fbp_path`, `filter_path` + `filter_value` (accept only
matching bodies). No signature check; `update_webhook_integration` replaces `mapping` as a whole.

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
| Purchases counted twice (thank-you tag + webhook) | Event ids differ. Thank-you tag must use `purchase_<same order id as the webhook>` (Stripe: session id `cs_…`; Shopify: order id not `#1001`; PayPal: order id). Or keep only one source. |
| Webhook events have no ad attribution | No passthrough vid and the email isn't known from the browser. Enable `decorate_checkouts` (and publish), pass `trackflow('vid')` to the provider, or capture the email earlier (form `identify`). |
| Test payments not tracked | Expected: test/sandbox mode ignored unless `config.accept_test: true` (log reason says so). |
| All webhooks `rejected` | Wrong secret type (PayPal needs the **Webhook ID**; Stripe needs the endpoint's `whsec_`, a different one per endpoint), or the secret was rotated. |
| Shopify webhook log full of `duplicate` | Both `orders/create` and `orders/paid` registered. Same order id → same event id, so it still counts once, but register only one topic. |
| CRM stage changes ignored | Stage not in `stage_map` (often internal ids), or custom map without `keep_defaults`. Check `get_crm_log` reasons. |
| CRM events ignored "No email, phone…" | Webhook carries ids only: add `api_token`, or include contact fields / `mapping`. |
| Offline rows rejected | Purchase without value, rows >90 days, bad currency, no identifier. Always `dry_run` first. |
| Sheet sync errors "Got an HTML page" | Sheet not public; share as "Anyone with the link" or publish to web as CSV. |
| Mapped webhook duplicates | `event_id_path` missing. |
| Server events never reach a destination | Destination in browser-only mode or missing server fields → event status `no_destination` (see reports-and-audit-data.md). |
