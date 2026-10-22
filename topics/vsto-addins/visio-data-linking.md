---
title: Linking Visio shapes to data rows and refreshing
description: Read when Visio code links shapes to data recordset rows or refreshes linked data - primary key, row matching, refresh conflicts, shape data matching, edited values, column changes.
---

# Linking Visio shapes to data rows and refreshing

Measured with Visio Pro 2019 C2R x64, German region settings, COM from PowerShell, recordsets from `.xlsx` via ACE (connection: [visio-data-recordsets](visio-data-recordsets.md)). Calls with `out` arrays (`GetLinkedDataRecordsetIDs`, `GetShapesLinkedToData*`, `GetPrimaryKey`) fail in PowerShell late binding; call them from C# (`Add-Type` with the interop assembly).

## Primary key: set it

`DataRecordsets.Add` gives no primary key: rows match **by position** on refresh (saved as `RowOrder='1'`). Sort or insert a row in the source and every linked shape silently shows another row's data. Set a key column right after `Add`:

```csharp
recordset.SetPrimaryKey(Visio.VisPrimaryKeySettings.visKeySingle, new[] { "ID" });  // enum: RowOrder 1, Single 2, Composite 3
```

Saved as `<PrimaryKey ColumnNameID='ID'><RowKeyValue RowID='1' Value='a1'/>...`. With a key, reordered source rows keep their Visio row IDs and links.

Key matching on `Refresh()`:

| Source change | Result |
|---|---|
| Rows reordered | links kept |
| Key changes case only (`a1` -> `A1`) | same row, link kept: matching ignores case |
| Two keys differ only in case (`abc`, `ABC`) | both shapes lose their link, already on an unchanged refresh |
| Key duplicated (second row `a1`) | shape linked to `a1` loses its link; not restored when the duplicate is removed again |
| One row with empty key | treated as key `""`, link kept |
| Two rows with empty key | the linked shape loses its link |
| Row removed | shape loses its link, keeps its values |
| Key `0007`, `1E3`, `-5`, `12.3` in text cells | kept as text, matched |

- Visio row IDs are never reused (a removed or failed row burns its ID).
- A shape that lost its link stays in `DataRecordset.GetAllRefreshConflicts()`; `GetMatchingRowsForRefreshConflict(shape)` gives the competing rows (duplicate key) or nothing (row removed). `Refresh()` by code shows no conflict UI. Relink: `shape.LinkToData(recordsetId, rowId, false)`.

## Linking

- `Shape.LinkToData(recordsetId, rowId, applyDataGraphic)` and `Page.LinkShapesToDataRows(recordsetId, rowIds, shapeIds, applyDataGraphic)` link directly, no matching field needed.
- `Selection.AutomaticLink(recordsetId, columnNames, fieldTypes, fieldNames, flags, out shapeIds)` matches by value, e.g. column `Name` = shape data label `Name` (`visAutoLinkCustPropsLabel` = 2, flags `visAutoLinkSelectedShapesOnly` 1 + `visAutoLinkNoApplyDataGraphic` 4). Returns the number of linked shapes.
- A column fills the existing shape data row whose **label** equals the column name. The row name does not count: column `Sig3` and a row `Prop.Sig3` with label `Signal 3` give a new row `Prop._VisDM_Sig3`. Columns without a matching label become new rows `Prop._VisDM_<column>`.
- Linking writes values over formulas (`Prop.F = Prop.A&"!"` becomes `"99"`).
- Into typed rows: fixed list (type 1) gets `INDEX(n,Prop.X.Format)`; a value not in the list gives `INDEX(0,...)` = first entry, without warning. Number (type 2): `12` from `5 mm` -> unit lost; text `7,5` is parsed with the region (`7.5`). Date (type 5): text `2026-10-22` and `22.10.2026` -> `DATETIME(46317)`.
- A shape data row with `Invisible = TRUE` and a matching label is filled like any other and stays hidden; no dialog appeared, neither by code (`AlertResponse` 0 and 1) nor when the user dragged the row from the External Data window onto the shape.
- One shape linked to two recordsets with the same column name (e.g. both have `ID`) shares one shape data row: the second link overwrites the first one's value.
- Find multi-links: `Shape.GetLinkedDataRecordsetIDs(out ids)` (recordsets of a shape), `Page.GetShapesLinkedToDataRow(recordsetId, rowId, out shapeIds)` (shapes of a row), `Page.GetShapesLinkedToData(recordsetId, out shapeIds)`.

## Refresh and linked shapes

- Changed values reach linked shapes; the row is matched by key (above).
- A linked value the user edits in the shape (shape data window or `FormulaU`) is **kept** on refresh; the shape's other linked values still update. `recordset.RefreshSettings = visRefreshOverwriteAll` (1) overwrites edited values too. Default `RefreshSettings` = 0. Repair one shape: `LinkToData` to the same row again restores the row's values, and later refreshes update them again (`BreakLinkToData` first is not needed).
- Column added in the source: appears in the recordset and as a new shape data row in every linked shape. Column removed: its shape data row is removed. Column renamed: = removed + added (`_VisDM_X` gone, `_VisDM_Y` new).
- Hidden column (`DataColumn.SetProperty(visDataColumnPropertyVisible, false)` before linking): no shape data row is created. Made visible later: the row appears at once with the current value, but refresh does not update it until the shape is linked again (`LinkToData`).
- Text of 400 characters came through ACE complete (2 rows; longer text below 8 short rows not tested, ACE guesses types from the first rows).

## Relative Data Source

- `DataRecordsets.Add` with `Data Source=data.xlsx` (also `.\data\x.xlsm`, `..\other\x.xlsx`) resolves against the **document folder** (saved document).
- `DataRecordset.Refresh()` and Data > Refresh All in the UI resolve it against the **current folder of the Visio process**. Wrong folder: `Unable to connect to the data.` (UI: `Error 80004005: ... could not find the object '...\AppData\Local\Temp\VSO<hex>.xlsx'`), or silently another folder's `data.xlsx`.
- The current folder depends on how documents were opened: the File Open dialog sets it to the document's folder; Visio started by COM, and a running Visio that got a document by double-click in Explorer, stayed at `C:\WINDOWS\system32`. With documents from two folders open, the last File Open decides. Once a `Refresh()` succeeded although the folder was `system32` (not reproduced, cause unknown).
- Workaround: set `DataConnection.ConnectionString` to the absolute path, `Refresh()`, set the relative string back. Saved file keeps the relative string. Each `Add` gets its own `DataConnection` (setting one does not change the other). Visio's own Refresh All stays unreliable.
- The error looks random (works after save, close, reopen; works in a copied folder; the Link Data wizard works only after browsing to the file): all match the current folder rule. `DataConnection.FileName` (read-only) is empty for `Add` by code and set by the wizard; setting it in `connections.xml` did not change the error (VisGuy forum thread "DatarecordSet with relative path to Excel worksheet", 2022). _Unverified:_ that the wizard's browse dialog sets the current folder; behavior of Visio before 2019.
- Tests: read another process's current folder from its PEB (x64: `NtQueryInformationProcess` -> `PEB+0x20` ProcessParameters -> `+0x38` CurrentDirectory); set it inside Visio with `doc.ExecuteLine("ChDir ""<path>""")`.

## Source open in Excel

- ACE reads a workbook open in Excel: it gets the **saved** state; unsaved edits are not seen.
- While Excel has it open, another process cannot open the file for writing (sharing violation); reading works.
