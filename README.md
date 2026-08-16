# elementor-reference

Standing reference for WordPress + Elementor builds where pages are **generated
as JSON and imported**, rather than assembled by hand in the editor.

## What's here

**[ELEMENTOR-BUILD-PLAYBOOK.md](ELEMENTOR-BUILD-PLAYBOOK.md)** — the playbook.
When generating beats hand-building, the WXR import technique that creates
complete Elementor pages in one go, the JSON traps that fail silently, the
checks worth automating, and the content rules that stop copy going missing.

## How to use it

Copy the playbook into each new project's `docs/` at the start of the build, and
**add to it as that build teaches something new**. The copy in the project is
the working one; changes worth keeping come back here.

Write every rule with *how it was found*. A rule with its reasoning survives
contact with a deadline; a rule without one gets ignored the first time it is
inconvenient.

## Why it exists

Almost every entry in the gotchas section produces a broken page with **no error
raised** — the JSON parses, the import succeeds, and the page is silently wrong.
Each one cost a round trip to find. Reading them first is much cheaper than
rediscovering them.

## Scope

Deliberately client-neutral, so it can live in the open and be shared between
builds. Project-specific bindings — attachment ids, form ids, template ids, menu
slugs — belong in the project's own `PROJECT.md`, never here.
