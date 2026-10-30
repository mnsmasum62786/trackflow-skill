# Installing TrackFlow per platform

How to get the tag onto a site and how to fire real ecommerce and lead events on each platform.
Tracker API details: [tracker-js.md](tracker-js.md). Tags/recipes: [tag-manager.md](tag-manager.md).
Checkout webhooks: [integrations.md](integrations.md).

Platforms with a TrackFlow guide (dashboard Install page and docs): **WordPress, WooCommerce, Shopify,
Webflow, Framer, Squarespace, Wix, Next.js, Custom HTML**, plus Google Tag Manager. GoHighLevel,
ClickFunnels and generic React/Vue SPAs have no dedicated TrackFlow guide; the steps below for them are
general platform knowledge, marked as such.

---

## 0. Before you touch the site (every platform)

1. **Get the snippet**: `install {"action": "get_install_snippet", "site_id": N, "mode": "…"}`.
   - `standard` → `<script async src="https://TRACKFLOW_HOST/t.js" data-site="YOUR_SITE_KEY"></script>`,
     JS global `trackflow`. Simplest; matches every example in this file.
   - `auto` → stealth or custom-domain loader with a **random JS global** (field `global`). If you use
     it, replace `trackflow` in every snippet below with that name. See
     [first-party-domain.md](first-party-domain.md).
   Hand the customer the returned `html` verbatim.
2. **Find existing pixels**: `tag_manager {"action": "inspect_site", "site_id": N, "url": "https://example.com/"}`
   on home, product, checkout and thank-you pages. Ask the customer which apps/plugins installed Meta,
   TikTok, Google, etc.
3. **Decide one source per event.** TrackFlow's browser tag fires the platform pixel for every
   connected destination in Browser + server mode. A pixel also installed by a theme, app, plugin,
   sales channel or GTM fires its own events with different IDs → the platform counts conversions twice.
   Choose per platform:
   - remove/disable the old pixel (best), **or**
   - keep the old pixel and set the TrackFlow destination to **Server only (API)** mode (then remove the
     old pixel's duplicate *server* events, if any), **or**
   - keep the old setup and don't send the same events through TrackFlow.
4. Turn on only the automatic tracking the customer wants (`update_site` → `auto_pageview`,
   `auto_forms`, `ecommerce_datalayer`); everything else is tags.
5. After install: `get_site` → `last_seen_at` set (the tag loaded), then `list_events`.

### Universal event snippets (any platform that lets you add JavaScript)

> **Function name:** every snippet in this file calls `trackflow(...)`, which exists only with the
> `standard` tag (or `stealth: false`). With a stealth / custom-domain / proxy tag, replace `trackflow`
> with the `global` name returned by `get_install_snippet` everywhere, including the PHP and Next.js
> examples below.

Use real values from the page/order; never hard-code amounts. Queue stub first if the tag may not have
loaded yet.

```html
<script>
  window.trackflow = window.trackflow || function () { (window.trackflow.q = window.trackflow.q || []).push(arguments); };

  // Product page
  trackflow('track', 'ViewContent', { content_ids: ['SKU-123'], content_type: 'product', content_name: 'Trail runner 2', value: 59.95, currency: 'EUR' });

  // Add to cart (call from the add-to-cart success handler)
  trackflow('track', 'AddToCart', { content_ids: ['SKU-123'], contents: [{ id: 'SKU-123', quantity: 1, item_price: 59.95 }], content_type: 'product', value: 59.95, currency: 'EUR' });

  // Checkout started
  trackflow('track', 'InitiateCheckout', { content_ids: ['SKU-123', 'SKU-456'], num_items: 2, value: 129.90, currency: 'EUR' });

  // Order confirmation: once per order; ID shared with any webhook for the same order
  trackflow('track', 'Purchase',
    { value: 129.90, currency: 'EUR', content_ids: ['SKU-123', 'SKU-456'], content_type: 'product', num_items: 2, order_id: 'ORD-1042' },
    { em: 'customer@example.com', ph: '+49 151 23456789', fn: 'Jane', ln: 'Doe', zp: '10115', country: 'DE' },
    { id: 'purchase_ORD-1042' });

  // Lead form: from the form's success callback, not the submit click
  trackflow('track', 'Lead', { value: 0, currency: 'USD', content_name: 'Contact form' }, { em: 'lead@example.com', ph: '+1 415 555 0134' });
</script>
```

Without code access, the same events come from **tags** (no site edits): `apply_recipe` `ecommerce`
(storefronts without a dataLayer; WooCommerce/Shopify-aware selectors), `lead_form` (every successful
form), `booking` (Calendly/Cal.com), `newsletter`, or individual `page_view` / `click` / `form_submit`
tags. If the store already pushes a **GA4 ecommerce dataLayer**, `ecommerce_datalayer: true` is enough.

---

## 1. Plain HTML / static site / any template

1. Paste the snippet inside `<head>` of the shared layout/header include so every page has it.
2. Keep `data-site` on the same tag (standard mode). Don't change `async`.
3. Use the universal snippets for events; for purchases prefer a server source (webhook / server API).

```html
<head>
  <meta charset="utf-8">
  <title>Example</title>
  <script async src="https://TRACKFLOW_HOST/t.js" data-site="YOUR_SITE_KEY"></script>
</head>
```

---

## 2. WordPress

**Install (pick one):**
- Plugin (recommended, survives theme updates): WPCode → *Code Snippets → Header & Footer* → paste in
  **Header** → Save.
- Child theme `functions.php`:
  ```php
  add_action('wp_head', function () {
    echo '<script async src="https://TRACKFLOW_HOST/t.js" data-site="YOUR_SITE_KEY"></script>' . "\n";
  }, 1);
  ```

**Leads:** form builders are detected by `form_submit` tags with presets: Contact Form 7 (`cf7`),
Elementor Pro (`elementor`), Gravity Forms (`gravity`), WPForms (`wpforms`), Ninja (`ninja`), Fluent
(`fluent`), Formidable (`formidable`), HubSpot (`hubspot`). `apply_recipe lead_form` covers all of them
(newsletter/search/login/cart/checkout forms excluded). Turn on `auto_forms` for match quality.

**Pitfalls:**
- Remove "Meta Pixel for WordPress", PixelYourSite, TikTok plugins, Site Kit ads tags or page-builder
  pixels for platforms TrackFlow now loads (or switch those destinations to Server only).
- Caching/optimisation plugins (WP Rocket, LiteSpeed, Autoptimize, Perfmatters): clear cache; exclude
  the TrackFlow script URL from "delay JavaScript" / combine / defer features, or the tag fires late or
  never.
- Consent plugins: Complianz, CookieYes, Cookiebot are auto-detected when `consent_mode` is on.

---

## 3. WooCommerce

1. Install the tag as for WordPress.
2. **Purchases, server-side (recommended):** checkout integration for WooCommerce
   (`integrations` → `list_checkout_providers`, then `create_checkout_integration` with provider
   `woocommerce`). Give the customer the returned webhook URL:
   *WooCommerce → Settings → Advanced → Webhooks → Add webhook*: Status **Active**, Topic **Order
   created** (and/or **Order updated**), Delivery URL = TrackFlow URL, a **Secret** → paste the same
   secret into TrackFlow (`update_checkout_integration`). Counted when status is `processing` or
   `completed`. Event ID = `purchase_<order ID>`.
3. **Tie orders to the ad click** (save the visitor ID on the order):
   ```php
   // functions.php or a snippets plugin
   add_action('woocommerce_checkout_create_order', function ($order) {
     if (!empty($_COOKIE['_tf_vid'])) $order->update_meta_data('_tf_vid', sanitize_text_field($_COOKIE['_tf_vid']));
   });
   // Block (Store API) checkout: use the woocommerce_store_api_checkout_update_order_meta hook instead.
   ```
4. **Funnel events (ViewContent, AddToCart, InitiateCheckout):**
   - If a GA4/GTM plugin already pushes GA4 ecommerce events → `ecommerce_datalayer: true`.
   - Otherwise `apply_recipe ecommerce` (knows WooCommerce product, cart, checkout and order-received
     selectors).
5. **Optional browser Purchase on the order-received page** (only if you want a browser copy too). Use
   the **same ID as the webhook** so they deduplicate:
   ```php
   add_action('woocommerce_thankyou', function ($order_id) {
     $order = wc_get_order($order_id);
     if (!$order || $order->get_meta('_trackflow_sent')) return;
     $order->update_meta_data('_trackflow_sent', 1); $order->save();
     $custom = [
       'value' => (float) $order->get_total(), 'currency' => $order->get_currency(),
       'content_ids' => array_values(array_map(fn($i) => (string) $i->get_product_id(), $order->get_items())),
       'num_items' => $order->get_item_count(), 'order_id' => (string) $order->get_order_number(),
     ];
     $user = [ 'em' => $order->get_billing_email(), 'ph' => $order->get_billing_phone(),
       'fn' => $order->get_billing_first_name(), 'ln' => $order->get_billing_last_name(),
       'ct' => $order->get_billing_city(), 'zp' => $order->get_billing_postcode(), 'country' => $order->get_billing_country() ];
     ?>
     <script>
       window.trackflow = window.trackflow || function () { (window.trackflow.q = window.trackflow.q || []).push(arguments); };
       trackflow('track', 'Purchase', <?php echo wp_json_encode($custom); ?>, <?php echo wp_json_encode($user); ?>,
         { id: <?php echo wp_json_encode('purchase_' . $order->get_id()); ?> });
     </script>
     <?php
   });
   ```
   The TrackFlow docs' older version uses the order *key* as ID; that only deduplicates with a mapped
   webhook whose `event_id_path` is `order_key`, **not** with the checkout integration
   (`purchase_<order ID>`). If the integration's `order_id_field` is `number`, use the order number.
   If the dataLayer bridge handles `purchase`, its ID is `purchase_<transaction_id>`: make sure the
   plugin's `transaction_id` is the same order ID, and don't add this PHP snippet as well.

**Pitfalls:** Facebook for WooCommerce / TikTok for WooCommerce / PixelYourSite also fire Purchase →
disable their pixel or events. Orders paid later (bank transfer, redirects that skip the thank-you
page) are only caught by the webhook.

---

## 4. Shopify

**Install (storefront):** *Online Store → Themes → ⋯ → Edit code → Layout → `theme.liquid`* → paste the
snippet on the line just before `</head>` → Save.

**Checkout and thank-you limitation:** Shopify's checkout and order-status ("Thank you") pages do not
render `theme.liquid`. With Checkout Extensibility, third-party code there can only run as an app pixel
or a custom pixel under *Settings → Customer events*, inside a sandboxed iframe. **TrackFlow ships no
Customer Events pixel**; do not paste the TrackFlow tag into Customer Events (it would run in the sandbox,
not on the storefront page). Track purchases **server-side**:

1. `create_checkout_integration` with provider `shopify` → webhook URL.
2. Customer: *Settings → Notifications → Webhooks → Create webhook*, Event **Order payment**
   (`orders/paid`) or **Order creation** (`orders/create`) — pick one, Format **JSON**, URL = TrackFlow.
   Copy the signing secret shown under the webhooks list into TrackFlow
   (`update_checkout_integration`). Cancelled orders are skipped; test orders only with
   `config.accept_test: true`. Event ID = `purchase_<order id>`.
3. Stitch orders to the ad click: save the visitor ID as cart attribute `_tf_vid` (it lands in the
   order's `note_attributes`). Add after the TrackFlow tag in `theme.liquid`:
   ```html
   <script>
   (function () {
     var G = 'trackflow';               // or the stealth global from get_install_snippet
     var n = 0;
     (function save() {
       var tf = window[G], vid = tf && tf._tf ? tf('vid') : '';
       if (!vid) { if (++n < 40) setTimeout(save, 500); return; }   // waits for load and consent
       fetch('/cart.js').then(function (r) { return r.json(); }).then(function (cart) {
         if (cart.attributes && cart.attributes._tf_vid === vid) return;
         return fetch('/cart/update.js', { method: 'POST', headers: { 'Content-Type': 'application/json' },
           body: JSON.stringify({ attributes: { _tf_vid: vid } }) });
       }).catch(function () {});
     })();
   })();
   </script>
   ```
   Click IDs in the order's landing URL and the order's browser IP/user agent are used too.

**Funnel events on the storefront:** `apply_recipe ecommerce` (Shopify product/cart/AJAX add-to-cart
aware), or Liquid on the product template, e.g.:
```liquid
{% if template contains 'product' %}
<script>
  window.trackflow = window.trackflow || function () { (window.trackflow.q = window.trackflow.q || []).push(arguments); };
  trackflow('track', 'ViewContent', {
    content_ids: ['{{ product.selected_or_first_available_variant.id }}'], content_type: 'product',
    content_name: {{ product.title | json }},
    value: {{ product.selected_or_first_available_variant.price | divided_by: 100.0 }},
    currency: {{ cart.currency.iso_code | json }}
  });
</script>
{% endif %}
```
(Shopify prices are in cents, hence `divided_by: 100.0`. `content_ids` must use the same ID format as
the Meta/TikTok catalog, which for Shopify-synced catalogs is often not the bare variant ID; check the
catalog.)

**Pitfalls:**
- Shopify's **Facebook & Instagram** and **TikTok** sales channels install their own pixel (and often
  their own server events) that also report Purchase. Disconnect that pixel / set data sharing to the
  lowest level, or keep it and don't connect the same platform in TrackFlow. Otherwise double counting.
- Remove pixel code pasted in `theme.liquid` or in custom pixels under Customer events.
- Google & YouTube channel / GA4 apps: same rule for Google Ads/GA4 destinations.
- Theme switch = reinstall the snippet in the new theme.

---

## 5. Wix

1. *Settings → Custom code* (under *Advanced* / *Development & integrations*) → **+ Add Custom Code**.
2. Paste the snippet, name it "TrackFlow", **All pages**, **Load code once**, place in **Head** → Apply.
3. Requires a premium plan and a connected domain; custom code does not run in the Editor preview →
   test on the published site.

**Events:** Wix forms → `form_submit` preset `wix` (or `lead_form` recipe). Wix Stores checkout is
Wix-hosted; TrackFlow has **no Wix Stores integration**. Options: a `page_view` tag on the
thank-you/order-confirmation URL (value only if readable from the page), or send purchases from your own
backend with the server events API. Wix also exposes store events to custom code through its own
analytics API; that is a Wix feature, untested with TrackFlow — verify with `?tf_debug=1` before relying
on it.

**Pitfall:** Wix *Marketing Integrations* (Facebook Pixel & CAPI, TikTok, Google) fire their own events →
disconnect them, or set the TrackFlow destinations to Server only.

---

## 6. Squarespace

1. *Settings → Advanced → Code Injection* (newer dashboards: *Settings → Developer tools → Code
   injection*) → **Header** field → paste → Save. Needs a plan with code injection.
2. Test in a private window on the live site (injection may not run while logged in/editing).

**Purchases (Squarespace Commerce):** the same screen has an **Order Confirmation Page** field. It
accepts the JavaScript API, and Squarespace replaces order placeholders there (for example `{orderId}`,
`{orderGrandTotal}`, `{customerEmailAddress}` — confirm the exact names and number formats in
Squarespace's current docs before using them):
```html
<script>
  window.trackflow = window.trackflow || function () { (window.trackflow.q = window.trackflow.q || []).push(arguments); };
  trackflow('track', 'Purchase', { value: Number('{orderGrandTotal}'), currency: 'USD', order_id: '{orderId}' },
    { em: '{customerEmailAddress}' }, { id: 'purchase_{orderId}' });
</script>
```
Set the real store currency. **Leads:** `form_submit` preset `squarespace`.
**Pitfall:** Squarespace's built-in Facebook/Google integrations (Marketing → …) duplicate pixels.

---

## 7. Webflow

1. *Site settings → Custom code → Head code* → paste → Save changes.
2. **Publish**. Custom code only runs on the published site (paid site plan required).

**Leads:** native forms → `form_submit` preset `webflow` (waits for the `.w-form-done` success state).
Buttons: give them an ID/class or `data-track` attribute in the Designer and track with a click tag.
**Ecommerce:** no TrackFlow-specific Webflow Ecommerce integration; use `apply_recipe ecommerce` /
page tags, or the server API from your backend.

---

## 8. Framer

*Site Settings → General → Custom Code → End of `<head>` tag* → paste → Publish. Framer navigates
without full reloads; route changes are tracked automatically. Forms: preset `framer`.

---

## 9. GoHighLevel (no TrackFlow guide — general platform knowledge)

- Funnels/websites: open the funnel/website **Settings → Tracking Code → Header code** → paste → Save
  (applies to every step). Per-step tracking code also exists; use the funnel-wide header to avoid
  double installs.
- Leads: put a `page_view` tag with condition on the thank-you step URL (`page_path` equals
  `/thank-you`), event `Lead`, or a `form_submit` tag (verify success detection in preview; GHL forms
  embedded on other sites are iframes and cannot be read from the parent page).
- Pipeline stages / closed deals: TrackFlow's CRM integration supports **GoHighLevel**
  ([integrations.md](integrations.md)).
- `inspect_site` detects GoHighLevel pages. Remove the Facebook/TikTok pixel IDs from GHL's own
  integration settings if TrackFlow loads those pixels.

## 10. ClickFunnels (no TrackFlow guide — general platform knowledge)

- ClickFunnels 2.0: site/funnel **Settings → Tracking code → Header code** → paste.
- Purchases: `page_view` tag on the order-confirmation step (read value/order ID from the page or
  URL with variables), or a server source. ClickFunnels checkout links are recognised by `inspect_site`.
- Remove pixels added in ClickFunnels' own tracking-code boxes for platforms TrackFlow now loads.

---

## 11. Next.js (App Router / Pages Router)

```tsx
// app/layout.tsx (App Router)
import Script from 'next/script';
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Script src="https://TRACKFLOW_HOST/t.js" data-site="YOUR_SITE_KEY" strategy="afterInteractive" />
      </body>
    </html>
  );
}
// pages/_app.tsx (Pages Router): same <Script … strategy="afterInteractive" /> after <Component {...pageProps} />
```

For a stealth/custom-domain snippet: put its `<script async src=…>` URL in `src` (no `data-site`) and
its stub inline: `<Script id="tf-stub" strategy="beforeInteractive">{`window.G=window.G||function(){(window.G.q=window.G.q||[]).push(arguments)};`}</Script>`
with `G` = the returned global.

Calling from client components:
```js
'use client';
export function trackflow(...args) {
  if (typeof window === 'undefined') return;
  window.trackflow = window.trackflow || function () { (window.trackflow.q = window.trackflow.q || []).push(arguments); };
  return window.trackflow(...args);
}
// <button onClick={() => trackflow('track', 'Lead', { value: 49, currency: 'USD' })}>Get the guide</button>
```

Client-side navigation is tracked automatically (one PageView per new URL when `auto_pageview` is on;
`page_view` tags re-run). Server-only events (renewals, refunds): server events API from a route handler,
API key in an env var, never in the browser.

## 12. React / Vue / Angular / other SPAs (general)

- Snippet in `index.html` `<head>` (Vite/CRA/Vue CLI/Angular `src/index.html`).
- Don't call `trackflow('page')` on every route change — the History API is already watched (that would
  double PageViews). Use `page` only for step changes that don't change the URL.
- Fire conversions from success handlers, not `useEffect` on mount (React StrictMode runs effects twice
  in development). For idempotency pass `{ id: 'purchase_<order id>' }`.
- Hash routers: `#/route` counts as a new page; `#section` does not.

## 13. Google Tag Manager

TrackFlow does not need GTM, but can be loaded through it:
1. New tag → **Custom HTML** → paste the snippet unchanged (including `data-site`, or the stub + loader
   for stealth/custom-domain snippets) → trigger **All Pages** (or **Initialization – All Pages**) →
   publish the GTM container.
2. **Pause** GTM's Meta Pixel / TikTok / other pixel tags for platforms TrackFlow now loads.
3. GA4 ecommerce events already pushed to `dataLayer` → `ecommerce_datalayer: true` (no GTM tags
   needed). Custom `dataLayer` events can trigger TrackFlow `custom_event` tags (`source: datalayer`).
4. Sending a custom event from GTM:
   ```html
   <script>
     window.trackflow = window.trackflow || function () { (window.trackflow.q = window.trackflow.q || []).push(arguments); };
     trackflow('track', 'Schedule', { value: {{Booking Value}}, currency: 'EUR' });
   </script>
   ```
**Pitfalls:** when GTM itself is blocked (ad blockers) or held back by a consent setup, TrackFlow loaded
through it is blocked too; a direct `<head>` install is more robust. Don't install both hard-coded and
via GTM: the second copy is skipped by the double-install guard, but whichever loads first wins.

## 14. Hosted checkouts and booking tools (Stripe, PayPal, Paddle, Lemon Squeezy, Gumroad, ThriveCart, SamCart, Calendly, Cal.com)

The thank-you page often never runs your scripts. Use a checkout integration (signed webhook) and
`update_container_settings {"decorate_checkouts": true}` so checkout/booking links carry the visitor ID.
Details: [integrations.md](integrations.md).

---

## 15. Duplicate-pixel checklist (do this on every install)

| Source of a second pixel | Where to turn it off |
|---|---|
| Theme/template code | Search the theme for `fbq(`, `ttq.`, `gtag(`, `snaptr(`, `pintrk(` |
| WordPress plugins | Meta Pixel for WP, PixelYourSite, Facebook for WooCommerce, TikTok for WooCommerce, Site Kit |
| Shopify | Facebook & Instagram / TikTok / Google channels; Customer events custom pixels |
| Wix / Squarespace | Built-in marketing integrations |
| GTM | Pixel tags in the container |
| GHL / ClickFunnels | Pixel ID fields and tracking-code boxes |

The TrackFlow Debugger lists pixel requests not linked to a TrackFlow event — that is the duplicate.

## 16. Verify

1. `get_site` → `last_seen_at` recent (tag loads).
2. Open the live site in a normal browser (no ad blocker) → `list_events` shows the events.
3. Trigger each conversion once with real-looking data → `get_event`: value, currency, order_id, user
   data present; deliveries `delivered`.
4. Meta/TikTok Test Events with the destination's test code → "deduplicated" for browser + server.
   Remove test codes afterwards.
5. Webhook purchases: `get_checkout_log` / `test_checkout_payload`.
