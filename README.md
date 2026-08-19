# elementor-reference

Standing references for builds where the output is **generated from a script
and imported**, rather than assembled by hand in an editor.

## What's here

**[ELEMENTOR-BUILD-PLAYBOOK.md](ELEMENTOR-BUILD-PLAYBOOK.md)** — WordPress and
Elementor. When generating beats hand-building, the WXR import technique that
creates complete Elementor pages in one go, the JSON traps that fail silently,
the checks worth automating, and the content rules that stop copy going missing.

**[LOTTIE-ICON-PLAYBOOK.md](LOTTIE-ICON-PLAYBOOK.md)** — animated icon sets
generated as Lottie JSON. How to measure a reference set instead of matching it
by eye (including the layer-scale trap that makes stroke widths lie), what
actually makes a set look like a set, the geometry and layer-order traps that
render without error, and the loop check that failed seven of ten icons the
first time it ran.

> The repo name is Elementor's, but the shelf is wider than that now. If it
> grows much further, split it — the naming is the only thing holding it back.

## How to use it

Copy the relevant playbook into each new project's `docs/` at the start of the
build, and **add to it as that build teaches something new**. The copy in the
project is the working one; changes worth keeping come back here.

Write every rule with *how it was found*. A rule with its reasoning survives
contact with a deadline; a rule without one gets ignored the first time it is
inconvenient.

## Why it exists

Almost every trap in here produces broken output with **no error raised** — the
JSON parses, the import succeeds, the animation plays, and the result is
silently wrong. Each one cost a round trip to find. Reading them first is much
cheaper than rediscovering them.

The other recurring lesson, in both playbooks: **verify by rendering, and read
values off something that already works** rather than reasoning about what they
should be. And when a measurement contradicts what someone can plainly see,
suspect the measurement.

## Scope

Deliberately client-neutral, so it can live in the open and be shared between
builds. Project-specific bindings — attachment ids, form ids, template ids, menu
slugs — belong in the project's own `PROJECT.md`, never here.
