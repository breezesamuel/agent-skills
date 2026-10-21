---
name: seo-bulk-page-enrichment
description: Turn hundreds or thousands of thin, orphaned HTML pages into indexable, linkable, monetizable and measurable pages with one batch script. Use when a user asks to "bulk enrich pages", "fix thin content at scale", "add internal links to 1000 pages", "add og images / FAQ schema / hreflang to a static site", "batch SEO remediation", or needs a repeatable, idempotent transformation applied to a whole static site.
---

# Bulk page enrichment

One script, one pass, six deterministic transformations applied to every page in
a static site — without breaking a single page, and without touching content
that is already there.

## The six transformations

1. **Open Graph / Twitter card** — `og:image`, `og:image:width/height`,
   `og:site_name`. Without `og:image`, every social share renders as a bare link.
2. **FAQPage JSON-LD** — a second `application/ld+json` block next to the
   existing `WebApplication` block. Rich results, zero page-weight cost.
3. **Depth block** — "How to use" + FAQ + related links, written in the page's
   own i18n mechanism so it is genuinely multilingual, not a translation of one
   English paragraph.
4. **Internal links** — 3–5 sibling pages from the same category + a hub link.
   Fixes the "orphan page" problem that no amount of content depth solves.
5. **Ad slot** — exactly one container per page, inert until an ad account is
   configured.
6. **First-party tracking** — a `sendBeacon` ping per page view and per
   meaningful click, to your own endpoint.

## Find the anchor points before you write the script

Read one real page first and count the anchors:

```python
for a in ["</body>", "</script>", "twitter:card", "og:image", "class=\"k\"", "<body"]:
    print(a, html.count(a))
```

You want exactly one `</body>` and one `twitter:card` per page. If the counts
are 0 or >1, your injection will land in the wrong place on some pages — fix
the selector, don't add a fallback.

## i18n: inject through the page's own dictionary

Most multi-language static pages already carry a dictionary like:

```js
window.__I = {"howto":{"zh":"…","en":"…","ar":"…"}}
```

and mark translatable nodes with `data-i="key"`. If you add a key to that
dictionary and use `data-i` in your injected markup, **your content becomes
trilingual for free** — it follows the language switcher.

Do that instead of writing a Chinese-only paragraph into an English page.

```python
m = re.search(r"window\.__I=(\{.*?\});", html, re.S)
d = json.loads(m.group(1))
d.update(NEW_KEYS)                      # {"howtoTitle": {"zh":…, "en":…, "ar":…}, …}
html = html[:m.start(1)] + json.dumps(d, ensure_ascii=False) + html[m.end(1):]
```

Parse it with `json.loads` before you write it back. If it fails, the page's
JavaScript is now broken and every page silently dies.

## Idempotency: mark what you processed

```python
MARK = "<!-- bds-enriched:v1 -->"
if MARK in html:
    return {"status": "skipped"}
...
html = html.replace("<body", MARK + "<body", 1)
```

Put the marker inside `<body>`, not in `<head>` — you want it to survive
head-only rewrites.

Re-running must be a no-op. If it isn't, you will double-inject on the next
run and ship duplicates to production.

## The trap that kills the whole batch: a syntax error in injected JS

This is the single most common way a batch enrichment breaks a site. One
unbalanced brace in the injected `<script>` makes the browser discard **that
script block only** — so the page still renders, but your tracking is dead and
nothing errors visibly.

The real example from this job:

```js
// BROKEN — the outer try{ is never closed with a catch
try{ var b=JSON.stringify({...});
    if(navigator.sendBeacon){ sendBeacon('/api/track',b) }
    else{ fetch('/api/track',{...}).catch(function(){}) };
    try{BDS_TRACK('pv')}catch(e){}      // ← this is the *inner* try
    document.addEventListener(...);
};                                       // ← closes the outer try, no catch → SyntaxError
```

A `try` without a matching `catch` is a **parse error**, so the entire block
is rejected at parse time — `window.BDS_TRACK` never gets defined.

Two defences, both required:

1. **Assert the shape after writing.** Check the function you meant to define
   is actually there:
   ```python
   assert "window.BDS_TRACK=function" in html
   ```
2. **Verify in a real browser, not by reading the string.** Load the page and
   check the DOM:
   ```js
   ({
     hasTracker: typeof window.BDS_TRACK === 'function',
     jsonld: [...document.querySelectorAll('script[type="application/ld+json"]')]
               .map(s => { try { return JSON.parse(s.textContent)['@type'] } catch { return 'BAD' } }),
     adSlot: !!document.querySelector('[data-ad-slot]'),
   })
   ```
   `BAD` in the JSON-LD list means your injected JSON is malformed. A missing
   `hasTracker` means the script block did not parse.

Run the browser check on a handful of pages from different templates, not just
the one you developed against.

## Ad slots: inert until configured

```html
<div class="ad-slot" data-ad-slot="tool-bottom" style="min-height:90px"></div>
```

```js
if (!window.BDS_ADS || !window.BDS_ADS.client) return;   // no account → no-op
```

Ship the container without a hardcoded publisher id. A placeholder id loads an
ad script that never fills — worse than no slot, because it burns the
impression and looks broken.

## Tracking: one endpoint, first-party

```js
var b = JSON.stringify({ e: e, d: d || {}, p: location.pathname, r: document.referrer || '', t: Date.now() });
if (navigator.sendBeacon) navigator.sendBeacon('/api/track', b);
else fetch('/api/track', { method:'POST', headers:{'content-type':'application/json'}, body:b, keepalive:true }).catch(function(){});
```

Server side, bucket by day and cap the size:

```
track_YYYYMMDD = { pv, events, pages: {path: n}, sources: {host: n}, ev: {type: n} }
```

Give the endpoint a GET mode that returns the aggregate (top pages, top
referrers, event breakdown). This is what lets you answer "which pages convert"
instead of guessing.

Do not send PII. Referrer hosts and paths are enough.

## Internal links from real taxonomy

Derive the categories from the site's own homepage cards (`data-ct`, `data-zh`,
`data-s` attributes) rather than inventing them:

```python
same = [s for s, v in catalog.items() if v["cat"] == my_cat and s != me]
if len(same) < 5:
    same += [s for s in catalog if s.startswith(my_prefix)]   # prefix fallback
```

Links must be **real** — point at pages that exist and load. A link graph full
of 404s is worse than no links.

## Verification checklist

Run all of these before deploying:

- [ ] Count of processed + skipped + failed adds up to the number of pages
- [ ] Every page's injected JSON parses (`json.loads` on the dictionary, `JSON.parse` on the JSON-LD)
- [ ] `<script>` / `</script>` counts match on every page
- [ ] Exactly one `<body` per page
- [ ] `window.__I` is valid JSON on every page
- [ ] A real browser renders the depth block, the links, the ad slot and the tracker
- [ ] The tracker function actually exists at runtime (not just present in the source)
- [ ] Re-running the script changes zero files (idempotent)
- [ ] No page grew past the point where it needs a `noindex` review (see below)

## The scaled-content trap

If you apply the same depth block to 1,500 pages, you have just created 1,500
near-identical pages. Search engines call that **scaled content abuse**, and it
can deindex the whole site — not just the new pages.

Mitigations, in order of effect:

- Make the injected text **derive from each page's own data** (title, category,
  pain point), so it differs meaningfully per page
- Keep the *structure* identical and the *content* specific — that's what
  "template" means; identical content is what gets punished
- Do not create a new URL per variant. Language variants need real,
  indexable URLs (`/en/`, `/ar/`) with proper `hreflang` — a client-side
  language switcher is **not** international SEO
- Never add pages to a site that already has a thin-content problem; fix the
  existing pages first

## Anti-patterns

- Regex surgery on HTML without counting the anchors first
- Injecting a `<script>` without checking it parses
- Rewriting the whole file when a targeted `str.replace` would do
- Non-idempotent scripts that double-inject on re-run
- Shipping an ad slot with a placeholder publisher id
- Adding 1,000 pages of identical text to a site Google already distrusts
