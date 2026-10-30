# Linking server orders to the ad click (visitor-id passthrough)

When an order, booking or lead arrives by webhook, TrackFlow must find the **visitor** who placed it: that
visitor holds the ad click ids (fbclid/fbc, gclid, ttclid…), browser, IP and journey. A matched server event
inherits all of them, so the platform credits the right campaign. The surest link is the TrackFlow visitor
id, **`tf_vid`**, carried inside the order.

The visitor id lives in the first-party cookie `_tf_vid` (also `localStorage`); page code reads it with
`trackflow('vid')` (stealth installs: `window.<global>('vid')`). It is a short code of letters, digits,
`_` and `-`. TrackFlow uses a passed-through value **only if a visitor with that id exists on the same
site**, so a wrong, stale or merchant-owned value in the same field does no harm.

Checkout integrations and their passthrough fields: [integrations.md](integrations.md). Refund and
cancellation events use the same matching: [lifecycle-events.md](lifecycle-events.md).

---

## 1. Matching order and `matched_by`

For every server-side event (checkout presets, mapped webhooks, lifecycle events) TrackFlow tries these in
order and stops at the first hit:

| # | Step | `matched_by` |
|---|---|---|
| 1 | Visitor id in the payload (passthrough field, cart attribute, order meta, `vid_path`) | `vid` |
| 2 | **Order binding**: an earlier event of the same order id (usually the thank-you page / Shopify Customer Events Purchase) or the stored order row already knows the visitor. Works without any passthrough when the thank-you page fired | `order` |
| 3 | **Provider ids** remembered from earlier webhooks of the same order (e.g. a Stripe refund carries only the payment intent) | `provider_ref` |
| 4 | Hashed **email**, then | `email` |
| 5 | hashed **phone** of a known visitor | `phone` |
| 6 | No visitor, but ad click ids found in the order's **landing URL** (Shopify `landing_site`, WooCommerce order-attribution entry URL, mapped `landing_url_path`) are attached to the event | `landing_url` |
| 7 | Nothing: the event still goes out with the hashed contact details it carries | `none` |

`matched_by` (plus `vid`) is shown by `test_checkout_payload`, `get_checkout_log` results,
`test_webhook_payload` and the `/hooks/checkout` response. Steps 1–2 give the best results; `email`/`phone`
only works when the buyer was identified in the browser before; `landing_url` and `none` mean weaker
attribution. Audit the mix (see [audit-playbook.md](audit-playbook.md)).

---

## 2. What to set up per checkout

| Checkout | Passthrough |
|---|---|
| Shopify | Automatic with the theme tag (cart attributes, §3) plus order binding from the Customer Events Purchase. See [install-cms.md](install-cms.md) §4 |
| Stripe Payment Links, Buy Button, Pricing Table; Lemon Squeezy, Paddle, Gumroad, ThriveCart, SamCart, PayPal Standard, Calendly, Cal.com | Automatic with link decoration: container setting `decorate_checkouts` (+ publish). Buy Button / Pricing Table get `client-reference-id` |
| Form / booking iframes and links: Typeform, Jotform, Tally, HubSpot, Zoho Forms, GoHighLevel | Automatic with `decorate_checkouts` (provider in the list, or `true`): the URL gets `tf_vid` (Typeform: hidden field in the hash / `data-tf-hidden`). The form needs a **hidden field named `tf_vid`** |
| Forms on the site's own pages (Gravity Forms, WPForms, Elementor, Webflow, HubSpot property…) | Automatic: add a hidden field named `tf_vid` (§3) |
| WooCommerce | PHP snippet (§4.1) |
| Stripe Checkout Sessions created by the merchant's server | Server code (§4.2) |
| PayPal Orders v2 | `purchase_units[].custom_id` (§4.3) |
| SSLCommerz | `value_a` + server report (§4.4) |
| Razorpay | `notes.tf_vid` (§4.5) |
| Whop | checkout configuration `metadata.tf_vid` |
| bKash, Nagad (no webhook) | Merchant's server reports the purchase (§5) |
| ClickFunnels 2.0, systeme.io, Teachable, Kajabi | No passthrough: order binding (thank-you page with the order id) and email/phone. ClickFunnels reads a contact custom attribute `tf_vid` when present |

Where a provider has one free-text field (PayPal `custom_id`, SSLCommerz `value_a`), write the visitor id
only if the merchant doesn't already use that field.

---

## 3. Automatic passthrough by the browser tag

- **Shopify cart attributes.** On Shopify storefronts the theme tag writes hidden cart attributes
  `_tf_vid`, `_tf_fbc`, `_tf_fbp`, `_tf_gclid` (via `/cart/update.js`, once a cart exists and tracking has
  consent). They reach the order's `note_attributes`, which the Shopify order webhook reads. Buyers don't see
  them. "Buy it now" / express checkouts can skip the cart: order binding and email cover most of those. No
  hand-written cart script is needed any more.
- **Hidden `tf_vid` inputs.** Any input named `tf_vid` (also `tfvid`, or a name ending in `/tf_vid`) is
  filled with the visitor id, including forms rendered later (popups, builders, SPAs) and same-origin
  iframes without `src`. Existing values are never overwritten; filled only after consent. The form tool's
  own webhook / CRM record then carries it: point the integration's visitor-id path (`vid_path`, CRM
  `mapping.vid_path`, or the CRM's auto-read `tf_vid` field) at it.
- **Link, embed and iframe decoration** (`decorate_checkouts`, a draft container setting: publish it).
  Links, Stripe embeds, Typeform embeds, the Calendly inline widget and provider iframes get the vid,
  also when added to the page later. An iframe that has **already loaded** keeps its URL (changing it
  reloads the form); set container setting `decorate_reload_iframes: true` to rewrite those too. Lazy
  `data-src` attributes are always decorated. Details: [tag-manager.md](tag-manager.md) §7.

---

## 4. Code for server-created checkouts (give it to the merchant or their developer)

Server code only sees the `_tf_vid` cookie when the request goes to the site's own domain. If the
checkout API runs elsewhere, send `trackflow('vid')` from the page instead. Placeholders only.

### 4.1 WooCommerce (classic + block checkout)

Saves the visitor id and Meta click cookies as order meta `tf_vid`, `tf_fbc`, `tf_fbp`; the WooCommerce
order webhook carries them (the older key `_tf_vid` is still read). Install with the free **Code
Snippets** plugin (paste without the first two lines, "Run snippet everywhere", Save and Activate) or as
the file `wp-content/mu-plugins/trackflow-vid.php` (theme/plugin updates can't remove it).

```php
<?php
defined('ABSPATH') || exit;

function trackflow_save_visitor_on_order($order) {
    $map = ['_tf_vid' => 'tf_vid', '_fbc' => 'tf_fbc', '_fbp' => 'tf_fbp']; // cookie => order meta key
    foreach ($map as $cookie => $meta_key) {
        if (empty($_COOKIE[$cookie]) || $order->get_meta($meta_key)) continue; // keep an existing value
        $value = substr(preg_replace('/[^A-Za-z0-9_.\\-]/', '', wp_unslash($_COOKIE[$cookie])), 0, 200);
        if ($value !== '') $order->update_meta_data($meta_key, $value);
    }
}
// Classic checkout
add_action('woocommerce_checkout_create_order', function ($order, $data) {
    trackflow_save_visitor_on_order($order);
}, 10, 2);
// Block checkout (Store API)
add_action('woocommerce_store_api_checkout_update_order_from_request', function ($order, $request) {
    trackflow_save_visitor_on_order($order);
}, 10, 2);
```

Check: place a test order, open it in WooCommerce → Orders with **Custom fields** on (Screen Options) →
`tf_vid` is there.

### 4.2 Stripe Checkout Sessions

| Field | Why |
|---|---|
| `client_reference_id` | Read on `checkout.session.completed` |
| `metadata.tf_vid` | Kept on the session |
| `payment_intent_data.metadata.tf_vid` | Payment mode: refunds and disputes can be traced too |
| `subscription_data.metadata.tf_vid` | Subscription mode: renewals and cancellations can be traced too |

```js
// Node.js (Express + cookie-parser + stripe)
const vid = String(req.cookies._tf_vid || '').replace(/[^A-Za-z0-9_-]/g, '').slice(0, 64);
const tf = vid ? { tf_vid: vid } : {};
const session = await stripe.checkout.sessions.create({
  mode: 'payment',
  line_items: [{ price: 'price_123', quantity: 1 }],
  success_url: 'https://example.com/thank-you?session_id={CHECKOUT_SESSION_ID}',
  client_reference_id: vid || undefined,
  metadata: tf,
  payment_intent_data: { metadata: tf }, // subscriptions: subscription_data: { metadata: tf }
});
```

### 4.3 PayPal Orders v2

Set `purchase_units[].custom_id = <vid>` when creating the order (server, or `custom_id: trackflow('vid')`
in the JS SDK `createOrder`). The PayPal integration reads it from the capture.

### 4.4 SSLCommerz

Add `'value_a' => $vid` to the session-init POST (use `value_b`–`value_d` if `value_a` is taken).
SSLCommerz returns it in the IPN / validation response. After the IPN handler has validated the payment
(status `VALID`), report the purchase as in §5 with `visitor_id = value_a` and
`event_id = purchase_<tran_id>`.

### 4.5 Razorpay

Put `notes: { tf_vid: <vid> }` on the order (server) or in the Checkout.js options
(`notes: { tf_vid: trackflow('vid') }`). The `razorpay` checkout preset reads
`payload.payment.entity.notes.tf_vid` (order notes too). With a mapped webhook instead, set `vid_path` to
`payload.payment.entity.notes.tf_vid` (`payment.captured`) or `payload.order.entity.notes.tf_vid`
(`order.paid`).

---

## 5. Gateways without webhooks: report from the merchant's server (bKash, Nagad, SSLCommerz)

Right after the payment is confirmed (bKash Execute Payment `transactionStatus: "Completed"`, Nagad verify
`status: "Success"`, SSLCommerz validated `VALID`), the callback request comes from the buyer's browser to
the merchant's domain, so it carries `_tf_vid`, `_fbp`, `_fbc`. Send them to the server events API
([rest-api.md](rest-api.md) §2) with an API key. Use `purchase_<order id>` as the event id so a thank-you
page Purchase for the same order counts once.

```js
// Node.js, after bKash Execute / Nagad verify succeeds. Keep the API key on the server.
const vid = String(req.cookies._tf_vid || '').replace(/[^A-Za-z0-9_-]/g, '').slice(0, 64);
await fetch('https://TRACKFLOW_HOST/api/v1/events', {
  method: 'POST',
  headers: { Authorization: 'Bearer ' + process.env.TRACKFLOW_API_KEY, 'Content-Type': 'application/json' },
  body: JSON.stringify({
    site_key: process.env.TRACKFLOW_SITE_KEY,
    event_name: 'Purchase',
    event_id: 'purchase_' + order.id,
    visitor_id: vid || undefined,
    fbp: req.cookies._fbp, fbc: req.cookies._fbc,
    ip: req.ip, user_agent: req.get('user-agent'),
    url: 'https://example.com/checkout/complete',
    user_data: { em: order.email, ph: order.phone },
    custom_data: { value: order.total, currency: 'BDT', order_id: String(order.id) }
  })
});
```

These gateways send no refund/cancel webhooks; a COD-style cancellation flow needs a store integration
(WooCommerce/Shopify) or a mapped webhook with a `status_map`.
