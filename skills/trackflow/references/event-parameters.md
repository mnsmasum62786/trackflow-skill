# TrackFlow event model, parameters and send / don't-send rules

Source of truth: TrackFlow's own behaviour (adapters, browser plugins, ingestion). Items marked **[vendor]** are general
ad-platform knowledge, not something TrackFlow enforces; check the platform's current docs before relying on them.

Contents: 1 event names · 2 event → platform mapping · 3 custom_data schema · 4 property → platform fields ·
5 user data & hashing · 6 click IDs & automatic fields · 7 send / don't send · 8 dedup · 9 event time · 10 test events ·
11 standard vs custom events

---

## 1. Canonical event names

TrackFlow's canonical names **are Meta's standard events**. Use them everywhere (JS API, tags, server API, webhooks,
offline/CRM); every adapter translates them. Names are **case-sensitive** (`lead` ≠ `Lead`), max 64 chars; an empty name
becomes `PageView`.

`PageView` `ViewContent` `Search` `AddToCart` `AddToWishlist` `InitiateCheckout` `AddPaymentInfo` `Purchase` `Lead`
`CompleteRegistration` `Contact` `CustomizeProduct` `Donate` `FindLocation` `Schedule` `StartTrial` `SubmitApplication`
`Subscribe`

Also recognised by some adapters: `Download` (TikTok standard; custom on Meta). Anything else is a **custom event**.

**Lifecycle events** (created by store/checkout/booking webhooks, never by tags): `OrderRefunded`, `OrderCancelled`,
`SubscriptionRenewed`, `SubscriptionCancelled`, `BookingCancelled`, `BookingNoShow`, with `custom_data.lifecycle`
(`refund`, `cancel`, `chargeback`, `renewal`, `subscription_cancel`, `booking_cancel`, `no_show`), `order_id`, a positive
`value` and for refunds `partial`, `remaining_value`, `original_value`. They **never count as conversions** and reach
a platform only when its destination has `send_lifecycle` on. Don't send these names from the browser. Details:
[lifecycle-events.md](lifecycle-events.md).

Which names count as "conversions" in TrackFlow reports is a per-site setting (default `Purchase`, `Lead`). It never
changes what is sent to ad platforms.

Ways events enter TrackFlow (all go through the same pipeline):

| Source | Event ID | Notes |
|---|---|---|
| Browser tracker `trackflow('track', name, customData?, userData?, {id?, cv?})` | `options.id` or random | Fires browser pixels **and** a server copy with the same ID |
| Container tags (`send_event` action, built via MCP) | tag's `event_id` template or random | Parses prices like `$1,299.00` and symbols → ISO currency; an ID already fired in this browser is not re-sent |
| Shopify Customer Events code (`get_install_snippet` mode `shopify_pixel`) | `purchase_<numeric Shopify order id>` for purchases | Shopify's own events → `PageView` (needs `auto_pageview`), `ViewContent`, `Search`, `AddToCart`, `InitiateCheckout`, `AddPaymentInfo`, `Purchase` (need `ecommerce_datalayer` or a published tag with that name), extras `ViewCategory`, `ViewCart`, `AddShippingInfo` (container setting `shopify_events`) |
| GA4 ecommerce dataLayer bridge (site setting, off for new sites) | `purchase_<transaction_id>` for purchases | Maps `view_item`→ViewContent, `add_to_cart`→AddToCart, `begin_checkout`→InitiateCheckout, `add_payment_info`→AddPaymentInfo, `purchase`→Purchase, `generate_lead`→Lead, `sign_up`→CompleteRegistration, `search`/`view_search_results`→Search, `add_to_wishlist`→AddToWishlist |
| Server events API `POST /api/v1/events` (Bearer API key) | `event_id` or random | Fields: `site_key`, `event_name`, `event_id`, `url`, `referrer`, `user_data`, `custom_data`, `visitor_id`, `ip`, `user_agent`, `fbc`, `fbp`, `click_ids{}`; one event per request, ≤ 64 KB |
| Webhooks / checkout / CRM / offline sources | mapped field or provider-prefixed random | Can carry `event_time` (backdated) and `action_source` |

`action_source` values TrackFlow understands: `website` (default), `physical_store`, `phone_call`, `system_generated`,
`email`, `chat`, `app`, `other`. A webhook event with no URL defaults to `system_generated`.

---

## 2. Event name mapping (TrackFlow → platform)

Browser and server copies always get the **same** translated name, so dedup works. "—" = not sent to that platform
(skipped on both sides). "slot" = only sent if you assign it a custom slot in `custom_events` (Nextdoor 1–10). "snake_case" = the TrackFlow name converted (`StartTrial` → `start_trial`).

| TrackFlow | Meta | TikTok | GA4 | Microsoft (UET action) | Snapchat | Pinterest API (Tag) | Reddit (pixel) | Quora | OpenAI | Nextdoor CAPI (pixel = UPPER) | Taboola (default) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| PageView | PageView | `Pageview` ¹ | `page_view` | pageLoad ² | `PAGE_VIEW` | `page_visit` (`pagevisit`) | `PAGE_VISIT` (`PageVisit`) | `ViewContent` (browser only) | `page_viewed` | `page_view` | `page_view` (browser only) |
| ViewContent | ViewContent | ViewContent | `view_item` | `view_item` | `VIEW_CONTENT` | `view_content` | `VIEW_CONTENT` | ViewContent | `contents_viewed` | `view_content` | `view_content` |
| Search | Search | Search | `search` | `search` | `SEARCH` | `search` | `SEARCH` | Search | custom `search` | `search` | `search` |
| AddToCart | AddToCart | AddToCart | `add_to_cart` | `add_to_cart` | `ADD_CART` | `add_to_cart` | `ADD_TO_CART` | AddToCart | `items_added` | `add_to_cart` | `add_to_cart` |
| AddToWishlist | AddToWishlist | AddToWishlist | `add_to_wishlist` | `add_to_wishlist` | `ADD_TO_WISHLIST` | `add_to_wishlist` | `ADD_TO_WISHLIST` | AddToWishlist | custom | `add_to_wishlist` | snake_case |
| InitiateCheckout | InitiateCheckout | InitiateCheckout | `begin_checkout` | `begin_checkout` | `START_CHECKOUT` | `initiate_checkout` | CUSTOM | InitiateCheckout | `checkout_started` | `initiate_checkout` | `start_checkout` |
| AddPaymentInfo | AddPaymentInfo | AddPaymentInfo | `add_payment_info` | `add_payment_info` | `ADD_BILLING` | `add_payment_info` | CUSTOM | AddPaymentInfo | custom | slot | snake_case |
| Purchase | Purchase | `CompletePayment` | `purchase` | `purchase` | `PURCHASE` | `checkout` | `PURCHASE` | Purchase | `order_created` | `purchase` | `make_purchase` |
| Lead | Lead | Lead | `generate_lead` | `generate_lead` | `SIGN_UP` | `lead` | `LEAD` | `GenerateLead` | `lead_created` | `lead` | `lead` |
| CompleteRegistration | CompleteRegistration | CompleteRegistration | `sign_up` | `sign_up` | `SIGN_UP` | `signup` | `SIGN_UP` (`SignUp`) | CompleteRegistration | `registration_completed` | `sign_up` | `complete_registration` |
| Contact | Contact | Contact | `contact` | `contact` | — | `contact` | CUSTOM | Generic | custom | slot | snake_case |
| Schedule | Schedule | Schedule | `schedule` | `schedule` | `RESERVE` | `schedule` | CUSTOM | Generic | `appointment_scheduled` | slot | snake_case |
| StartTrial | StartTrial | StartTrial | `start_trial` | `start_trial` | `START_TRIAL` | `start_trial` | CUSTOM | Generic | `trial_started` | slot | snake_case |
| Subscribe | Subscribe | Subscribe | `subscribe` | `subscribe` | `SUBSCRIBE` | `subscribe` | CUSTOM | Generic | `subscription_created` | `subscribe` | snake_case |
| SubmitApplication | SubmitApplication | SubmitApplication | `submit_application` | `submit_application` | `SIGN_UP` | `submit_application` | CUSTOM | Generic | custom | slot | snake_case |
| CustomizeProduct / FindLocation | same | same (custom) | snake_case | snake_case | — | `customize_product` / `find_location` | CUSTOM | Generic | custom | slot | snake_case |
| Donate | Donate | Donate (custom) | `donate` | `donate` | — | `custom` | CUSTOM | Generic | custom | slot | snake_case |
| Custom name (e.g. `VideoWatched`) | custom event, same name | same name (custom) | `video_watched` | `video_watched` | — (unless a Snap name e.g. `CUSTOM_EVENT_1`, `LOGIN`) | `custom` | `CUSTOM` + `custom_event_name` | Generic (or skip) | `custom` + `custom_event_name: video_watched` | `custom_conversion_n` if slotted | `video_watched` (or skip) |

¹ TikTok browser uses `ttq.page()` (no event ID) → server PageView is skipped while the pixel loads.
² Microsoft: UET records page loads itself; server PageView skipped while the tag loads.

Mapping-only platforms (no name translation; you map each TrackFlow event to an ID, unmapped = not sent on that side):

| Platform | Browser mapping field | Server mapping field | Format |
|---|---|---|---|
| Google Ads | `conversion_labels` (`Purchase=AbC-D_efG123`) | `conversion_actions` (`Purchase=987654321`) | one `Event=value` per line, comma/semicolon list or JSON; event lookup is case-insensitive |
| LinkedIn | `conversion_ids` (Insight Tag "Event-specific" conversion IDs) | `conversion_rules` (Conversions-API rule IDs or `urn:lla:llaPartnerConversion:<id>`) | same |
| X | `event_ids` (shared by pixel and API) | `event_ids` | `Event=tw-<pixel>-<event>` |
| Nextdoor | `custom_events` slots 1–10 | same | slot number |
| Taboola / Outbrain | `event_names` overrides (case-sensitive names configured in Realize / Amplify) | same | `Purchase=make_purchase` |
| Microsoft | `event_actions` overrides (`Lead=submit_lead_form`) | same | — |

LinkedIn conversion types per event (for choosing the rule type in Campaign Manager): PageView KEY_PAGE_VIEW, ViewContent
VIEW_CONTENT, Search SEARCH, AddToCart ADD_TO_CART, AddToWishlist ADD_TO_LIST, InitiateCheckout START_CHECKOUT,
AddPaymentInfo ADD_BILLING_INFO, Purchase PURCHASE, Lead LEAD, CompleteRegistration COMPLETE_SIGNUP, Contact CONTACT,
Donate DONATE, Schedule BOOK_APPOINTMENT, StartTrial START_TRIAL, SubmitApplication SUBMIT_APPLICATION, Subscribe SUBSCRIBE.
X event types: PageView "Page view", AddToCart "Add to cart", AddPaymentInfo "Added payment info", Lead, Purchase; anything
else maps to an X Custom event.

---

## 3. custom_data schema (what TrackFlow reads)

Empty values (`""`, `null`) are dropped before sending.

| Key | Type | Rule / TrackFlow behaviour |
|---|---|---|
| `value` | number | Major units (`129.90`, not cents). Converted with `Number()`. A **numeric string** (`"49.90"`) is fine; a string with symbols or separators (`"$1,299.00"`) is **not parsed** by the JS API or server API (only container tags parse prices). |
| `currency` | string | ISO 4217. Uppercased and validated: an invalid code (`"$"`, `"EURO"`) is dropped. **If `value` is set and the currency is missing or invalid, TrackFlow uses the site's `default_currency`, else `USD`.** Set `default_currency` with `update_site` for non-USD shops. |
| `content_ids` | string[] | Product IDs / SKUs; match catalog IDs for catalog/dynamic ads. Used to build items when `contents` is absent (quantity 1). |
| `contents` | `[{id, quantity, item_price}]` | Preferred item list (wins over `content_ids`). `item_price` = unit price. |
| `content_type` | string | `product` or `product_group`. |
| `content_name` | string | Product/page name (single-item name on GA4, OpenAI, Microsoft). |
| `content_category` | string | Category. |
| `num_items` | number | Total quantity; computed from `contents` quantities when absent (Snap, Pinterest, Reddit, Taboola, X). |
| `order_id` | string | Order reference. **Drives transaction-ID dedup** (Google Ads, GA4, Taboola, Snap/Pinterest/Reddit order fields). |
| `search_string` | string | Search term (Search events). |
| `status` | string | Registration status (Snapchat, X browser). |
| `predicted_ltv` | number | Snapchat, Pinterest. |
| `plan_id` | string | OpenAI subscription/trial and custom events. |
| any other key | — | **Sent verbatim to Meta only** (Meta passes the whole `custom_data`); other platforms ignore unknown keys. |

---

## 4. Property → platform field (server API copy)

| TrackFlow | Meta CAPI | TikTok Events API | Google Ads (Data Manager) | GA4 MP | Microsoft UET CAPI | Snapchat CAPI | Pinterest CAPI |
|---|---|---|---|---|---|---|---|
| value | `custom_data.value` | `properties.value` | `conversionValue` | `value` | `customData.value` (+`ecommTotalValue` on purchase) | `custom_data.value` | `custom_data.value` (string) |
| currency | `custom_data.currency` | `properties.currency` | `currency` | `currency` (USD default) | `currency` (USD default) | `currency` | `currency` |
| contents / content_ids | as sent | `contents[{content_id, quantity, price}]` | `cartData.items[{itemId, quantity, unitPrice}]` (+`merchantProductId`/`merchantId` with Merchant Center ID) | `items[{item_id, quantity, price}]` ≤ 200 | `itemIds`, `items[{id, quantity, price}]` | `content_ids`, `contents[{id, quantity, item_price}]` | `content_ids`, `contents[{id, quantity, item_price}]` |
| content_type | as sent | `content_type` | — | — | `pageType` (derived from event) | `content_type` | — |
| content_name / category | as sent | — | — | `items[0].item_name` / `item_category` (1 item) | `items[0].name` / `ecommCategory` | `content_name` / `content_category` | `content_name` / `content_category` |
| num_items | as sent | — | — | — | — | `num_items` (string) | `num_items` |
| order_id | as sent | `order_id` | `transactionId` = order_id ‖ event_id | `transaction_id` (Purchase: order_id ‖ event_id) | `transactionId` | `order_id` | `order_id` |
| search_string | as sent | `query` | — | `search_term` | `searchTerm` | `search_string` | `search_string` |

| TrackFlow | LinkedIn | Reddit | X | Quora | Taboola S2S | Outbrain S2S | Nextdoor | OpenAI |
|---|---|---|---|---|---|---|---|---|
| value | `conversionValue.amount` (only with currency) | `metadata.value` | `value` (2-dec string) | `conversion.value` (**account currency, no currency sent**) | `revenue` | `orderValue` | `custom.order_value` = `USD12.34` (needs currency) | `data.amount` (**integer minor units**) |
| currency | `currencyCode` | `metadata.currency` | `price_currency` | — | `currency` (supported list only) | `currency` (supported list only) | in `order_value` | `data.currency` (required for amount) |
| items | — | `metadata.products[{id, quantity, item_price, category}]`, `item_count` | `contents[{content_id, content_price, num_items}]`, `number_items` | — | `quantity` | — | `product_context[{id, quantity, item_price}]` | `data.contents[{id, quantity, amount, currency}]` ≤ 50 |
| order_id | — | `metadata.order_id` | — | — | `orderid` = order_id ‖ event_id | `orderId` | `custom.order_id` | — |
| search_string | — | — | `description: "search: …"` | — | — | — | — | — |

---

## 5. User data: keys, hashing, normalisation

All `user_data` is normalised and **SHA-256 hashed on TrackFlow's server**; only hashes are stored and forwarded. A value
that is already 64-char lowercase hex is treated as pre-hashed and passed through (normalise it yourself first).

| Key | Aliases accepted | Normalisation before hashing |
|---|---|---|
| `em` | `email`, `e_mail`, `mail` | trim, lowercase; **dropped if not a valid email** |
| `ph` | `phone`, `tel`, `telephone`, `mobile`, `phone_number` | digits only; country dial code added if the number starts with 0 or is a 10-digit US/CA number **and** `country` is known (supported: US CA GB AU NZ IE DE FR ES IT NL IN BD PK AE SA SG MY PH ZA NG BR MX); otherwise leading zeros stripped → **send E.164 (`+<cc><number>`)** |
| `fn`, `ln` | `first_name`/`firstname`/`fname`/`given_name`; `last_name`/`lastname`/`lname`/`surname`/`family_name` | trim, lowercase, NFKC, letters only |
| `name` | — | full name split into `fn` + `ln` on first space |
| `ct` | `city` | lowercase, a–z only (accents dropped: `Zürich`→`zrich`; send ASCII) |
| `st` | `state`, `region` | US state names → 2-letter code; otherwise first 2 letters → **send 2-letter codes** |
| `zp` | `zip`, `zipcode`, `postal_code`, `postcode` | lowercase, no spaces/hyphens, **first 5 chars** |
| `country` | `country_code` | ISO 3166-1 alpha-2; common full names mapped, **unrecognised names dropped** |
| `db` | `dob`, `birthday` | digits only → send `YYYYMMDD` |
| `ge` | `gender` | first letter (`f`/`m`) |
| `external_id` | `user_id` | trim, then hashed. If absent, TrackFlow uses SHA-256 of the TrackFlow visitor ID |

Platform-specific hash variants TrackFlow derives automatically (you send plain `em`/`ph`):

| Variant | Rule | Used by |
|---|---|---|
| `em` | trim + lowercase | Meta, TikTok, Snapchat, Pinterest, LinkedIn, X, Nextdoor, OpenAI |
| `em_g` | gmail/googlemail: dots removed from local part | Google Ads, GA4 |
| `em_r` | all domains: dots and `+tag` removed | Reddit, Microsoft |
| `em_gp` | gmail/googlemail: dots and `+tag` removed | Quora |
| `ph` | digits with country code, no `+` | Meta, TikTok, Snapchat, Pinterest, OpenAI |
| `ph_e164` | SHA-256 of `+<digits>` | Google Ads, GA4, Microsoft, Reddit, X |

Server-side identifier coverage per platform:

| Platform | Hashed identifiers sent | Raw identifiers sent |
|---|---|---|
| Meta | em ph fn ln ct st zp country db ge external_id | IP, UA, fbp, fbc, lead_id |
| TikTok | email, phone, external_id | IP, UA, ttclid, ttp |
| Google Ads | email (em_g), phone (ph_e164) — **no address server-side** | gclid/gbraid/wbraid, IP, UA |
| GA4 | sha256_email_address, sha256_phone_number | client_id, session_id, IP, UA |
| Microsoft | em (em_r), ph (e164) | msclkid, anonymousId (`_uetvid` or hashed visitor ID), IP, UA |
| Snapchat | em ph fn ln ge ct st zp country external_id | IP, UA, ScCid, `_scid` |
| Pinterest | em ph fn ln ge db ct st zp country external_id | IP, UA, epik |
| LinkedIn | email; first+last name (pair only) | li_fat_id, **IPv4 only** |
| Reddit | email (em_r), phone (e164), external_id | IP, UA, `_rdt_uuid`, click id (none for physical_store) |
| X | email, phone (e164) | twclid, `_twpid`, IP + UA |
| Quora | email only (phone/name/geo not sent) | IP, UA, referrer, qclid |
| Nextdoor | email fn ln ct st zp country db ge external_id (**no phone**) | IP, UA, ndclid |
| OpenAI | email, phone, external_id, fn, ln (geo browser-only) | IP, UA, oppref, obref |
| Taboola / Outbrain | none (click-ID postback only) | tblci / ob_click_id |

Browser side: the tracker keeps identified user data **raw in the visitor's own browser storage** and passes it to pixels
for advanced matching; the pixels hash it themselves (Meta, TikTok, Snap, Reddit, X, Nextdoor, Microsoft, Google
enhanced conversions, Pinterest email, Quora email on init). OpenAI and Taboola get browser-side SHA-256 hashes.
Auto form capture (site setting `auto_forms`) reads email/phone/name/city/state/zip/country fields and never password,
card, CVC, IBAN, SSN or similar fields.

---

## 6. Click IDs and automatically filled fields

TrackFlow fills these itself — do not ask the user for them:

| Field | Source |
|---|---|
| IP, user agent | the browser request (server API: `ip`, `user_agent`) — sent **unhashed** |
| `fbp` | `_fbp` cookie (the tracker creates it if missing) |
| `fbc` | `_fbc` cookie, or built from `?fbclid=` as `fb.1.<ms>.<fbclid>` |
| landing-URL click IDs, kept 90 days as first-party cookies | `gclid gbraid wbraid dclid msclkid ttclid ScCid epik twclid rdt_cid li_fat_id qclid obclid tblci irclickid oppref ndclid OutbrainClickId` |
| pixel cookies read by plugins | `_ttp`, `_scid`, `_epik`, `_uetvid`, `_rdt_uuid`/`_rdt_cid`, `_twclid`/`_twpid`, `li_fat_id`, `quora_qclid`, `__oppref`/`__obref`, `_ga` client/session id, gclid from gtag |
| `external_id` | hashed TrackFlow visitor ID when you send none |
| page URL, referrer | `event_source_url` / `page_location` etc. (raw) |

Server-side events (API, webhook, CRM, offline) with a `visitor_id` inherit that visitor's stored click IDs, IP and UA,
and identifiers learned earlier in the journey (e.g. a form email) are merged into later events.
Server API: click IDs (`fbc`, `fbp`, `ttclid`, `ttp`, `gclid`, `gbraid`, `wbraid`, `msclkid`, …) may be sent top-level or
inside `click_ids`; top-level wins.

---

## 7. Send / don't send

### Required and recommended per key event

| Event | Required | Strongly recommended | Notes |
|---|---|---|---|
| **Purchase** | `value` (number), `currency` (ISO 4217), `order_id` | `contents` (id, quantity, item_price) or `content_ids`, `content_type: product`, `num_items`; user data `em`, `ph` (E.164), `fn`, `ln`, `ct`, `st`, `zp`, `country`, `external_id` | Use an order-based event ID (`purchase_<order>` or `options.id`) wherever the same order can be reported twice (thank-you page + webhook). Fire once, after payment succeeded. |
| **Lead** | — | `em` and/or `ph`; `value`+`currency` if leads have a value; `content_name`/`content_category` for the form | Fire on confirmed submit (success message/redirect), not on button click. |
| **CompleteRegistration** | — | `em`, `ph`, `external_id` (your user ID); `status` optional | Snapchat/Reddit/GA4 treat it as sign-up. |
| **AddToCart** | `content_ids`/`contents` | `value`, `currency`, `content_type`, `num_items` | Value = price × qty added. |
| **InitiateCheckout** | — | `value`, `currency`, `contents`, `num_items` | |
| **AddPaymentInfo** | — | `value`, `currency` | Snap `ADD_BILLING`; Reddit as CUSTOM; Nextdoor only via a custom slot. |
| **ViewContent** | `content_ids` (catalog ads) | `content_type`, `content_name`, `content_category`, `value`, `currency` | |
| **Search** | `search_string` | — | |
| **Subscribe / StartTrial** | — | `value`, `currency`, `predicted_ltv` (Snap/Pinterest), `plan_id` (OpenAI) | |
| **Contact / Schedule / SubmitApplication** | — | `em`/`ph` | Limited coverage: not on Snap (Contact); Nextdoor needs a custom slot; Quora sends Generic. |
| **Custom events** | — | keep names stable, PascalCase or snake_case, ≤ 40 chars (GA4 limit) | See §11. |

Value rule of thumb: send `value` + `currency` **together on every event that has money attached**; never only one.

### Don't send

| Don't | Why (TrackFlow behaviour / platform rule) |
|---|---|
| Raw PII (email, phone, name, address) in `custom_data` | Only `user_data` is hashed. Meta receives `custom_data` verbatim → raw PII reaches Meta. **[vendor]** Meta and Google prohibit PII in event parameters. |
| PII in URLs (`?email=…`, `?phone=…`), page paths or referrers | URLs are stored and forwarded raw (`event_source_url`, GA4 `page_location`). **[vendor]** GA4 forbids PII collection. |
| PII in event names or custom parameter names | Names go to every platform as-is. |
| `value` as a formatted string (`"$1,299.00"`, `"1.299,00"`) via JS API / server API | Not parsed: Meta gets the raw string (likely rejected), other platforms drop the value. Send `1299.00`. |
| `value` in cents/minor units | Every platform expects major units (TrackFlow converts to minor units itself for OpenAI). |
| `value` without (valid) `currency` | Becomes the site's `default_currency` (else USD); set `default_currency` for non-USD shops. |
| Non-ISO currency (`"$"`, `"Tk"`, `"Euro"`) | Only uppercased + cut to 3 chars, never validated. Quora ignores currency (account currency); Taboola/Outbrain drop value for unsupported currencies. |
| Purchase without `order_id` / stable event ID | Reloaded thank-you pages and browser + webhook copies cannot be deduplicated; Google Ads/GA4 fall back to the random event ID. |
| The same conversion from two sources with different event IDs | TrackFlow and platforms count both (e.g. a thank-you-page tag plus a Stripe webhook). Pick one source or share the ID. |
| Phone numbers without country code (unless `country` is in the dial-code list) | Hash won't match. Send E.164. |
| Full country/state names, long postcodes | Unknown country names are dropped; states cut to 2 letters; zips cut to 5 chars. Send ISO-2 country, 2-letter state. |
| Health, financial, or other sensitive-category data (conditions, treatments, loan/credit details, religion, sexual orientation) in names, parameters or URLs | **[vendor]** Meta restricts sensitive data and limits event/parameter sharing for health & financial advertisers; Google bans it for personalised ads. |
| Events for visitors who declined marketing consent, from your server | Consent filtering applies only to browser events; server/API events are trusted. |
| Hand-installed copies of the same pixel (theme, plugin, GTM, Shopify/Wix channel) | Different event IDs → double counting. Remove them or set the destination to server-only. |
| A test code / test mode left on in production | While set, **all** server events go to the platform's test view or validator (see §10). |
| Meta `lead_id` unless the lead came from Meta Lead Ads | It is sent unhashed as the CRM lead ID. |

---

## 8. Deduplication

**Inside TrackFlow:** an event with the same `event_id` **and** `event_name` as one already stored for the site is
dropped as a duplicate (`{"duplicate": true}`) — no time limit, per site (setting `dedup`, default on). The browser
tracker gives the pixel and the server copy the same ID, so each action is stored and sent once.

**Order-based IDs:** dataLayer purchases use `purchase_<transaction_id>` and remember sent orders in the browser;
container tags with an `event_id` template refuse to re-fire an ID already sent from that browser; JS API `options.id`
and server `event_id` let you use the order ID. Use the **same ID string** on every source of the same order.

**Per platform** (browser ↔ server):

| Platform | Dedup key | Behaviour |
|---|---|---|
| Meta | `event_name` + `event_id` (`eventID` in pixel) | Every event incl. PageView sent from both sides. **[vendor]** Meta pairs copies received within 48 h. |
| TikTok | `event` + `event_id` | PageView browser-only while pixel loads. |
| Google Ads | `transaction_id` = order_id ‖ event_id, **per conversion action** | `click_import` (default): browser events with a label go via tag only; server sends only events the browser never saw. `multi_source` (allowlisted accounts only): uploads join the tag's action. |
| GA4 | none by event ID; `transaction_id` for purchases | While the tag loads, server sends only non-browser events (webhook, offline, CRM, API). |
| Microsoft | `eventId` + same `eventName` | PageView browser-only (UET pageLoad). |
| Snapchat | pixel `client_dedup_id` = CAPI `event_id`; purchase `transaction_id` = `order_id` | 48 h window. |
| Pinterest | `event_id` + same name | first copy within 48 h kept. |
| LinkedIn | `event_id` = `eventId` | Insight Tag copy kept; needs separate Insight-Tag and CAPI conversion rules. |
| Reddit | `conversionId` = `metadata.conversion_id` | same type, website channel, within 2 days. |
| X | Events Manager event id + `conversion_id` | only mapped events, both sides. |
| Quora | `event_id` | PageView browser-only. |
| Taboola | event name within click (`tblci`) context + `orderid` | S2S copy kept; S2S needs `tblci`. |
| Outbrain | none exists | each event sent one way: with Outbrain click ID → S2S only; without → pixel only. |
| Nextdoor / OpenAI | `event_id` (OpenAI keeps the first copy) | |

`retry_event_delivery` re-sends a stored event to every enabled destination; safe for ID-deduped platforms, but avoid
retrying already-delivered events to GA4 / Taboola / Outbrain (no event-ID dedup).

---

## 9. Event time windows

TrackFlow accepts `event_time` (ms, seconds or ISO) only from sources that support backdating (webhooks, CRM, offline);
values more than 90 days old or more than 60 s in the future are replaced with "now". The server events API has no
`event_time` field (events are stamped on arrival). Checkout Purchases are stamped with the time of the `purchase_on`
stage (paid / fulfilled / created); refunds and cancellations with the time they happened.

Platform limits checked by TrackFlow before every server delivery (older events → `skipped` "Too old for <Platform>
(max N days)"):

| Platform | Limit |
|---|---|
| GA4 | 72 h |
| Meta | 7 days (62 days for `physical_store`) |
| TikTok, Pinterest, Reddit, OpenAI | 7 days |
| Snapchat | 37 days |
| LinkedIn, Google Ads, Microsoft | 90 days (Google Ads value restatements: ≤ 54 days after the conversion) |
| Outbrain | events > 5 min old carry their own `timestamp` |

---

## 10. Test events

- `send_test_event` / `test_destination` create one **server** event flagged test (test email, placeholder IP/UA,
  optional USD value). It is delivered **only to destinations that have a test field set**; others report `skipped`.
- Preview links (`create_preview_link`) flag browser events as test; their **server** copies follow the same rule, but the
  **browser pixels still fire normally (live)**.
- While a test field is set, **every** server event of that destination (not only test ones) goes to test/validation.
  Remove it before going live.

| Platform | Test field | Effect |
|---|---|---|
| Meta | `test_event_code` | Events Manager → Test Events only |
| TikTok | `test_event_code` | Test Events only |
| Google Ads | `validate_only` | validated, nothing ingested |
| GA4 | `debug` | sent to `/debug/mp/collect`, nothing recorded (use it: `/mp/collect` returns 2xx even for bad payloads) |
| Snapchat | `test_event_code` / `validate_only` | test view / `/events/validate` |
| Pinterest | `test_mode` | `?test=true`, not recorded |
| Reddit | `test_id` | Event Testing |
| Nextdoor | `test_event` | marked as test |
| OpenAI | `validate_only` | checked, not saved |
| Quora | `debug` | **warnings only — events are still processed live** |
| LinkedIn, X, Microsoft, Taboola, Outbrain | none | test events never sent; verify with real traffic or the platform's own debugger |

---

## 11. Standard vs custom events

- Prefer a standard name whenever one fits: platforms optimise on them without extra setup and TrackFlow can translate them.
- Custom names reach: Meta (custom event — **[vendor]** turn it into a Custom Conversion in Events Manager to optimise on
  it; custom conversions can also be rules on standard events + URL/parameters), TikTok (same name), GA4/Microsoft
  (snake_case; **[vendor]** mark it as a key event / conversion goal), Pinterest (`custom`), Reddit (`CUSTOM`), OpenAI
  (`custom`), Quora (`Generic`), Taboola (snake_case). They need explicit mapping on Google Ads, LinkedIn, X,
  Nextdoor (slot 1–10), Taboola/Outbrain (if Realize/Amplify names differ). They are **never sent** to
  Snapchat (except Snap's own names like `CUSTOM_EVENT_1`).
- Google Ads has no event names: each conversion action needs its own label (browser) and action ID (server).
- Keep custom names short (GA4 truncates to 40 chars, must start with a letter) and free of PII.
