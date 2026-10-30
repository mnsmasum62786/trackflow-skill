# Refunds, cancellations, renewals and no-shows (lifecycle events)

Store, checkout, booking and custom webhooks tell TrackFlow more than "an order was paid". Refunds,
cancellations, chargebacks, subscription renewals and cancellations, booking cancellations and no-shows
become **lifecycle events**. TrackFlow records them, takes them out of net revenue in its reports and,
**only when the customer turns it on per destination**, tells the ad platforms.

Where they come from: [integrations.md](integrations.md) (checkout presets, mapped webhooks with a
`status_map`, Cal.com). How the order is tied to the ad click: [visitor-id.md](visitor-id.md).

---

## 1. The events

| What happened | `event_name` | `event_id` | `custom_data` (besides `lifecycle`) |
|---|---|---|---|
| Full or partial refund | `OrderRefunded` | `refund_<order id>_<refund id>` (no refund id: a key from the amount and time/body) | `order_id`, `value` = refunded amount (always positive), `currency`, `partial` (true/false), `remaining_value` (what is left of the order after all refunds so far), `original_value`, `refund_id` |
| Order cancelled | `OrderCancelled` | `cancel_<order id>` | `order_id`, `value` = order total, `remaining_value: 0`, `original_value` |
| Chargeback / dispute | `OrderCancelled` | `chargeback_<order id>` | as cancel, `lifecycle: "chargeback"` |
| Subscription renewed | `SubscriptionRenewed` | `renewal_<invoice / charge id>` | `value` = renewal amount, `currency`; the subscription id is in `order_id` when stored and is sent to platforms as `subscription_id` |
| Subscription cancelled | `SubscriptionCancelled` | `subcancel_<subscription id>` | no value |
| Booking cancelled | `BookingCancelled` | `bookingcancel_<invitee uuid / booking uid>` | no value (Cal.com: `booking_uid`) |
| Booking no-show | `BookingNoShow` | `noshow_<invitee uuid / booking uid>` | no value |

`custom_data.lifecycle` is one of `refund`, `cancel`, `chargeback`, `renewal`, `subscription_cancel`,
`booking_cancel`, `no_show`. Origin is `webhook`, `action_source` `system_generated`, and the event time is
when the refund/cancellation happened (not when the webhook arrived).

Rules TrackFlow applies (checkout presets and mapped webhooks alike):

- **Never conversions.** Lifecycle events never count as conversions, whatever the conversion list says.
  `set_conversion_events` drops these names and returns `ignored` + a `note`; `get_conversion_events` lists
  them separately under `lifecycle_events`. They add nothing to Revenue or Conversions.
- **Only for tracked orders.** A refund, cancellation or chargeback is created only when the order's
  `Purchase` was tracked (by any source: thank-you page, Shopify Customer Events code, webhook). Otherwise
  the result is `ignored` ("No tracked Purchase for order …").
- **Order lookup by any id.** Refund and dispute webhooks often carry only a payment/charge/capture id.
  TrackFlow remembers every provider id seen for an order (Stripe payment intent, charge, invoice,
  subscription, customer; PayPal capture; Paddle transaction…) and follows it back to the order.
- **Counted once.** Provider retries reuse the event id (duplicate). Running totals (Stripe
  `amount_refunded`, Lemon Squeezy `refunded_amount`) are turned into the new refunded amount only. A refund
  after a cancellation, or a cancellation after a full refund, is ignored; a second cancel is `duplicate`.
- **Contact data:** the hashed email/phone stored with the order is reused (refund payloads rarely carry
  any). No visitor id, email or phone at all → `ignored`.
- **Test-mode** refunds are ignored unless the integration has `config.accept_test: true`.

---

## 2. Sending them to ad platforms (opt-in per destination)

Every destination has the Advanced field `send_lifecycle` ("Send refunds and cancellations"), **off by
default**. While it is off, lifecycle events are still stored and reported; the delivery to that
destination is `skipped` with the reason **"Refunds and cancellations are off for this destination"**.

Turning it on changes what the platform reports and optimises on, so **only do it when the customer asks
for it** (explain the effect per platform first):

```json
destinations {"action": "update_destination", "site_id": 12, "destination_id": 7,
              "config": {"send_lifecycle": true}}
```

`list_platforms` shows the field on every platform with a `why` that says exactly what that platform
receives. TrackFlow never sends a zero or negative value and never sends a second, lower "correction"
Purchase.

| Platform | With `send_lifecycle` on |
|---|---|
| **GA4** | Refunds, cancellations and chargebacks → GA4 `refund` event, `transaction_id` = order id, positive `value` + currency (full reversal: no items, GA4 refunds the whole transaction; partial: the refunded items when known). GA4 lowers that transaction's revenue. Needs the order id. Renewals and subscription/booking events → custom events (`subscription_renewed`, …). Within 72 h only. |
| **Google Ads** | **Partial refund** → value restatement of the original Purchase to `remaining_value` (same transaction id and conversion action), only for a Purchase this destination uploaded successfully (looked up within 54 days). Google wants it ≥ 24 h after the conversion: until then the delivery is `skipped` with **"Queued: … sending after <time>"** and an hourly sweep sends it when due. Max 54 days after the conversion. **Full refunds, cancellations and chargebacks are skipped**: the Data Manager API TrackFlow uses cannot retract a conversion (that needs the Google Ads API). Renewals/other lifecycle events only when mapped under Conversion action IDs (e.g. `SubscriptionRenewed=<action id>`). |
| **Meta, TikTok, Reddit** | Custom events `OrderRefunded`, `OrderCancelled`, `SubscriptionRenewed`, … with `order_id` and a positive value. No way to undo a Purchase: use them for custom conversions or audience exclusions. Meta/TikTok/Reddit: within 7 days. |
| **OpenAI** | Custom events `order_refunded`, `order_cancelled`, `subscription_renewed`, … within 7 days. |
| **Microsoft Ads** | UET custom events with action `order_refunded` / `order_cancelled` (or your override), positive value; restating/retracting needs the Microsoft Advertising API, which TrackFlow doesn't use. Within 90 days. |
| **Custom destination (webhook)** | Forwarded like any event; `custom_data.lifecycle` says which kind. |
| **Outbrain, Taboola** | Named conversions (Outbrain `OrderRefunded`…, Taboola `order_refunded`…, or your mapped names), positive value, no order id, **only for visitors with that network's click id**. Create the conversions and keep them out of optimisation. |
| **LinkedIn, X, Spotify, Nextdoor** | Only when the event name is mapped (LinkedIn Conversions API rule ids, X event ids, Spotify custom slots, Nextdoor custom conversion slots, e.g. `OrderRefunded=3`). Otherwise skipped with the reason. |
| **Snapchat, Pinterest, Quora, AppLovin** | Always skipped (fixed event lists, no named custom events). Leave `send_lifecycle` off. |

---

## 3. Maximum event age per platform (all server deliveries)

Late events (refunds, recovered purchases from webhooks, CRM and offline rows) are checked before
sending. Older ones are `skipped` with **"Too old for <Platform> (max N days)"** instead of being sent and
rejected. This applies to every event, not only lifecycle ones.

| Platform | Max age |
|---|---|
| Meta | 7 days (62 days for `physical_store` offline events) |
| GA4 | 72 hours |
| TikTok, Pinterest, Reddit, OpenAI | 7 days |
| Snapchat | 37 days |
| LinkedIn, Google Ads, Microsoft Ads | 90 days |

A stricter platform rule (e.g. Google Ads' 54-day adjustment window) still applies on top.

---

## 4. In TrackFlow reports

| Where | Fields |
|---|---|
| `get_overview` `totals` | `refunds`, `net_revenue` (= revenue − refunds), `cancelled_orders` |
| `get_attribution` | `totals {revenue, refunds, net_revenue, cancelled_orders}`, and the same three fields per `sources[]` / `campaigns[]` row, attributed to the **source of the order's Purchase** (first- or last-touch per `model`) |

`refunds` = refunded amounts + the value of cancelled/charged-back orders (minus what was already
refunded), **only for orders whose Purchase counted as a conversion**. `cancelled_orders` counts distinct
orders (a cancel and a chargeback of one order count once). Blocked and test events are excluded.

---

## 5. Which sources send them

| Source | Refunds | Cancellations / chargebacks | Other |
|---|---|---|---|
| Shopify | `refunds/create` | `orders/cancelled` | – |
| WooCommerce | `order.updated` with `refunds[]` or status `refunded` | status `cancelled` (and `failed`) | – |
| Stripe | `charge.refunded` | `charge.dispute.created` | `invoice.paid` (billing_reason `subscription_cycle`) → renewal, `customer.subscription.deleted` |
| PayPal | `PAYMENT.CAPTURE.REFUNDED` | `PAYMENT.CAPTURE.REVERSED`, `CUSTOMER.DISPUTE.CREATED` | `PAYMENT.SALE.COMPLETED` (first sale of an unknown subscription = Purchase, later = renewal), `BILLING.SUBSCRIPTION.CANCELLED` |
| Paddle Billing | `adjustment.created` / `adjustment.updated` (action refund, status approved) | adjustment action chargeback | `transaction.completed` with origin `subscription_recurring` → renewal, `subscription.canceled` |
| Lemon Squeezy | `order_refunded`, `subscription_payment_refunded` | – | `subscription_payment_success` (billing_reason renewal), `subscription_cancelled` |
| Gumroad | Ping with `refunded=true` | Ping with `disputed=true` | `is_recurring_charge` → renewal |
| ThriveCart | `order.refund` | – | `order.subscription_payment`, `order.subscription_cancelled` |
| SamCart | Refund, Partial Refund | – | Subscription Charged, Subscription Canceled |
| Razorpay | `refund.processed` | `payment.dispute.created` | `subscription.charged` (paid_count > 1), `subscription.cancelled` |
| ClickFunnels 2.0 | `one-time-order.refunded` / `order.refunded` | – | `subscription.invoice.paid`, `subscription.canceled` |
| systeme.io | – | `SALE_CANCELED` → OrderCancelled | – |
| Teachable | `Transaction.refunded` | – | `Sale.subscription_canceled` |
| Whop | `refund.created` | `dispute.created` | `payment.succeeded` with a renewal billing reason, `membership.updated` / `membership.deactivated` cancellations |
| Calendly | – | `invitee.canceled` → BookingCancelled (reschedules skipped) | `invitee_no_show.created` → BookingNoShow |
| Cal.com (mapped webhook, provider `cal`) | – | `BOOKING_CANCELLED` → BookingCancelled | `BOOKING_NO_SHOW_UPDATED` (an attendee marked no-show) → BookingNoShow |
| Mapped / custom webhook | `status_map` → `refund` (+ `refund_amount_path`) | `status_map` → `cancel` / `chargeback` | see [integrations.md](integrations.md) §5 |
| Kajabi | – | – | purchases only |

Tell the customer to select these events when registering the webhook; `list_checkout_providers` returns
the exact list per provider (`events`).

---

## 6. Cash on delivery (COD) stores, e.g. Bangladesh

Many COD orders are refused or never delivered. Two options, set with the store integration's
`config.purchase_on` ([integrations.md](integrations.md) §2):

**Option 1 (recommended): Purchase at order placement, cancel later.**
1. `update_checkout_integration {"config": {"purchase_on": "created_cod", …}}` (send the full config).
   The Purchase goes out when the order is placed, so ads learn fast. The thank-you page Purchase and the
   webhook share `purchase_<order id>`, so it counts once.
2. With the customer's OK, `send_lifecycle: true` on the destinations.
3. The merchant marks refused/undelivered orders **Cancelled** in the store → the webhook sends
   `OrderCancelled`: GA4 gets a `refund`; Meta, TikTok etc. get the custom event (suggest a Meta custom
   conversion on `OrderCancelled` to see cancellation rates, or an exclusion audience). Google Ads skips it
   (no retraction via the Data Manager API).

**Option 2 (quality mode): Purchase only when delivered.** `purchase_on: "completed"`. Ads optimise on
real sales but learn later. The thank-you page must then **not** send `Purchase` (send e.g.
`InitiateCheckout` or a custom `PlaceOrder` there), otherwise that one counts first and the delivered one
is a duplicate. The webhook Purchase is stamped with the completion time; platform age limits (§3) still
apply to it.

For gateways without webhooks (bKash, Nagad, SSLCommerz) the merchant's server reports the purchase:
[visitor-id.md](visitor-id.md) §5.
