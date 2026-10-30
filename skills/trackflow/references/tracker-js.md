# TrackFlow browser tracker (`t.js`)

The tracker is one async script. It loads the connected platforms' browser tags (Meta Pixel, TikTok
Pixel, gtag, …), gives every event one `event_id`, fires the browser tags with that ID and sends the
same event to TrackFlow, which forwards it server-side (Conversions APIs) with the same ID. That shared
ID is what lets each platform deduplicate the browser and server copies.

Everything below is the externally visible behaviour of the current tracker (`trackflow.version === '2.0'`).

---

## 1. The install snippet

Always get the tag from the tool, never type it by hand:

```json
install {"action": "get_install_snippet", "site_id": 12, "mode": "standard"}
```

Returns `{ mode, src, global, html, notes[] }` (or `{ mode, error }`). Give the customer `html` exactly.

| `mode` | What `html` looks like | JS global | Blocker resistance | Use when |
|---|---|---|---|---|
| `standard` | `<script async src="https://TRACKFLOW_HOST/t.js" data-site="YOUR_SITE_KEY"></script>` | `trackflow` | Lowest (well-known path on the TrackFlow domain) | Simplest install; on-page code already calls `trackflow(...)` |
| `stealth` | queue stub + `<script async src="https://TRACKFLOW_HOST/<prefix>/<file>.js"></script>` | random (e.g. `qx7abc`) unless stealth is off | Random path, but the TrackFlow domain can still be blocked | No custom domain yet, want some resistance |
| `custom_domain` | queue stub + `<script async src="https://data.example.com/<prefix>/<file>.js"></script>` | random unless stealth is off | High (first-party subdomain, server-set cookies) | After a custom domain is `active` ([first-party-domain.md](first-party-domain.md)) |
| `proxy` | queue stub + `<script async src="/<prefix>/<file>.js"></script>` | random unless stealth is off | Highest (same origin) | Only with a working same-origin proxy (`get_proxy_config`) |
| `auto` (default) | `custom_domain` if a domain is `active`, else **`stealth`** | random unless stealth is off | | |

**Critical:** stealth is **on by default** for every site's first-party settings. So `mode: "auto"`
(and `stealth` / `custom_domain` / `proxy`) returns a tag whose JS API is **not** `window.trackflow` but
a random global name, shown in the result's `global` field and in `notes`. `window.trackflow` is then
**undefined**. Every on-page call (`trackflow('track', …)`, theme snippets, GTM Custom HTML, the Shopify
cart-attribute snippet) must use that global name instead. To keep the name `trackflow`, either install
`mode: "standard"`, or set `install {"action": "update_first_party_settings", "site_id": 12, "stealth": false}`
and fetch the snippet again.

Non-standard snippets have the site key built into the script URL, so they carry no `data-site`
attribute. The standard tag needs `data-site` on the same `<script>` element (a `?k=YOUR_SITE_KEY`
query on the `src` also works).

### Where to paste

Inside `<head>` on **every** page (shared header/layout). `async` never blocks rendering. One copy per
page. Platform-specific steps: [install-cms.md](install-cms.md).

### The queue stub (calls before the script has loaded)

Non-standard snippets include it. With the standard tag, define it yourself before any inline call:

```html
<script>
  window.trackflow = window.trackflow || function () { (window.trackflow.q = window.trackflow.q || []).push(arguments); };
</script>
```

Queued calls are replayed in order once the tracker starts. The stub is harmless if the tracker has
already loaded (`||` keeps the real function). Replace `trackflow` with the stealth global when the
snippet has one.

### What happens on load

1. Reads the site key, the global name and its own base URL (it sends to the host it was loaded from).
2. **Double-install guard:** if a tracker for the same site key (or the same global) is already running
   on the page, this copy exits silently. Only the first copy to run tracks.
3. Fetches the site config (connected browser tags, settings, published tag container). Unknown site
   key (HTTP 404): nothing is ever sent. Config unreachable: runs with defaults (no pixels, no automatic
   events, explicit `track` calls still sent).
4. If `auto_pageview` is on, queues a `PageView` (flagged when the navigation was a reload).
5. Loads the tag engine if the site has published tags; loads the preview badge in preview mode.
6. Hooks `window.dataLayer` (see §6.4).
7. Checks consent. When allowed: sets IDs/cookies, loads each connected platform's browser plugin,
   then sends all queued events.

---

## 2. Global function and commands

`trackflow(command, ...args)` (or the stealth global). All commands are safe to call through the stub.

| Command | Signature | Returns | What it does |
|---|---|---|---|
| `track` | `trackflow('track', eventName, customData?, userData?, options?)` | event ID (string) once loaded; `undefined` through the stub | Sends one event: browser tags + server copy with one shared `event_id`. |
| `identify` | `trackflow('identify', userData)` | – | Attaches customer details to this visitor (kept in the browser and merged into every later event, also on later page loads). No event is sent. |
| `page` | `trackflow('page', customData?)` | – | Sends a `PageView` for the current URL and, if the URL changed, runs the container's page tags. Rarely needed (SPA routes are automatic). |
| `consent` | `trackflow('consent', true \| false)` | – | Stores the visitor's marketing-consent choice in the browser; overrides any detected CMP. `false` stops all tracking for that browser even when consent mode is off. |
| `vid` | `trackflow('vid')` | visitor ID or `''` | The TrackFlow visitor ID (`''` until tracking has started, e.g. before consent). Pass it to checkouts (Stripe `client_reference_id`, Shopify cart attribute `_tf_vid`, …). |
| `decorate` | `trackflow('decorate', url)` | URL string | The URL with the visitor ID added in the provider's passthrough field (needs the tag engine and `decorate_checkouts`; otherwise returns the URL unchanged). |
| `get` | `trackflow('get', variableName)` | value | Reads a tag-container variable. |
| `set` | `trackflow('set', name, value)` | – | Sets a runtime value that container variables can read. |
| `event` | `trackflow('event', name, data?)` | – | Fires container tags whose trigger is `custom_event` with that `name` (source `dom`). Does **not** send anything by itself. |

Properties: `trackflow.version` (`'2.0'`), `trackflow.debug` (boolean), `trackflow._tf` (true on the
real tracker), `trackflow.platform(type, impl)` (used by the platform plugins, not by site code).

### `track` arguments

| Argument | Type | Notes |
|---|---|---|
| `eventName` | string | Standard name (`Lead`, `Purchase`, …) or a custom one. **Case-sensitive**, max 64 chars (longer is cut). |
| `customData` | object | `value`, `currency`, `content_ids`, `contents`, … (§3). Keys whose value is `null`/`undefined`/`''` are dropped (`0` is kept). |
| `userData` | object | Same as calling `identify` first (§4). Pass `null` to skip. |
| `options.id` | string | Your own event ID (max 64 chars). Use an order/booking ID so a webhook or server event for the same action deduplicates. Default: random. |
| `options.cv` | boolean | Force this event to count (`true`) / not count (`false`) as a conversion in TrackFlow's own reports. Does not change what platforms receive. |

```js
// Lead with value and the visitor's details (hashed server-side)
trackflow('track', 'Lead', { value: 49, currency: 'USD' }, { em: 'jane@example.com', ph: '+1 415 555 0134' });

// Purchase that deduplicates with a checkout webhook for the same order
trackflow('track', 'Purchase',
  { value: 129.90, currency: 'EUR', content_ids: ['SKU-123', 'SKU-456'], content_type: 'product', num_items: 2, order_id: 'ORD-1042' },
  { em: 'jane@example.com' },
  { id: 'purchase_ORD-1042' });

// Custom event
trackflow('track', 'QuoteRequested', { value: 500, currency: 'USD' });
```

Send each conversion exactly once, **after** it succeeded (successful API response / confirmation
shown), not on the button click.

---

## 3. Custom data (`customData`) schema

Canonical keys (Meta naming). The server maps them to each platform's fields.

| Key | Type | Meaning / rules |
|---|---|---|
| `value` | number | Major units (`129.90`, not `12990`, not `"€129,90"`). Non-numeric strings are not converted into a value. |
| `currency` | string | ISO 4217, uppercased, 3 letters. **If `value` is sent without `currency`, `USD` is assumed.** |
| `content_ids` | string[] | Product IDs/SKUs. Must match the product catalog IDs for catalog/dynamic ads. |
| `contents` | `[{ id, quantity, item_price }]` | Per-line items. Preferred over `content_ids` for platforms that take line items; if absent they are built from `content_ids` with quantity 1. |
| `content_type` | string | `product` or `product_group`. |
| `content_name` | string | Product/page name. Never put PII here. |
| `content_category` | string | Product category. |
| `num_items` | number | Total quantity. |
| `search_string` | string | Search term for `Search`. |
| `order_id` | string | Order reference. Google Ads, GA4 and others use it as transaction ID for purchase dedup. |
| `predicted_ltv`, `status`, `tax`, `shipping`, `plan_id` | various | Read by some platform adapters when present. |

Not mapped from `track()`: GA4-style `items[]`, `transaction_id`, `item_id`. Use `contents`,
`content_ids` and `order_id`. (GA4 ecommerce *dataLayer pushes* are translated automatically; §6.4.)

---

## 4. User data (`userData` / `identify`)

Raw values are sent to TrackFlow over HTTPS and normalised + SHA-256 hashed **on the server** before
storage or forwarding. 64-char hex values are treated as already hashed.

| Key | Meaning | Also accepted |
|---|---|---|
| `em` | email | `email`, `e_mail`, `mail` |
| `ph` | phone (include country code when you can) | `phone`, `tel`, `telephone`, `mobile`, `phone_number` |
| `fn` / `ln` | first / last name | `first_name`, `firstname`, `fname`, `given_name` / `last_name`, `lastname`, `lname`, `surname`, `family_name` |
| `name` | full name, split on the first space into `fn` + `ln` | |
| `ct` | city | `city` |
| `st` | state/region | `state`, `region` |
| `zp` | ZIP/postcode | `zip`, `zipcode`, `postal_code`, `postcode` |
| `country` | country (names become ISO codes) | `country_code` |
| `db` | date of birth `YYYYMMDD` | `dob`, `birthday` |
| `ge` | gender `f`/`m` | `gender` |
| `external_id` | your user/customer ID | `user_id` |

- Keys are case-insensitive; unknown keys are ignored.
- If no `external_id` is given, the server uses a hash of the visitor ID.
- Identified data persists for the visitor: in the browser (`localStorage` key `_tf_ud`, raw until it
  reaches the server) and on the server (hashed), so later events (e.g. a Purchase on a later page)
  carry it too.
- When the Meta plugin is loaded, known details are also passed to the Pixel as advanced matching.

---

## 5. Standard event names

Recognised as standard by the browser tags: `PageView`, `ViewContent`, `Search`, `AddToCart`,
`AddToWishlist`, `InitiateCheckout`, `AddPaymentInfo`, `Purchase`, `Lead`, `CompleteRegistration`,
`Contact`, `CustomizeProduct`, `Donate`, `FindLocation`, `Schedule`, `StartTrial`, `SubmitApplication`,
`Subscribe`. Any other name is sent as a custom event. Names are translated per platform automatically
(e.g. TikTok `Purchase` → `CompletePayment`). `lead` ≠ `Lead`. Only `Purchase` and `Lead` count as
conversions in TrackFlow reports by default (change with `set_conversion_events`).

---

## 6. Automatic behaviour

New sites have `auto_pageview`, `auto_forms` and `ecommerce_datalayer` **off**. Turn on with
`sites → update_site {"settings": {...}}` only when the customer wants that tracking. Settings reach
visitors within about a minute (config is cached briefly).

### 6.1 PageView (`auto_pageview`)

- One `PageView` per page load, sent first.
- A reload's PageView is flagged; with `block_reloads` (default on) the server stores it as `blocked`
  ("Page reload") **and the browser tags skip it too**, so platforms never get a browser copy without its
  server twin.
- SPA route changes: `pushState`, `replaceState`, back/forward (`popstate`) and `hashchange` are watched.
  A new PageView is sent only when the URL actually changes; plain `#section` anchors are ignored, hash
  routes starting `#/` or `#!/` count. Several history calls in one tick count once. The referrer of the
  new page is the previous URL. Meta's own automatic history PageView is disabled to avoid duplicates.
- Without `auto_pageview`, route changes still run the container's `page_view` / `history_change` tags.

### 6.2 Form identity capture (`auto_forms`)

On every `<form>` submit (capture phase), reads fields and calls `identify` with what it recognises:
email (only if it looks like an address), phone, first/last/full name, ZIP, city, state, country. Fields
are recognised by `type`, `autocomplete`, and words in `name`/`id`/placeholder/`aria-label`/label text.
Never read: password, hidden, file, checkbox, radio, and anything whose hints mention card, CVC, IBAN,
SSN, account number, security, expiry, routing. **It does not send an event**: a Lead needs a
`form_submit` tag or a `track` call.

### 6.3 Click IDs, UTMs and cookies

On each page (once tracking is allowed) the tracker stores, on the registrable domain
(`.example.com`, so all subdomains share them; `SameSite=Lax`, `Secure` on https):

| Cookie | Lifetime | Content |
|---|---|---|
| `_tf_vid` | long-lived (also mirrored to `localStorage`) | visitor ID |
| `_tf_sid` | 30 min rolling | session ID |
| `_fbp` | 90 days | Meta browser ID (created by TrackFlow if missing) |
| `_fbc` | 90 days | Meta click ID, built from `fbclid` in the URL |
| `_tf_<id>` | 90 days | from URL params `gclid`, `gbraid`, `wbraid`, `dclid`, `msclkid`, `ttclid`, `ScCid` (→`_tf_sccid`), `epik`, `twclid`, `rdt_cid`, `li_fat_id`, `qclid`, `obclid`, `OutbrainClickId` (→`_tf_obclid`), `tblci`, `irclickid`, `oppref`, `ndclid`, `aleid`, `alart` |

Every server copy carries `fbp`, `fbc`, `ttp` (TikTok `_ttp`), the stored click IDs and first-party IDs
the platform tags keep (e.g. Snap, Pinterest, Microsoft). Cookies blocked: an in-memory fallback is used
for the page.

UTMs are not stored separately: the server classifies source/medium/campaign from the page URL
(`utm_*`, `fbclid`, `gclid`/`gbraid`/`wbraid`, `ttclid`, `msclkid`) and referrer, and keeps the
visitor's last meaningful (non-direct) touch for attribution.

With a custom domain or same-origin proxy, the server also sets these cookies via HTTP headers
(plus HttpOnly backups); see [first-party-domain.md](first-party-domain.md).

### 6.4 GA4 ecommerce dataLayer bridge (`ecommerce_datalayer`)

The tracker wraps `dataLayer.push` and also reads pushes made before it loaded.

| dataLayer `event` | Sent as | Counted as conversion by the bridge |
|---|---|---|
| `view_item` | `ViewContent` | no |
| `search`, `view_search_results` | `Search` (`search_string` from `search_term`) | no |
| `add_to_cart` | `AddToCart` | yes |
| `add_to_wishlist` | `AddToWishlist` | no |
| `begin_checkout` | `InitiateCheckout` | yes |
| `add_payment_info` | `AddPaymentInfo` | yes |
| `purchase` | `Purchase` | yes |
| `generate_lead` | `Lead` | yes |
| `sign_up` | `CompleteRegistration` | yes |

From `ecommerce`: `value` (or Σ `price × quantity` of `items`), `currency` (default `USD`), `content_ids`
(`item_id`/`id`/`sku`/`item_name`), `contents`, `num_items`, `content_type: product`, `content_name` /
`content_category` for single-item pushes, `order_id` from `transaction_id`.

- **Purchase event ID = `purchase_<transaction_id>`**, and a browser-side list of sent order IDs stops
  a reopened thank-you page from firing the Pixel again. A checkout webhook for the same order uses the
  same ID, so the order counts once — only if `transaction_id` equals the order ID the webhook reports.
- Customer details in the same push are used: `user_data` (incl. `user_data.address`), `billing_info`,
  `customer`, `ecommerce.customer`, `ecommerce.billing` with keys `email`, `phone`/`phone_number`,
  `first_name`, `last_name`, `full_name`/`name`, `city`, `state`/`region`, `postal_code`/`zip`,
  `country`/`country_code`, `customer_id`/`user_id`.
- `{ ecommerce: null }` resets are ignored (no `event`).
- Only pushes with a string `event` are read. Non-ecommerce events can still fire `custom_event` tags.

```js
dataLayer.push({ ecommerce: null });
dataLayer.push({
  event: 'purchase',
  ecommerce: { transaction_id: 'ORD-1042', value: 129.90, currency: 'USD',
    items: [{ item_id: 'SKU-123', item_name: 'Olive oil', price: 64.95, quantity: 2 }] },
  user_data: { email: 'jane@example.com', phone_number: '+15125550134' }
});
```

### 6.5 Consent

With `consent_mode` on, nothing is sent, no pixels load and no IDs/cookies are set until consent is
granted. Detected automatically: Cookiebot (marketing), CookieYes (advertisement), OneTrust (`C0004`),
Complianz (marketing), Google Consent Mode (`ad_storage` via `dataLayer`). Re-checked on CMP events and
every 1.5 s. Declined: queued events are discarded. Revoked later: browser tags are told (Meta `revoke`).
Any other banner: call `trackflow('consent', true/false)`. Server/webhook events are not filtered by
consent mode.

### 6.6 Bots, reloads, blocking

The config carries the server's bot pattern; for bot user agents the browser tags are skipped and the
server stores the event as `blocked` ("Bot traffic"). Headless/automated browsers (Puppeteer,
Playwright, Selenium, HeadlessChrome) count as bots, so **test in a normal browser**. Custom blocking
rules (IP, URL, user agent, referrer, event name): [first-party-domain.md](first-party-domain.md) ("Blocking rules and data hygiene").

### 6.7 Cross-domain

With `cross_domains` set (`update_site`), clicks on links to those domains (and their subdomains) get
`?_tfx=<vid>.<sid>.<fbc>`; the tracker on the other domain continues the visitor/session and removes the
parameter from the address bar. Same snippet and **same site key** on every domain. Subdomains of one
root domain need nothing. Server redirects, form posts and `window.open` are not decorated.

### 6.8 Declarative tracking

There is no `data-*` attribute API in the tracker itself. Clicks, forms, scroll, visibility, video and
dataLayer events are tracked with **tags** (tag manager). Adding attributes such as
`data-track="signup"` to elements is still useful: a click tag can target `[data-track="signup"]`.

---

## 7. Browser pixels and event-ID sharing

For each enabled destination in **Browser + server** or **Browser only** mode, the tracker loads that
platform's plugin (`<base>/p/<type>.js`), which loads the vendor's official tag with the configured ID.
Every event reaches each plugin with the **same ID** that goes to the server.

| Platform | Browser tag | Dedup key passed |
|---|---|---|
| Meta | `fbevents.js` | `eventID` (event name + ID) |
| TikTok | `events.js` | `event_id` |
| Google Ads | gtag.js | `transaction_id` = `order_id` or event ID |
| GA4 | gtag.js | none (GA4 has no browser/server dedup; see ad-platforms.md) |
| Microsoft UET | `bat.js` | `event_id` |
| Snapchat | Snap Pixel | `client_dedup_id` |
| Pinterest | Pinterest Tag | `event_id` |
| LinkedIn | Insight Tag | `event_id` |
| Reddit | Reddit Pixel | `conversionId` |
| X | `twq` | event ID + `conversion_id` |
| Quora, Spotify, Nextdoor, OpenAI, AppLovin | vendor pixel | `event_id` / `dedupe_id` |
| Taboola | `_tfa` | `orderid` + click ID |
| Outbrain | `obApi` | none (no pixel↔S2S dedup ID) |

Meta specifics: uses `trackSingle` / `trackSingleCustom` so it **only fires TrackFlow's pixel ID**, never
other pixels already on the page; `autoConfig` (automatic button/microdata events without IDs) is off
unless the destination enables it; the pixel's automatic SPA PageView is disabled.

Server side, TrackFlow stores and forwards each `event_id` + event name **once per site** (`dedup`,
default on). A repeat returns `"duplicate": true` and is not sent again.

---

## 8. Debugging

| Tool | How |
|---|---|
| Console log | Add `?tf_debug=1` to any page URL; every send, identify and consent decision is logged with prefix `[trackflow]`. Per page load. |
| Debug events | While debug is on, the tracker dispatches `trackflow:debug` `CustomEvent`s on `window` (`config`, `consent`, `plugin`, `pixel`, `event`, `skipped`, `log`). |
| Network tab | Expect the script, the config request, `p/<platform>.js` plugins and POSTs to the collect endpoint (`sendBeacon`, fallback `fetch`). Config/collect 404 = wrong site key. `ERR_BLOCKED_BY_CLIENT` = ad blocker. |
| Debugger extension | TrackFlow Debugger (Chrome, Developer mode, Load unpacked): install check, per-event browser pixel vs server delivery comparison, unlinked pixel requests (duplicate pixels). Download from the TrackFlow docs (Tagging → Debugger). |
| Preview | `publishing → create_preview_link` returns a URL with `?tf_preview=<token>`: this browser session loads the **draft** container, shows a preview badge, and every event is flagged `test`. `?tf_preview=off` leaves preview. |
| Test delivery | Test-flagged events are delivered **only to destinations that have a test code / test mode set**; others record them as skipped. |
| Server view | `reports → list_events` (`test: "only"` for preview traffic), `get_event` for the exact payload and each platform's response. |
| Is it installed? | `get_site` → `last_seen_at` is updated when the config loads or a browser event arrives. |

Limits: collect body 32 KB; about 120 events per minute per IP per site (HTTP 429 beyond); URL 1000
chars; event name and event ID 64 chars.

---

## 9. Common mistakes

| Mistake | Effect | Fix |
|---|---|---|
| Calling `trackflow(...)` with a stealth / custom-domain snippet | `ReferenceError` or calls silently queued to a function nobody reads | Use the `global` name from `get_install_snippet`, or turn stealth off |
| Old Meta/TikTok pixel left in theme, plugin, app or GTM | Every browser event counted twice (different IDs, no dedup) | Remove it, or set the TrackFlow destination to Server only |
| Two TrackFlow tags with different site keys on one page | Both track | One site key per website |
| Calling `track` from the standard tag's inline script before it loads, without the stub | `trackflow is not defined` | Add the queue stub |
| `value` as text (`"$49"`) or in cents | Wrong/missing revenue | Number in major units |
| `value` without `currency` | Silently reported as USD | Always send both |
| Firing Purchase on button click / every thank-you reload | Inflated conversions | Fire after success; pass `{ id: 'purchase_<order id>' }` |
| Browser Purchase ID ≠ webhook ID for the same order | Order counted twice | Same `purchase_<order id>` on both sides |
| `lead` vs `Lead` | Custom event, not the standard Lead | Exact standard names |
| Expecting `auto_forms` to send a Lead | Only identity is captured | Add a `form_submit` tag or `track('Lead')` |
| GA4 `items` passed to `track()` | Products not mapped | Use `contents` / `content_ids` (or the dataLayer bridge) |
| `ecommerce_datalayer` on **and** a recipe/tag sending the same events | Double events | One source per event |
| Testing with an ad blocker / headless browser | Nothing arrives / everything "Bot traffic" | Normal browser, private window, no extensions |
| PII in event names, `content_name`, URLs | PII leaks unhashed | PII only in user data |
| React StrictMode effects in development | Event fired twice in dev | Fire from handlers, or pass a stable `options.id` |
| Preview events expected in ad accounts | Skipped without a test code | Set the destination's test event code while testing, then remove it |
