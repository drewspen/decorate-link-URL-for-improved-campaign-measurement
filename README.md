# decorate-link-URL-for-improved-campaign-measurement

A **Google Tag Manager (GTM) container export** (JSON) — not a single tag template — solving a specific, tricky attribution problem: getting campaign (UTM) data and the GA4 anonymous client ID **across an iframe boundary** into an embedded CRM lead-gen form (e.g., a Pardot or HubSpot form embedded via iframe), so the CRM can capture that context on the same page view where the visitor actually converts. Per the repository's own description, this builds on the author's earlier UTM-persistence work and is inspired by an [Analytics Mania blog post by Julius Fedorovicius](https://www.analyticsmania.com/post/transfer-utm-parameters-google-tag-manager/) on transferring UTM parameters between pages.

This README is based on a direct inspection of the container export's actual contents (7 tags, 1 trigger, 7 variables, 3 folders, 5 built-in variables, and 2 bundled custom templates), cross-checked against the repository's own (detailed) description.

## The core problem this solves

Browsers treat an `<iframe>` as a completely separate document/origin from the page that embeds it — so if your marketing site's URL has `?utm_campaign=spring_sale`, and that page embeds a Pardot or HubSpot form inside an iframe, **the form has no way to see that query string**. It doesn't know what campaign brought the visitor, or what GA4 anonymous client ID identifies their session — both of which are extremely useful to capture as hidden-field values on the form so the resulting CRM lead record carries full attribution. This container solves that by **decorating the URLs of iframe (and other) elements on the parent page** with the parent's own persisted UTM values and GA4 client ID, so that when the iframe loads (or reloads) with those values now present in *its own* URL, GTM running *inside* the iframe can read them and persist them to the iframe's own first-party cookies — from which your CRM form's hidden fields can finally read them.

## The full data path (four containers, two GTM installations)

The pattern requires **two separate GTM implementations** — one on the parent page, one running inside the iframe/CRM page — connected by URL decoration:

1. **Parent page: capture the incoming UTM values.** `Parent write UTMs to session cookies` (using the bundled **Write URL query strings 2 cookies** custom template — this author's newer, consolidated evolution of the per-parameter `Cookie Creator` approach used in the earlier UTM/click-ID containers) runs as a **setup tag** on `Parent GA4 Initialization`, writing the incoming `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, and `utm_term` values to session cookies scoped to `{{Root Domain}}`.

2. **Parent page: find iframe/link elements pointing at your CRM.** A trigger, `Parent target URL link hostname found on page` (`DOM_READY`, gated by a Custom JavaScript variable of the same name), only fires once the page actually contains an element whose URL matches a **hard-coded allowlist of CRM hostnames** — as delivered, just `www.pardot.com` and `www.hubspot.com`.

3. **Parent page: decorate matching element URLs with UTM values and the GA4 client ID.** Two Custom HTML tags — `Parent decorate URL link with UTM names and values from cookies` and `Parent decorate URL link with _ga cookie name and value` — both fire on that trigger and share nearly identical logic: scan every `<iframe>`, `<a>`, `<form>`, `<area>`, `<script>`, and `<img>` element on the page (checking `src`/`data-src`/`href`/`action` as appropriate), and for any whose URL hostname is in the permitted list, append the relevant query parameters (the four persisted UTM cookies, or `_ga=<client id>`) — **without duplicating any parameter that's already present in the URL.**

4. **Inside the iframe: capture what the parent just appended.** A second, independent GTM container runs *inside* the iframe/CRM page itself. `iframe Write UTMs to session cookies` (the same custom template, configured identically) runs as a setup tag on `iframe GA4 Initialization`, capturing the UTM values now present in the iframe's own URL (because the parent decorated them in) into the iframe's own session cookies — this time scoped to `{{Page Hostname}}` rather than a root domain, since the CRM page typically lives on an entirely separate vendor domain.

5. **Inside the iframe: recover the GA4 client ID, with a fallback.** Two variables support this: `iframe obtain _ga client id from URL query string` reads `_ga` directly from the iframe's URL (populated by step 3's decoration), while `iframe Google Analytics Anonymous Browser Client ID Value` uses GTM's built-in **Analytics Storage** variable type (`dataField: client_id`) as a fallback source. Per the repository's description, the intended logic is: *use the URL-decorated value if present; otherwise fall back to the Analytics Storage–sourced value.*

6. **On the CRM form itself: read the persisted cookie into a hidden field.** `iframe populate form utm_campaign hidden field from cookie EXAMPLE` is explicitly labeled **EXAMPLE** — a minimal illustration (reading the `utm_campaign` cookie into a form input matching `input[data-sc-field-name="utm_campaign"]`) that the repository's description explicitly says will vary significantly depending on your specific form implementation and is meant to be adapted, not used verbatim.

## ⚠️ Note that no consent gating is configured on any tag

All seven tags in this export have `consentStatus: NOT_SET` — unlike this author's cookie-persistence containers (which generally require `analytics_storage` or `functionality_storage`), **nothing here is gated behind GTM's consent mode**. If your site requires consent before setting cookies or reading/writing the DOM this way, you'll need to add that yourself.

## The hard-coded CRM hostname allowlist — you must edit this

All three DOM-scanning scripts (the trigger-gating variable and both decoration tags) share this exact list:

```javascript
var permittedHosts = [ 'www.pardot.com', 'www.hubspot.com' ];
```

This is a **safety allowlist**, not a discovery mechanism — the scripts only decorate URLs whose hostname exactly matches one of these two strings. If your CRM iframe is served from a different hostname (e.g., a custom-branded Pardot/HubSpot domain, or a different CRM vendor entirely), **nothing will be decorated until you edit this array** in all three places (the gating variable, and both decoration tags) to include your actual iframe hostname(s). This matches the repository description's own guidance: *"the hosts are a list of possibilities... Alter and narrow these as needed."*

## The `Write URL query strings 2 cookies` custom template

This is a more consolidated, single-tag alternative to the per-parameter `Create`/`Create 1st`/`Rewrite 1st` × `Cookie Creator` pattern used in this author's earlier UTM/click-ID containers. Instead of three separate tags per parameter, **one tag** exposes a checkbox for each of roughly 30 known campaign/click-ID parameters — covering the standard UTMs (`utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `utm_id`, `utm_marketing_tactic`, `utm_source_platform`, `utm_creative_format`) alongside a long list of ad-platform click IDs (`gclid`, `gclsrc`, `dclid`, `gbraid`, `wbraid`, `fbclid`, `msclkid`, `ttclid`, `ttcid`, `twclid`, `li_fat_id`, `rdt_cid`, `srsltid`, `dclid`, `epic`, `igshid`, `ScCID`/`ScCid`, `cid`) — plus `cookieDuration`, `persistentCookieExpires`, `rootDomain`, and `uriDecode` settings. **In this export, only the five classic UTM parameters are enabled** (`true`); every click-ID checkbox is left `false`. `cookieDuration` is set to `sessionCookie` on both tags (no persistent first-touch variant is configured here, unlike the discrete UTM-persistence container).

## Custom templates

| Template | Purpose |
|---|---|
| **Write URL query strings 2 cookies** | The consolidated multi-parameter cookie-writer described above. |
| **Get Root Domain** | Extracts the registrable root domain from `{{Page Hostname}}`, used as the parent page's cookie `Domain`. |

## Variables

| Variable | Type | Purpose |
|---|---|---|
| `Root Domain` | Custom (Get Root Domain) | Root domain for the parent page's cookies. |
| `GA4 Measurement Stream ID` | Constant | Placeholder GA4 Measurement ID (`G-12345678`) shared by both GA4 Initialization tags. |
| `Parent target URL link hostname found on page` | Custom JavaScript | Scans the parent page's DOM for any element pointing at a permitted CRM host; gates the decoration trigger. |
| `Parent Google Tag Shared Configuration Settings` | Google Tag Config Settings | Empty scaffold for the parent GA4 Initialization tag. |
| `iframe Google Tag Shared Configuration Settings` | Google Tag Config Settings | Empty scaffold for the iframe GA4 Initialization tag — see the client-ID gap noted above. |
| `iframe obtain _ga client id from URL query string` | URL (Query component, key `_ga`) | Reads a `_ga` value from the iframe's own URL, if the parent successfully decorated it in. |
| `iframe Google Analytics Anonymous Browser Client ID Value` | Built-in "Analytics Storage" (`client_id` field) | Fallback source for the GA4 client ID inside the iframe. |

## Prerequisites

1. **Two separate GTM containers (or two workspaces within your GTM setup)** — one for the parent site, one for the iframe/CRM-hosted page, each published independently since they run on different origins.
2. **A real GA4 Measurement ID**, replacing the placeholder `G-12345678` in `GA4 Measurement Stream ID`.
3. **The actual hostname(s) your CRM iframe is served from**, to replace the `www.pardot.com`/`www.hubspot.com` placeholders in all three DOM-scanning scripts.
4. **Familiarity with your specific CRM form's hidden-field markup**, since `iframe populate form utm_campaign hidden field from cookie EXAMPLE` is explicitly a stub you'll need to adapt (and likely gate behind a visibility trigger, per the repository description) rather than something that will work unmodified.
5. Per the repository's own description, **this container does not address linking a CRM form conversion back to the parent page** — for that, the author points to [Simo Ahava's `postMessage()`/cross-site-iframe guidance](https://www.teamsimmer.com/2023/05/02/how-do-i-use-the-postmessage-method-with-cross-site-iframes/), which is outside the scope of this export.

## Getting started

### Import into Google Tag Manager

1. In GTM, go to **Admin** → **Import Container**.
2. Choose `decorate link URL for improved campaign measurement.json` from this repository.
3. Select the target container and **choose a new workspace** (recommended) rather than overwriting an existing one, so you can review the merge before publishing.
4. Choose **Merge**, and review the import summary — the export's account/container IDs and public ID (`GTM-ABCDEFGH`) are placeholder/scrubbed values, so GTM will import everything by name into your own container.
5. Confirm the import. Both bundled custom templates will import alongside the 7 tags, 1 trigger, and 7 variables.
6. **Repeat for your iframe/CRM-side GTM container**, since the "Parent" and "iframe" tags are meant to live in two separate GTM installations, both delivered together in this single export for convenience.

### Post-import checklist

- **Replace the placeholder CRM hostnames** in all three DOM-scanning scripts with your actual iframe host(s).
- **Set your real GA4 Measurement ID.**
- **Wire up the client-ID fallback** by populating `iframe Google Tag Shared Configuration Settings`'s `client_id` field, per the gap noted above.
- **Adapt the EXAMPLE hidden-field script** to your actual CRM form's markup, and attach it to an appropriate visibility trigger.
- **Add consent gating** if your compliance posture requires it — nothing is currently gated.
- **Test end-to-end in Preview mode on both containers**: confirm the parent page's UTM/`_ga` cookies get set, confirm the iframe element's `src` gets decorated with those values once the permitted-host trigger fires, confirm the iframe's own GTM container then captures those decorated values into its own cookies, and confirm your form's hidden field(s) populate correctly.

## Notes

- This container's core DOM-decoration logic runs via **Custom HTML/inline script**, because — per the repository's own explanation — GTM's sandboxed JavaScript environment's limited API surface can't perform the kind of broad DOM element scanning and attribute rewriting this technique requires.
- Credited by the author to a technique by [Julius Fedorovicius / Analytics Mania](https://www.analyticsmania.com/post/transfer-utm-parameters-google-tag-manager/), building on this author's own earlier UTM-persistence work.
- This is an unofficial, personal automation export and is not affiliated with or endorsed by Google, Salesforce/Pardot, or HubSpot. Always review a container export's tags, triggers, and variables — and test thoroughly in a sandbox workspace — before merging it into a production GTM container.

## License

No license file is currently included in this repository. Check with the repository owner before reusing this container export in a commercial or redistributed context.
