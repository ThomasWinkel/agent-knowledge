---
title: Project layout and configuration
description: Read when creating a Zensical site, putting several sites in one repository, or sharing content between sites - config file, docs_dir/site_dir rules, snippets, TOML limits.
---

# Project layout and configuration

- One project per config file; the config's folder is the project root. Several sites in one repository = several config files, each built with `-f <config>`.
- `docs_dir` and `site_dir` must lie inside the project root, else `Error: ... must be within project root`. `docs_dir = "."` is not supported.
- Two configs in one folder work (e.g. `zensical.en.toml` with `docs_dir = "en"`, `site_dir = "site/en"`, and a second one for `de`); they share `.cache/`.
- `.cache/` is created next to the config and gitignores itself.
- `zensical.toml` is native; `mkdocs.yml` is only for compatibility.
- TOML cannot read environment variables: no per-build switch (e.g. offline on/off) inside one config.
- Snippets (`pymdownx.snippets.base_path`) resolve from the current directory, not the config's folder. Fragile; do not use them to share content between sites.

```toml
[project]
site_name = "MyProduct"
docs_dir = "docs"      # default
site_dir = "site"      # default
nav = [ { "Home" = "index.md" }, { "Guide" = ["guide/index.md", "guide/setup.md"] } ]
```
