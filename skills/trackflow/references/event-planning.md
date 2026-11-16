# Event planning from a website link

Use this when the customer gives only a website address ("track my site", "set up tracking for
example.com") or doesn't know which events to track. You work out what kind of business the site is,
recommend the events that matter for it, show the plan, and build only what the customer approves.

The flow is the same in every AI app connected to the TrackFlow MCP server. Ask as little as possible: the
site and TrackFlow already tell you most things.

## 1. Find the site and what is already there

1. `sites → list_sites`; match the link's host to `domain`. Ignore `www.` and the path. A subdomain belongs
   to its root site (`shop.example.com` → site `example.com`); `inspect_site` accepts the root domain and its
   subdomains. Not found: tell the customer the site isn't in their TrackFlow account and offer `create_site`
   + install ([setup-playbook.md](setup-playbook.md) Phase 1). Several matches: ask which one.
2. `sites → get_site`:
   - `last_seen_at` empty means the tag isn't working yet, so say so before planning;
   - note `platform`, `auto_pageview`, `auto_forms`, `ecommerce_datalayer` and `default_currency`;
   - `conversion_events` starts as `Purchase`, `Lead` on every new site (a default, not the customer's
     choice). Set it from the approved plan in step 6.
3. `destinations → list_destinations`: which ad platforms are connected and `ready`. The plan sends events
   to these. If none are connected, plan anyway and say which credentials you'll need.
4. `tag_manager → list_tags` and `reports → list_events` (`limit: 50`): what is already tracked. Don't plan
   an event twice; mark existing ones as "already tracked".

## 2. Look at the website (read-only)

1. `tag_manager → inspect_site` on the home page (no `url`). Read:
   - `title`, `description`, `headings`, `json_ld_types`, `meta` (OpenGraph `og:type`, product price and
     currency) (what the business is);
   - `platform`, `existing_tags`;
   - `forms` (`kind`, `fields`, `builder`), `embedded_forms`, `ctas` (button text, `tel:`, `mailto:`,
     WhatsApp);
   - `checkout_links` (providers), `datalayer`, `iframes`, `suggestions`;
   - `links` (the site's own pages, up to 60; `links_total` counts every link on the page).
2. From `links`, pick up to 6 pages that show how the business converts, and inspect each. Choose pages whose
   path or text matches:

   | Business | Pages to inspect |
   |---|---|
   | Shop | a product (`/products/…`, `/product/…`, `/shop/…`), a collection or category, `/cart`. Never place an order. |
   | Leads | `/contact`, `/quote`, `/estimate`, `/get-started`, `/free-consultation` |
   | Bookings | `/book`, `/appointment`, `/schedule`, `/reserve` |
   | Services | `/services/…`, `/pricing`, `/plans` |
   | Real estate | `/listings/…`, `/properties/…`, `/homes/…` |
   | Other | `/donate`, `/apply`, `/careers`, `/signup` |

   Checkout and thank-you pages are rarely linked. Try the platform's usual paths (`/checkout`, `/cart`,
   `/thank-you`, `/order-received`, `/success`) with `inspect_site`. A 404 just means it isn't there.
3. `inspect_site` doesn't run JavaScript. Content a page builds in the browser (many single-page apps, Wix or
   Framer sections, pop-ups, cart drawers) can be missing. Plan from what you see and confirm selectors later
   in preview. If no prices are readable (no JSON-LD product, no `meta` price), say that purchase values are
   checked in preview. The `ecommerce` recipe then needs `price_selector` / `total_selector` options, or the
   site's own dataLayer or checkout webhook supplies them.

Never submit forms, add to cart or start a checkout to "test" a live site.

## 3. Work out the type of website

Combine the signals. The strongest come first:

| Signal | Points to |
|---|---|
| `json_ld_types` `Dentist`, `MedicalClinic`, `Physician`, `MedicalBusiness`, `Optician` | Dental / medical clinic |
| `HVACBusiness`, `Plumber`, `Electrician`, `RoofingContractor`, `HomeAndConstructionBusiness`, `GeneralContractor`, `HousePainter`, `Locksmith`, `MovingCompany` | Home services |
| `RealEstateAgent`, `RealEstateListing`, `Residence`, `Accommodation` with prices or listings | Real estate |
| `LegalService`, `Attorney`, `AccountingService`, `FinancialService`, `InsuranceAgency` | Professional / legal / finance |
| `Restaurant`, `FoodEstablishment`, `CafeOrCoffeeShop` | Restaurant |
| `Hotel`, `LodgingBusiness`, `TouristTrip` | Hotel / travel |
| `AutoDealer`, `AutoRepair`, `Car`, `Vehicle` | Automotive |
| `ExerciseGym`, `HealthClub`, `BeautySalon`, `DaySpa`, `SportsActivityLocation` | Fitness / beauty |
| `EducationalOrganization`, `Course`, `School` | Education / courses |
| `SoftwareApplication`, `WebApplication` | SaaS / app |
| `Event`, `EventVenue` with ticket links | Events / tickets |
| `NGO`, donate links or a donation provider | Nonprofit |
| `Product` / `Offer`, `og:type` `product`, add-to-cart CTAs, `checkout_links` to a store checkout, platform `shopify` / `woocommerce` / `bigcommerce` / `magento`, a GA4 ecommerce dataLayer | Ecommerce |
| Words in `title`, `description`, `headings`, link text | "dentist, dental, implants, braces" → dental. "AC repair, heating, furnace, HVAC" → HVAC. "plumbing, drain" → plumbing. "homes for sale, listings, realtor, rent" → real estate. "attorney, lawyer, injury" → legal. "free trial, pricing per month, sign up" → SaaS. "menu, reservations" → restaurant. "enroll, course, webinar" → education. "donate" → nonprofit |
| CTAs | "Get a quote / free estimate / call now" → local services. "Book appointment / schedule" → appointment business. "Request a demo / start free trial" → SaaS / B2B. "Add to cart / buy" → shop. "View listings / schedule a showing" → real estate |

A site can be two types: for example a clinic that also sells products. Plan for both and say so.

When the signals don't settle it, ask **one** short question, with your best guess first ("This looks like a
dental clinic that takes bookings online. Is that right?"). Otherwise state the type and why in one line, and
move on.

## 4. Choose the events

Use TrackFlow's standard names: `PageView`, `ViewContent`, `Search`, `AddToCart`, `InitiateCheckout`,
`AddPaymentInfo`, `Purchase`, `Lead`, `CompleteRegistration`, `Contact`, `Schedule`, `StartTrial`,
`Subscribe`, `SubmitApplication`, `FindLocation`, `Donate`, `AddToWishlist`. TrackFlow translates them for
every platform. Use a custom name only when no standard event fits.

Each plan has:
- **Primary conversions:** what the business is paid for. These count as conversions and the ads optimise on
  them. Usually 1–3.
- **Supporting events:** steps before the conversion, used for audiences and as fallbacks to optimise on.
- `PageView` on every page (`auto_pageview`) for retargeting, unless the customer declines it.

| Type | Primary conversions | Supporting events | How TrackFlow captures them |
|---|---|---|---|
| **Ecommerce** | `Purchase` (value, currency, order id, contents) | `ViewContent` (product), `AddToCart`, `InitiateCheckout`, `AddPaymentInfo`, `Search`; `CompleteRegistration` (account) and `Lead` (newsletter sign-up) if present. A newsletter `Lead` is recommended but not a conversion, unless the customer wants their ads to optimise on sign-ups | Shopify: Customer Events code + `ecommerce_datalayer`. GA4 ecommerce dataLayer present: `ecommerce_datalayer`. Otherwise `apply_recipe ecommerce`. Add a store webhook for purchase backup ([integrations.md](integrations.md)) |
| **Home services** (HVAC, plumbing, electrical, roofing, cleaning, pest control, landscaping) | `Lead` (quote / estimate / service request form), `Contact` (phone click: often the main conversion on mobile), `Schedule` (online booking) | `ViewContent` on service pages, `FindLocation` (directions / map click), WhatsApp or SMS click as `Contact` | `lead_form` recipe or `form_submit` tag; `click` on `a[href^="tel:"]`; booking widgets (ServiceTitan, Housecall Pro, Jobber, Calendly) via the `booking` recipe or `iframe_message` / form tags |
| **Dental / medical clinic** | `Schedule` (appointment booked), `Lead` (new-patient or contact form), `Contact` (phone click) | `ViewContent` on treatment pages, `FindLocation` | Booking embeds (Calendly, Cal.com, practice booking widgets) through the `booking` recipe or a thank-you page; `form_submit`; tel click. **Health rules below** |
| **Real estate** | `Lead` (contact agent, property inquiry, home valuation), `Schedule` (showing / viewing) | `ViewContent` on listing pages with the listing ID as `content_ids`, `Search` (property search), `Contact` (call / WhatsApp), `CompleteRegistration` (saved search / account), `SubmitApplication` (rental application) | `form_submit` on inquiry forms; `page_view` tag on listing pages with a listing-ID variable; tel click |
| **Legal / finance / insurance / consulting** | `Lead` (consultation, quote, case review), `Contact` (call), `Schedule` (consultation booked) | `ViewContent` on practice-area pages, `SubmitApplication` (loan or insurance application) | `lead_form`, tel click, `booking` recipe. **Sensitive data rules below** |
| **SaaS / B2B** | `CompleteRegistration` or `StartTrial` (sign-up), `Lead` (demo request), `Purchase` / `Subscribe` (paid plan) | `ViewContent` on pricing, `Contact` | Sign-up / demo forms; payments by checkout webhook (Stripe, Paddle, Lemon Squeezy); `decorate_checkouts` for hosted checkouts |
| **Restaurant** | `Schedule` (reservation), `Purchase` (online order) | `ViewContent` on the menu, `FindLocation`, `Contact` | Reservation / ordering widgets are often third-party: cross-domain or webhook; tel click |
| **Hotel / travel** | `Purchase` (booking) | `Search`, `ViewContent` (room / trip), `InitiateCheckout` | Booking engines usually live on another domain: `cross_domains` + `decorate_checkouts`, a webhook if available |
| **Education / courses / coaching** | `Purchase` (enrolment), `Lead` (application / enquiry), `CompleteRegistration` (webinar or free lesson) | `ViewContent` on the course page | Checkout webhooks (Kajabi, Teachable, ThriveCart…), forms |
| **Events / tickets** | `Purchase` (ticket) | `ViewContent` (event page), `InitiateCheckout` | Ticket platforms are usually off-site: cross-domain or webhook |
| **Fitness / beauty / salon** | `Schedule` (class / appointment), `Purchase` (membership, package), `Lead` (free trial pass) | `ViewContent` on class / service pages, `Contact` | Booking widgets via the `booking` recipe or form / iframe tags |
| **Automotive** | `Lead` (vehicle inquiry, test drive, finance application), `Schedule` (service appointment) | `ViewContent` on vehicle pages with the stock number as `content_ids`, `Search`, `Contact` | Forms, tel click, `page_view` tag with a vehicle-ID variable |
| **Nonprofit** | `Donate` (with value) | `Lead` (newsletter), `CompleteRegistration` (volunteer) | Donation platforms via webhook or thank-you page |
| **Content / blog / media** | `Subscribe` or `Lead` (newsletter) | `ViewContent` on articles; scroll depth only if asked | `newsletter` recipe |

Only plan events the site really supports. No form → no `Lead` from a form. No phone link → no phone-click
`Contact`. Each event in the plan names the page or button you found it on.

**Values:**
- Shop events carry real order values.
- For leads, bookings and calls, don't invent a value. Offer to add one if the customer gives an average
  ("What is a new patient worth to you on average?"). Platforms optimise better with values, but a wrong
  value misleads them.

**Health, finance and other sensitive businesses** (dental, medical, legal, finance, insurance):
- **Generic events:** use generic events and names (`Schedule`, `Lead`). Never put treatments, conditions,
  case types or financial details in event names, `content_name`, custom properties or URLs.
- **Meta restrictions:** Meta may restrict data from sites it classifies as health or finance, including
  limits on custom events and parameters. Warn the customer if events seem to be missing in Meta.
- **User data:** send customer details only in user-data fields (hashed), never in event properties
  ([event-parameters.md](event-parameters.md) "Don't send").

## 5. Show the plan and get approval

Show one short message in the customer's language, with no jargon:

```
Your site looks like a dental clinic (online booking, a new-patient form, a phone number on every page).
Here's what I recommend tracking. Your Facebook and Google Ads connections will receive these:

  #  Event       When it's sent                                   Conversion?   Recommended
  1  PageView    every page                                       –             ✅
  2  Schedule    an appointment is booked (booking widget)        ✅ main        ✅
  3  Lead        the "New patient" form is sent                    ✅             ✅
  4  Contact     someone taps the phone number                     ✅             ✅
  5  ViewContent a treatment page is opened                        –             optional
  6  FindLocation the "Get directions" link is clicked             –             optional

Reply "approve" to set up the recommended ones, or tell me the numbers you want (e.g. "1–4 and 6").
```

Rules:
- **Mark each row:** recommended rows ✅, the rest "optional". Each row says what the visitor does, not
  how you'll track it.
- **Existing tracking:** list events the site already sends as "already tracked" and leave them out of the
  numbering.
- **No destinations:** if no platform is connected, add one line saying which credentials you'll ask for
  after approval.
- **Ad accounts:** mention anything that changes what ad accounts receive or count (for example, "Calls
  will count as conversions in Google Ads").
- **Questions:** put the questions you still need under the table, in the same message, at most 4. Typical
  ones:
  - which ad platforms (if none are connected);
  - the currency (shops without a readable price and no `default_currency`; set it with `update_site`);
  - who takes payments (for a webhook backup);
  - publish now or preview first.
- **Wait:** don't build anything until the customer answers. "Approve", "yes", "go ahead" or "all" means
  the recommended rows. A list of numbers means exactly those.
- **Publishing:** approving the plan is not approval to publish unless the customer says so. Ask in the
  same message if it isn't clear: "Should I publish when done, or send you a preview first?"

## 6. Build the approved events

Follow [setup-playbook.md](setup-playbook.md) Phases 3–8 with the approved list:

1. **Settings:** turn on `auto_pageview` and/or `ecommerce_datalayer` (`update_site`) only for approved
   events.
2. **Recipes first** where one fits: `ecommerce`, `lead_form`, `booking`, `newsletter`. Then specific tags
   for the rest: tel or WhatsApp `click`, `page_view` on listing pages with an ID variable, booking
   thank-you pages ([tag-manager.md](tag-manager.md)).
3. **Conversions:** `conversions → set_conversion_events` with the approved primary conversions.
4. **Server sources:** add store or checkout webhooks where the plan needs them (purchase backup, off-site
   payments) ([integrations.md](integrations.md)).
5. **Preview and test:** `create_preview_link`, walk the funnel in a normal browser (no real orders or
   form spam), then check `list_events` (`test: "only"`).
6. **Publish** as the customer instructed, then verify live with `list_events` / `get_event`.
7. **Report:** report back in the same table, with a ✅ for each event that now arrives, and anything
   the customer still has to do (connect a platform, register a webhook).

If the customer comes back later with "track my site" for the same site, start at step 1: existing tags
show what's done, so plan only the gaps.
