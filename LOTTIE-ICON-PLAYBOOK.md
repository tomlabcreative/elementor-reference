# Lottie icon playbook

Cross-project reference for building animated icon sets in code — generating
Lottie JSON from a script rather than drawing it in After Effects.

Written after a ten-icon set for a heating client (Aug 2026), which took six passes
and a lot of client review to get right. Almost everything below is a thing
that went wrong first, and the reason it went wrong is kept with it: a rule
with its cause survives, a bare rule gets ignored the moment it's inconvenient.

> **When NOT to use this.** If the budget will carry a motion designer, that is
> the better answer and this document should not talk you out of it. See §8.

---

## 1. Measure the reference. Do not eyeball it.

If the client has named a site they want to match, the reference files are
usually downloadable, and they will answer questions that arguing about taste
never will.

Getting them: the animations are usually fetched by a page script, so the URL
sits in a data attribute rather than the markup. Look for the player's config
blob and pull the path out of it:

```js
document.querySelectorAll('.some-lottie-class').forEach((e) => {
  const cfg = (e.getAttribute('data-lottie-settings') || '')
    .replace(/\\\//g, '/').replace(/\\"/g, '"');
  const m = cfg.match(/"json_url":"([^"]+)"/);
  if (m) console.log(m[1]);
});
```

Cache them in `references/` with a README saying they are reference only and
must not ship.

Then measure. Useful numbers, in rough order of how much they explain:

| Measure | What it tells you |
|---|---|
| **Effective stroke width** (see §2) | Whether the set has one weight or several |
| Layers per icon | How much of the "slickness" is parts count |
| Strokes vs fills | Whether it is line art or filled shapes |
| Distinct fill colours | How disciplined the accent is |
| Duration and fps | The rhythm of the set |
| Animated properties per icon | How the motion is distributed |

---

## 2. The layer-scale trap

**This is the single most expensive mistake in the whole build, and it was made
twice.**

Read the stroke widths straight out of a Lottie file and they will look wildly
inconsistent — one reference set appeared to run 0.6 to 6.4. That
led to building a three-tier weight system, which the client immediately
rejected as too bold.

The widths vary because **the layers are scaled**. A designer draws at whatever
size is convenient and sets the stroke so the *rendered* result matches. A 0.6
stroke on a layer scaled 308% and a 6.4 stroke on a layer scaled 30% both land
in the same place.

Always compute the effective width:

```js
const effective = strokeWidth * (200 / doc.w) * (layerScale / 100);
```

Corrected that way, every stroke across the fourteen reference icons fell
between **1.85 and 2.24**. One weight. The client's very first note — "theirs
are consistent, ours are not" — had been right all along; the measurement was
what was wrong.

**Lesson beyond Lottie:** when a measurement contradicts what someone is
plainly seeing, suspect the measurement.

---

## 3. What actually makes a set look like a set

Ranked by how much difference each made:

1. **One stroke weight, everywhere.** Enforce it in the build (§6).
2. **Line art, not filled shapes.** The reference ran 164 strokes to 20 fills.
   Bodies are transparent outlines; only the accent is filled. Filling bodies
   white makes icons read heavy and flat, and it causes stacking bugs.
3. **One accent element per icon**, always a fill, always the part carrying the
   most movement. Sixteen of those twenty fills were the single accent colour.
4. **Shared components.** The reference used an identical boiler under both its
   servicing and repairs icons — same body, same dial, same pulse. Build one
   cabinet, one dial, one drop, one set of legs, and have every icon use them.
   This is what stops ten drawings looking like ten drawings.
5. **Draw outlines of solid forms, not centre-lines.** A tap drawn as a
   centre-line gooseneck reads as a desk lamp, because a line has no thickness.
   The same object drawn as a closed silhouette reads immediately.
6. **Polyrhythm.** Three motions on different clocks at once — a fast pulse, a
   slow drift across the whole loop, one directional move. Nothing landing on
   the same beat is most of what "slicker" means.

---

## 4. Traps that produce silently wrong output

Each of these renders without error.

| Trap | What happens | Fix |
|---|---|---|
| **Layer order** | Lottie draws **first-in-array ON TOP**, for layers and for items within a group. An accent listed before its body covers the body's own outline | List tools first, bodies next, accents last. Treat this as the default check whenever two parts touch — it caused **four separate faults** in one build |
| **Anchor vs position** | A point renders at `position + (point − anchor)`. Setting the anchor to the pivot while drawing the shape around the origin cancels the position out and drops the part at 0,0 | Draw rotating parts around the ORIGIN, keep anchor `[0,0]`, and carry them into place with position |
| **Double offset** | A shape drawn in absolute coordinates on a layer that is also positioned gets its offset applied twice | Absolute shapes go on layers at position `[0,0]`; only local-coordinate shapes get a position |
| **Bezier handle length** | Handles at the full radius overshoot at the sides and flatten the curve between them. This is what makes a "round" bottom look flat | A circular arc needs handles of **0.5523 × radius** (kappa). Nothing about moving the vertices will fix it |
| **Smoothed vertices** | Smoothing every point turns a flame into a leaf and a teardrop into an almond | Sharp features need **zero-length tangents** at that vertex. A teardrop is a corner at the apex and a true arc at the base |
| **Symmetry** | A shape widest at its middle reads as an almond however sharp the tip is | Push the widest point off-centre — low for a teardrop, low for a flame |
| **Overlapping parts** | Separate stroked rectangles meeting each other draw an internal edge at every junction, which reads as clutter | Either trace the whole object as one closed path, or give the part on top a white fill so it occludes what it crosses |
| **Trim-path loops** | Animating trim start/end between fixed values snaps back at the loop point | Animate the trim **offset** through a full 360 instead — offset wraps, so first and last frame are identical by construction |
| **Partial rotations** | A cog rotating 0→136° jumps back at the loop | Whole turns only |

---

## 5. Motion notes

- **Match the pulse to the object.** A 4-keyframe snap over 11 frames is right
  on a small dial and reads as a sudden blast on a large disc. Large elements
  want a slow swell across the whole loop.
- **Rotation beats squash.** The reference animated its flame as two lens
  shapes counter-rotating ±7°, with no scaling at all. Squash-and-stretch on an
  icon reads as wobble.
- **Stagger with `ip`/`op`.** Parts that should appear and leave — falling
  drops — get their own visibility window rather than an opacity fade.
- **Vary the durations** across the set, roughly 2–3s, so a grid of icons does
  not pulse in unison.

---

## 6. Make the build refuse bad output

The checks are cheap and each one caught something real.

```
one stroke weight       reject any icon containing a second
accent discipline       exactly one accent-coloured fill (a twin counts as one)
bounds                  every vertex inside the canvas, allowing for motion
loop seam               every animated property returns to its start value
```

**The loop check is the one to write first.** "It stops and starts" is a
testable claim, and on its first run it failed **seven of ten** icons — two
cogs and a valve wheel stopping part-way through a turn, and a tank that filled
then jumped back to empty. Only one of those had been noticed by eye.

```js
const first = prop.k[0].s, last = prop.k[prop.k.length - 1].s;
// rotation and trim offset are exempt on whole turns: 360 and 0 are the same place
const ok = first.every((v, i) => wraps
  ? Math.abs(v - last[i]) % 360 < 0.01
  : Math.abs(v - last[i]) < 0.01);
```

Layers with their own `ip`/`op` window are exempt — a staggered drop is
supposed to appear and leave.

---

## 7. Reviewing the work

- **Render and screenshot. Every time.** Every serious fault in this build
  passed structural validation. Stacking, silhouette and proportion problems
  are invisible to every numeric check.
- **Show it beside the reference at matched size.** Faults that are invisible
  alone are obvious in a pair.
- **Build a review harness once** — the icons at display size, slow motion,
  freeze, single-frame step, a loop-seam test that flips between first and last
  frame, and a background grid. It pays for itself in one round.
- **Check it at the size it will actually be used.** Detail that reads at 190px
  becomes a smudge at 96px. If the design calls for small icons, that caps how
  much detail is worth drawing — and it may be the layout that needs to change,
  not the icon.
- **Take the client's eye seriously.** On this build the client was right and
  the measurement was wrong, twice. They cannot always name the cause — "the
  lines are inconsistent" turned out to mean "there is no weight hierarchy",
  and "it looks like a gas pump" meant "you have drawn a centre-line" — but the
  observation was sound both times. Diagnose the cause, do not argue with the
  symptom.

---

## 8. Know the ceiling

Check the reference's provenance before promising to match it. Internal names
and layer names give it away: one reference set was numbered
`1_Heating` through `14_Air conditioning`, and several layer names decoded as
mis-encoded Russian — a bespoke set commissioned from a motion designer.

Hand-authored geometry closes most of the gap and will not close all of it. The
honest measure is parts count: the finished hand-authored set carried ~5.8 moving
parts per icon against the reference's 9.07.

Say so early. And note that a finished set is itself a precise brief — if the
client does commission the work, the exact subjects, palette, weight, timing
and motion are all already specified.

---

## 9. Shape of the generator

```
assets/lottie/<version>/
  _generate.js     primitives, motion helpers, shared components, the ten icons,
                   and the audit — one file, run to regenerate everything
  *.json           output, overwritten on every run
  README.md        the rules this set is built on, and what is not matched
references/<name>/ the reference set, cached, marked do-not-ship
exports/<name>/    a clean copy for sending out, with upload notes
```

Keep each pass in **its own folder**. Version numbers in filenames get confusing
fast; a folder makes it obvious which set is current, and makes it safe to keep
the previous one while a review is running.

Regenerating everything from one script means a fix to a shared component — a
teardrop, a set of legs — lands on every icon that uses it at once. Several of
the notes in the final review rounds were one-line changes for that reason.
