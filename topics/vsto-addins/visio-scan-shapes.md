---
title: Scanning all shapes of a Visio document fast
description: Read when Visio code must read cells of many shapes across all pages (find shapes by type or key, build an index or tree) - cost per COM call, selection by master + Page.GetResults, row index gotcha.
---

# Scanning all shapes of a Visio document fast

Reading cells shape by shape (`foreach` over `Page.Shapes`, `CellExistsU`, `CellsU(..).ResultStr`) costs a COM call each, also in-process, and gets slower per call as the document grows. Select the shapes by master and read all key cells of a page with one `Page.GetResults` call: about 250x faster on 20,000 shapes.

## Measured

VSTO add-in (in-process), Debug build, stopwatch in a marker event handler, 3 runs each. Per page: 100 instances of 2 "device" group masters (3 sub-shapes, `User.Type = "Device"` + 4 key cells + 10 more `Prop` rows), 60 lines with `User.Type = "Line"`, 200 plain rectangles, 40 groups of 5 (no `User` cells). Top-level shapes only.

| Variant | 2 pages, 800 shapes | 50 pages, 20,000 shapes |
|---|---|---|
| B `foreach` shape, read `ID` only | 94-99 ms | 2,459-2,532 ms |
| C `foreach` + `CellExistsU("User.Type", visExistsAnywhere)` | 104-106 ms | 5,368-5,895 ms |
| A C + `ResultStr` of the type, 4 key cells of each device | 275-357 ms | 35,615-60,694 ms |
| D `CreateSelection(visSelTypeByMaster)` per device master, 4 `CellsU` reads each | 139-145 ms | 9,540-10,004 ms |
| E D, but `GetIDs` + one `Page.GetResults` per page | 5-8 ms | 154-158 ms |

- Same 5,000 keys in A, D and E.
- B grows linearly; every variant reading cells one by one grows faster (A: 0.34 ms per shape at 800, 1.8-3.0 ms at 20,000). Cause not found; same effect for writes: [visio-performance-switches](visio-performance-switches.md#measured-the-switches-gave-nothing-the-batch-call-did).

## Recipe

Find the masters once per scan (all document masters, also `Name.<n>` copies), then per page one selection per master and one batch read.

```csharp
string[] keyCells = { "Prop.Unit", "Prop.Name" };
var noCast = Visio.VisUnitCodes.visNoCast;

// Masters whose shape has the type; SRC of the key cells taken from the master shape.
var masters = new List<(Visio.Master Master, short[] Src)>();
foreach (Visio.Master m in doc.Masters)
{
    var ms = m.Shapes[1];
    if (ms.CellExistsU["User.Type", (short)Visio.VisExistsFlags.visExistsAnywhere] == 0) continue;
    if (ms.CellsU["User.Type"].ResultStr[noCast] != "Device") continue;
    var src = new List<short>();
    foreach (var name in keyCells)
    {
        var c = ms.CellsU[name];
        src.Add(c.Section); src.Add(c.Row); src.Add(c.Column);
    }
    masters.Add((m, src.ToArray()));
}

foreach (Visio.Page page in doc.Pages)
{
    var stream = new List<short>();   // 4 shorts per cell: shape ID, section, row, cell
    foreach (var (master, src) in masters)
    {
        var sel = page.CreateSelection(Visio.VisSelectionTypes.visSelTypeByMaster,
                                       Visio.VisSelectMode.visSelModeSkipSub, master);
        sel.GetIDs(out Array ids);
        foreach (int id in ids)
            for (int i = 0; i < src.Length; i += 3)
            {
                stream.Add((short)id); stream.Add(src[i]); stream.Add(src[i + 1]); stream.Add(src[i + 2]);
            }
    }
    if (stream.Count == 0) continue;
    Array sids = stream.ToArray();
    Array units = new object[] { (short)noCast };   // one entry: used for all cells
    page.GetResults(ref sids, (short)Visio.VisGetSetArgs.visGetStrings, ref units, out Array results);
    // results: strings in stream order, keyCells.Length per shape
}
```

## Gotchas

- **Row index of an instance can differ from its master: the batch read then returns another cell's value without error.** Measured: `DeleteRow` of the inherited `User` row 0 in an instance moved the next rows down (row 1 -> 0). An instance whose section turned local (row added or deleted, [visio-cell-values](visio-cell-values.md#master-instances)) also misses rows the master gets later. Guard: put a self-identifying value into the batch (e.g. a key cell whose result starts with a fixed prefix) and read shapes that fail the check one by one; or fall back to `CellsU` where exactness matters more than speed.
- `AddRow(section, 0, ...)` on an instance with inherited rows did not insert in front: the row got the next free index (13 of 13).
- Selection by master finds only instances of the listed masters: shapes without a master, or of a master the code did not list, are missed.
- `ids` from `GetIDs` are `int`; the SRC stream takes `short`: IDs above 32,767 do not fit. _Unverified_ what Visio does then.
- PowerShell: a function named `Group` loses against the alias of `Group-Object`; `Page.DropMany` needs the out array initialized (`$ids = [int16[]]@()`, passed `[ref]`), `$null` gives `Object reference not set`.

Measured with Visio Pro 2019 C2R x64 (2026-10). _Unverified:_ Release build, shapes with many formulas, documents with data graphics or containers.
