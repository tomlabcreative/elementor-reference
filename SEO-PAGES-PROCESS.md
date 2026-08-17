# Migrating SEO content pages into an Elementor design

How a site's existing SEO content gets pulled in, reshaped to a new design, kept
native in Elementor, and imported into WordPress in bulk. Written up from the
Greenwoods build (Aug 2026) — **128 posts and pages across four imports**,
including 56 location landing pages — so the process is repeatable rather than
remembered.

Read alongside [ELEMENTOR-BUILD-PLAYBOOK.md](ELEMENTOR-BUILD-PLAYBOOK.md), which
covers the JSON traps. This document is the order of operations.

---

## The shape of it

```
1  Scrape the live site        →  content/*.md, SEO in front matter
2  Reconcile with client copy  →  CONTENT-STATUS.md, redirect map
3  Build ONE page by hand      →  the design, signed off in Elementor
4  Read the settings back out  →  scripts/lib/elementor.js primitives
5  Write a page spec           →  slugs, H1 rules, related links
6  Generate every page         →  generated/<type>/*.json
7  Check coverage              →  source vs built, per page
8  Emit WXR                    →  one XML, complete pages
9  Import to draft             →  Tools → Import → WordPress
```

**The order is the point.** Content first, design second, generation third. Doing
design before content is what causes copy to go missing — see stage 7.

---

## Stage 1 — Scrape the live site into structured markdown

Ask Claude to fetch every live URL and write one markdown file per page, with the
SEO fields in YAML front matter:

```yaml
---
title: "Oil Boilers"
slug: oil-boilers
url: https://client.co.uk/oil-boilers/
meta_title: "New Oil Boilers | Oil Boiler Installations - Greenwoods"
meta_description: "…"
focus_keyphrase: "New Oil Boilers Preston"
h1: "Oil Boiler Installation, Repairs & Servicing"
robots: "index, follow"
scraped: 2026-08-12
---
```

Rate-limit it (1 req/sec) and log failures. Greenwoods: **62 URLs, all 200, no
redirects.**

Also capture, in the same pass:

- a **URL inventory CSV** — every live URL with its status and meta
- the **heading hierarchy** per page, so structure is checkable later
- **content images** with their source URLs and dimensions
- the **JSON-LD** — it reveals what schema exists, and what doesn't

Ask for a similarity analysis across page families. Greenwoods' 49
`gas-oil-boiler-installations-*` pages turned out to be **90% identical** — the
same template with the town swapped. That single fact changed the migration
strategy, and it was free to discover.

> **This markdown becomes the source of truth.** Every later stage reads from it,
> never from the live site again. One scrape, many builds.

---

## Stage 2 — Reconcile against the client's own content

The client's Word docs and the live site will disagree. Get the disagreements
written down before anything is built.

Ask Claude to produce a **content status document** covering:

- which pages have copy ready and which are still owed
- **URL changes and the redirects they need** — a page splitting in two, a slug
  changing, a live page absent from the new sitemap
- meta fields over the SERP limits
- **keyword cannibalisation** — two pages targeting one term
- factual contradictions between sources

Greenwoods turned up a page claiming the company served "from Birmingham to the
Lake District" when everything else said Preston, four different spellings of the
business name, and 27 live landing pages missing from the migration list.

**None of that is design work, and all of it is launch-blocking.** Better to find
it now than during a go-live.

### The rule that saves the most rework

> **A reference mockup is a design guide, not the content layout.** It decides
> hierarchy, spacing and component style. The *client copy* decides which blocks
> exist. If the copy has content the mockup has no slot for, add a slot.

On this build a mockup was briefly treated as the content list, and roughly half
of one document's copy had nowhere to go.

---

## Stage 3 — Build one page by hand, in Elementor

Do not generate anything yet. Build the homepage — or one representative page —
in Elementor, by hand, and get it signed off.

This page is the **design system**. Everything generated later inherits from it:
palette, type scale, section padding, card treatment, button styles, the way a
photo band alternates sides.

Set the site kit up first and bind colours through `__globals__`
(`globals/colors?id=primary`) rather than inlining hex. Only the four system
slots — `primary`, `secondary`, `text`, `accent` — have predictable ids; custom
globals get random hashes, so use explicit hex for anything beyond those four.

---

## Stage 4 — Read the settings back out, don't reason about them

**The single highest-value habit in this whole process.**

Once that page works, read its actual settings out of the running install rather
than working out what they ought to be:

- **In the editor tab:** `elementor.elements` holds the live model. Any control
  can be read back.
- **On the front end:** Elementor prints settings into `data-settings` on the
  widget wrapper.

That is how two silent bugs were pinned down on this build. Both had been guessed
at wrongly first:

| Guessed | Actual |
|---|---|
| Lottie `source: "media"` | `source: "media_file"` — anything else silently renders Elementor's own `default.json` |
| Overlay opacity left at default | Defaults to `0.5`, so a "solid" gradient wash arrives at half strength |

Then have Claude turn those confirmed settings into **shared primitives** — a
small library the generators call, so every page is assembled from the same
functions instead of hand-written JSON:

```js
// scripts/lib/elementor.js
const px   = (size) => ({ unit: 'px', size, sizes: [] });
const box  = (t,r,b,l) => ({ unit:'px', top:`${t}`, right:`${r}`, bottom:`${b}`, left:`${l}`, isLinked:'' });
const gap  = (n) => ({ unit:'px', size:'', column:`${n}`, row:`${n}`, isLinked:'1' });
const link = (url) => ({ url, is_external:'', nofollow:'', custom_attributes:'' });

const container = (id, settings, elements, inner = true) => …
const widget    = (id, widgetType, settings) => …
const section   = (id, title, bg, elements, padTop = 80, padBottom = 80) => …
```

Once `section()` and `container()` are right, they are right on all 56 pages.

---

## Stage 5 — Write the page spec

One file holding the decisions, separate from the layout code:

- **slug per page**, and which slugs are live and must be preserved
- **which pages have ranking to protect** (`liveOnly`) versus which are new
- **related-service links** — three siblings per page
- **H1 rules** — Greenwoods appended the location to the six H1s that lacked one
- **front-matter → Yoast field mapping**

### The migration rule that matters most

> **On a migration the URL is the ranking asset.** Preserve slugs exactly. Carry
> meta titles and descriptions over as-is **even when they exceed the limits** —
> changing them changes what Google shows.

Greenwoods carried 48 over-length meta titles across unchanged, deliberately. The
suggested trims sit in a separate document for the SEO team to apply as a
decision, not as a side effect of the build.

**H1s can change** where the copy is being rewritten anyway — there's no "before"
to protect once the page is new.

---

## Stage 6 — Generate

Now Claude writes the generator. One function per section type, assembled per
page from the spec:

```
buildHero() · buildBand() · buildBandWithCard() · buildValues()
buildAccreditations() · buildRelated() · buildCta()
```

**When generating beats hand-building:**

| Pages | Approach |
|---|---|
| 1–2 | Build in Elementor. Generating costs more than it saves |
| 3–10 | Hand-written JSON per section, imported individually |
| 10+ | **Generator + WXR.** Below this the setup does not pay back |

The 56 landing pages took roughly the same effort as the first one did.

**Use placeholders deliberately.** Every photograph on the Greenwoods service
pages points at one known placeholder attachment. Swapping it later is a one-line
change per image, and nothing was blocked waiting on photography.

---

## Stage 7 — Check coverage before you look at anything

The check that matters most, and the one that is always skipped:

> **List every heading and paragraph in the source document and confirm each one
> reached the built page. Read the raw source, not your own parser** — otherwise a
> parser bug hides itself.

```
Checked 1301 content blocks across 56 landing pages.
59 deliberate omissions (the manufacturer logo strip, blocked on assets).
Every other heading and paragraph on every landing page is present.
```

This was learned twice on one project, both times losing content. The second time:
a parser read only the first two paragraphs per page, because the page it was
written against happened to be one of three that only had two. **53 pages lost 19
paragraphs each — and a distribution check had already printed "max 30" and been
read straight past.**

So:

1. **Never generalise a content shape from one sample. Print the distribution first.**
2. **Run coverage per page as it's built**, not at the end.
3. **Do not de-duplicate a page you're carrying over.** If the live page repeats a
   paragraph, repeat it — dropping it drops word count on a page whose whole job
   is to hold its position.

Worth automating alongside it: element-id uniqueness, no legacy `section`/`column`
elements, row widths summing under 100%, every referenced asset returning 200, and
WCAG contrast on every text/background pair.

---

## Stage 8 — Emit WXR

**This is the lever.** WordPress's importer takes postmeta, and Elementor stores a
page's entire layout in `_elementor_data` as a JSON string. So one XML file
creates dozens of *complete* pages — layout, SEO, slugs — in a single import.

Minimum postmeta per Elementor page:

```
_elementor_edit_mode       builder
_elementor_template_type   wp-page
_elementor_version         0.4
_wp_page_template          elementor_header_footer
_elementor_data            <JSON array of root containers, as a string>
```

Yoast alongside it:

```
_yoast_wpseo_title
_yoast_wpseo_metadesc
_yoast_wpseo_focuskw
_yoast_wpseo_canonical
_yoast_wpseo_meta-robots-noindex    1
```

---

## Stage 9 — Import

**Tools → Import → WordPress → Run Importer.**

**Import to draft** for anything not yet seen rendered. Greenwoods imported the
nine service pages as drafts and the 56 landing pages published, because the
landing pages were carrying live content across unchanged.

### "Download and import file attachments"

| Situation | Tick it? |
|---|---|
| Migrating blog posts with images off an old site | **Yes** — without it every image stays hotlinked to the old domain |
| Pages referencing images already in the Media Library by attachment id | **No** — there's nothing for it to act on |

### Re-importing

The importer matches on GUID and **skips anything that already exists**. To
re-import you must delete the old page *and empty the trash* — a trashed page
still holds its slug, so the import lands as `-2`.

### Import order

Anything referenced by id must exist first. A Loop Grid's `template_id` is a
WordPress post id that doesn't exist until the loop item is imported. Import the
loop item, read the id off the install, then set it. **Never invent one.**

Same for `header`, `footer` and `loop-item` templates — they cannot be imported
through Saved Templates ("this source does not support import"). Use Theme Builder
→ the matching tab.

---

## What the human still has to do

Generation does not remove judgement. Every build ends with a list like this:

- **Photography** — placeholders need real images
- **Regulated claims** — review scores, years trading, accreditations, finance
  figures. **Build the layout, leave the claim out, and say what's needed to put
  it back.** Greenwoods built OFTEC and Gas Safe as fact on every service page;
  the OFTEC number came from client artwork, the Gas Safe number is still unknown
- **Internal links** pointing at pages that don't exist until the import runs
- **The redirect map** — signed off before go-live, not after

---

## The five rules worth memorising

1. **Scrape once into structured markdown.** Every later stage reads from it.
2. **Read settings off something that works.** JSON that parses is not a page that
   works — every serious bug on this build passed structural validation.
3. **The mockup decides look; the copy decides blocks.**
4. **Preserve URLs and meta exactly on a migration.** The URL is the ranking asset.
5. **Check coverage against the raw source, per page, as you go.**
