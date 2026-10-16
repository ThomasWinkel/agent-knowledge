---
title: Heading anchors and link checks
description: Read when linking to a heading of a Zensical site from outside (app help, other sites), choosing stable heading IDs, or checking that links and anchors exist - explicit IDs, slug rule, strict build.
---

# Heading anchors and link checks

## Explicit ID

```markdown
## Gleiche auswählen {#select-same}
```

- Needs `attr_list` in `[project.markdown_extensions]`. Works in every language; the `id` is used as given.
- Use it for any heading something outside the site links to: auto IDs change with the heading text and differ per language.

## Auto ID (slug rule)

Python-Markdown `markdown.extensions.toc.slugify(text, "-")`: Unicode NFKD -> ASCII (umlaut -> base letter, `ß` dropped), punctuation dropped, lowercase, spaces -> `-`.

| Heading | ID |
|---|---|
| `Count (Selection) & More: 2 Items!` | `count-selection-more-2-items` |
| `Gleiche auswählen senkrecht` | `gleiche-auswahlen-senkrecht` |
| `Größe ändern: Übersicht` | `groe-andern-ubersicht` |

A check script can call the same function (`uv run python -c "from markdown.extensions.toc import slugify; print(slugify('My Heading', '-'))"`).

## Link check in the build

- Internal links with anchor (`[x](guide/page.md#my-id)`) are checked: missing anchor -> `Warning: anchor does not exist` with file and line; `zensical build -s` then fails (`Aborted because --strict flag is set`).
- Only links inside the Markdown sources are checked. To check targets kept outside (e.g. an app's help table), generate a page with a link per target and build with `-s`. _Unverified_ as a full recipe.
