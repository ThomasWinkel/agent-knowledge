---
title: Zensical
description: "Zensical static site generator (successor of Material for MkDocs): project layout, offline/file:// sites, heading anchors and link checks, multiple languages."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# Zensical

Zensical is the static site generator by the Material for MkDocs team (Rust core, Python package `zensical`, Material theme). It reads `zensical.toml` natively and `mkdocs.yml` for compatibility. Read before setting up a Zensical site, building it for offline use (file://, installer), linking to its headings from outside, or planning more than one language. Docs: https://zensical.org/docs/.

## Key facts

- Alpha (0.0.x), releases every few days. Pin the exact version (e.g. `uv.lock`) and update deliberately; after an update re-check the offline gotchas. Latest version: `https://pypi.org/pypi/zensical/json` (`info.version`).
- One project = one config file; `docs_dir`/`site_dir` must lie inside the config's folder: [project-layout](project-layout.md).
- Offline sites need `[project.plugins.offline]` plus several fixes, or pages load from CDNs: [offline-sites](offline-sites.md).
- `## Heading {#my-id}` sets a stable anchor; auto IDs follow Python-Markdown's `toc.slugify`; the build checks anchors of internal links: [anchors](anchors.md).
- No i18n support: one config per language, joined by the language selector: [multiple-languages](multiple-languages.md).
- Commands: `zensical build -f <config> -s` (`-s` = strict: fail on warnings), `zensical serve -f <config>`.
- Verified with Zensical 0.0.69 on Windows (2026-10). Statements marked _Unverified_ are untested.

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Heading anchors and link checks](anchors.md) — Read when linking to a heading of a Zensical site from outside (app help, other sites), choosing stable heading IDs, or checking that links and anchors exist - explicit IDs, slug rule, strict build.
- [Multiple languages](multiple-languages.md) — Read when a Zensical site needs more than one language - no i18n support, one config per language, language selector, staying on the same page online vs offline.
- [Offline sites (file://, no external requests)](offline-sites.md) — Read when a Zensical site must open via file:// (e.g. from an installer) or make no external requests - offline plugin, CDNs to remove, polyfill bug, Mermaid, checks, browser test.
- [Project layout and configuration](project-layout.md) — Read when creating a Zensical site, putting several sites in one repository, or sharing content between sites - config file, docs_dir/site_dir rules, snippets, TOML limits.
<!-- END GENERATED CONTENTS -->
