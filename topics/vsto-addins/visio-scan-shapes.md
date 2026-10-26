---
title: Scanning all shapes of a Visio document fast
description: Read when Visio code must read cells of many shapes across all pages (find shapes by type or key, build an index or tree) - release COM objects, batch reads by master, GetResults failures that block a page.
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
- PowerShell: a function named `Group` loses against the alias of `Group-Object`; `Page.DropMany` needs the out array initialized (`$ids = [int16[]]@()`, passed `[ref]`), `$null` gives `Object reference not set`.

_Unverified:_ Release build, shapes with many formulas, documents with data graphics or containers.
