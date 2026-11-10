---
title: Visio hyperlinks to pages and shapes - Visio and PDF export
description: Read when Visio shapes link to another page or shape (Hyperlink SubAddress) and the links must also work in PDF export, or survive page renames.
---

# Visio hyperlinks to pages and shapes

A shape's `Hyperlink.<row>.SubAddress` names a page (`Page-2`) or a shape on a page (`Page-2/Sheet.5`). Visio follows both; **PDF export keeps only page links.**

Measured 2026-11, Visio 2019 C2R x64, COM from PowerShell (`Hyperlink.Follow()`, `Document.ExportAsFixedFormat` to PDF), PDF link annotations read with `pypdf`.

## Visio vs PDF

| `SubAddress` | Visio (`Follow`) | PDF export |
|---|---|---|
| `<page>` (local `Name`) | goes to the page, whole page in view | `/Link` to that page, `/XYZ` left 0, top of page, zoom 0 (viewer keeps its zoom) |
| `<page>` (`NameU` that differs from `Name`) | goes to the page | **no link** |
| `<page>/<shape>` | goes to the page, view centered on the shape (zoom whole page, shape not selected) | **no link** (also for a shape on the same page) |
| `<page>/<shape>/200` | as `<page>/<shape>`; zoom ignored | no link |
| own page | stays | link to the own page |
| page name with `/` (`E/F`) | goes to the page | no link |
| unknown page (e.g. renamed) | nothing, no error | no link |
| `Invisible` = 1 | `Follow()` by code still goes | **no link** |

- Only the first hyperlink row of a shape reaches the PDF: a page link in row 2 after a shape link in row 1 was missing; an invisible page link in row 1 + a shape link in row 2 gave none.
- Link area in the PDF = the shape's own box: for a group its `Width` x `Height`; a sub-shape outside it (e.g. a text block beside a 5 x 5 mm symbol) gets no link area.
- Formulas work in both cells: `Invisible = NOT(TheDoc!User.showLinks)` switches the PDF link on and off with a document cell; `SubAddress = User.target` with `User.target` = the cross-page formula below gave the same links.
- Every link appeared twice in the PDF (two identical `/Link` annotations, same rect), with and without structure tags / document properties. Harmless in viewers.
- So for PDF: link to the page, by its local name. To reach the shape inside Visio, use code (e.g. `Window.CenterViewOnShape`).

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
- _Unverified:_ Ctrl+click / context menu in the UI, other PDF viewers' handling of zoom 0, Visio versions other than 2019.
