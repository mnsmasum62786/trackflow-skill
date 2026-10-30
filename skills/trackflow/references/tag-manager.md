# TrackFlow tag manager (built-in tag engine)

TrackFlow has its own tag manager (no Google Tag Manager needed). Everything is data: **variables**,
reusable **triggers**, **tags** (trigger(s) + optional conditions + actions) and **container settings**.
You edit a **draft**; visitors run the **published version**. Nothing you build reaches visitors until
`publish_version`.

Tools live in two MCP groups (call as `{"action": "<name>", "site_id": N, ...}`):

| Group | Actions |
|---|---|
| `tag_manager` | `get_tagging_reference`, `inspect_site`, `list_variables`, `create_variable`, `update_variable`, `delete_variable`, `list_tag_triggers`, `create_tag_trigger`, `update_tag_trigger`, `duplicate_tag_trigger`, `delete_tag_trigger`, `list_tags`, `create_tag`, `update_tag`, `delete_tag`, `list_recipes`, `apply_recipe`, `update_container_settings`, `get_container` |
| `publishing` | `create_preview_link`, `list_preview_links`, `delete_preview_link`, `publish_version`, `list_versions`, `rollback` |

Trigger actions are named `*_tag_trigger` (`create_tag_trigger`, not `create_trigger`). If your client
shows other names, use `{"action": "help"}` on the group to see the real list.

`get_tagging_reference` returns the full live vocabulary (variable types, built-ins, trigger types, form
presets, operators, action types, container settings, recipes). Read it once per session before building.

---

## 1. Workflow (always in this order)

1. `list_sites` → `site_id`. Check `get_site.last_seen_at`: empty means the tag is not installed yet
   (give the customer `get_install_snippet`).
2. `inspect_site` on every relevant page (home, product, cart, checkout, thank-you, contact). Use the
   selectors, form builders, product data and existing pixels it reports.
3. `list_tags`, `list_tag_triggers`, `list_variables` to see what already exists. Don't duplicate.
4. Build in the draft: `create_variable` → (`create_tag_trigger` or inline trigger) → `create_tag`, or
   `apply_recipe`. Read the `warnings` array in every create/update result (unknown variable names).
5. `create_preview_link` → the customer (or you, in a normal non-headless browser) opens it and performs
   the action → `list_events` with `test: "only"` → `get_event` to check names, values, IDs.
6. `publish_version` with `notes` **only when the customer asked for it to go live** (or approved). If
   they want to review, leave the draft and hand over the preview link + a summary.
7. After publishing, verify real events with `list_events` (`test: "exclude"`). `rollback` if needed.

---

## 2. Data model

| Object | Key fields | Rules |
|---|---|---|
| Variable | `name`, `type`, type fields, `transform`, `default`, `fallback`, `notes` | Name `^[A-Za-z_][A-Za-z0-9_]{0,63}$`, unique per site, may not equal a built-in name. Unknown fields are rejected. |
| Trigger | `name` (1–120, unique per site), `trigger` = `{type, …fields}`, `notes` | Reusable; one trigger can fire many tags. Returned with `used_by` (tags using it). Cannot be deleted while used (409). |
| Tag | `name` (≤120), `trigger`, `conditions`, `actions`, `once`, `enabled`, `priority`, `notes` | `actions` must be a non-empty array. Disabled tags are not served. Higher `priority` runs first. |
| Container settings | `decorate_checkouts`, `decorate_reload_iframes`, `shopify_events` | Part of the draft: **needs `publish_version`** like any tag change. |
| Version | `version`, `notes`, `live`, `tags`, `variables`, `has_custom_js`, `via` (`dashboard` / `api` / `system`), `created_at` | Every publish/rollback creates a new numbered version. `list_versions` shows the newest 100. |

### Tag `trigger` field

One item or an array (any item fires the tag). Each item is either:

- a reference `{"trigger_id": 12}` (other keys ignored; the id can also be the trigger's exact name), or
- an inline definition `{"type": "click", "selector": "#buy", "trigger_name": "Buy button"}`. It is saved
  as a reusable trigger automatically. An identical saved definition (same keys/values) is reused;
  otherwise a new trigger is created, named `trigger_name` or auto-named (e.g. `Click – #buy`).

`list_tags` returns `trigger` resolved as `[{trigger_id, trigger_name, type, …fields}]`.

Editing a shared trigger (`update_tag_trigger`) changes **every** tag that uses it. To change one tag
only, `duplicate_tag_trigger` (creates `"<name> (copy)"`) and point that tag at the copy.

### `update_*` semantics

| Action | Behaviour |
|---|---|
| `update_tag` | Only given fields change, but `trigger`, `conditions`, `actions` are **replaced as a whole**. Pause with `enabled: false` instead of deleting. |
| `update_variable` | Fields are merged into the config; `null` removes a field. Passing `type` **resets** the config to only the fields you send. |
| `update_tag_trigger` | `name`, `notes`, or `trigger` (new definition replaces the old one). |

### Templates

Any string in actions or conditions may contain `{{variable}}`.

- A string that is exactly one reference (`"{{price}}"`) keeps the raw type (number, array, object).
- Mixed text (`"Plan {{plan}}"`) is joined as a string; empty variables become `""`.
- Dot paths read inside objects: `{{event_data.payload.email}}`, `{{checkout.value}}`.
- Referencing a variable that doesn't exist still saves, but the API returns a warning. Fix it.

---

## 3. Variable types

Common optional fields on every variable: `transform` (one or a list), `default` (value when empty),
`fallback` (name of another variable to try when empty; chains allowed, a variable cannot fall back to
itself).

Transforms: `trim`, `lowercase`, `uppercase`, `number`, `int`, `price`, `currency`, `digits`, `count`,
`first`, `json`. `price` parses `"₹1,299.00"` → `1299`, `"1.299,00 €"` → `1299`, `"$12.50"` → `12.5`.
`currency` detects the ISO code from symbols or codes (`$` → USD, `€` → EUR, `৳`/`Tk` → BDT…).

| Type | Required | Optional | Reads |
|---|---|---|---|
| `dom_text` | `selector` | `multiple`, `in_event` | Text of first match (`content` attr for `<meta>`/itemprop). `multiple:true` → array. `in_event:true` searches inside the triggering element/form first. |
| `dom_attr` | `selector`, `attribute` | `in_event` | Attribute of first match; alternatives `"data-id\|data-product-id"`. |
| `dom_value` | `selector` | `in_event` | Value of input/select/textarea (checkbox → true/false). |
| `url` | – | `part`, `param`, `source` | `part`: `full` (default) `path` `host` `query` `hash` `origin` `param` (needs `param`). `source`: `page` (default) `click` (clicked link) `referrer`. |
| `cookie` | `cookie` | – | First-party cookie value. |
| `js_var` | `path` | – | Global JS variable by dot path, e.g. `Shopify.checkout.total_price`. |
| `local_storage` / `session_storage` | `key` | `path` | Storage value (JSON parsed; `path` picks a field). |
| `datalayer` | `key` | `event` | Dot path into the latest dataLayer push that has it (only pushes with that `event` if set). The push that fired a dataLayer trigger is checked first. |
| `json_ld` | `field` | `schema_type` | schema.org JSON-LD. Product (default): `name`, `id` (sku/productID/mpn/gtin), `sku`, `brand`, `category`, `image`, `description`, `price`, `currency`, `availability`, `url`, `gtin`. `schema_type: "Order"`: `order_id`, `total`, `currency`. Other fields = dot path. |
| `meta_tag` | `meta` | – | `content` of `<meta name\|property\|itemprop="…">`. |
| `open_graph` | `property` | – | `"title"` → `og:title`; `"price:amount"` → `og:price:amount` or `product:price:amount`. |
| `constant` | `value` | – | Fixed value (string, number, object, array). |
| `regex` | `source`, `pattern` | `group`, `flags` | Regex on another variable (`source` = variable name, dot paths OK). Returns group 1 by default (whole match if no group). Default flags `i`. |
| `lookup` | `source`, `map` | `match` | `{input: output}` table; `match`: `equals` (default) `contains` `regex`. |
| `element_count` | `selector` | – | Number of matching elements. |
| `click_element` | – | `attr`, `closest`, `find` | The element that fired the trigger. `closest` goes up to a selector, then `find` goes down. `attr`: `text` (default) `value` `href` `id` `class` or any attribute (`"a\|b"`). |
| `form_field` | one of `field`/`label`/`selector` | `form` | Field of the triggering form: `field` = name/id (`"billing_email\|email"`), `label` = visible label/placeholder contains, or `selector`. Password and card fields are never read. |
| `page_title`, `referrer`, `user_agent` | – | – | `document.title`, referrer (or previous SPA URL), UA. |
| `stored` | `bag` | `key`, `pluck` | Value saved by a `store` action: whole bag or `bag[key]`. `pluck` maps an array of objects to one field. |
| `event_data` | – | `path` | Data carried by the trigger (postMessage payload, dataLayer push, CustomEvent detail, form-builder response). |
| `calc` | `op`, `a`, `b` | – | `add` `subtract` `multiply` `divide` `min` `max` on two templates/numbers, rounded to 2 decimals. |

```json
{"action": "create_variable", "site_id": 12, "name": "order_total", "type": "dom_text",
 "selector": ".order-total .amount", "transform": "price", "fallback": "order_total_js"}
```

### Built-in variables (exist on every site; reserved names)

| Names | Value |
|---|---|
| `page_url`, `page_path`, `page_host`, `page_query`, `page_hash` | Current URL parts (`page_path` has **no** query string; query/hash without `?`/`#`). |
| `page_title`, `referrer`, `user_agent`, `language` | Page and browser. |
| `event`, `event_name`, `event_data` | Trigger type; custom/dataLayer event name or form preset; trigger payload (dot paths). |
| `click_text`, `click_url`, `click_id`, `click_classes`, `click_element_tag` | Clicked link/button. |
| `form_id`, `form_name`, `form_action`, `form_classes`, `form_signature` | Submitted form. `form_signature` = id + name + class + action (+ parent classes), handy for regex. |
| `form_preset`, `form_label` | Builder that reported success (`cf7`, `elementor`, `hubspot`…); readable label (preset, else form id/name). |
| `form_user` | Customer details read from the form/message (`em`, `ph`, `fn`, `ln`…). Use as `user_data`. |
| `form_field_count` | Visible fields in the form. |
| `scroll_percent`, `video_status`, `video_percent`, `video_title`, `video_url`, `video_provider` | Scroll/video triggers. |
| `message_origin`, `visible_percent`, `timer_count`, `error_message`, `error_source`, `file_extension`, `link_host`, `history_source`, `copy_text` | Data of the matching trigger type. |
| `vid`, `sid` | TrackFlow visitor id and session id. |
| `product_name`, `product_id`, `product_price`, `product_currency` | Product data: JSON-LD → meta tags → page (`h1`, `[itemprop=price]`, `[data-product-id]`). |

---

## 4. Trigger types

| `type` | Fields (required **bold**) | Fires on |
|---|---|---|
| `page_view` | – | Each page load (DOM parsed) and each SPA route change. |
| `history_change` | – | SPA route changes only (pushState/replaceState/popstate/hashchange). |
| `dom_ready` | – | DOMContentLoaded. |
| `window_loaded` | – | window load. |
| `click` | `selector`, `text`, `text_match`, `target` | Click on an element matching `selector` **or inside it**; or a link/button whose visible text matches `text`. `text_match`: `contains` (default) `equals` `starts_with` `regex`. Both given → either matches. `target`: `any` (default) `link` `button`. Needs at least one of selector/text/target. |
| `form_submit` | `selector`, `preset`, `wait_for`, `success_selector`, `success_text`, `success_url`, `request_url`, `timeout_ms` | **Successful** submissions (see below). `selector` may match the form, an ancestor wrapper, or an element inside it. |
| `element_visible` | **`selector`**, `threshold`, `repeat`, `min_ms` | Element scrolled into view (threshold 0–1, default 0.5). `repeat:true` fires every time; `min_ms` = must stay visible that long. Watches new elements too. |
| `custom_event` | **`name`**, `source` | DOM CustomEvent on window/document (jQuery events too), `trackflow('event', name, data)`, or a dataLayer push with `event == name`. `source`: `any` (default) `dom` `datalayer`. A name starting with `/` is a regex (`"/^quote_/"`). |
| `timer` | **`interval_ms`** (≥1000), `limit` | Every interval, up to `limit` times per page (default 1). |
| `scroll_depth` | `percents` | Vertical scroll thresholds, default `[25, 50, 75, 90]`. |
| `video` | `providers`, `statuses`, `percents` | HTML5 `<video>/<audio>` and YouTube. `statuses`: `start` `progress` `pause` `complete` (default start, progress, complete); `percents` default `[25, 50, 75]`. |
| `iframe_message` | `preset`, `origin`, `contains` | `window.postMessage`. `preset`: `calendly` `cal` `typeform` `tally` `jotform` `hubspot`. `origin`/`contains` = substrings of sender origin / serialized message. |
| `copy` | `selector` | Text copied (optionally only inside `selector`). |
| `file_download` | `extensions` | Clicks on file links (default pdf, zip, rar, 7z, gz, doc(x), xls(x), ppt(x), csv, txt, rtf, epub, mp3, mp4, mov, avi, dmg, exe, apk, msi, pkg) or `<a download>`. |
| `outbound_link` | `exclude` | Clicks on links to other domains (cross-domain list and `exclude` hosts skipped). |
| `error` | `contains` | Uncaught JS errors (max 10 per page). |

### form_submit: how success is detected

`preset`: `auto` (default: native forms + every builder/iframe preset), `native`, one preset id, or a
list. Presets:

| Preset | Builder | Success signal |
|---|---|---|
| `cf7` | Contact Form 7 | `wpcf7mailsent` |
| `elementor` | Elementor Pro | `submit_success` |
| `gravity` | Gravity Forms | `gform_confirmation_loaded` |
| `wpforms` | WPForms | `wpformsAjaxSubmitSuccess` |
| `ninja` | Ninja Forms | `nfFormSubmitResponse` |
| `fluent` | Fluent Forms | `fluentform_submission_success` |
| `formidable` | Formidable | `frmFormComplete` |
| `hubspot` | HubSpot embeds | `onFormSubmitted` message / `hs-form-event` |
| `webflow` | Webflow | `.w-form-done` visible |
| `framer` | Framer | 2xx form request |
| `squarespace` | Squarespace | `/api/form/` request or success text |
| `wix` | Wix | 2xx form API request |
| `typeform`, `jotform`, `tally` | Embeds | postMessage (form-submit / submission-completed / Tally.FormSubmitted) |
| `calendly`, `cal` | Booking embeds | `calendly.event_scheduled` / `bookingSuccessful` |

`wait_for` (native forms only):

| Value | Fires when |
|---|---|
| `auto` (default) | Immediately if the page navigates; for AJAX forms when a 2xx non-GET request is seen, the URL changes, or the form is removed/hidden. Builders use their own signal. |
| `none` | Immediately on submit (counts failed validation too). |
| `request` | 2xx response from `request_url` (substring; default: the form's action path). |
| `element` | `success_selector` becomes visible. |
| `text` | `success_text` appears on the page. |
| `url_change` | URL changes (optionally to one containing `success_url`). |

Gives up after `timeout_ms` (default 10000). A builder form (CF7, Elementor…) with `preset: "native"`
only fires if the page navigates; use `auto` or the builder's preset.

### Firing limits and duplicate guards

- `once`: `""` (every time), `page`, `session` (sessionStorage), `visitor` (localStorage, last 200 tag ids).
- Throttle per tag: clicks / file downloads / outbound links 0.7 s; form_submit and iframe_message 2 s
  (per form). Double clicks and builders that report success twice don't create two events.
- A tag fires at most once per trigger signal even if several of its triggers match.

---

## 5. Conditions

A leaf `{var, op, value}` or a group `{all:[…]}`, `{any:[…]}`, `{not: …}`; a plain array means all.
Max nesting depth 6. `value` may be a template.

| `op` | True when |
|---|---|
| `equals` | Equal as text (case-insensitive), or as numbers when both numeric |
| `contains`, `starts_with`, `ends_with` | Text tests |
| `regex` | Pattern matches (flag `i` unless `case_sensitive: true`) |
| `gt`, `gte`, `lt`, `lte` | Numeric (price strings like `"$1,299"` are parsed) |
| `exists` | Variable is not empty |
| `in` | Variable equals one item of a list `value` |
| `matches_css` | Triggering element matches / is inside the selector in `value` (no `var`) |

Prefix `not_` to negate (`not_regex`, `not_exists`). Aliases: `eq`, `=`, `starts`, `ends`, `>`, `<`, `>=`,
`<=`, `matches` (= regex). Regex is syntax-checked at save time.

```json
{"all": [
  {"var": "page_path", "op": "regex", "value": "^/(checkout|cart)"},
  {"any": [{"var": "product_price", "op": "gt", "value": 0},
           {"var": "event_name", "op": "equals", "value": "cf7"}]}
]}
```

---

## 6. Action types (run in order)

| `type` | Required | Optional | Does |
|---|---|---|---|
| `send_event` | `event_name` (≤64) | `custom_data`, `user_data`, `event_id`, `is_conversion` | Sends the event to every destination's browser tag **and** the server copy, one shared event id. `custom_data`: canonical keys → templates, or one template resolving to an object. `user_data`: `{em, ph, fn, ln, ct, st, zp, country, external_id}` → templates, or `"{{form_user}}"`. `event_id`: template, sanitized to `[\w.:@-]`, ≤64 chars. `is_conversion` is **ignored** (conversions are chosen per event name with `set_conversion_events`). |
| `store` | `bag` (1–40 of `[\w-]`) | `values`, `storage`, `ttl_minutes`, `append`, `clear`, `replace` | Save to a first-party bag (localStorage; `storage:"session"` for sessionStorage). Merges unless `replace:true`. `append`: key of an array to push onto (keeps last 50). `clear:true` deletes. TTL default 4320 min (3 days). Read with a `stored` variable. |
| `identify` | `user_data` | – | Attach customer details (`em, ph, fn, ln, name, ct, st, zp, country, external_id`) to the visitor; later events carry them (hashed server-side). |
| `set_variable` | `name`, `value` | `persist` | Override a variable for the rest of the page; `persist:"session"` keeps it in sessionStorage. |
| `custom_js` | `code` (string ≤20000) | – | Body of `function(tf, event)`. `tf = {track, identify, get, set, store, vid, log}`; `event = {type, name, data, element, form, tag}`. Runs only while site setting `allow_custom_js` is on (default on); off → stripped from the served container. Errors are caught. Prefer variables + `send_event`. |

Canonical `custom_data` keys: `value`, `currency`, `content_ids`, `contents[{id, quantity, item_price}]`,
`content_type`, `content_name`, `content_category`, `num_items`, `order_id`, `search_string`, `status`,
`predicted_ltv`. Cleaning: `value` parsed as a price and rounded to 2 decimals (dropped if not a number);
`currency` → ISO code (if a value exists and currency is missing or invalid, the site's `default_currency` (else USD)); empty
`content_ids`/`contents` items dropped.

### event_id and dedup

- An `event_id` already sent **from this browser** (same event name, last 100 kept) is skipped → a
  reloaded thank-you page doesn't send a second Purchase.
- The server stores each `event_id` + `event_name` once per site (site setting `dedup`, keep on). A
  checkout webhook using the same id (`purchase_<order id>`) counts once with the browser event.
- **If any variable in the `event_id` template is empty, a random id is used** → no dedup. Make sure the
  order id variable resolves (test in preview).

---

## 7. Container settings: checkout, form and embed decoration; Shopify extras

`update_container_settings {"site_id": N, "decorate_checkouts": true | false | [providers]}` appends the
TrackFlow visitor id to third-party checkout/booking/form links, embeds and iframes so their webhooks can
be tied to the ad click ([visitor-id.md](visitor-id.md)).

| Provider | Added |
|---|---|
| `stripe` | Payment Links (`buy.stripe.com`): `client_reference_id=<vid>`; Buy Button / Pricing Table embeds (`<stripe-buy-button>`, `<stripe-pricing-table>`): `client-reference-id` attribute; Stripe.js `redirectToCheckout` gets `clientReferenceId` |
| `paypal` | PayPal Standard (`cgi-bin/webscr`, donate): `custom=<vid>`; PayPal forms get hidden `custom` |
| `lemonsqueezy` | `checkout[custom][tf_vid]=<vid>` |
| `gumroad` | `tf_vid=<vid>` |
| `thrivecart` | `passthrough[tf_vid]=<vid>` |
| `samcart` | `utm_content=<vid>` (only if none set) |
| `paddle` | Billing `Paddle.Checkout.open` `customData.tf_vid`; Classic `passthrough=<vid>` |
| `calendly` | `utm_content=<vid>` on links, inline widgets and `Calendly.init*` |
| `cal` | `metadata[tf_vid]=<vid>` |
| `typeform` | Hidden field `tf_vid` (URL hash on `/to/` links, `data-tf-hidden` on embeds): add a hidden field named `tf_vid` in the form |
| `jotform` | `tf_vid=<vid>` on links/iframes (prefills a hidden field named `tf_vid`) |
| `tally` | `tf_vid=<vid>` on `/r/` and `/embed/` links/iframes (hidden field `tf_vid`) |
| `hubspot` | `tf_vid=<vid>` on `hsforms.com` links/iframes (hidden property `tf_vid`) |
| `zoho` | `tf_vid=<vid>` on Zoho Forms links/iframes (hidden field `tf_vid`) |
| `gohighlevel` | `tf_vid=<vid>` on GoHighLevel / LeadConnector form, survey, quiz and booking widgets (hidden/custom field `tf_vid`) |
| `hidden_input` | Forms posting to another domain get `<input type=hidden name="tf_vid">` |

Never overwrites an existing value; only after consent. Content added later (SPA views, popups,
JS-rendered buttons and embeds) is decorated as it appears; lazy `data-src` iframe attributes always.
An iframe that already loaded keeps its URL (rewriting it reloads the form) unless
`decorate_reload_iframes: true`. **These are draft settings: publish to apply.** The engine (and therefore
decoration) only loads when the published container has at least one enabled tag. For custom flows the
site can call `trackflow('decorate', url)` / `trackflow('vid')`. Independently of this setting, the tag
fills any on-page input named `tf_vid` with the visitor id.

`shopify_events` (Shopify Customer Events code only, [install-cms.md](install-cms.md) §4): `true` = also
send `ViewCategory`, `ViewCart`, `AddShippingInfo`; a list of some of them; `false` / `null` = none.
The funnel events follow the site setting `ecommerce_datalayer` (or a published tag sending that name),
`PageView` follows `auto_pageview` (or published `page_view` tags).

---

## 8. Recipes

`list_recipes`, then `apply_recipe {"site_id": N, "recipe": "<id>", "options": {...}}`. Recipes add
variables + tags (+ settings) to the draft. Re-applying updates the recipe's own items (and re-enables
its tags); variables you created with the same name are kept unless `options.overwrite: true`. Result:
`variables_created/updated/kept`, `tags_created/updated`, `next`.

| Recipe | Creates | Options |
|---|---|---|
| `ecommerce` | ~45 `ec_*` variables (JSON-LD → OpenGraph/meta → DOM fallbacks) and 5 tags: **Ecommerce – ViewContent** (page_view on product pages, once/page); **Ecommerce – AddToCart** (click on add-to-cart selector or button text, value = price × qty, appends to `tf_cart` bag); **Ecommerce – InitiateCheckout** (page_view on checkout path, stores a `checkout` snapshot); **Ecommerce – checkout snapshot on place order** (click/submit on place-order, refreshes snapshot + `identify` from `{{form_user}}`); **Ecommerce – Purchase** (thank-you URL/markup + order evidence, `event_id: purchase_{{ec_order_id}}`, value from snapshot else page total, clears bags). | `currency` (fallback, default USD), `add_to_cart_selector`, `price_selector`, `product_path_regex`, `checkout_path_regex`, `thank_you_path_regex`, `total_selector`, `thank_you_total_selector`, `order_id_selector`, `place_order_selector`, `overwrite` |
| `lead_form` | Tag **Lead – form submissions**: form_submit `preset: auto`, excludes newsletter/search/login/register/cart/checkout/comment/coupon/password/wishlist forms (by `form_signature`) and Calendly/Cal, sends `Lead` with `content_name: {{form_label}}`, `user_data: {{form_user}}`. | `event_name` (default Lead), `form_selector`, `exclude_regex`, `wait_for`, `success_selector`, `success_text` |
| `booking` | 3 `bk_*` variables; tags **Booking – Calendly** (`event_id: calendly_<invitee uuid>`) and **Booking – Cal.com** (`event_id: cal_<booking uid>`), event `Schedule`. Adds `calendly` and `cal` to `decorate_checkouts`. | `event_name` (default Schedule) |
| `newsletter` | Tag **Newsletter – sign-up**: form_submit on forms whose signature matches Mailchimp/Klaviyo/MC4WP/Brevo/ConvertKit/MailerLite/Omnisend/beehiiv/Substack…, sends `Lead` with `content_category: "newsletter"`. | `event_name` (default Lead; e.g. CompleteRegistration, Subscribe), `form_selector` |

Recipes contain WooCommerce/Shopify-style default selectors. After applying, check in preview that
value, currency and order id resolve on the customer's theme; pass selectors as options if not.
Don't apply `ecommerce` when the site already pushes a GA4 ecommerce dataLayer: turn on the site setting
`ecommerce_datalayer` instead (built-in bridge, Purchase id `purchase_<transaction_id>`).

---

## 9. Publishing, versions, rollback, preview

| Action | Params | Notes |
|---|---|---|
| `get_container` | `which`: `draft` (default) / `published` | Draft result has `unpublished_changes`, `published_version`, `has_custom_js`, `allow_custom_js`. |
| `publish_version` | `notes` (≤500) | Snapshots the draft as version N+1 and makes it live. Returns `version`, `tags`, `variables`, `has_custom_js`, `custom_js_runs`. Visitors get it on their next page load (config cached up to ~1 min). |
| `list_versions` | – | Newest first: `live`, `notes`, counts, `has_custom_js`, `via`. |
| `rollback` | `version` (number from list_versions), `notes`, `restore_draft` (default true) | Publishes a **new** version with that content ("Rollback to vN"). By default also resets the draft to it, **discarding unpublished draft edits** (pass `restore_draft: false` to keep them). |
| `create_preview_link` | `url` (default home; must be on the site domain, a subdomain or a cross-domain), `hours` (default 2, 0.25–24) | Returns `url` with `?tf_preview=pv_…`. That browser session runs the **draft**; every event is stored with `is_test`. `?tf_preview=off` leaves preview. |
| `list_preview_links` / `delete_preview_link` | `token` | Active links; revoke one early. |

Preview behaviour to know:
- Browser pixels still fire for preview events (they reach Meta/TikTok… as browser hits).
- Server copies of preview events are **skipped** for destinations without a test mode configured
  (`test_event_code` etc.); with one set, they go to the platform's test view.
- Preview events are excluded from TrackFlow's business numbers; filter with `list_events test:"only"`.
- Test in a normal browser. Headless/automated browsers (Playwright, Puppeteer, `HeadlessChrome`) are
  dropped as bots (`blocked`, reason `Bot traffic`).
- Debug mode: `?tf_debug=1` on any page prints what the tracker sends in the console.

---

## 10. Cookbook (copy-ready)

### A. Button click → Lead

`inspect_site` the page, take the CTA's `selector` (prefer `#id`, `[data-*]`, `href` patterns).

```json
{"action": "create_tag", "site_id": 12, "name": "Lead – Get a quote button",
 "trigger": {"type": "click", "selector": "#get-quote, a[href*='/quote']", "trigger_name": "Click – quote button"},
 "once": "page",
 "actions": [{"type": "send_event", "event_name": "Lead",
              "custom_data": {"content_name": "{{click_text}}"}}]}
```

By text when there is no stable selector:
`"trigger": {"type": "click", "text": "book a call", "text_match": "contains", "target": "button"}`.
Tell the customer a click is intent, not a confirmed lead; prefer the form or thank-you page if one exists.
Phone / WhatsApp: selector `a[href^="tel:"]` / `a[href*="wa.me"], a[href*="whatsapp.com"]` → `Contact`.

### B. Form submit → Lead (with customer data)

```json
{"action": "create_tag", "site_id": 12, "name": "Lead – contact form",
 "trigger": {"type": "form_submit", "selector": "#contact-form", "preset": "auto"},
 "actions": [{"type": "send_event", "event_name": "Lead",
              "custom_data": {"content_name": "{{form_label}}"},
              "user_data": "{{form_user}}"}]}
```

AJAX form that shows a message without navigating:
`"trigger": {"type": "form_submit", "selector": "#contact-form", "wait_for": "text", "success_text": "Thank you"}`.
Optional: `update_site settings.auto_forms: true` to attach form identities to the visitor (match quality),
only if the customer wants form capture. For all forms at once use `apply_recipe lead_form`.

### C. Thank-you page → Purchase with value from the page

```json
[
 {"action": "create_variable", "site_id": 12, "name": "ty_total", "type": "dom_text",
  "selector": ".order-total .amount, [data-order-total]", "transform": "price"},
 {"action": "create_variable", "site_id": 12, "name": "ty_currency", "type": "dom_text",
  "selector": ".order-total .amount", "transform": "currency", "default": "EUR"},
 {"action": "create_variable", "site_id": 12, "name": "ty_order_id", "type": "regex",
  "source": "page_url", "pattern": "order-received/(\\d+)", "fallback": "ty_order_id_dom"},
 {"action": "create_variable", "site_id": 12, "name": "ty_order_id_dom", "type": "dom_text",
  "selector": ".order-number strong", "transform": "trim"},
 {"action": "create_tag", "site_id": 12, "name": "Purchase – thank-you page",
  "trigger": {"type": "page_view"}, "once": "page",
  "conditions": {"all": [
     {"var": "page_url", "op": "regex", "value": "order-received|thank-you"},
     {"var": "ty_order_id", "op": "exists"}]},
  "actions": [{"type": "send_event", "event_name": "Purchase",
               "event_id": "purchase_{{ty_order_id}}",
               "custom_data": {"value": "{{ty_total}}", "currency": "{{ty_currency}}",
                               "order_id": "{{ty_order_id}}"}}]}
]
```

(Each object is one call.) Use `page_url` (not `page_path`) when the marker is in the query string. If
the thank-you page shows no total, use the snapshot pattern (store the total on checkout with a `store`
action, read it with a `stored` variable) or `apply_recipe ecommerce`. If a checkout webhook will also
report the order, the order id here must be **the same id the webhook uses** (see integrations.md).

### D. Custom event with event_id dedup

Site code (developer adds one line) — either form works:

```js
trackflow('event', 'quote_sent', { quote_id: 'Q-1042', value: 250, currency: 'USD' });
// or
dataLayer.push({ event: 'quote_sent', quote_id: 'Q-1042', value: 250, currency: 'USD' });
```

```json
{"action": "create_tag", "site_id": 12, "name": "Lead – quote sent",
 "trigger": {"type": "custom_event", "name": "quote_sent", "source": "any"},
 "actions": [{"type": "send_event", "event_name": "Lead",
              "event_id": "quote_{{event_data.quote_id}}",
              "custom_data": {"value": "{{event_data.value}}", "currency": "{{event_data.currency}}"}}]}
```

The same `event_id` + same `event_name` sent later by a server source (webhook, API) is deduplicated.
Listen to several names with a regex name: `"name": "/^(quote_sent|quote_signed)$/"`.

### E. Other quick tags

| Goal | Tag |
|---|---|
| Pricing page views | `page_view` + `{var:"page_path", op:"starts_with", value:"/pricing"}` → `ViewContent`, `once:"session"` |
| PDF downloads | `{type:"file_download", extensions:["pdf"]}` → `Lead` with `content_name:"{{click_url}}"` |
| Video completes | `{type:"video", statuses:["complete"]}` → `VideoComplete` with `content_name:"{{video_title}}"` |
| Engaged readers | `{type:"scroll_depth", percents:[75]}` → custom `EngagedVisit` |
| Calendly/Cal bookings | `apply_recipe booking` |

---

## 11. Gotchas

| Problem | Why / fix |
|---|---|
| Nothing changed on the site | Draft not published. `get_container` → `unpublished_changes: true` → `publish_version` (if the customer wants it live). Wait ~1 min for config cache. |
| Event counted twice in Meta/TikTok | Hard-coded pixel code, a plugin, the Shopify/Wix channel app or a GTM tag already fires the same event with a different event id. `inspect_site` → `existing_tags` (gtm, ga4, meta_pixel, tiktok_pixel). Remove the old event or don't add it here. Also: `auto_pageview` on **and** a `page_view` tag sending `PageView`; or a click tag + a thank-you tag for the same action. |
| Link click lost / double counted | The event is sent with `sendBeacon` and normally survives navigation, but the platform's own browser request may be cut off. If the link leads to a thank-you page, track that page instead of the click (not both). |
| Selector stops matching | Generated classes (`css-1a2b3c`, `framer-xyz`, `elementor-element-5f2c`) change on redeploy. Prefer ids, `data-*`, `name`, `href*=` patterns or button `text`. Re-run `inspect_site` after theme changes. |
| Value missing / 0 | Selector or JSON-LD not found on that page → variable empty → `value` dropped. Add `fallback` chains and `transform: "price"`; verify with `get_event` in preview. |
| Purchase not deduplicated | Empty variable in `event_id` → random id. Or the browser id and the webhook id differ (`purchase_1234` vs `purchase_#1234`). Event names must match exactly (case). |
| Regex never matches | JSON needs double backslashes (`"\\d+"`). `page_path` has no query string. Regex is unanchored (`/thank` also matches `/thanks-page`; use `^/thank-you/?$`). Case-insensitive by default. A `custom_event` regex name needs the leading `/`. |
| Form tag fires on failed validation | `wait_for: "none"` or a click trigger on the submit button. Use `form_submit` with `auto` / `text` / `element`. |
| Form inside an iframe from another domain | Can't be read. Use `iframe_message` presets (Calendly, Cal, Typeform, Tally, Jotform, HubSpot) or the provider's webhook (integrations.md). |
| Shared trigger edit broke another tag | `update_tag_trigger` changes all `used_by` tags. Duplicate first. |
| `decorate_checkouts` has no effect | Not published, or the container has no enabled tags (engine not loaded), or no consent yet. A form iframe that loaded before the tag ran keeps its URL: `decorate_reload_iframes: true`. The form needs a hidden field named `tf_vid`. |
| `once: "visitor"` fires again | It's localStorage; cleared storage / other browser = new visitor. |
| Rollback lost my edits | `rollback` resets the draft by default. Use `restore_draft: false` to keep unpublished work. |
| custom_js doesn't run | Site setting `allow_custom_js` was switched off (default on); turn it back on with `update_site` only when the customer asks. |
| Tests show `blocked` | Headless browser / script UA = bot, or a blocking rule (office IP). Test in a normal browser. |
