---
title: Visio page border and footer that follow the page
description: "Read when a Visio template needs a border or title block that fits every page size, orientation and drawing scale, or footer fields (title, page x of y, date)."
---

# Visio page border and footer that follow the page

A background page does not fit pages of different sizes: put the border as a master instance on every foreground page, sized by formulas.

Measured 2026-11-05, Visio 2019 C2R x64 build 20527, COM (`Visio.InvisibleApp`).

## Facts

- A background page keeps its own size: foreground `PageWidth` 420 -> 210 mm, its background page stays 420 x 297 mm. One background page serves pages of any size, so it cannot follow them. (Page Setup / Design > Size in the UI: not measured.)
- Page field functions in a shape formula or text field:
  - `PAGECOUNT()` counts foreground pages only (2 foreground + 1 background -> 2). `PAGENUMBER()&" / "&PAGECOUNT()` shows `3 / 3` on the last of three pages.
  - `TITLE()`, `CREATOR()` (File > Info: Title, Author; COM `Document.Title`, `.Creator`), `PAGENAME()`.
  - `DOCLASTSAVE()` / `DOCCREATION()` of a never saved document: the creation time.
  - Date picture: `FORMAT(DOCLASTSAVE(),"{{yyyy-MM-dd}}")` -> `2026-11-05`. Without braces `"yyyy-mm-dd"` gave `2026-04-05` (`mm` = minutes); `"YYYY-MM-DD"` gave `2026-11-` (`DD` unknown).
- A master reading `ThePage!Prop.X` or `ThePage!User.X.Prompt` needs the rows on the **master's** page sheet, else the formula write fails with `#NAME?`. Dropped on a page without them, Visio adds the rows (also a `Prop` row; write its label yourself). See [visio-cross-shape-references](visio-cross-shape-references.md).
- Lock the border's layer on the page **after** the drop (`visLayerLock`): a drop onto a page whose layer of that name is already locked fails ([visio-layers](visio-layers.md), Locked layer). An unlocked master layer arrives unlocked. On a locked layer `Window.SelectAll` takes nothing (0 shapes).
- `Page.Duplicate()`: `PageAdded` comes with the shapes already on the page, so "page has a border" tells a duplicate from a new page. `Pages.Add()` + a drop in the `PageAdded` handler: one Undo removes page and border; Redo brings the border back once (skip the handler while `Application.IsUndoingOrRedoing`).

## Recipe

Group master; the group covers the page, so sub-shape coordinates are page coordinates. Paper sizes times the page scale factor keep margins right on scaled pages (1:5 -> 10 mm margin = 50 mm drawing units).

```text
Group:
  Width   = GUARD(ThePage!PageWidth)        PinX    = GUARD(ThePage!PageWidth*0.5)
  Height  = GUARD(ThePage!PageHeight)       PinY    = GUARD(ThePage!PageHeight*0.5)
  LocPinX = GUARD(Width*0.5)                LocPinY = GUARD(Height*0.5)
  User.s  = ThePage!DrawingScale/ThePage!PageScale
  User.x0 = 20mm*User.s   User.x1 = Width-10mm*User.s     (ISO 5457: left 20 mm, others 10 mm)
  User.y0 = 10mm*User.s   User.y1 = Height-10mm*User.s
  Geometry: lines between x0/y0 and x1/y1 (group geometry = the border)
  LockMoveX/Y, LockWidth/Height, LockRotate = 1; SelectMode = 0 (group only)
Sub-shape (footer cell, text):
  LocPinX = Width*0  LocPinY = Height*0
  PinX = Sheet.<group>!User.x0 + ...   Width = (...)*Sheet.<group>!User.s
  text field: TITLE(), PAGENAME(), PAGENUMBER()&" / "&PAGECOUNT(), FORMAT(DOCLASTSAVE(),"{{yyyy-MM-dd}}")
```

- `Sheet.<group>!` references inside the master are remapped on each drop.
- Text sizes (`Char.Size` in pt) and line weights are paper units: no scale factor.
- New pages: an add-in drops the master in `PageAdded` (foreground only, no border yet), then locks the layer.
