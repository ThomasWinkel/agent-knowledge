---
title: Long strings in Visio ShapeSheet cells
description: Read when storing long text (JSON, encoded data) in Visio ShapeSheet cells from code - no 64k limit via the API, what survives save and reopen, sources claiming 64k.
---

# Long strings in Visio ShapeSheet cells

Measured with Visio 2019 C2R x64 (16.0 build 20430), COM from PowerShell. Written as a quoted literal:

```powershell
$cell.FormulaU = '"' + $text + '"'      # inner " must be doubled
$cell.ResultStr(0).Length                 # read back
```

- **In memory:** every tested cell took 32,768,000 characters and returned all of them; the search stopped there. Cells: `User.x`, `User.x.Prompt`, `Prop.x` (value), `Prop.x.Label`, `Prop.x.Prompt`, `Prop.x.Format`, on the DocumentSheet, a PageSheet and a shape. `GUARD("...")` around the literal: same (tested up to 10,000,000).
- **Save and reopen:** 1,000,000 characters in `User.x`, `User.x.Prompt`, `Prop.x` (shape) and `User.x` (DocumentSheet) came back complete, both `.vsdx` and `.vsd`. Save and open each took under 1 s.
- No difference at 65,535 / 65,536 / 65,537: there is no 64k limit for cells set by code.
- **Row names:** `AddNamedRow` took User row names up to 200 characters (the search stopped there); `CellsU("User.<name>")` finds them.
- **Real data in a master:** 8 JSON cells (85 KB in total) in one master with 100 instances: `.vsdx` 22,709 bytes vs 15,397 without (instances inherit, the text is stored once); writing all 8 cells 75 ms; reading one 27 KB cell through each of 100 instances 331 ms (`FormulaU`) out of process -> read it once per master if the instances inherit it.

- **Undo keeps every old value:** a document `User` cell with 317,029 characters (JSON, 6,500 entries) rewritten by code: write 5-16 ms, read 1-2 ms, Undo 1 ms (out of process); Visio's private memory grew about 2 MB per write up to +35 MB after 20 writes (default 20 Undo levels), not more after 60. Data that must not be undone: [visio-undo-scopes](visio-undo-scopes.md#data-outside-undo).

## Sources claiming 64k

- Visio MVP blog (bVisual, Shape Data): "the value can be up to 64k characters", "64k characters can be entered or viewed". Probably a limit of the UI (Shape Data window, ShapeSheet window); _unverified_.
- Old Microsoft docs: `Shape.Text` and `Data1/2/3` up to 64 KB. Via the API `Shape.Text` took 1,000,000 characters in memory; saving that file did not finish (not analyzed).

## Gotchas

- Test strings like `'x' * n` compress almost completely in `.vsdx` (ZIP): file sizes say nothing about real data.
- _Unverified:_ the UI limit, and how long strings affect recalculation or ShapeSheet window performance.
