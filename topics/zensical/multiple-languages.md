---
title: Multiple languages
description: Read when a Zensical site needs more than one language - no i18n support, one config per language, language selector, staying on the same page online vs offline.
---

# Multiple languages

- No i18n in Zensical (0.0.69): `mkdocs-static-i18n` is not supported (backlog https://github.com/zensical/backlog/issues/1, open); the roadmap lists i18n, not in development. Re-check https://zensical.org/roadmap/ before planning.
- `theme.language` sets one language per project (`<html lang>`, theme texts).

## One config per language

```
docs/
  zensical.en.toml   docs_dir = "en", site_dir = "site/en", theme.language = "en"
  zensical.de.toml   docs_dir = "de", site_dir = "site/de", theme.language = "de"
  en/  de/           sources (assets per language)
  site/en/  site/de/ output
```

Build each config: `zensical build -f docs/zensical.en.toml -s`, then the same for `de`. Both share `.cache/`.

## Language selector

```toml
[project.extra]
alternate = [
  { name = "English", link = "../en/", lang = "en" },
  { name = "Deutsch", link = "../de/", lang = "de" },
]
```

- `link` is relative to the site root (`../de/` = sibling site) and rendered relative to each page. The head also gets `<link rel="alternate" hreflang>`.
- Staying on the same page: the theme's JS reads the other language's `sitemap.xml` and opens the same path (anchor kept). Works only over http(s), with `site_url` set in each config (sitemap is empty without it) and the directory form `../de/`. Then the same explicit heading IDs in all languages keep the anchor too ([anchors](anchors.md)).
- Offline (file://) the sitemap is blocked: `../de/` opens a folder listing, `../de/index.html` the other language's home page. No link form fits both; for one build that serves both, a custom selector is needed. _Unverified_ which fix works best.
