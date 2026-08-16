# Elementor build & import playbook

Cross-project reference for WordPress + Elementor builds where pages are
generated as JSON rather than assembled by hand. Written after a 130-page
heating and plumbing build (Aug 2026); intended to be **copied into each new project and added to**.

Everything here was learned by getting it wrong first. The gotchas section in
particular is a list of things that silently produce a broken page — no error,
no warning, just wrong output — so it is worth reading before writing any JSON.

> **Keeping it useful.** When a build teaches something new, add it here and say
> how it was found. A rule with its reasoning survives; a rule without one gets
> ignored the first time it is inconvenient.

---

## 1. When to generate rather than build by hand

| Pages | Approach |
|---|---|
| 1–2 | Build in Elementor. Generating costs more than it saves |
| 3–10 | Hand-written JSON per section, imported individually |
| 10+ | **Generator script + WXR import.** Below this the setup does not pay back |

The build this came from crossed that line at the service pages and never went back.
The 56 landing pages took roughly the same effort as the first one did.

---

## 2. The WXR import — the big lever

WordPress's importer takes postmeta, and Elementor stores a page's entire layout
in `_elementor_data` as a JSON string. So a single WXR file can create dozens of
**complete** pages — layout, SEO fields, slugs — in one import.

Minimum postmeta for an Elementor page:

```
_elementor_edit_mode       builder
_elementor_template_type   wp-page
_elementor_version         0.4
_wp_page_template          elementor_header_footer
_elementor_data            <the JSON array of root containers, as a string>
```

Yoast alongside it:

```
_yoast_wpseo_title
_yoast_wpseo_metadesc
_yoast_wpseo_focuskw
_yoast_wpseo_canonical
_yoast_wpseo_meta-robots-noindex    1
```

### "Download and import file attachments"

Tick it **only** when the WXR declares `<item>` entries of
`wp:post_type = attachment`. That is the case when migrating media off an old
site. If every image is already in the library and referenced by attachment id,
leave it unticked — there is nothing for it to act on.

### Re-importing

The importer matches on GUID and **skips anything that already exists**. To
re-import a page you must delete the old one *and empty the trash* — a trashed
page still holds its slug, so the import lands as `-2`.

Plan for this: import to **draft** first on anything you have not seen render.

---

## 3. Elementor JSON gotchas

Each of these produces a silently wrong page. No error is raised.

| Thing | The trap | The fix |
|---|---|---|
| **Container width** | `_element_width` is a *widget* control. On a container it does nothing | `width: {unit:'custom', size:'auto'}` plus `_flex_size`, `_flex_grow`, `_flex_shrink` |
| **Lottie source** | `source: 'media'` is not a valid option value. Elementor silently renders its own `default.json` | `source: 'media_file'`, and `source_json` needs the real attachment `id`, not just a url |
| **Background overlay** | `background_overlay_opacity` defaults to **0.5**, so a "solid" wash arrives at half strength | Set it explicitly to `1` |
| **Wrapping rows** | Widths are a % of the container, gaps are fixed px. As the container narrows the gaps take a bigger share, so a row tuned at 1240 still drops a card at 1100 | A fixed-count row gets `flex_wrap: 'nowrap'` with `flex_wrap_tablet: 'wrap'`. Cards shrink instead of dropping |
| **Image sizing** | Setting `height` + `object-fit` without a `width` leaves the image free to render at natural size. With `_flex_shrink: 0` nothing pulls it back and it overflows its column | Set `width` in px and let height follow the aspect ratio |
| **Element ids** | Must be unique **within** a document. They may repeat across documents | Generate from a per-page counter. Assert uniqueness before writing |
| **Template type** | `header`, `footer` and `loop-item` cannot be imported through Saved Templates — you get "this source does not support import" | Import via Theme Builder → the matching tab |
| **Loop grid** | `template_id` is a WordPress post id that does not exist until the loop item is imported | Import the loop item first, then set it. Never invent one |
| **Global colours** | Only the four system slots (`primary`, `secondary`, `text`, `accent`) have predictable ids. Custom globals get random hashes | Bind the four; use explicit hex for the rest |
| **Media re-uploads** | WordPress keeps both copies and appends `-1`, `-2`. Anything referencing the old attachment keeps working but drifts out of sync with what is live | Read ids off the install before each build, not from memory |

### Third-party embeds and forms

- Forms go in the native **`shortcode`** widget, never an HTML widget.
- A vendor `<script>` loader is the one case for the **`html`** widget — no
  native widget emits a script tag.
- Saving a `<script>` needs the `unfiltered_html` capability. Administrators on
  a single site have it; Editors and Multisite users do not, and WordPress
  silently strips the tag.
- **Plugin styling is not reachable from Elementor.** Scope CSS to the widget
  with `selector`, and keep inputs at 16px so iOS does not zoom on focus.
- Both render as a placeholder in the editor. Judge them on the front end.

---

## 4. Verify by rendering, not by validating

The single most expensive habit to break: JSON that parses is not a page that
works. Every serious bug on the build this came from passed structural validation.

**Read settings off something that already works** rather than reasoning about
what they should be:

- In the editor tab: `elementor.elements` holds the live model. Any control can
  be read back.
- On the front end: Elementor prints settings into `data-settings` on the
  widget wrapper.

That is how the Lottie `media_file` value and the overlay opacity were pinned
down. Both had been guessed at wrongly first.

For animations and generated graphics, render them and measure. A bounds check
on the source is not enough — a Lottie file can be structurally perfect and draw
entirely off-canvas.

---

## 5. The checks worth automating

Cheap to write, and each one caught a real defect:

| Check | Catches |
|---|---|
| Element id uniqueness and length | Duplicate ids that break a page on import |
| No legacy `section` / `column` elType | Architecture drift |
| Row width sum + gaps < 100% | Columns dropping to the next line |
| Every referenced asset returns 200 | Broken images, missing Lottie files |
| WCAG contrast on every text/background pair | Unreadable copy, before anyone sees it |
| **Content coverage: source document vs built page** | The big one — see below |

### The content coverage pass

Before calling any page done, list every heading and paragraph in the source
document and confirm each one reached the build. **Read the raw source, not your
own parser** — otherwise a parser bug hides itself.

This was learned twice on the same project:

1. A mockup was treated as the page's content list, so roughly half of one
   document's copy had nowhere to go and silently vanished.
2. A landing-page parser read only the first two paragraphs, because the page
   sampled happened to be one of three that only had two. 53 pages lost 19
   paragraphs each. **A distribution check had already printed "max 30" and it
   was read straight past.**

Never generalise a content shape from one sample. Print the distribution first.

---

## 6. Content and SEO rules

- **A reference mockup is a design guide, not the content layout.** It decides
  look — hierarchy, spacing, component style. The client copy decides what
  blocks exist. If the copy has content the mockup has no slot for, add a slot.
- **Regulated claims get flagged, not transcribed.** Review scores, years
  trading, accreditations, finance figures, service promises. Build the layout,
  leave the claim out, and say what is needed to put it back.
- **On a migration the URL is the ranking asset.** Preserve slugs exactly.
  Meta titles and descriptions carry over as-is even when they exceed the
  limits — changing them changes what Google shows.
- **H1s can change** when the copy is being rewritten anyway; there is no
  "before" to protect once the page is new.
- **Do not de-duplicate a page you are carrying over.** If the live page repeats
  a paragraph, repeat it. Dropping it drops word count on a page whose whole job
  is to hold its position. Tidying is a separate, later conversation.
- **Never invent an install binding** — attachment ids, form ids, template ids,
  menu slugs, registration numbers. Read them off the install and record them.

---

## 7. Repository shape that worked

```
content/           source copy, one .md per page, front matter carrying SEO
generated/         Elementor JSON, subdivided per page
  <page>/          only the current version of each section
scripts/           generators and checks
  lib/             shared Elementor primitives, parsers, page specs
docs/              conventions, this playbook, kit spec
references/        design mockups
assets/            production assets
```

**Only the current version of a file stays on disk.** Delete the superseded one
in the same commit that adds its replacement — git holds the history, and a
folder of `v1` through `v5` makes it genuinely ambiguous which to import. That
ambiguity cost one wasted import on this build.

Exception: a file that is a *different component* rather than an older one.

---

## 8. Audit — the build this came from, August 2026

**Delivered:** 74 commits · 34 Elementor templates · homepage (8 sections),
10 service pages, three standalone pages, a news page, header, footer, 23
migrated blog posts and 56 location landing pages. Four WXR imports covering 128
posts and pages.

### What worked

- **Generating rather than hand-building.** The 56 landing pages cost about
  what the first one did.
- **Scraping source content once** into structured markdown with the SEO in
  front matter. Every later build read from it instead of from the live site.
- **Reading the live site to settle questions** — the gradient value, the Lottie
  settings, the fact that the landing pages repeat themselves. Every time this
  replaced a guess it saved a round trip.
- **Writing the check that would have caught the bug**, immediately after the
  bug. Both coverage checkers exist because something got through.

### What cost time

- **Asserting a fix without rendering it.** The header wrapping issue took three
  rounds because the first two "fixes" were reasoning, not measurement.
- **Generalising from one sample**, twice, both times losing content.
- **Not acting on a signal already in front of me** — the distribution output
  said "max 30 unique paragraphs" and the build still read two.
- **Shell quoting.** Several edits were mangled by backticks and `$` inside
  heredocs. Use the file-edit tool for code, not shell string surgery.

### What to do differently next time

1. Run the content coverage pass **per page as it is built**, not at the end.
2. Print the distribution of any content shape before writing a parser for it.
3. Import one page and look at it before generating the other fifty-five.
4. Set up the checks in `scripts/` on day one — they are what makes generating
   safe at volume.
