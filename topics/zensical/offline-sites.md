---
title: Offline sites (file://, no external requests)
description: Read when a Zensical site must open via file:// (e.g. from an installer) or make no external requests - offline plugin, CDNs to remove, search shim, Mermaid, checks, browser test.
---

# Offline sites (file://, no external requests)

One build can serve both installer (file://) and web server.

## Config

```toml
[project.extra]
polyfills = [ "assets/vendor/iframe-worker-shim.js" ]
# Zensical <= 0.0.71: the string form renders an empty <script src="">; use the table form:
# polyfills = [ { path = "assets/vendor/iframe-worker-shim.js", type = "text/javascript", async = false, defer = false } ]

[project.theme]
font = false            # no Google Fonts
features = [ "navigation.footer", "search.highlight" ]   # but never "navigation.instant": not supported offline

[project.markdown_extensions]
# leave out pymdownx.emoji (twemoji from jsdelivr) and pymdownx.arithmatex (MathJax)

[project.plugins.offline]
```

- `[project.plugins.offline]` forces `use_directory_urls = false` (links end in `.html`) and writes the search index as `search.js` (search works over file://).
- `zensical new` template loads from CDNs: Google Fonts, iframe-worker shim (unpkg), MathJax (arithmatex), twemoji (emoji). Remove or replace each.
- iframe-worker shim (needed for search over file://): save https://unpkg.com/iframe-worker@1.0.4/shim/index.js (MIT, keep its LICENSE) under `docs/assets/vendor/`. The file name must contain `iframe-worker`, otherwise the theme still loads it from unpkg.
- Mermaid: the bundle loads it from unpkg only if the global `mermaid` is undefined -> ship a local `mermaid.min.js` (~3.5 MB) via `extra_javascript`.
- The bundle still contains CDN URLs (glightbox, ace, pyodide, mathjax); they are fetched only when the feature is used.
- 0.0.72 added the MkDocs privacy plugin (downloads external assets at build time; source: release notes 0.0.72). _Unverified_ as a replacement for the fixes above.

## Check

No external URL in the output:

```sh
grep -rhoE '(src|href)="https?://[^"]+' site | sort -u
```

Browser test over file:// with Playwright and the installed Edge (no browser download). Search input and results live in an open shadow root: typing works after the click, but light-DOM selectors find no results.

```sh
uv run --no-project --with playwright python test.py
```

```python
from playwright.sync_api import sync_playwright
import pathlib
with sync_playwright() as p:
    page = p.chromium.launch(channel="msedge").new_page()
    external = []
    page.on("request", lambda r: r.url.startswith(("file:", "data:", "blob:")) or external.append(r.url))
    page.goto(pathlib.Path("site/index.html").resolve().as_uri())
    page.click("button.md-search__button")   # search input is injected by JS: click first, then type
    page.keyboard.type("install")
    page.wait_for_timeout(2000)
    hits = page.evaluate("[...document.querySelectorAll('*')].find(e => e.shadowRoot)"
                         ".shadowRoot.querySelectorAll('a[href]').length")
    assert hits > 0 and not external, (hits, external)
```

## Gotchas

- Over file://, `fetch`/XHR of local files is blocked (CORS, origin `null`): features that read `sitemap.xml` do nothing (e.g. language selector staying on the page, [multiple-languages](multiple-languages.md)).
- Opening `file:///...page.html#anchor` from a Windows program can lose the `#anchor`: [vsto-addins/ribbon-click-help](../vsto-addins/ribbon-click-help.md).
