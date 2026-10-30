# First-party tracking domain, ad-blocker resistance and blocking rules

Ad blockers and Safari mostly target well-known tracking hosts and paths. TrackFlow can serve the
tracker, its config and its collect endpoint from a **subdomain of the customer's own site**
(e.g. `data.example.com`), Stape-style, with a certificate issued automatically. Advanced options
(stealth paths, renamed global, same-origin proxy) exist for developers.

All tools are in the `install` group (`install {"action": "...", ...}`); per-operation names also work.

| Tool | Purpose |
|---|---|
| `add_domain` `{site_id, host}` | Register the subdomain; returns the ONE DNS record to create |
| `verify_domain` `{site_id, host \| domain_id}` | Check DNS now; returns what is missing/wrong |
| `domain_status` `{site_id, host?}` | Domains + statuses, stealth paths, global name, proxy prefix (proxy secret masked) |
| `get_install_snippet` `{site_id, mode}` | `auto` \| `custom_domain` \| `proxy` \| `stealth` \| `standard` \| `shopify_pixel` (Shopify Customer Events code; loads from the custom domain once it is active, so **re-copy it into Shopify after a domain goes live**) |
| `get_proxy_config` `{site_id, platform}` | Ready-to-paste same-origin proxy config (contains the proxy secret) |
| `update_first_party_settings` `{site_id, stealth?, global?, rotate_paths?, rotate_secret?}` | Stealth on/off, global name, path/secret rotation |
| `remove_domain` `{site_id, host \| domain_id}` | Disconnect a domain (ask first) |

---

## 1. The 3-step custom-domain flow

**Step 1 — add the subdomain** (you do it):
```json
install {"action": "add_domain", "site_id": 12, "host": "data.example.com"}
```
Result `domain`: `{ id, host, status: "pending_dns", cookie_domain, records: [ {type, name, fqdn, zone, value, proxy} ], alternative_records: [...], message }`.

Host rules (the call is rejected otherwise):
- A subdomain of the site's domain **or** of a domain in the site's `cross_domains` list
  (cookies are scoped to that registrable domain).
- Not the site itself and not `www` ("Use a dedicated subdomain such as data.example.com").
- Not a TrackFlow hostname. Each hostname can be connected once (409 "already connected").
- Max **5** custom domains per site.
- Good names: `data.`, `t.`, `m.`, `stats.` + the site domain. Avoid words like `track`/`pixel`/`analytics`
  in the label (blocklists match them).

**Step 2 — the customer creates ONE DNS record** (their job; give exact values from `records[0]`):

| Field | Value |
|---|---|
| Type | `CNAME` |
| Name / Host | `records[0].name` — the label **relative to the DNS zone**, e.g. `data` for `data.example.com`, `data.shop` for `data.shop.example.com` in the `example.com` zone |
| Value / Points to / Target | `records[0].value` — TrackFlow's edge hostname (currently `edge.webanalyticssolution.com`; always copy it from the result) |
| Proxy (Cloudflare) | **DNS only** (grey cloud) |
| TTL | Auto / default |

Tell the customer where: "Log in where your domain's DNS is managed (GoDaddy, Namecheap, Cloudflare,
Hostinger, Squarespace Domains, Route 53…) → DNS records → Add record." Then:
```json
install {"action": "verify_domain", "site_id": 12, "host": "data.example.com"}
```
Repeat until `dns.ok` is `true` (status `verified`). TrackFlow also re-checks pending domains in the
background, so the customer only has to create the record.

Fallback only when the provider cannot create a CNAME on that name: `alternative_records` = an `A`
record to TrackFlow's edge IP **plus** a `TXT` record `_edge-verify.<label>` with the domain's
`edge-…` token (proves ownership because the IP is shared). Both are required.

**Step 3 — swap the tag** (customer pastes, you fetch) once `status` is `active`:
```json
install {"action": "get_install_snippet", "site_id": 12, "mode": "auto"}
```
`auto` returns the custom-domain loader once a domain is active (before that: stealth). Example shape:
```html
<script>window.qx7abc=window.qx7abc||function(){(window.qx7abc.q=window.qx7abc.q||[]).push(arguments)};</script>
<script async src="https://data.example.com/<prefix>/<file>.js"></script>
```
Tell the customer: remove the old TrackFlow tag, paste this in its place inside `<head>` on every page.
- The site key is built into the URL (no `data-site`).
- **The JS global is the random name in `global`** (stealth is on by default). `window.trackflow` is not
  defined. Every existing on-page call (`trackflow('track', …)`, theme/GTM snippets) must be changed to
  that name — or turn stealth off first
  (`update_first_party_settings {"stealth": false}`) and re-fetch the snippet to keep `trackflow`.
- Leaving the old tag too is not double tracking (one copy per site key runs), but whichever loads
  first wins, possibly the blockable one. Remove it.
- Tags built in TrackFlow's tag manager are unaffected (they run inside the tracker).

---

## 2. Statuses and timing

| Status | Meaning | Typical duration / what to do |
|---|---|---|
| `pending_dns` | Waiting for the record | Usually minutes after the customer saves it; up to 48 h at some providers. Re-checked in the background (roughly every 10 min) for **14 days**; after that only `verify_domain` checks it. The dashboard page also re-checks every ~20 s while open. |
| `verified` | DNS correct; server block + Let's Encrypt certificate being created | Usually **2–5 minutes**. The domain does not serve yet: do not hand out the snippet. |
| `active` | Live: `https://<host>` serves the tracker; certificate renews automatically | Swap the tag (step 3). |
| `error` | Certificate/server setup failed (`last_error`) | Retried automatically with back-off (10 min, doubling, max 24 h). An already-active domain keeps serving with its current certificate during renewal problems. Fix the cause (§5), then `verify_domain`. |

`domain_status` shows per domain: `status`, `dns` (last check: `cname`, `a`, `aaaa`, `txt_found`,
`problems[]`), `checked_at`, `verified_at`, `active_at`, `cert_expires_at`, `last_error`, `attempts`,
`next_attempt_at`, `message`.

A custom domain only serves that site's tracking files (and a `robots.txt` disallowing crawlers);
the dashboard, login and API are never reachable on it.

---

## 3. Install modes compared

| Mode | Served from | Global | Blocked by domain lists? | Server-set cookies | Needs |
|---|---|---|---|---|---|
| `standard` | TrackFlow host, `/t.js` | `trackflow` | Most likely | No | Nothing |
| `stealth` | TrackFlow host, random path | random (if stealth on) | Path rules avoided; TrackFlow domain still blockable | No | Nothing |
| `custom_domain` | `data.example.com/<prefix>/…` | random (if stealth on) | Survives most lists | Yes, on `.example.com` | One CNAME |
| `proxy` | `example.com/<prefix>/…` (relative URL) | random (if stealth on) | Hardest to block | Yes | Proxy rule on the customer's server/CDN |

---

## 4. Stealth paths, global name and rotation (API/MCP only, not in the dashboard)

Every site has a random path prefix and random file names (script, Shopify pixel loader, config,
collect, plugins dir, extensions dir), visible in `domain_status.settings.paths` (`shopify_pixel` is the
file the Shopify Customer Events code loads). The same paths work on the TrackFlow host
(stealth), the custom domain and through a proxy. With stealth on, the served script has comments
stripped and the API is under `settings.global`.

```json
install {"action": "update_first_party_settings", "site_id": 12, "stealth": false}
install {"action": "update_first_party_settings", "site_id": 12, "global": "qx7"}
install {"action": "update_first_party_settings", "site_id": 12, "rotate_paths": true}
install {"action": "update_first_party_settings", "site_id": 12, "rotate_secret": true}
```

| Change | Consequence |
|---|---|
| `stealth` / `global` | On-page code calling the old global breaks. `global` must be a plain identifier without tracking words (`track`, `pixel`, `analytic`, `collect`, `trackflow`). |
| `rotate_paths` | Old paths stop working at once: the installed stealth/custom-domain/proxy snippet (and a Shopify Customer Events code) must be replaced immediately. Use when a filter list starts blocking the site's paths. |
| `rotate_secret` | Every installed proxy config must be updated (`get_proxy_config` again). |

Ask before rotating on a live site.

---

## 5. Same-origin proxy (developers)

`get_proxy_config` with `platform` = `nginx` | `apache` | `cloudflare` | `vercel` | `nextjs` | `netlify`
| `wordpress` (aliases: `worker`, `next`, `wp`, `php`, `htaccess`). Returns
`{ platform, path_prefix, origin, where, config, notes[], test_url }`.

| Platform | Where it goes | Notes |
|---|---|---|
| nginx | `location ^~ /<prefix>/` inside the site's `server {}` | Sends the key, client IP, host; `nginx -t && systemctl reload nginx`; behind a CDN set up `real_ip` first |
| apache | `<VirtualHost *:443>` (mod_proxy, mod_proxy_http, mod_ssl, mod_headers, mod_rewrite) | Not allowed in `.htaccess`; shared hosting → WordPress option or a custom domain |
| cloudflare | Worker on route `example.com/<prefix>/*` (same zone as the site) | Store key with `wrangler secret put EDGE_KEY`; good for Cloudflare-proxied sites incl. Shopify-on-Cloudflare |
| vercel | `vercel.json` rewrites | Secret travels in the server-side destination path |
| nextjs | `middleware.ts` | Explicit headers; `EDGE_KEY` env var |
| netlify | `netlify.toml` redirect, status 200 | Visitor IP from `x-nf-client-connection-ip` |
| wordpress | must-use plugin `wp-content/mu-plugins/edge-proxy.php` | Exclude the prefix from page caching/CDN |

Then: open `test_url` (must return JavaScript, not the site's 404) → `get_install_snippet` `mode: "proxy"`
→ `<script async src="/<prefix>/<file>.js"></script>`. Only TrackFlow's own two cookies are forwarded.
The config contains the per-site **proxy secret** (proves requests come through the customer's proxy so
the forwarded visitor IP is trusted): hand over the whole config, never quote the secret alone; rotate
it if it leaks.

---

## 6. Server-set cookies (custom domain or verified proxy)

| Cookie | Script-readable | Lifetime | Purpose |
|---|---|---|---|
| `_<prefix>` | No (HttpOnly) | 400 days | Durable visitor ID; restores `_tf_vid` if script storage was cleared |
| `_<prefix>c` | No (HttpOnly) | 90 days | Click-ID bag (fbp, fbc, ttp, ad click IDs) added to server copies |
| `_tf_vid` | Yes | 400 days | Visitor ID |
| `_fbp`, `_fbc` | Yes | 90 days | Meta browser / click IDs |
| `_ttp` | Yes | 390 days | TikTok browser ID |
| `_tf_<click id>` | Yes | 90 days | gclid, gbraid, wbraid, dclid, msclkid, ttclid, sccid, epik, twclid, rdt_cid, li_fat_id, qclid, obclid, tblci, irclickid |

All `Secure; SameSite=Lax; Path=/`, `Domain=.<registrable domain>` (host-only through a proxy that
doesn't forward the host). Set only on responses to accepted events, and with consent mode on only for
consented events.

**Limits (tell the customer honestly):**
- **Safari** caps cookies set by a server whose IP differs from the site's own IP at **7 days**. A custom
  subdomain points at TrackFlow's server → 7-day cap applies. A same-origin proxy avoids it.
- Some blockers uncloak CNAMEs and block subdomains pointing at known tracking hosts (the A + TXT option
  has no CNAME to follow, but doesn't avoid Safari's IP check).
- The platforms' own scripts (`connect.facebook.net`, `analytics.tiktok.com`, `googletagmanager.com`…)
  can still be blocked; the **server copy still arrives**. Platforms relying on the browser for some
  events (GA4 browser events; TikTok/Microsoft/Quora page views; Outbrain without click ID) lose those.
- Filter lists can target a specific site; rotate paths if that happens.
- Consent still applies: with consent mode on, nothing is tracked and no cookie is set before consent.

---

## 7. Troubleshooting

`verify_domain` returns `dns.problems[]` in plain language. Map them:

| Symptom / problem text | Cause | Fix |
|---|---|---|
| "No DNS record found … yet" | Record not created, wrong name, or still propagating | Check the name is relative to the zone (`data`, not `data.example.com` when the provider appends the domain → `data.example.com.example.com`); wait a few minutes; verify again |
| "… resolves to <IPs> (Cloudflare proxy). … set this record to DNS only" | Cloudflare orange cloud | Click the cloud → grey (**DNS only**) |
| "… is a CNAME to X. Point it to <edge host> instead." | Typo or old target (e.g. a previous Stape/other tagging host) | Edit the CNAME value |
| "… is a CNAME to <edge host>, but that name does not resolve … yet" | Edge host resolution lagging | Wait, verify again |
| "The A record is correct. … also add a TXT record `_edge-verify.<host>` …" | A-record route without ownership proof | Add the TXT (value = the `edge-…` token), or switch to the CNAME |
| "Remove the AAAA record(s) …" | An IPv6 record on the name (often a wildcard `*` AAAA or a parked-domain default) | Delete AAAA records for that name |
| "… resolves to <IPs>, not <edge IP>" | An A record from hosting/parking/wildcard | Delete it and create the CNAME (a CNAME can't coexist with other records on the same name) |
| Record created in the wrong zone | Subdomain delegated elsewhere, or record added at the registrar while DNS is hosted elsewhere (Cloudflare, hosting panel) | Create it where the domain's **nameservers** point (the provider listed in the NS records) |
| Status `error` after `verified` | Certificate issuance failed | Check `last_error`. If the domain has **CAA** records that don't allow Let's Encrypt, add `0 issue "letsencrypt.org"` at the zone; ensure the CNAME is still DNS-only; it retries automatically |
| `active` but no events | Old tag still first / snippet not swapped / on-page code uses `trackflow` with a stealth global | Replace the tag; use the `global` name |
| Works except Safari attribution after a week | Safari 7-day cap | Expected; same-origin proxy if it matters |
| Pending for > 14 days | Background checks stopped | Fix DNS, then `verify_domain` |

**Removing a domain:** first move the site to another snippet (`get_install_snippet` `standard` or
`stealth`) and have the customer install it; then `remove_domain` (confirm with the customer). Tracking
through that hostname stops and its certificate/server block are removed shortly after.

---

## Blocking rules and data hygiene

Blocked events are stored in the Events log with status `blocked` and a reason, and are **never
delivered** to any destination.

### Built-in filters (site settings via `sites → update_site {"settings": {...}}`)

| Setting | Default | Effect |
|---|---|---|
| `block_bots` | on | Browser events with no user agent or a crawler/monitor/automation UA (search & SEO crawlers, AI crawlers like GPTBot/ClaudeBot/CCBot/Bytespider, link-preview fetchers, uptime monitors, HeadlessChrome/Puppeteer/Playwright/Selenium, curl/wget/python-requests/axios…). Reason "Bot traffic". The browser tags skip these too. |
| `block_reloads` | on | The PageView of a page reload. Reason "Page reload". Browser tags skip it too. The visit still updates the visitor's journey. |
| `consent_mode` | off | Browser events without marketing consent are never sent; the server also blocks any event explicitly marked declined ("No marketing consent"). |
| `dedup` | on | Each `event_id` + event name is stored and sent once per site. Keep on. |

Built-ins apply to **browser** events only.

### Custom rules (all origins: browser, server API, webhooks, CRM, offline)

```json
install {"action": "create_blocking_rule", "site_id": 12, "type": "ip", "pattern": "203.0.113.0/24", "note": "Office"}
install {"action": "list_blocking_rules", "site_id": 12}
install {"action": "update_blocking_rule", "site_id": 12, "rule_id": 3, "enabled": false}
install {"action": "delete_blocking_rule", "site_id": 12, "rule_id": 3}
```

| `type` | Matches | Pattern examples |
|---|---|---|
| `ip` | Visitor IP: exact IPv4, IPv4 CIDR, or wildcard; an exact IPv6 address is also accepted (no IPv6 ranges) | `203.0.113.7`, `10.0.0.0/8`, `192.168.1.*` |
| `url` | Page URL, case-insensitive. `*` = wildcard; without `*` it matches anywhere in the URL | `/admin`, `*staging.example.com*`, `*?preview=*` |
| `user_agent` | UA contains text (case-insensitive) | `HeadlessChrome`, `MyMonitor` |
| `referrer` | Referrer contains text (case-insensitive) | `spam-domain.example` |
| `event` | Event name equals pattern exactly (case-sensitive) | `AddToWishlist` |

- First matching enabled rule blocks; reason `Rule: <type> <pattern>`; each rule counts its hits.
- Takes effect immediately. Prefer `enabled: false` to deleting.
- Webhook events carry no visitor IP, so IP rules never match them.
- A blocked event still occupies its event ID: a later event with the same ID and name is treated as a
  duplicate and not sent.

Typical rules: office/agency IP (keeps internal tests out of ad optimisation), staging URLs using the
live snippet, a QA monitor's UA, pausing one event name while its tag is fixed.

**Related:** consent behaviour and CMP detection — [tracker-js.md](tracker-js.md) §6.5; event
statuses — [reports-and-audit-data.md](reports-and-audit-data.md).
