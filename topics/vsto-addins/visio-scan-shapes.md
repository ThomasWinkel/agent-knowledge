---
title: Scanning all shapes of a Visio document fast
description: Read when Visio code reads cells or row names of many shapes (find, index, compare, event handlers) - release COM objects, foreach vs index, batch reads, slow Shape.Parent, GetResults failures.
---

# Scanning all shapes of a Visio document fast

Reading cells shape by shape gets slower per call as the document grows, because every `Shape` and `Cell` object stays alive in .NET until the garbage collector runs. **Release them right after use** (`Marshal.ReleaseComObject`): an exact scan of 20,000 shapes took 4.6 s instead of 17-18.5 s. A batch read (`Page.GetResults`) is faster still, but a failing call can block `GetResults` on that page for the rest of the session.

Live objects also make **`Delete` and `Save` slower**, long after the scan ([measured](#live-objects-slow-delete-and-save)): release in every loop that touches many shapes, also in helpers that read one cell (`CellsU[...]` returns a new `Cell` object each time). Do not keep `Cell` objects in a list to write later: keep page ID, shape ID and cell name.

**Take collection items by index, never by `foreach`.** An object that came out of a `foreach` over a Visio collection (`Documents`, `Pages`, `Shapes`, ...), and every object and cell read through it, was 3.5-7x slower in-process ([measured](#foreach-vs-index)). The .NET wrapper (RCW) of a COM object is cached per object: once a document was first taken by `foreach`, a later `ActiveDocument` returns the same slow wrapper.

## Recipe: exact scan, one shape at a time

Looks at every top-level shape, assumes nothing about masters or row positions.

```csharp
using System.Runtime.InteropServices;

const short anywhere = (short)Visio.VisExistsFlags.visExistsAnywhere;

var pages = doc.Pages;
for (int p = 1; p <= pages.Count; p++)   // by index, not foreach
{
    var page = pages[p];
    var shapes = page.Shapes;            // top-level shapes only
    int n = shapes.Count;
    for (int i = 1; i <= n; i++)
    {
        var shape = shapes[i];
        if (shape.CellExistsU["User.Type", anywhere] != 0)
        {
            var cell = shape.CellsU["User.Type"];
            string type = cell.ResultStr[Visio.VisUnitCodes.visNoCast];
            Marshal.ReleaseComObject(cell);
            // ... read further cells the same way: get, read, release
        }
        Marshal.ReleaseComObject(shape);
    }
    Marshal.ReleaseComObject(shapes);
    Marshal.ReleaseComObject(page);
}
```

## Measured

The tables of drawings 1 and 2 were measured with `foreach` over `Pages` (shapes by index): absolute times there are probably too high ([foreach vs index](#foreach-vs-index)); the ratios between variants hold (_unverified_).

VSTO add-in (in-process), Debug build, stopwatch in a marker event handler, 3 runs each, Visio Pro 2019 C2R x64 (2026-10).

Drawing 1: 50 pages, 20,000 top-level shapes; 5,000 instances of 2 "device" group masters (3 sub-shapes, `User.Type = "Device"`, 4 key cells, 10 more `Prop` rows), 3,000 lines, 10,000 plain rectangles, 2,000 groups of 5.

| Variant (drawing 1) | 800 shapes | 20,000 shapes |
|---|---|---|
| `foreach` shape, read `ID` only | 94-99 ms | 2,459-2,532 ms |
| `foreach` + `CellExistsU("User.Type", visExistsAnywhere)` | 104-106 ms | 5,368-5,895 ms |
| + `ResultStr` of the type, 4 key cells per device, nothing released | 275-357 ms | 35,615-60,694 ms |
| `CreateSelection(visSelTypeByMaster)` per device master, 4 `CellsU` reads each | 139-145 ms | 9,540-10,004 ms |
| same selection, `GetIDs` + one `Page.GetResults` per page | 5-8 ms | 154-158 ms |

Drawing 2: 50 pages, 20,000 top-level shapes; per page 90 master instances + 40 copies of a device shape **without master** (6,500 devices; one self-identifying key cell `User.Key` = `"dev|"&...`), 20 instances grouped by the user (must not be found), the rest as drawing 1; 125 instances with a deleted inherited row (row indices shifted), 1 with the key row index out of range.

| Variant (drawing 2, all exact: same 6,500 keys) | 20,000 shapes |
|---|---|
| one by one: exists + type + key, nothing released | 17,169-18,565 ms |
| one by one, `ReleaseComObject` on every shape and cell | 4,559-5,155 ms (first run after opening: 11,488 ms) |
| batch per master (value checked by its prefix, failures one by one) + all other shapes one by one, not released | 6,225-7,257 ms |
| same, released | 3,020-3,103 ms |
| batch per master + only shapes without master one by one (assumes instances of other masters have no such cells) | 1,186-1,810 ms |

- Per-call cost grows with the number of live COM objects, not with the document: releasing them made the one-by-one scan 4x faster. The same probably explains growing write costs: [visio-performance-switches](visio-performance-switches.md#measured-the-switches-gave-nothing-the-batch-call-did) (_unverified_).
- `CreateSelection(visSelTypeByMaster, mode, master)` on a page found only top-level instances, never instances inside a user's group, in every mode (`visSelModeSkipSub`, `SkipSuper`: 70 of 70 top-level; `OnlySub`, `OnlySuper`: 0). Shapes without master are not found by it.

## Finding rows by name prefix (e.g. all `User.Link_*` rows)

No call returns all row names of a section: read them one by one. Cost per call stays about 10 us when every object is released, so this is affordable even in a large document; reading the found cells costs as much again.

```csharp
const short user = (short)Visio.VisSectionIndices.visSectionUser;
short rows = shape.SectionExists[user, 0] != 0 ? shape.RowCount[user] : (short)0;
for (short r = 0; r < rows; r++)
{
    var cell = shape.CellsSRC[user, r, 0];
    string name = cell.RowNameU;          // "" for an unnamed row
    Marshal.ReleaseComObject(cell);
    if (name.StartsWith("Link_", StringComparison.Ordinal)) { /* ... */ }
}
```

Measured as above (in-process, Debug, Visio 2019, 2026-10): drawing 2 with 21 User rows on each of the 6,500 devices (13 named `Link_*`), 3 runs:

| Variant | 20,000 shapes |
|---|---|
| `SectionExists` + `RowCount` of the User section only | 302-311 ms (first run after opening: 6,660 ms) |
| all row names, `CellsSRC(...).RowNameU` (136,500 rows) | 2,355-3,809 ms |
| all row names, `Section[s][r].NameU` | 2,664-3,849 ms |
| names cached per master, reused when the instance has the same `RowCount` (not exact) | 1,582-1,959 ms |
| one list cell per shape naming the rows instead (exists + `ResultStr`) | 359-362 ms |
| row names + 2 cells of each of the 84,500 found rows (`Prompt` `ResultStr`, `FormulaU`) | 5,884-7,338 ms |

- The first touch of all shapes after opening a large document is slow whatever is read (here 6.7 s, 11.5 s in the scan above).
- A list cell is fast but must be kept in step with the rows by hand; a per-master cache misses rows an instance renamed or replaced with the same count.

## Reading many cells of every shape (e.g. a full compare)

Visio Pro C2R x64 build 20625, Debug add-in, in-process, 2026-11. Drawing: 20 pages x 200 instances of 8 group masters (3-5 sub-shapes each) = 27,521 shapes incl. sub-shapes; all `Prop` + `User` rows of each (303,009 rows), 478,009 cells read (`Prop` Value, Label, Invisible; `User` Value); every object released. ms, run 1 / runs 2-3:

| Variant | ms |
|---|---|
| first pass after opening (structure) | 11,099 |
| structure: `SectionExists`, `RowCount`, `CellsSRC(...).RowNameU` per row | 2,848 / 5,066-5,123 |
| + `Shape.MasterShape` per shape (master key) | 3,060 / 5,619-5,735 |
| + per cell `IsInherited` only | +3,569 / +6,350-6,473 |
| + per cell `IsInherited`, `FormulaU`, `ResultStr` | +10,732 / +19,324-19,734 |
| + `Shape.GetFormulasU` + `Shape.GetResults` per shape | +10,463 / +10,462-10,558 |
| + `Page.GetFormulasU` + `Page.GetResults` per page (one stream of ~24,000 cells) | +9,759 / +9,971-10,052 |

- With many cells per shape the batch calls cost about as much per cell as one-by-one reads in a fresh pass (~20 us per cell for formula + result); they only stay stable over repeated passes. A page-wide stream is no faster than one per shape. Same formulas and results in all variants.
- Why the one-by-one variants got slower in passes 2-3: unknown.
- **"Local or inherited" by comparing the formula with the master shape's is wrong for sub-shapes**: inherited formulas name the instance's own sheet IDs (`Sheet.22!FlipX` in the instance, `Sheet.5!FlipX` in the master): 23,000 of 476,009 inherited cells differed this way. Map `Sheet.<id>!` from master to instance IDs first, or use `Cell.IsInherited`. `IsInherited` was 0 on exactly the 2,000 cells set locally.

## Cost per cell call

What makes a read slow is the **formula text**, not the number of calls: `FormulaU` (also in `GetFormulasU`) costs ~5x `ResultStr`. Read formulas only where needed; for "is it empty?" see [visio-cell-values](visio-cell-values.md#empty-cells-types-and-errors-on-read) (result first).

Add-in in-process, Debug, Visio Pro 2019 C2R x64, 2026-11; 20 pages x 200 group instances (4,001 top-level shapes), all 135,009 `User` rows (Value column), one pass per kind after a warm-up pass, every object released:

| Call per cell | ms (135,009 cells) | ~us per cell |
|---|---|---|
| `CellsSRC` + release (base of the lines below) | 1,242-1,648 | 9-12 |
| + `RowNameU` | +490-1,510 | 4-11 |
| + `Section[s][r].NameU` instead (`Row` object) | 3,312 total | 25 |
| + `FormulaU` | +5,980-6,180 | 44-46 |
| + `ResultStr(visNoCast)` | +1,080-1,450 | 8-11 |
| + `ResultIU` | +280-870 | 2-6 |
| + `Error` / `IsInherited` | +130-640 | 1-5 |
| `CellExistsU(name)` / `CellsU(name)` | 1,283-1,301 / 817-835 | 6-10 |
| `Shape.GetResults` per shape (strings, no cast), all rows | 688-706 | 5 |
| `Shape.GetFormulasU` per shape, all rows | 5,884-6,233 | 44-46 |

- A batch of results (`GetResults`, `visGetStrings`, units `visNoCast`) reads the same text as `ResultStr(visNoCast)` per cell, **except errors**: `1/0` and `NA()` read `0,0000` in a number cell, `""` in a string cell (no error text, no flag). To get exactly the one-by-one text: trust a batch result only if it has a letter, is not `FALSE` and does not start with a zero number (`^\s*[-+]?0+([.,]0*)?(\D|$)`); read every other cell again one by one (`ResultStr`, and `FormulaU` if it could be empty). Checked alike on 270,018 `User` Value and Prompt cells of the drawing above and on 1,646 test cells.
- Batch only the cells you need, after the row names: reading Value + Prompt of all rows with `GetFormulasU` + `GetResults` made a scan slower (10.7 s) than one-by-one reads of the needed cells (6.3 s); a `GetResults` batch of the needed cells only, after the names one by one: 4.1 s (2.7 s of it row names).

## Top-level check: not by `Parent`

`Shape.Parent` is slow on a big drawing: **~0.9 ms per call** (8,002 shapes: 7.1-7.5 s), while `ContainingShape` costs ~0.01 ms (72-83 ms), same drawing and setup as above. In event handlers that ask "is it top-level?" per event, `Parent` was most of the handler time (18,500 `CellChanged` events: 11.8 s -> 3.1 s with the check below). Same answer as `shape.Parent is Visio.Page` for page shapes, sub-shapes, master shapes of all levels, page, master and document sheets:

```csharp
static bool IsTopLevel(Visio.Shape shape)
{
    // Master shape or sheet, document sheet. A top-level master shape is contained by the master's sheet,
    // which is of page type too.
    if (shape.ContainingPageID == -1)
        return false;
    // A page's sheet (Parent: the page); its ContainingShape is null.
    if (shape.Type == (short)Visio.VisShapeTypes.visTypePage)
        return true;
    var container = shape.ContainingShape;
    var top = container.Type == (short)Visio.VisShapeTypes.visTypePage;
    Marshal.ReleaseComObject(container);
    return top;
}
```

- Why `Parent` is slow: unknown (not measured: whether it grows with the shape count of the page).

## foreach vs index

Same drawing and setup as the table above (27,521 shapes, 303,009 `Prop` + `User` rows), every object released, each variant in a fresh Visio unless noted, 2026-11. ms:

| Variant | ms |
|---|---|
| `RowNameU` of all rows; document by `ActiveDocument`, pages + shapes by index | 5,113-5,489 (first pass after opening 9,617-17,567) |
| same, pages by `foreach` (same session, right after the line above) | 39,150 |
| same as line 1, but the document first taken by `foreach` over `Documents` (then by `ActiveDocument`, same wrapper) | 37,297-39,373 |
| full read (row names, `GetFormulasU` + `GetResults` per shape, text, master), `foreach` over masters, pages, shapes, connects | 75,825-76,170 |
| same, all by index | 21,491-22,897 |
| same by index + `IsInherited` per cell (390,681 cells), no `Master` for sub-shapes of instances | 24,248-24,859 |

- Cost stays with the object: a `foreach` over `Documents` after the document was already taken by `ActiveDocument` changed nothing (5.3 s before and after); `GC.Collect` changed nothing.
- Cause not known; `foreach` goes through `IEnumVARIANT`.
- **The whole add-in shares the cached wrapper.** A `VisioIsIdle` handler that took `Application.Documents` by `foreach` and kept each document (to attach event handlers) made a later scan of that document, taken by index in another command, 3.7x slower: 57-58 s instead of 15-16 s (20 pages, 4,001 shapes, 218,181 cells). By index in the idle handler: 15-16 s on every run. Which wrapper a document gets depends on which call creates it first while no live wrapper exists: a document already taken by `ActiveWindow.Document` (another idle handler, run first) kept its fast wrapper through the `foreach`; once its wrapper was collected, the `foreach` made a new, slow one. So a scan can be fast or slow depending on idle timing. Rule: take items by index everywhere, above all where the object is kept.
- The same idle handler's own scan got faster too: 28.5 s -> 15.2 s (first pass after opening) and 32.2 s -> 4.2 s (later pass).

- Guard against regressions: a unit test that compiles the add-in sources with Roslyn (`CSharpCompilation` + the Visio interop from the GAC as reference; other compile errors ignored) and fails on every `ForEachStatement` whose expression type is in `Microsoft.Office.Interop.Visio`, every LINQ extension call (`this` parameter `IEnumerable`) on such a type, and every implicit conversion of one to `IEnumerable`. A text search misses `var` and LINQ. Fail also on a `foreach` whose type is unresolved, so a missing reference cannot hide one.

## Live objects slow Delete and Save

VSTO add-in, Debug, Visio Pro 2019 C2R x64, 2026-11. Drawing: 20 pages x 200 group instances (4,001 top-level shapes). Columns: scan = one pass over all shapes and sub-shapes reading their layers (`Shape.Layer[i].Name`); delete step = one more such pass, then `Shape.Delete` of 20 shapes and `Page.Delete` of one page with 200 shapes; save = `Document.Save` after the whole sequence (more passes and layer changes before). Fresh Visio each, ms:

| Variant | scan | delete step | save |
|---|---|---|---|
| `foreach` over shapes and sub-shapes, nothing released | 13,115 | 28,346 | ~9,000 |
| by index, nothing released | 8,445 | 58,814 | 25,000-28,000 |
| by index, nothing released, `GC.Collect` + `WaitForPendingFinalizers` before the delete | 8,644 | 19,085 | 5,700 |
| by index, every `Shape`, `Shapes`, `Layer`, `Cell` released | 904 | 1,319 | 850 |

- Switching to index loops alone made deleting and saving slower here; releasing made everything fast. Why the index-taken objects cost more than enumerated ones while alive: unknown.
- A `Page.Delete(1)` with many live objects of that page's shapes is the slowest case seen (_unverified_ which part of the step dominates).
- **Objects kept on purpose cost the same.** A list of 4,000 `Cell` objects kept to write later (one per shape) made `CellExistsU` on every other shape 3.6x slower: one pass over 4,001 shapes 98-104 ms before, 374 ms while kept, 102 ms right after releasing them (same setup, one Visio session). A loop that keeps one cell per shape slows itself as it grows. Keep where the cell is instead (page ID, shape ID, cell name) and get it again for the write:

```csharp
// Read: keep the text and the place, not the Cell object.
var cell = shape.CellsU[cellName];
var item = (PageId: pageId, ShapeId: shape.ID, Text: cell.ResultStr[Visio.VisUnitCodes.visNoCast]);
Marshal.ReleaseComObject(cell);
// Write later: get the cell again.
var page = document.Pages.ItemFromID[item.PageId];   // release Pages, Shapes, page, shape, cell after use
var target = page.Shapes.ItemFromID[item.ShapeId];
```

## Batch read by master: how and why it is risky

Take the SRC of the key cells from the master shape, select the instances per master and page, read all with one `Page.GetResults` (`visGetStrings`). Check each value (e.g. a fixed prefix), read failures one by one.

```csharp
var c = master.Shapes[1].CellsU["User.Key"];
short[] src = { c.Section, c.Row, c.Column };
page.CreateSelection(Visio.VisSelectionTypes.visSelTypeByMaster, Visio.VisSelectMode.visSelModeSkipSub, master)
    .GetIDs(out Array ids);
var stream = new List<short>();                    // 4 shorts per cell: shape ID, section, row, cell
foreach (int id in ids) { stream.Add((short)id); stream.AddRange(src); }
Array sids = stream.ToArray();
Array units = new object[] { (short)Visio.VisUnitCodes.visNoCast };   // one entry: used for all
page.GetResults(ref sids, (short)Visio.VisGetSetArgs.visGetStrings, ref units, out Array results);
```

- **An instance's row index can differ from its master's.** `DeleteRow` of an inherited row moves the following rows down (row 1 -> 0); the batch read then returns another cell's value without error. An instance whose section turned local ([visio-cell-values](visio-cell-values.md#master-instances)) also misses rows the master gets later.
- **One missing cell fails the whole call** (missing row, missing section, missing shape ID): COM exception `An exception occurred.` or `Unexpected end of file.`, no partial results.
- **After such a failure, `GetResults` on that page can keep failing** with `Out of memory. Try closing Visio windows or other applications you aren't using.`, in-process too: seen after a one-entry stream with a missing row (also one past the last row). Other pages, `Shape.GetResults` and `CellsSRC` still worked; `ReleaseComObject` of the page and a new `Page` object did not help. From PowerShell, a new process worked again. After a failing multi-entry stream (one missing row among 130 cells) later calls on that page worked (with garbage collection in between). Rule not found: send only cells that certainly exist.
- `GetIDs` returns `int`, the SRC stream takes `short`: IDs above 32,767 do not fit (_unverified_ what Visio does then).

## Gotchas

- `AddRow(section, 0, ...)` on an instance with inherited rows did not insert in front: the row got the next free index (13 of 13).
- PowerShell: `Page.DropMany` arrays and helper names that clash with aliases: [visio-powershell-com](visio-powershell-com.md).

_Unverified:_ Release build, shapes with many formulas, documents with data graphics or containers.
