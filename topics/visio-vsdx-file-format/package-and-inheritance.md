---
title: VSDX package parts and master inheritance
description: Read when code finds pages, masters and top-level shapes in a .vsdx, or merges a shape's ShapeSheet rows with its master's rows as Visio does.
---

# VSDX package parts and master inheritance

Measured 2026-11 with Visio Pro 2019 C2R x64 files: XML read by `System.IO.Compression` + `XDocument` (net48), compared row by row with COM (`Shape.CellsSRC`) on the same file: 135,031 User rows, 0 differences in keys and row index.

## Parts

Namespaces: main `http://schemas.microsoft.com/office/visio/2012/main`, `r:` = `http://schemas.openxmlformats.org/officeDocument/2006/relationships`.

| Part | Holds |
|---|---|
| `visio/document.xml` | `DocumentSheet` (the document's ShapeSheet), `StyleSheets` |
| `visio/_rels/document.xml.rels` | targets `pages/pages.xml`, `masters/masters.xml`, `windows.xml` (relationship types `.../visio/2010/relationships/pages` etc.) |
| `visio/pages/pages.xml` | `<Page ID NameU Name>` with `PageSheet` and `<Rel r:id>` |
| `visio/pages/_rels/pages.xml.rels` | `r:id` -> `pageN.xml` (the number is not the page ID) |
| `visio/pages/pageN.xml` | `PageContents/Shapes/Shape`, sub-shapes nested in `Shape/Shapes` |
| `visio/masters/masters.xml` + `_rels/masters.xml.rels` | `<Master ID NameU>` -> `masterN.xml` |

- Find parts through the rels, not by file name; resolve `Target` relative to the folder of the part.
- Shape attributes: `ID` (= COM `Shape.ID`), `NameU`, `Name`, `Type`, `Master` (master ID, on an instance's top shape), `MasterShape` (shape ID inside the master, on sub-shapes).
- Page `ID` = COM `Page.ID`.

## Rows of an instance

```csharp
// inherited: rows of the master's only top-level shape (MasterContents/Shapes/Shape), in order.
// local: rows of the instance's Section. Row and cell names compare ignoring case.
List<Row> Merge(List<Row> inherited, List<Row> local)
{
    var byName = local.ToDictionary(r => r.Name, StringComparer.OrdinalIgnoreCase);
    var result = new List<Row>();
    foreach (var row in inherited)
    {
        if (!byName.TryGetValue(row.Name, out var own)) { result.Add(row); continue; }
        if (own.Deleted) continue;                       // <Row N="x" Del="1"/>
        result.Add(row.WithCells(own.Cells));            // local cells over inherited ones, by N
    }
    var names = new HashSet<string>(inherited.Select(r => r.Name), StringComparer.OrdinalIgnoreCase);
    result.AddRange(local.Where(r => !r.Deleted && !names.Contains(r.Name)));   // new local rows after
    return result;
}
```

- This order gave the same row index as COM for every row (named sections such as `User`).
- A local row lists only the cells it overrides; the rest come from the master row.
- A shape without `Master` has only its local rows (minus `Del` rows).
- Unverified: masters with several top-level shapes (instance = group, which shape it inherits from), rows inherited from style sheets (`LineStyle`/`FillStyle`/`TextStyle`), unnamed sections (Geometry, Connection: rows by `IX`).
