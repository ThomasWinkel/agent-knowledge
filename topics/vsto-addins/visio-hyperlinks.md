---
title: Visio hyperlinks to pages and shapes - Visio and PDF export
description: Read when Visio shapes link to another page or shape (Hyperlink SubAddress) and the links must also work in PDF export, or survive page renames.
---

# Visio hyperlinks to pages and shapes

A shape's `Hyperlink.<row>.SubAddress` names a page (`Page-2`) or a shape on a page (`Page-2/Sheet.5`). Visio follows both. **PDF export resolves a shape only in the form `<page>/Sheet.<ID>`**; by its name (`Page-2/MyShape`) the link is dropped. For a shape link the PDF target is the shape's top-left corner, not just the page top.

Measured 2026-11, Visio 2019 C2R x64 (build 20527), COM from PowerShell (`Hyperlink.Follow()`, `Document.ExportAsFixedFormat` to PDF), PDF link annotations read with `pypdf`.

## Visio vs PDF

| `SubAddress` | Visio (`Follow`) | PDF export |
|---|---|---|
| `<page>` (local `Name`) | goes to the page, whole page in view | `/Link` to that page, `/XYZ` left 0, top of page, zoom 0 (viewer keeps its zoom) |
| `<page>` (`NameU` that differs from `Name`) | goes to the page | **no link** |
| `<page>/Sheet.<ID>` | goes to the page, view centered on the shape (zoom whole page, shape not selected) | `/Link`, `/XYZ` left/top = the shape's top-left corner (incl. line width), zoom 0: viewer scrolls the shape to the top-left of the window, keeps its zoom. Works for any shape name (custom name, master instance) |
| `<page>/<shape name>` (custom `Name` or `NameU`, master instance `MyMaster` / `MyMaster.7`) | as `Sheet.<ID>` | **no link** (also for a shape on the same page) |
| `<page>/<shape>/200` | zoom ignored | name form: no link; `Sheet.<ID>` form _unverified_ |
| `ExtraInfo` `zoom=400` | _unverified_ | zoom not passed (zoom 0) |
| own page | stays | link to the own page |
| page name with `/` (`E/F`) | goes to the page | no link |
| unknown page (e.g. renamed) | nothing, no error | no link |
| `Invisible` = 1 | `Follow()` by code still goes | **no link** |

- Only the first hyperlink row of a shape reaches the PDF: a page link in row 2 after a (name-form) shape link in row 1 was missing; an invisible page link in row 1 + a shape link in row 2 gave none.
- Link area in the PDF = the shape's own box: for a group its `Width` x `Height`; a sub-shape outside it (e.g. a text block beside a 5 x 5 mm symbol) gets no link area.
- Formulas work in both cells: `Invisible = NOT(TheDoc!User.showLinks)` switches the PDF link on and off with a document cell; `SubAddress = User.target` with `User.target` = the cross-page formula below gave the same links.
- Every link appeared twice in the PDF (two identical `/Link` annotations, same rect), with and without structure tags / document properties. Harmless in viewers.
- So for PDF: page by its local name, shape by `Sheet.<ID>`. Let the target build its own address, `User.myTarget = PAGENAME()&"/Sheet."&ID()`, and link to it: `SubAddress = Pages[TargetPage]!Sheet.5!User.myTarget`. `ID()` gives `Sheet.12` (no decimals), a pasted copy gets its own ID, a page rename follows (measured). Visio exports no zoom; only post-processing the PDF could set one (e.g. `/FitR`, _unverified_).

## Survive a page rename

A constant `SubAddress` breaks silently on rename (Visio and PDF). Keep the page name live with a formula:

```
' in the target shape:   User.myPage = PAGENAME()
' in the linking shape:  Hyperlink.Row_1.SubAddress = Pages[TargetPage]!Sheet.5!User.myPage
```

Visio rewrites the `Pages[..]` reference on rename ([visio-cross-shape-references](visio-cross-shape-references.md) for what else breaks it). `PAGENAME()`, `PAGENAME(0)` and `PAGENAME(1)` all returned the local `Name` (also with `Name` `Bloc`, `NameU` `Buni`) - the one PDF export needs.

## Gotchas

- `User.page` cannot be created: `The name 'page' is a reserved system name.`
- A reference `Pages[B]!T!User.x` to a shape named `T` failed (`Bad ShapeSheet name.`); `Pages[B]!Sheet.<ID>!User.x` worked.
- _Unverified:_ Ctrl+click / context menu in the UI, other PDF viewers' handling of zoom 0, Visio versions other than 2019, `Sheet.<ID>` of a sub-shape in a group, target page with a different drawing scale.
