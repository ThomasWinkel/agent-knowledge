---
title: Scanning all shapes of a Visio document fast
description: Read when Visio code reads cells or row names of many shapes across all pages (find, index, compare) - release COM objects, batch reads, IsInherited, GetResults failures.
---

# Scanning all shapes of a Visio document fast

Reading cells shape by shape gets slower per call as the document grows, because every `Shape` and `Cell` object stays alive in .NET until the garbage collector runs. **Release them right after use** (`Marshal.ReleaseComObject`): an exact scan of 20,000 shapes took 4.6 s instead of 17-18.5 s. A batch read (`Page.GetResults`) is faster still, but a failing call can block `GetResults` on that page for the rest of the session.

## Recipe: exact scan, one shape at a time

Looks at every top-level shape, assumes nothing about masters or row positions.

```csharp
using System.Runtime.InteropServices;

const short anywhere = (short)Visio.VisExistsFlags.visExistsAnywhere;

foreach (Visio.Page page in doc.Pages)
{
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
}
```

## Measured

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
