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
- Hold the border with `GUARD` only, no lock cells: with `LockMoveX/Y` (or `LockWidth/Height`) on the border, Select All + move fails **as a whole** (`Shape protection, container, and/or layer properties prevent complete execution of this command.`, the other shapes do not move either); with only `GUARD(...)` on `PinX/PinY/Width/Height` the move succeeds, the others move, the border stays (COM `Selection.Move`; mouse drag not measured). `LockDelete` blocks deleting a selection that holds it the same way.
- Not selectable at all: a locked layer (but Redo of a new page loses page layers, [visio-layers](visio-layers.md) Gotchas) or `LockSelect = 1`, which acts only with document protection (`Document.Protection = 2`, visProtectShapes; then Select All skips it and `Window.Select` throws `Shape is locked against selection.`; Redo keeps it). Group members: [visio-group-sub-selection](visio-group-sub-selection.md).
- `Page.Duplicate()`: `PageAdded` comes with the shapes already on the page, so "page has a border" tells a duplicate from a new page. `Pages.Add()` + a drop in the `PageAdded` handler: one Undo removes page and border; Redo brings the border back once (skip the handler while `Application.IsUndoingOrRedoing`).
- The drop in `PageAdded` **selects the border**: the window already shows the new page (page tab `+`, Insert > New Page, also COM `Pages.Add()` in a visible Visio), so the Shape Data window shows the border, not the page. Deselect it right after the drop in each drawing window that shows the page: `window.Select(shape, visDeselect)` (window `Type` 1, `Window.Page.ID` = page ID). Redo of the new page selects it again (the handler is skipped there). Measured by COM, visible Visio, `Window.Selection` after `DoCmd(visCmdInsertPageTab)` / `DoCmd(visCmdNewForegroundPage)`.

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
  no lock cells (GUARD holds it); SelectMode = 0 (group only)
Sub-shape (footer cell, text):
  LocPinX = Width*0  LocPinY = Height*0
  PinX = Sheet.<group>!User.x0 + ...   Width = (...)*Sheet.<group>!User.s
  text field: TITLE(), PAGENAME(), PAGENUMBER()&" / "&PAGECOUNT(), FORMAT(DOCLASTSAVE(),"{{yyyy-MM-dd}}")
```

- `Sheet.<group>!` references inside the master are remapped on each drop.
- Text sizes (`Char.Size` in pt) and line weights are paper units: no scale factor.
- New pages: an add-in drops the master in `PageAdded` (foreground only, no border yet).
- Template that shows its document stencil (so a deleted border can be placed again): call `Document.OpenStencilWindow()` before `SaveAsEx`; `visio/windows.xml` then holds a `WindowType='Stencil'` window without `Document` attribute, and a new drawing from the template opens with `Document Stencil` shown (measured, invisible Visio). Docked stencils (`Documents.OpenEx(path, visOpenRO + visOpenDocked)`) are saved the same way, with their full path in `Document`. The stencil chosen in the Shapes window of a new drawing is the stencil window saved with `visWSActive` (`0x4000000`) in its `WindowState`; the window opened last gets it, so `OpenStencilWindow()` after the docked stencils makes the document stencil the chosen one. To choose another: after opening all, `.Activate()` its window (child of the drawing window: `drawingWindow.Windows`, match `.Document`), then `SaveAsEx` (measured: invisible build Visio, new drawing in a visible Visio shows the flag on that window only).
